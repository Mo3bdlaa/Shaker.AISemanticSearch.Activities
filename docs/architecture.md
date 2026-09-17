# Architecture

## The big picture

```
                    one writer at a time (file lock)
 Machine A ────────────────────────────────────────────┐
   Ingest*  1.lock 2.pull 3.write local 4.checkpoint   │
            5.publish (atomic rename) 6.release        ▼
                                          \\share\rag\store.db   (canonical)
 Machine B ────────────────────────────────────────────┘
   Retrieve*  1.sync local copy (only if share newer)  ▲
              2.query the LOCAL copy                   │
 Machine C ────────────────────────────────────────────┘
```

Two hard rules drive the whole design:

1. **Never run the SQLite engine over SMB.** The engine only ever opens *local* files.
   The share holds a plain file that is copied whole in both directions.
2. **Readers must never see a half-written file.** Publishing is copy-to-`store.db.new`
   then atomic rename-replace; syncing is copy-to-`.tmp` then atomic replace locally.

## SQLite schema

Created by `helpers/InitializeStore.xaml` (WAL mode, `busy_timeout`, `synchronous=NORMAL`):

| Table | Columns (key ones) | Purpose |
|---|---|---|
| `documents` | `doc_id` PK, `source`, `content_hash`, `created_utc`, `updated_utc` | One row per document. `content_hash` is the **doc-level** hash used for skip-unchanged (see below). |
| `chunks` | `id` PK, `doc_id`, `chunk_index`, `content`, `embedding` BLOB, `dim`, `model`, `content_hash`, `metadata` | One row per text chunk (free-text path) or per sheet row (structured path; `content` = the row JSON). `embedding` = unit-normalized Float32, 4 bytes/dim. |
| `chunk_fields` | `id` PK, `chunk_id`, `doc_id`, `field_key`, `field_value`, `field_mode`, `embedding` BLOB, `dim` | Structured path only: one row per non-empty cell. `field_mode` ∈ {`exact`, `semantic`}. Exact rows have NULL embedding by design. `field_value` is trimmed at ingest. |
| `meta` | `key` PK, `value` | `embedding_model`, `embedding_dim` — the store's identity, stamped by the first ingest. Later ingests and every retrieval are **checked against it** and fail on mismatch, so a store can never mix two models. See [error-handling.md](error-handling.md#embedding-identity-guards-fail-fast-no-retry). |

Indexes: `ix_chunks_doc(doc_id)`, `ix_fields_key(field_key)`, `ix_fields_chunk(chunk_id)`.

## Embeddings

- Any OpenAI-style endpoint: `POST {url}` body `{"model":"...","input":["t1","t2"]}` →
  `{"data":[{"embedding":[...],"index":0}, ...]}` (handled in `helpers/EmbedTexts.xaml`).
- Responses are re-ordered by `index`, then **unit-normalized**, so cosine similarity
  reduces to a dot product at query time.
- Stored as Float32 BLOBs (`Single()` → `Buffer.BlockCopy` → bytes); decoded the same way when scoring.
- Batched (`in_BatchSize` texts per HTTP call) with retry + backoff. Permanent 4xx
  (except 408/429) fails fast instead of burning retries.

## Concurrency model

- **Write lock** (`helpers/AcquireWriteLock.xaml`): opens the lock file with `FileShare.None`
  and KEEPS THE HANDLE. The held handle *is* the lock — if the process crashes, the OS/SMB
  server releases it, so there is no stale-lock cleanup. Timeout returns `out_lockOk=False`
  (no exception); callers report `LOCK NOT ACQUIRED` in `out_Errors` and exit gracefully.
- **Pull-for-write**: before writing, the canonical db is copied local and stale `-wal`/`-shm`
  side files are deleted (a leftover `-wal` would be replayed onto the fresh copy and corrupt it).
- **Checkpoint**: after writing, `PRAGMA wal_checkpoint(TRUNCATE)` + `ClearAllPools()` folds the
  WAL into the file and releases handles so the file can be copied.
- **Publish** (`helpers/PublishDatabase.xaml`): copy local → `store.db.new` on the share, then
  `File.Replace` (atomic). Retries up to 3× with backoff because a reader mid-copy can hold
  the target briefly.
- **Read sync** (`helpers/SyncLocalCopy.xaml`): copy share → local only when the share's
  `LastWriteTimeUtc` is strictly newer; copy goes to `.tmp` then atomic replace so a local
  reader never opens a half-copied file.
- The lock is released in a `Finally` block — it is held across pull/write/checkpoint/publish.

## Change detection (skip-unchanged)

Structured path. Two places compute the SAME doc-level hash — keep them in sync:

- `IngestStructuredData.xaml` → *Invoke Code - FilterUnchanged* (decides what to skip)
- `helpers/WriteStructuredChunks.xaml` → pass 0 (stamps `documents.content_hash`)

Formula: `SHA256( concat(rowContentJson + LF, per row in row order) + "|modes|" + sig )`
where `sig` = sorted distinct `fieldKey:mode` pairs of the doc's fields, `;`-joined, mode lower-cased.

Consequences:

- Re-running an unchanged workbook: zero embedding calls, zero writes, publish skipped.
- Changing row content OR flipping a column exact↔semantic changes the hash → the whole
  document is deleted and rewritten (all-or-nothing per document).
- If the two formulas ever drift apart, the failure mode is SAFE: documents are always
  rewritten (never wrongly skipped).

The free-text path chunks with `helpers/ChunkText.xaml`: fixed character windows with overlap,
snapped back to the nearest paragraph break, else sentence end, else whitespace (Latin and Arabic
terminators both recognised). A candidate boundary is only taken if it still leaves more than half
a window, so text with no boundaries falls back to the hard window rather than degenerating.

The free-text path (`IngestDocuments`) has per-document change detection instead:
`helpers/CheckDocument.xaml` compares `SHA256(text)` against `documents.content_hash`.

## Scoring

- **Free-text** (`helpers/ScoreChunks.xaml`): dot product between the query vector and every
  chunk embedding; top-N by score. `Content` returned is the stored chunk text / row JSON.
- **Query planning** (`RetrieveStructuredData` → *PlanQueryEmbedding*): before embedding, the
  local copy is read for `meta.embedding_model` and for each queried key's stored `field_mode`.
  Only semantic, non-filter keys are embedded, so an all-exact query costs no HTTP call, and the
  model guard fires before any request. If the store cannot be read yet, it falls back to
  embedding every non-filter key.
- **Structured** (`helpers/ScoreFieldsSemantic.xaml`): scans `chunk_fields` for the queried keys
  only (parameterized IN-list). Per key: exact → 1.0 on case-insensitive equality;
  semantic → dot product. A chunk's score = sum of key contributions ÷ number of **scored** keys,
  so matching more keys ranks higher. Winners are joined back to `chunks` to return whole rows.
  Keys passed in `in_FilterKeys` are *hard predicates* instead: a chunk survives only if every
  one of them matches exactly (trimmed, case-insensitive) regardless of its stored mode, and
  they contribute nothing to the score. A chunk matching all filters but no scored key is still
  returned with score 0; an all-filter query scores 1.0. No filter keys = original behaviour.

## Native SQLite loading

`Microsoft.Data.Sqlite` needs the native `e_sqlite3.dll`. `lib/EnsureSqliteNative.xaml` copies the
bundled `lib\e_sqlite3.dll` next to the running process (probing several candidate roots — project
dir, process base dir, the installed package folder via the loaded assembly) then calls
`SQLitePCL.Batteries_V2.Init()`. Idempotent; every public workflow invokes it first.
