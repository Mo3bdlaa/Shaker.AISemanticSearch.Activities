# Error-Handling Conventions

One pattern everywhere, one documented exception. If you add code, follow these.

## The standard pattern: flag + CheckTrue

Every `Invoke Code` block — in helpers AND inline in the public workflows — has this shape:

```vb
' <Name> - what it does.
' Contract: sets out_Success/out_ErrorMessage; the CheckTrue after throws.
out_Success = False
out_ErrorMessage = ""
Try
    ' ... the actual work ...
    out_Success = True
Catch ex As System.Exception
    out_ErrorMessage = "<Name> failed: " & ex.Message
    If ex.InnerException IsNot Nothing Then out_ErrorMessage &= " | Inner: " & ex.InnerException.Message
End Try
```

and is immediately followed by:

```xml
<ui:CheckTrue DisplayName="Check True - <Name>" ErrorMessage="[<errVar>]" Expression="[<okVar>]" />
```

Why this shape instead of letting exceptions bubble out of Invoke Code directly:
- The error message is **named and prefixed** (`"PullForWrite failed: ..."`), so a log line or an
  `out_Errors` entry tells you exactly which step died without opening the workflow.
- Inner exceptions are folded in — SQLite and HttpClient bury the useful message one level down.
- The flag contract makes each block independently testable and keeps the code readable:
  the happy path is the whole Try body, failures are one place at the bottom.

The `CheckTrue` re-throws inside the workflow, where the **outer TryCatch** of the public
workflows turns it into a graceful result (see below). Nothing is ever swallowed.

## Where thrown errors land

Public **ingest** workflows have one outer TryCatch around the whole write session:

- Catch → `out_Errors += "INGEST ABORTED: <message>"` (or `STRUCTURED INGEST ABORTED`).
  The canonical store is safe: publish happens last, so an aborted run never modifies it.
- Finally → `ReleaseWriteLock` always runs; the lock can never leak.
- Per-document errors in `IngestDocuments` are caught by an inner per-row TryCatch and
  accumulated (`out_Failed`, `out_Errors`) — one bad document doesn't sink the batch.

Public **retrieval** workflows are read-only and fail-fast: any error propagates to the caller
as an exception (there is nothing to roll back).

## The one deliberate exception: AcquireWriteLock

`AcquireWriteLock` does **not** throw on timeout. A busy lock is an expected outcome, not an
error: it returns `out_lockOk=False` + `out_lockErr`, and the ingest workflows branch on it,
returning `LOCK NOT ACQUIRED: ...` in `out_Errors` with no exception. Do not add a CheckTrue
to it — that would make the graceful branch unreachable (this exact bug existed and was removed).

## Embedding-identity guards (fail-fast, no retry)

A store belongs to exactly **one** embedding model. Vectors produced by two different models are
not comparable — cosine over them yields plausible-looking numbers that mean nothing — so a store
that mixes them is silently useless and cannot be un-mixed. Three guards enforce this:

| Message | Raised by | When |
|---|---|---|
| `Embedding model mismatch. This store was built with '<X>' but this ingest passed '<Y>'. ...` | `InitializeStore` (both ingest paths) | `meta.embedding_model` already records a different model. **Nothing is written.** |
| `Embedding dimension mismatch. This store was built with dim=<N> but this ingest passed dim=<M>. ...` | `InitializeStore` (both ingest paths) | `meta.embedding_dim` disagrees with `in_EmbeddingDim`. **Nothing is written.** |
| `Embedding model mismatch. This store was built with '<X>' but this query was embedded with '<Y>'. ...` | `ScoreChunks`, `ScoreFieldsSemantic` | `in_Model` disagrees with `meta.embedding_model` at retrieval. |

Model names are compared with the **provider/org prefix stripped**, case-insensitively: `BAAI/bge-m3`,
`bge-m3` and `sentence-transformers/BGE-m3` are all the same model. A version or tag suffix is
**not** stripped — `bge-large-en` and `bge-large-en-v1.5` are different models that happen to share
a dimension, and must not be treated as interchangeable.

The first ingest into an empty store **stamps** the identity; there is nothing to compare against, so
it never fails. A store with no recorded model (created before these guards existed) does not block
retrieval — the check is skipped rather than assumed-mismatched.

To change model or dimension, ingest into a **different store path**, or delete the store and
re-ingest from scratch. There is deliberately no override flag: the failure it prevents is silent.

Independently of the name check, `ScoreChunks` and `ScoreFieldsSemantic` also compare the stored
vector's actual length against the query vector's (`Embedding dimension mismatch` / `Embedding dim
mismatch: stored=..., query=... (key '<k>')`). That catches a corrupt or hand-edited BLOB even when
the recorded model name agrees.

## Retry policies (who retries what)

| Where | What | Policy |
|---|---|---|
| `EmbedTexts` | embedding HTTP call | `in_MaxRetries` attempts, 500ms × attempt backoff. Permanent 4xx (400–499 except 408/429) throws immediately. |
| `PublishDatabase` | copy + atomic replace on the share | 3 attempts, 750ms × attempt, on `IOException` only (reader briefly holding the file). |
| `AcquireWriteLock` | lock acquisition | spins every `in_RetryDelayMs` until `in_TimeoutMs`, then returns `lockOk=False`. |
| Everything else | | No retry — fail with a precise message; the caller decides. |

## Log conventions

- Every helper logs a start line ("Embedding Text...") and an end line ("Done").
- Decisions worth auditing get their own line: FilterUnchanged logs
  `skipped N unchanged doc(s); M row(s) to write`; both ingest paths log
  `All docs unchanged - skipping checkpoint and publish.` when nothing was written.
- `excludedLoggedData` in project.json masks `*password*` and `*apikey*` — keep argument names
  containing secrets matching those patterns.
