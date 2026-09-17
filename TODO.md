# TODO

Backlog for `AI_SemanticSearch`, roughly in priority order. Items marked **behaviour change**
alter results for existing consumers and need a version bump + release note.

## 1. Correctness & API

- [ ] **Let exact keys filter, not just score** — *behaviour change*, highest impact.
      In `helpers/ScoreFieldsSemantic.xaml` a row's score is the sum over queried keys ÷ key
      count, so an exact key contributes 1.0 or 0 but never excludes. A query of
      `{"Rank":"5","Name":"..."}` still returns rows whose `Rank` is not 5, just ranked lower —
      which is not what most people writing that query mean.
      Proposal: add an opt-in `in_FilterColumns` (or a `!` key prefix) that turns a key into a
      hard predicate, and renormalise the score across the remaining scoring keys.
      Document the default in `docs/public-api.md` either way — today it is a silent surprise.

- [ ] **Per-key weighting** — follow-on from the above. Averaging treats a matched ID and a
      fuzzy name match as equally important. Optional weights per key would help hybrid queries.

## 2. Repository hygiene

- [ ] **Rename the repo: `Shaker.AISemenitcSearch.Activities` → `Shaker.AISemanticSearch.Activities`.**
      "Semenitc" is a typo. The project itself is spelled correctly (`AI_SemanticSearch`), so it is
      only the repo name. Cheap now, annoying once anyone has cloned or referenced it.
      GitHub redirects the old URL, but update the remote afterwards.

- [ ] **Create a `main` branch and make it the default.** GitHub set the default to
      `claude/magical-wright-hnbxsc` because it was the first branch pushed. A feature-branch name
      as the public default is confusing, and `workflow_dispatch` only appears on the default branch.

- [ ] **Drop the stale `docs/AGENTS.md` entry** from `privateWorkflows` in `project.json` —
      the file is not in the package.

- [ ] **Add a LICENSE.** The repo is public with none, so by default nobody may legally reuse it.

## 3. Testing

- [ ] **Rebuild a test workflow without hardcoded values.** The original `Test/TestStructured.xaml`
      was removed before publishing because it embedded an internal host and path. Nothing now
      guards the invariant `docs/architecture.md` explicitly warns about: the doc-level hash is
      computed in **two** places (`IngestStructuredData.xaml` → *FilterUnchanged*, and
      `helpers/WriteStructuredChunks.xaml` → pass 0) and they must stay in sync.
      Take endpoint/paths as arguments or from Orchestrator assets.

- [ ] **Cover the identity guards**: model mismatch, dim mismatch, and skip-unchanged returning
      `out_RowsWritten = 0`. These are the behaviours most likely to regress silently.

## 4. Scaling (only when the store grows)

- [ ] **Vector scoring is a full scan.** `helpers/ScoreChunks.xaml` dot-products the query against
      every chunk embedding. At 1024 dims (4 KB/vector): ~10k chunks is imperceptible, ~100k means
      ~400 MB read per query, ~1M is not viable. There is no ANN index.
      When it hurts, `sqlite-vec` is the natural upgrade and keeps the single-file model.

- [ ] **Retrieval sync copies the whole `store.db`** whenever the share is newer
      (`helpers/SyncLocalCopy.xaml`). Hourly ingest into a large store means every robot re-copies
      it hourly. Store size drives both this and the item above — measure it before optimising.

- [ ] **Single global write lock** serialises all ingest. Correct for the design, but ingest
      throughput will not improve by adding robots. Fine unless ingest becomes the bottleneck.

## 5. Retrieval quality

- [ ] **Chunking cuts mid-word.** `helpers/ChunkText.xaml` uses fixed character windows — a
      deliberate trade (no tokenizer dependency, works for Arabic), but it splits sentences.
      Try paragraph/sentence boundaries first, falling back to a hard window. Cheap, and usually
      a real retrieval-quality win.

## 6. Packaging & CI

- [ ] **The package is ~130 MB.** UiPath bundles the full dependency closure
      (`SeparateRuntimeDependencies` + `IncludeSources` in `.project/design.json`). Slow to upload
      to Orchestrator. Try `IncludeSources: false`, or `--splitOutput` to separate runtime and
      design packages, and measure.

- [ ] **Harden `EnsureSqliteNative`.** It copies `lib/e_sqlite3.dll` next to the running process,
      probing several roots — this needs write permission to that directory, which is exactly what
      gets locked down for hardened robot service accounts. Also x64-only. Worth a clear error
      message when the copy fails, rather than a downstream SQLite load failure.

- [ ] **Decide the release trigger.** Releases are currently cut by CI from `projectVersion` in
      `project.json`, because tag pushes were rejected for the account that set this up. If you can
      push tags, the workflow already handles `v*` tags too — pick one and simplify
      `.github/workflows/build.yml`.

- [ ] *(optional)* **Workflow Analyzer warnings.** The build is error-free; remaining warnings are
      cosmetic — default activity names (`ST-MRD-002`), `dt` prefix conventions (`ST-NMG-009/011`),
      duplicate display names (`ST-NMG-004`), nesting depth over 7 (`ST-MRD-009`).
