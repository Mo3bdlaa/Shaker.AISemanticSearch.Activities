# Helper Internals

Every helper is a private workflow whose logic lives in a single `Invoke Code` (VB.NET) block,
followed by a `Check True` that converts the failure flag into an exception
(see [error-handling.md](error-handling.md)). This file tells you what each one does and the
non-obvious decisions inside, so you can fix them by hand.

## Locking

### helpers/AcquireWriteLock.xaml
Opens `in_LockPath` with `FileMode.OpenOrCreate` + `FileShare.None` and **keeps the stream open**
(`out_LockStream`) — the held handle IS the lock. Retries every `in_RetryDelayMs` until
`in_TimeoutMs`. Writes owner info + timestamp into the file (for humans only).

- **Does NOT throw on timeout** — returns `out_lockOk=False` + `out_lockErr`. Callers branch on it.
  This is the one deliberate exception to the CheckTrue pattern; don't "fix" it by adding one.
- Crash safety: OS/SMB releases the handle automatically — no stale-lock cleanup exists or is needed.

### helpers/ReleaseWriteLock.xaml
Disposes `io_LockStream` (null-safe). Called from `Finally` blocks so the lock is always released.

## Store lifecycle

### helpers/InitializeStore.xaml
Opens/creates the LOCAL db, sets PRAGMAs (WAL, busy_timeout, synchronous=NORMAL), and creates the
four tables + indexes if missing.

Then **verifies or stamps** the store's embedding identity: if `meta.embedding_model` /
`meta.embedding_dim` are already set and disagree with `in_EmbeddingModel` / `in_EmbeddingDim`, it
fails before anything is written; otherwise it records them. This is what makes a store single-model
— see [error-handling.md](error-handling.md#embedding-identity-guards-fail-fast-no-retry). It is
**not** a blind upsert: a rejected ingest leaves the recorded identity intact.

### helpers/PublishDatabase.xaml
Copies the local db to `<share>.new` then atomically swaps it in (`File.Replace`, or `File.Move`
for the first publish). Retries 3× with backoff on `IOException` (a reader mid-copy can hold the
target briefly). A leftover `.new` from a failed run is harmless — overwritten next time.

### helpers/SyncLocalCopy.xaml
Read-side pull: copies share → local only when the share file is strictly newer
(`LastWriteTimeUtc`; `File.Copy` preserves the source timestamp, so equality means "in sync").
Copies to `<local>.tmp` then atomically replaces, so concurrent local readers never see a torn file.

## Free-text ingest chain

### helpers/CheckDocument.xaml
`SHA256(text)` (lowercase hex) vs `documents.content_hash` → `out_NeedsWrite`. On any DB error it
reports failure and leaves `out_NeedsWrite=False` — never request a write it couldn't validate.

### helpers/ChunkText.xaml
Sliding-window chunker: `in_ChunkSize` characters per chunk, `in_Overlap` shared between
consecutive chunks. Returns `List(Of String)`.

### helpers/WriteDocumentChunks.xaml
One transaction: `DELETE FROM chunks WHERE doc_id=...`, upsert `documents`
(per-doc text hash), insert each chunk with its Float32 embedding BLOB.

## Structured ingest chain

### helpers/BuildFieldTable.xaml
Explodes the sheet into `out_Rows` (RowKey, DocId, Source, ContentJson) and `out_Fields`
(RowKey, FieldKey, FieldValue, Mode).
- `Mode` = `exact` iff the column is in `in_ExactColumns` (case-insensitive), else `semantic`.
- `FieldValue` is **trimmed** so exact matching isn't broken by stray spaces; `ContentJson`
  keeps the raw values.
- Empty cells stay in `ContentJson` but emit no field row (nothing to embed/match).
- `DocId` falls back to `<source>#row<N>` when the column is missing/empty.

### helpers/EmbedTexts.xaml
Shared by every path (rows, fields, queries). Batches `in_Texts` (`in_BatchSize` per POST),
Bearer auth if a key is set, reorders the response by `index`, **unit-normalizes** each vector,
verifies count(out) == count(in). Retry with 500ms×attempt backoff; permanent 4xx
(anything 400–499 except 408/429) throws immediately. Empty input is valid → empty output, no HTTP.
`ADAPT POINT 1/2` comments mark where to change the request/response shape for non-OpenAI servers.

### helpers/WriteStructuredChunks.xaml
ONE transaction, three passes:
- **Pass 0** (per distinct doc): delete old `chunk_fields` + `chunks`, upsert `documents` with the
  **doc-level hash** (content + field-mode signature — formula documented in
  [architecture.md](architecture.md#change-detection-skip-unchanged); MUST stay identical to
  FilterUnchanged in `IngestStructuredData.xaml`).
- **Pass 1** (per row): insert the row-JSON chunk with its embedding; remember `chunk_id`.
- **Pass 2** (per field): insert `chunk_fields`; semantic fields consume vectors from
  `in_FieldEmbeddings` **in field order** (alignment contract with BuildEmbedInputs);
  exact fields store NULL embedding.
- Any failure rolls the whole transaction back (the `Using txn` disposes uncommitted).

## Scoring

### helpers/ScoreChunks.xaml
Loads every chunk embedding, dot-products against the query vector (both unit-normalized →
this is cosine), sorts, returns top `in_TopN` as `Rank, DocId, ChunkIndex, Content, Score`.

### helpers/ScoreFieldsSemantic.xaml
Scans `chunk_fields` for the queried keys only (parameterized IN-list — keys are user input,
never concatenated). Per matching field: exact → 1.0 on case-insensitive equality; semantic →
dot product with that key's query vector. Chunk score = sum ÷ number of queried keys. Top-N chunk
ids are then joined back to `chunks` for the result rows (numeric ids → safe to concatenate).

## Native bootstrap

### lib/EnsureSqliteNative.xaml
Copies the bundled `lib\e_sqlite3.dll` next to the running process if not already there, probing:
`Environment.CurrentDirectory\lib`, `AppDomain.BaseDirectory\lib`, and the installed package folder
(located via the loaded `AI_SemanticSearch` assembly). Then `SQLitePCL.Batteries_V2.Init()`.
Idempotent; every public workflow calls it first.

## Gotcha log (things that already bit us — don't reintroduce them)

1. **`ChrW`/`Chr` (or any `Microsoft.VisualBasic` runtime function) inside Invoke Code** fails at
   runtime with BC30451 — the Invoke Code compiler doesn't import that namespace. Use `Convert.ToChar`,
   `Environment.NewLine`, etc.
2. **Duplicate namespace entries** in a workflow's `TextExpression.NamespacesForImplementation`
   break every Invoke Code in the file with BC31051 ("already been imported") at negative line numbers.
3. **Deletes inside a per-row loop** wiped sibling rows sharing a doc_id and orphaned their fields —
   deletes belong in pass 0, once per document.
4. **The two doc-hash formulas** (FilterUnchanged / WriteStructuredChunks pass 0) must stay in sync;
   drift is safe (always rewrites) but kills the skip optimization.
5. **`in_FieldEmbeddings` alignment**: vectors correspond to `Mode='semantic'` rows of `in_Fields`
   in row order. Reordering or filtering one side without the other throws the mismatch guard.
