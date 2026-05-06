# Dolibarr — Octopus Export

Custom Dolibarr module that pushes validated sales invoices and credit notes from Dolibarr to Octopus Online (Belgian bookkeeping SaaS), which then forwards them via Peppol.

## Language

### Domain

**Octopus**:
External Belgian bookkeeping SaaS and certified Peppol Access Point.
_Avoid_: "boekhouden API", "accountancy system"

**Octopus Relation**:
A customer/supplier party as known in Octopus, identified by VAT number.
_Avoid_: Octopus customer, Octopus client (Octopus uses "relatie")

**Octopus Account**:
A general-ledger account code in Octopus (e.g. `700000` for sales).
_Avoid_: ledger, GL code, grootboek

**Octopus VAT Code**:
A symbolic VAT classification used by Octopus (e.g. `21`, `06`, `IC`, `EX`, `MC`); not the same as a VAT rate %.
_Avoid_: VAT rate, BTW%, tarief

**Exportable Invoice**:
A Dolibarr `facture` of type 0 (sales) or 2 (credit note), in status VALIDATED, intended for push to Octopus.
_Avoid_: invoice, factuur (use only when scope is unambiguous)

**Export Queue**:
The internal queue of Exportable Invoices awaiting push to Octopus, including last attempt status and error.

### Octopus-side concepts

**Dossier**:
A bookkeeping file in Octopus (one per legal entity); identified by `dossierId`. All Octopus calls are scoped to a Dossier.

**Bookyear**:
A fiscal year inside a Dossier; identified by `bookyearKey.id`. Required on every invoice and book/send call.

**Journal**:
A bookkeeping journal (e.g. `V` = verkoop / sales) in which invoices are recorded. Sales export targets journal `V`.

**Document Sequence Number** (`documentSequenceNr`):
The position of an invoice inside `(Bookyear, Journal)`. **Client-supplied and required** when creating an invoice — Octopus does not auto-number. Together with `(Bookyear, Journal)` it uniquely identifies an Octopus invoice and is therefore our idempotency key.

**External Relation Id** (`externalRelationId`):
A client-supplied integer used to upsert an Octopus Relation idempotently. Set to Dolibarr `societe.rowid`.

**Book** (verb):
Commit one or more draft invoices in a Journal to the ledger via `POST /invoices/book`. Hard-capped at 24 calls/day → must batch (e.g. once daily).

**Send** (verb):
Deliver booked invoices to the customer via `POST /invoices/send`; tries Peppol, falls back to email when recipient is not Peppol-eligible.

### Process

**Validate (in Dolibarr)**:
Transition of an invoice from DRAFT to VALIDATED. Legal commit moment in BE; assigns final reference and locks lines. Triggers enqueue for Octopus export.

**Push**:
The act of creating one Exportable Invoice in Octopus (`POST /invoices`) plus uploading its PDF attachment. Does not Book or Send.

**Retry**:
A subsequent Push attempt after a previous failure, triggered manually from the invoice card or by the cron worker.

## Relationships

- A **Dolibarr Customer** (`societe`) maps 1:1 to an **Octopus Relation** via `externalRelationId = societe.rowid`; the upsert (`PUT /relations`) is performed before each first Push and is idempotent.
- An **Exportable Invoice** has many lines; each line carries one **Octopus Account** (per-product mapping) and one **Octopus VAT Code** (derived from rate + customer country + VAT validity).
- An **Exportable Invoice** flows: Validate → Push → Book (once-daily batch) → Send (Peppol or email). Book and Send are batched together once per accounting day.
- A failed Send is **not retried automatically**: the human operator decides whether to retry over Peppol or force email via the invoice card.

## Example dialogue

> **Dev:** When the user clicks Validate on a draft, do we Push to Octopus immediately and block on the API?
> **Domain expert:** No — Validate must always succeed locally. We enqueue the invoice and a worker performs the Push; if Octopus rejects, the Retry button lets the user re-try after fixing master data.

## Flagged ambiguities

- "Customer" is used in Dolibarr for `societe` and in Octopus for "relatie" — resolved: **Dolibarr Customer** vs **Octopus Relation**, linked by VAT number.
- "Account" is used for both bank/payment account and ledger code — resolved: ledger codes are **Octopus Account**; bank accounts are out of scope (no payment push).
