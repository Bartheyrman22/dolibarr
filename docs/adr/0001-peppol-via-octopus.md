# Peppol delivery via Octopus, not direct from Dolibarr

## Context

Belgian B2B invoicing must move to Peppol under the 2026 mandate. We use Octopus Online as our bookkeeping SaaS, and Octopus is itself a certified Peppol Access Point.

## Decision

Dolibarr pushes validated sales invoices and credit notes to Octopus via its REST API. Octopus, after we Book the invoice, performs the Peppol delivery via `POST /invoices/send` (with automatic email fallback when the recipient is not Peppol-eligible). Dolibarr never speaks Peppol or UBL/AS4 itself.

## Why

- One integration to build and maintain instead of two (Octopus accounting + Peppol AP).
- Octopus is already a certified Peppol AP — outsourcing certification, key rotation, and AS4 transport to them.
- Eliminates the risk of double-send / divergence between bookkeeping and the dispatched invoice (UBL is generated from the booked entry by Octopus, so what is sent equals what is booked).
- The 2026 BE mandate is satisfied by Octopus's compliance, not ours.

## Consequences

- Customers not present in Octopus, or invoices Octopus refuses, cannot be Peppol'd from this system. If that ever becomes a regular need, an alternative path (e.g. a separate Belgian AP) would have to be added — non-trivial.
- Send latency depends on Octopus's processing window; we accept once-daily delivery as the contract.
- We have no direct visibility into Peppol-side delivery state beyond what Octopus reports.
