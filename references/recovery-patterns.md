# Recovery and synchronization patterns

Read this reference when a collector must recover interrupted work, update an existing file, or read an authenticated browser session. Adapt the patterns to the target system instead of copying implementation-specific details.

## Pending-run recovery

Use a durable run record with a stable run ID, source boundary, output targets, a collection-complete marker or checkpoint, and a status such as `PENDING` or `SUCCESS`. Store record batches under that run ID. Use idempotent upserts keyed by source identity so replaying a batch is safe.

Suggested flow:

1. Read and validate source scope; recover any interrupted file replacement.
2. Before new collection, find pending runs. Resume collection-incomplete runs from their checkpoint or safely re-read their boundary; reconcile only collection-complete runs to every required output. Never publish a partial run merely because some batches were saved.
3. Build each output at a temporary path. Validate schema, row identity, and required links before replacing the destination.
4. Mark the run successful and advance its cursor only after all required outputs validate.
5. On failure, leave the run pending, retain the last known-good output, and report the failed boundary. Retry through the same recovery path next time.

If outputs include several files that cannot be replaced as one transaction, make each replacement independently recoverable and keep the run pending until the set is reconciled. State clearly that recovery is coordinated, not a cross-file atomic commit.

| Interruption point | Durable state | Next-run action |
| --- | --- | --- |
| Before any record is stored | No run or empty pending run | Re-read the source boundary |
| Between stored batches | Pending run with completed batches but collection incomplete | Resume the collection checkpoint or replay the boundary idempotently |
| After records commit, before output write | Pending run and source records | Rebuild the output from durable state |
| During replacement | Temporary file and/or swap backup | Validate candidates and restore the last valid copy |
| After outputs validate, before success marker | Pending run and valid outputs | Re-validate outputs, then mark success |
| After success marker | Successful run and published outputs | Start after the committed cursor, with configured overlap |

Never advance the cursor just because fetching finished. A cursor is a promise that all required work before its boundary is durable and recoverable.

## Cursor and pagination choices

- Prefer a source sequence, immutable event ID, or provider page token when available. For timestamp boundaries, add a deliberate overlap and deduplicate by stable identity to cover ties and late arrivals.
- Use source event time for collection boundaries and local collection time for audit/diagnostics; do not substitute one for the other.
- Stop early only if the provider guarantees the ordering used by the stop condition. If ordering is unknown or can change, scan to a verified end/retention boundary and deduplicate.
- When the source has finite retention, measure the gap since the last successful cursor. Warn or require a backfill decision when the gap may exceed that retention.
- Scope keys to their parent: `account_id + record_id` or `workspace_id + event_id` may be correct where a bare record ID is not.
- Persist a skipped identity only as long as its business condition is expected to remain true; recheck temporary conditions according to a defined policy.

## Editable spreadsheet synchronization

Treat source columns and user-owned columns as separate data with an explicit merge policy. Before a refresh or schema migration:

1. Read the workbook and validate the expected schema.
2. Bind each row to a durable source key or verified native link. If a row cannot be matched uniquely, stop and preserve the workbook for review.
3. Extract user-owned values before changing source-owned values.
4. Write a temporary workbook, restore the user-owned values by key, and verify headers, row counts, links, and representative preserved edits.
5. Back up and replace the workbook; keep recovery material until the replacement validates.

Names, row positions, and formatted display text are not stable identities. If an editable export cannot preserve a required manual field unambiguously, surface that constraint before rewriting it.

## Authenticated browser sessions

Use a browser session only when the user is authorized to access the source. Prefer supported APIs or exports. If the task specifically depends on the user's authenticated browser, use the session only to read the requested data and follow the source's documented access controls and rate limits. Do not collect or store passwords, expose cookies, bypass access controls or anti-bot mechanisms, or trigger messages and other unrelated actions. Keep credentials and session artifacts outside logs, source control, model prompts, and generated reports.

## Failure-injection checks

Test observable outcomes at the boundaries rather than only testing helper functions:

- Replay the same page, event, or pending batch; count and identity remain stable.
- Stop the process after each durable write, after temporary output creation, during replacement, and before the success marker; restart and verify recovery.
- Make output validation fail; the cursor stays fixed and the previous valid output remains available.
- Simulate a live lock, a stale lock, and a malformed fresh lock; ensure only a safe stale lock is reclaimed.
- Simulate rate limiting, timeout, authentication failure, malformed records, and an unknown business error; verify bounded retries and visible failure.
- Make optional enrichment fail; confirm raw records remain intact and the selected failure marker is visible.
- Return records out of order around the cursor; ensure pagination does not stop early unless ordering is guaranteed.
- Edit a manual spreadsheet field, refresh the corresponding source record, and verify the edit follows the stable key.
