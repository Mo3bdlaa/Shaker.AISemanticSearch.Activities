# TODO

Backlog for `AI_SemanticSearch`. Completed items are kept, checked, for context.

## Done

- [x] **Exact keys can now filter, not just score.** `RetrieveStructuredData` takes
      `in_FilterColumns`; keys listed there are hard predicates (exact, trimmed,
      case-insensitive, whatever mode they were ingested in) and contribute nothing to the
      score. Leaving it empty reproduces the old behaviour, so existing consumers are
      unaffected. Guarded by `Test/SmokeTest.xaml`.
- [x] **Chunking ends on natural boundaries.** `helpers/ChunkText.xaml` snaps each cut back to
      the nearest paragraph break, else sentence end, else whitespace (Latin and Arabic
      terminators), accepting a boundary only if it still leaves more than half a window.
      *(Correction to an earlier note: it already avoided splitting words — the gap was
      mid-sentence cuts.)*
- [x] **Native loader failures are now explicit.** `helpers/EnsureSqliteNative.xaml` used to
      fall through silently when the engine could not be found or copied, surfacing later as
      an opaque SQLite error. It now names the probed paths, calls out the robot account's
      write access when a copy is refused, and reports the x64 requirement when `Init()` fails.
- [x] **Package size fixed: 130 MB → ~2 MB.** *(Correction: this was never UiPath's dependency
      bundling.* The CI job downloaded the UiPath CLI into the checkout, and `uipcli` packs the
      whole working directory into `content/` — so `v1.1.0` shipped a 114 MB `uipcli.zip` plus
      an 87 MB extracted copy inside the library.) The CLI now lives in `RUNNER_TEMP`, and the
      build fails if the package exceeds 25 MB so it cannot regress.
- [x] **MIT license added**, Copyright (c) 2026 Mohamed Shaker; author stamped into the package
      metadata and credited in the README.
- [x] **Argument-driven smoke test** (`Test/SmokeTest.xaml`) replacing the one removed before
      publishing. No environment-specific values: every path and endpoint is an argument and
      the fixture is built in memory.
- [x] **Dropped the stale `docs/AGENTS.md`** entry from `privateWorkflows`.

## Needs you (no API access from here)

- [ ] **Rename the repo: `Shaker.AISemenitcSearch.Activities` → `Shaker.AISemanticSearch.Activities`.**
      "Semenitc" is a typo; the project itself is spelled correctly. *Settings → General →
      Repository name.* GitHub redirects the old URL, but update your local remote afterwards.
- [ ] **Make `main` the default branch.** *Settings → General → Default branch.*
- [ ] **Replace or delete the `v1.1.0` release.** Its `.nupkg` is the 130 MB one containing a
      copy of the UiPath CLI. It installs and runs correctly, but it is 65× larger than it
      should be. `v1.2.0` supersedes it.

## Open

- [ ] **Per-key weighting for structured queries.** Deliberately *not* done: hard filters
      addressed the actual problem, and an unused knob is permanent API surface on a published
      library. Worth adding only given a concrete case where averaging ranks badly.
- [ ] **Extend the smoke test to the identity guards** — model mismatch, dimension mismatch, and
      skip-unchanged returning `out_RowsWritten = 0`. It currently covers ingest errors and the
      filter behaviour only.
- [ ] **Guard the doc-hash invariant.** `docs/architecture.md` warns that the doc-level hash is
      computed in two places (`IngestStructuredData.xaml` → *FilterUnchanged*, and
      `helpers/WriteStructuredChunks.xaml` → pass 0) that must stay in sync. Drift fails safe
      (always rewrite, never wrongly skip), but nothing detects it.
- [ ] *(optional)* **Workflow Analyzer warnings.** The build is error-free; the rest are
      cosmetic — default activity names (`ST-MRD-002`), `dt` prefix conventions
      (`ST-NMG-009/011`), duplicate display names (`ST-NMG-004`), nesting depth over 7
      (`ST-MRD-009`). Mechanical churn across most files for no functional gain.

## Deferred until measured

Both are gated on store size — they only pay off well past ~100k chunks, and neither is worth
the disruption before that. Measure your real store first.

- [ ] **Vector scoring is a full scan.** `helpers/ScoreChunks.xaml` dot-products the query
      against every chunk embedding. At 1024 dims (4 KB/vector): ~10k chunks is imperceptible,
      ~100k means ~400 MB read per query, ~1M is not viable. `sqlite-vec` is the natural
      upgrade and keeps the single-file model, but it changes the store format.
- [ ] **Retrieval sync copies the whole `store.db`** whenever the share is newer
      (`helpers/SyncLocalCopy.xaml`). Hourly ingest into a large store means every robot
      re-copies it hourly.
- [ ] **Single global write lock** serialises all ingest. Correct for the design, but ingest
      throughput will not improve by adding robots.
