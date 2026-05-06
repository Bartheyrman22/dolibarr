# Octopus-leading document sequence number, discovered via `/invoices/modified`

## Context

Octopus requires a client-supplied `documentSequenceNr` per `(Bookyear, Journal)` on every invoice POST and does not auto-number. There is no `Idempotency-Key` header. The triple `(bookyearId, journalKey, documentSequenceNr)` is therefore the only natural idempotency anchor.

Two strategies are possible: (a) Dolibarr owns a counter table and is the sole authority; (b) Octopus is the authority — before each push we read the current max sequence from Octopus and increment in memory for the duration of the worker run. The closest endpoint that can return that max is `GET /invoices/modified`, which is rate-limited to **48 calls/day**.

## Decision

We use option (b): one `GET /invoices/modified` call at the start of each cron worker run, hold the next sequence number in PHP memory for that run, increment locally as we POST invoices in the same run. No persistent Dolibarr-side counter.

This caps the cron cadence at one run per ~30 minutes (48/day budget). Operationally, we run once daily for Push + Book + Send, so the budget is not strained.

## Why

- Removes a stateful Dolibarr-side counter that would otherwise need migration management, multi-instance locking, and a recovery path on drift.
- Octopus is already the source of truth for what is in the journal; making the client a second source invites divergence (e.g. accountant-side adjustments, deletions of draft invoices).
- The 48/day rate limit is acceptable because we batch: in-memory increment within a run amortises one `/modified` call across all the invoices pushed in that run.

## Consequences

- Cron cadence is hard-capped at ~48/day. Faster cadence (e.g. every 5 minutes) is not possible without a different discovery strategy.
- Concurrent worker runs would race; the design assumes a single worker.
- A failed POST after the in-memory counter advanced creates a transient gap; the next run re-reads max from Octopus and self-heals. Permanent gaps are tolerable in the V journal because Belgian audit cares about the invoice reference, not the journal sequence position.
