# AI_SemanticSearch — Library Documentation

A UiPath **library** implementing semantic search & retrieval over SQLite ("RAG without the G"):
ingest free-text documents or tabular sheets, embed them with any OpenAI-compatible embedding
endpoint, and retrieve by meaning (semantic), by precise value (exact), or both (hybrid).
Multi-machine safe: many robots can share one store on a network path.

## Documentation map

| File | Read it when you want to... |
|---|---|
| [architecture.md](docs/architecture.md) | Understand the store schema, the multi-machine concurrency model, and the change-detection (hash) logic |
| [public-api.md](docs/public-api.md) | Use the 4 public activities — arguments, behavior, examples, failure modes |
| [helpers.md](docs/helpers.md) | Fix or extend an internal helper — what each one does and how |
| [error-handling.md](docs/error-handling.md) | Follow (or debug) the error-handling conventions used everywhere |
| [TODO.md](TODO.md) | See known limitations and what is planned next |

## Public API (what consumers see)

| Activity | Purpose |
|---|---|
| **IngestDocuments** | Free-text path: chunk → embed → store. Input: DataTable `[DocId, Source, Text]`. |
| **IngestStructuredData** | Tabular path: each sheet row becomes an embedded record with per-field exact/semantic matching. |
| **RetrieveData** | Free-text semantic search across all stored chunks. Returns top-K rows. |
| **RetrieveStructuredData** | Hybrid per-field search with a JSON query `{"field":"value", ...}`. Returns whole rows. |

Everything under `helpers/` is **private**. The four activities above work with local paths too —
the "share" can be any folder.

`lib/e_sqlite3.dll` is the native SQLite engine shipped with the package; it is not a workflow.

## Quick start (consumer process)

1. Install the library package.
2. **Ingest** (structured example):
   - Read your sheet with *Read Range* → DataTable.
   - Add/choose a `DocId` column (rows sharing a value are one document, rewritten together).
   - Call **Ingest Structured Data** with store paths, endpoint URL, model (`BAAI/bge-m3`), dim (1024),
     and `in_ExactColumns` = the ID/numeric columns (everything else becomes semantic).
   - Check `out_Errors` — empty means success. `out_RowsWritten = 0` means everything was unchanged (skipped).
3. **Retrieve**:
   - Structured/hybrid: `in_QueryJson = "{""Name"":""جامعة هارفارد"",""Rank"":""5""}"` → whole rows back.
   - Free-text: `in_Query = "أفضل جامعة في أمريكا"` → best-matching chunks/rows.
   - Results DataTable: `Rank, DocId, ChunkIndex, Content, Score` (best first).

## Key facts to remember

- **The model must match**: retrieval must use the same embedding model the store was ingested with
  (recorded in the store's `meta` table). Mismatch = meaningless scores or a dim-mismatch error.
- **Exact vs semantic is decided at INGEST time**, per column, by `in_ExactColumns`.
  Changing it triggers a full rewrite of affected documents on the next ingest (mode is part of the doc hash).
- **Re-ingesting unchanged data is free**: documents whose content+mode hash matches are skipped
  before any embedding call.
- **Multilingual**: with bge-m3, Arabic queries match English data and vice versa.
- **Scores**: semantic = cosine similarity (~0.5–1.0 useful range); exact = 1.0 or 0;
  structured queries average across the queried keys.

## Building the package

The project targets the **Windows (legacy)** framework, so UiPath's workflow compiler only runs on
Windows — packing on Linux fails with *"Cannot execute Windows projects on Linux platform"*.

Builds therefore run in CI on a `windows-latest` runner ([.github/workflows/build.yml](.github/workflows/build.yml)),
with the UiPath Workflow Analyzer enabled.

Releases are cut from `projectVersion` in `project.json`, not by pushing a tag:

- **Bump `projectVersion`** and push → CI packs that version, publishes it as a
  [GitHub Release](../../releases), and creates the matching `v<version>` tag.
- **Push again without bumping** → CI packs `<projectVersion>-ci.<run number>` and uploads it as a
  workflow artifact only, so a published release is never silently overwritten.

To build locally instead, on a Windows machine with the UiPath CLI:

```powershell
uipcli package pack project.json -o output -v 1.1.0
```

Consume the resulting `.nupkg` by adding its folder as a custom NuGet feed in UiPath Studio
(*Settings → Manage Sources*), or by uploading it to Orchestrator.
