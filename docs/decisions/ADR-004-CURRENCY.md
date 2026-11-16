# ADR-004: Initial Currency and Amount Representation

- **Status:** Accepted
- **Date:** 2026-11-16

## Context
The reference project needs realistic multi-currency validation without implementing foreign-exchange conversion.

## Decision
Support ETB and USD initially. Both use two minor-unit decimal places. Store all amounts as integer minor units in a signed 64-bit integer. Currency is explicit and immutable for a payment. Do not use floating-point arithmetic.

## Consequences
- API amounts are integer minor units (for example, 1250 represents ETB 12.50).
- Currency support is validated against an allowlist.
- No FX rate lookup or conversion is performed.
- Adding currencies with different minor-unit exponents requires explicit currency metadata and additional tests.
