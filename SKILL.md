---
name: reliable-incremental-collector
description: Design and implement reliable incremental collectors for growing sources such as APIs, authenticated browser sessions, feeds, or queues, including durable deduplication, resumable runs, safe outputs, and bounded failure handling. Use for new collectors or substantial extensions; do not use for one-off exports or generic scraping without incremental state.
license: MIT
---

# Reliable Incremental Collector

Use this skill when a data source is read repeatedly and each run must avoid duplicates, survive interruption, and publish a trustworthy result. Keep the design specific to the user's source and destination; do not add a generic framework unless the request calls for one.

## Establish the contract before coding

Inspect the existing repository, schemas, entry points, and tests before proposing edits. State assumptions that affect correctness. Resolve these questions from the source or ask the user when they are product decisions:

- What stable source identity makes a record the same record across runs? Use a composite key when identity is scoped by tenant, job, account, or another parent.
- Which source timestamp or sequence defines the incremental boundary? Separate it from local acquisition time.
- What does the cursor mean: the newest fully published source value, a page token, a sequence, or a run checkpoint? Define the first-run lookback, overlap window, retention limit, and behavior after a long gap.
- Is the source ordered strongly enough to stop pagination early? Only stop on a boundary when that ordering is verified. Otherwise scan the relevant retention range and deduplicate locally.
- Which fields are source-owned, derived, or manually edited? Define the authoritative owner and a stable row binding before writing a spreadsheet or other editable output.
- Which transformations are optional, such as an LLM summary? Define their privacy boundary and whether one failure blocks the batch.

## Build the smallest durable state machine

Represent a run explicitly, for example `PENDING` followed by `SUCCESS`. Store enough information to distinguish a collection-incomplete run from a collection-complete run waiting for publication, and to resume either one without duplicating completed records.

1. Acquire a single-instance lock before any state or output mutation.
2. Discover and validate the source scope and its expected identity before collecting records.
3. Read pages or events using a bounded concurrency. Validate stable identity and boundary values; fail closed when required values are missing.
4. Insert records idempotently using the stable key. Persist bounded batches so a long optional transformation does not hold one transaction open.
5. Keep the run pending while collection, enrichment, and outputs are being reconciled. Mark collection complete only after the source boundary has been fully processed; advance the source cursor only after every required output has passed validation.
6. On startup, recover valid temporary or swap files and handle pending runs before collecting new data: resume collection-incomplete runs from their checkpoint or safely re-read their boundary, and rebuild outputs for collection-complete runs. Recovery must be idempotent.

Do not describe a database commit and several files as one cross-system atomic transaction. Use a pending run, validation, backups, and deterministic recovery to make the boundary observable and safe.

## Handle errors deliberately

Classify errors by consequence rather than catching everything:

- Retry only bounded transient failures such as rate limits, timeouts, and temporary server errors, with a small fixed retry budget and backoff.
- Skip a record only for a documented, exact business condition whose identity and scope are known. Record the skip so a later run does not repeatedly process it.
- Make skipped-record retention explicit: cache permanent skips only for durable conditions, and define when temporary or recheckable conditions become eligible again.
- Abort the affected run on unknown, malformed, authentication, schema, or identity errors. Do not silently turn data loss into a successful run.
- For optional enrichment, preserve the source record and store an explicit failure marker when policy allows degradation. Keep the first useful error for diagnostics.

Choose a conservative concurrency bound based on source limits and test ordering, throttling, and partial failure. Add user-facing concurrency configuration only when the workflow needs it. Send only the minimum fields required by an external model or service.

## Publish outputs safely

Before changing an existing output, create a durable backup when the operation can alter user data. Write a temporary artifact in the same destination area, validate its structure and row identity, then replace the destination atomically when the platform supports it. Keep a swap backup until validation succeeds; if the process stops between replacement and cleanup, validate both copies and restore the known-good one.

For editable outputs:

- Keep manual fields separate from source refresh fields, or merge them with an explicit field-ownership rule.
- Bind rows by stable identity or a verified source link, never by display order alone.
- Validate headers, sheet or table structure, row counts, links, and preserved manual values after migration or refresh.
- Treat native hyperlinks and formulas as distinct representations; verify the representation that users actually need.

Read [references/recovery-patterns.md](references/recovery-patterns.md) when implementing pending-run recovery, editable spreadsheet synchronization, browser-session access, or the associated failure tests.

## Acceptance checks

The implementation is ready only when tests or reproducible checks demonstrate:

- Repeating the same source page or event is idempotent, while the same source identity under different parents remains distinct.
- A crash after durable data insertion but before output publication leaves the run pending and is recovered on the next start.
- A failed output validation does not advance the cursor and does not destroy the previous valid output.
- A stale lock can be recovered safely, while a live or freshly malformed lock prevents a second writer.
- File-in-use, malformed output, missing identity, authentication, and unknown business errors fail visibly.
- Optional enrichment failure preserves the source record according to the chosen policy.
- Unsorted source results do not trigger unsafe early termination; proven ordering does.
- Retention gaps and first-run behavior are explicit, logged, and covered by tests.

Keep the final implementation narrow. Do not copy a source site's endpoints, credentials, personal data, machine paths, arbitrary batch sizes, or project-specific field names into this skill.
