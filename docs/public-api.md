# Public API Reference

All four activities share the same infrastructure arguments:

| Argument | Meaning |
|---|---|
| `in_SharePath` | Canonical `store.db` (network share or any folder). Never opened directly by the engine. |
| `in_LocalDbPath` | This machine's local copy (write copy for ingest, read copy for retrieval). Created automatically. Use different paths for ingest and retrieval. |
| `in_LockPath` (ingest only) | Write-lock file, conventionally next to the store (`store.lock`). |
| `in_EmbeddingUrl` | OpenAI-compatible `/v1/embeddings` endpoint. |
| `in_ApiKey` | Bearer token, empty for anonymous endpoints. Source from a Credential asset. |
| `in_Model` | Embedding model name. Must match the store's `meta.embedding_model` — compared with the provider prefix stripped, so `BAAI/bge-m3` and `bge-m3` are equivalent, but `bge-large-en` and `bge-large-en-v1.5` are not. Enforced at both ingest and retrieval. |

Retrieval results (both activities): DataTable `Rank (Int32), DocId (String), ChunkIndex (Int32),
Content (String), Score (Double)`, best first.

---

## IngestDocuments (free-text)

**Input**: `in_Documents` DataTable with columns `DocId`, `Source`, `Text` — one document per row.

**Behavior**, per document, inside one write-lock session:
1. `CheckDocument`: hash the text; skip if unchanged (`out_Skipped`).
2. `ChunkText`: sliding window of `in_ChunkSize` chars with `in_Overlap` overlap. Each cut is
   snapped back to the nearest paragraph break, else sentence end, else whitespace, so chunks
   end on a natural boundary when one is available within the window.
3. `EmbedTexts`: batch-embed all chunks.
4. `WriteDocumentChunks`: one transaction — delete the doc's old chunks, insert new ones.
5. Errors on one document do NOT stop the others (`out_Failed` + `out_Errors` accumulate).

**Outputs**: `out_Written`, `out_Skipped`, `out_Failed`, `out_Errors`.

---

## IngestStructuredData (tabular)

**Input**: `in_Sheet` DataTable (one record per row) + `in_ExactColumns` + `in_DocIdColumn` + `in_Source`.

**Per-column mode** — the core concept:
- Column listed in `in_ExactColumns` → `exact`: matched by (trimmed, case-insensitive) string
  equality at query time. No embedding stored — free. Use for IDs, ranks, codes, numbers, dates.
- Every other column → `semantic`: each cell value is embedded and matched by similarity.
  Use for names, descriptions, free text.

**Pipeline** (one lock session per call, all batched):
1. `BuildFieldTable`: explode the sheet → `rows` (RowKey, DocId, Source, ContentJson) and
   `fields` (RowKey, FieldKey, FieldValue trimmed, Mode).
2. `FilterUnchanged`: drop documents whose doc-level hash (content + field modes) matches the
   store — skipped docs cost nothing. Logged: "skipped N unchanged doc(s); M row(s) to write".
3. `EmbedTexts` ×2: all row JSONs; all semantic field values.
4. `WriteStructuredChunks`: ONE transaction — per distinct doc: delete old chunks+fields, upsert
   the document with its doc hash; then insert every row chunk and every field.
5. Checkpoint + publish only if something was written.

**Outputs**: `out_RowsWritten` (0 = everything unchanged), `out_Errors`.

**Rows sharing a DocId** form one document: they are deleted and rewritten together, and the
whole group is skipped/rewritten as a unit. Use a unique column as `in_DocIdColumn` if you want
per-row documents.

---

## RetrieveData (free-text semantic)

Embeds `in_Query`, dot-products against every chunk embedding, returns top `in_TopK`.
`Content` is the stored chunk text (free-text path) or the full row JSON (structured path) —
both are searched.

Best for: "find rows/passages about X" in any language. Not a ranking engine: "best/top X"
phrasing is not understood — scores measure similarity of meaning, not quality.

---

## RetrieveStructuredData (hybrid per-field)

`in_QueryJson` is a JSON object: `{"Name":"جامعة هارفارد","Rank":"5"}`.
Keys match ingested column names (case-insensitive); each key scores by its STORED mode
(exact = equality, semantic = similarity). Returns whole rows (`Content` = full row JSON).

**Scoring vs filtering** — the distinction that matters:

- By default **every** queried key only *scores*. A row whose `Rank` is not 5 is still returned
  for `{"Rank":"5","Name":"..."}`, just ranked lower. The score is the average across the
  scored keys, so rows matching more keys rank higher.
- `in_FilterColumns` (optional) turns the keys listed in it into **hard filters**: a row is
  returned only if every listed key matches the queried value exactly (trimmed,
  case-insensitive), *whatever mode it was ingested in*. Filter keys contribute nothing to the
  score, which is then averaged over the remaining keys. A query that is all filters scores 1.0.

So `in_QueryJson = {"Rank":"5","Name":"..."}` with `in_FilterColumns = ["Rank"]` means
*Rank must be 5, then rank those rows by name similarity* — usually what such a query intends.
Leaving `in_FilterColumns` empty reproduces the original behaviour exactly.

Tips:
- Combine a semantic key with filter keys to filter+search in one call.
- Filter by `docId` to scope results to one source workbook.
- An all-exact query makes no embedding HTTP calls at all.

---

## Failure modes at a glance

| Symptom in `out_Errors` | Meaning | Store state |
|---|---|---|
| `LOCK NOT ACQUIRED: ...` | Another machine held the write lock past `in_LockTimeoutMs`. | Untouched. Retry later. |
| `INGEST ABORTED:` / `STRUCTURED INGEST ABORTED: ...` | The run failed mid-way (embedding down, DB error, ...). | Canonical store untouched — publish never happened. Local copy may be partial; the next run re-pulls it. |
| `<DocId>: <error>` (IngestDocuments) | That one document failed; others proceeded. | Other docs written and published. |
| `INGEST ABORTED: ... Embedding model mismatch ...` | `in_Model` names a different model than the one this store was built with. | Untouched — the guard runs before any write. Ingest into a different store path, or delete and re-ingest. |
| `INGEST ABORTED: ... Embedding dimension mismatch ...` | `in_EmbeddingDim` disagrees with the store's recorded dimension. | Untouched, same as above. |
| Exception thrown (retrieval) | Store missing, endpoint down/misconfigured, or model/dim mismatch. | Read-only path — nothing modified. |

Model/dim mismatches are detailed in [error-handling.md](error-handling.md#embedding-identity-guards-fail-fast-no-retry),
including how model names are compared (provider prefix stripped, version suffix significant).
