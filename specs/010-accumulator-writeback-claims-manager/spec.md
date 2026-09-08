# Spec: Live Accumulator Write-back — Claims Manager

**Pod spec number:** claim-runner/010
**Cpod spec:** claim-runner/cpod/008 (Live Accumulator Write-back)
**Service scope:** Claims Manager only
**Status:** Allocated
**Sequencing:** Must not merge until pod spec claim-runner/009 (Data Service — Live Accumulator Write-back) is merged
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow
**Allocated:** 2026-09-08

---

## Goal

After each successful adjudication, Claims Manager writes the updated accumulator
balances back to the Data Service so that subsequent claims for the same member
reflect live year-to-date spending rather than the static seeded snapshot.

Currently, the Pricer returns an `accumulator_snapshot` showing
`_used_before` and `_used_after` values. Claims Manager surfaces these in the
response but does not update the Data Service. This spec closes that gap by
inserting a PATCH call to the Data Service accumulator endpoint introduced in
pod spec claim-runner/009.

---

## Out of Scope

- Family deductible or family OOP max accumulators
- Accumulator reset at plan year rollover
- Benefits Determiner or Pricer changes
- Any change to the Data Service API (that is covered by pod spec claim-runner/009)

---

## Functional Requirements

### FR-1 — Write-back in adjudication flow

After the Pricer returns and before writing the claim to the ledger, Claims
Manager computes the accumulator deltas from the Pricer response and calls the
Data Service.

Compute deltas:

```
deductible_delta = accumulator_snapshot.individual_deductible_used_after
                 - accumulator_snapshot.individual_deductible_used_before
oop_delta        = accumulator_snapshot.individual_oop_used_after
                 - accumulator_snapshot.individual_oop_used_before
```

Call `PATCH /members/{member_id}/accumulators` with these deltas.

If the PATCH returns `503` or the Data Service is unreachable, Claims Manager
returns `503` to the caller and does not write the claim to the ledger.

If the PATCH returns `404` (member no longer in the Data Service — unusual but
possible), Claims Manager returns `503` with a descriptive error and does not
write the claim.

### FR-2 — Write-back only on successful adjudications

Accumulator write-back occurs only when the claim produces a `PAID` or
`PARTIALLY_PAID` status. `DENIED`, `VALIDATION_ERROR`, and `CONFLICT` outcomes
do not trigger a PATCH call.

A `DENIED` claim has no Pricer response and no deltas to apply. A
`VALIDATION_ERROR` or `CONFLICT` claim never reaches the Pricer.

### FR-3 — Health check unchanged

`GET /health` continues to return `200 { "status": "UP" }`.

---

## Domain Model

### PATCH call from Claims Manager to Data Service

```
PATCH http://${DATA_SERVICE_URL}/members/{member_id}/accumulators
Content-Type: application/json

{
  "individual_deductible_delta": <number>,
  "individual_oop_delta": <number>
}
```

`DATA_SERVICE_URL` defaults to `http://localhost:8083`.

This endpoint was introduced in pod spec claim-runner/009. This spec must not
merge until pod spec claim-runner/009 is merged.

Called after the Pricer returns, before `POST /claims` writes the ledger entry.
If this call fails, the ledger write is aborted and the batch returns `503`.

---

## Integration

### Updated: Claims Manager → Data Service (inter-service contract change)

The existing `architecture/inter-service-contracts.md` section for Claims
Manager → Data Service must be updated to document the new PATCH endpoint.

### Existing routes unchanged

`POST /claims/batch`, `GET /claims/{claim_id}`, `GET /claims?member_id=` (pod
spec claim-runner/007), and `GET /health` on Claims Manager are not modified.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Additive to adjudication flow:** The PATCH call and delta computation are
  inserted after the Pricer returns. No existing adjudication behaviour changes.
- **No direct file access:** Claims Manager does not access the accumulator
  store directly; all data flows through the Data Service (constitution: Data Layer)

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| Copay-only claim (no deductible applied) | `deductible_delta = 0`; PATCH still called; `individual_deductible.used` unchanged |
| Claim that fully meets the deductible | `used` equals `limit`; `met` flips to `true` |
| Claim where OOP max is reached mid-claim | Pricer caps member liability; delta reflects capped amount; `individual_oop_max.met` flips to `true` |
| DENIED claim | No PATCH call; accumulators unchanged |
| VALIDATION_ERROR claim | No PATCH call; accumulators unchanged |
| Data Service returns `404` on PATCH | Claims Manager returns `503`; claim not written to ledger |
| Data Service unreachable during PATCH | Claims Manager returns `503`; claim not written to ledger |
| Two claims for same member submitted sequentially | Second claim reads the updated accumulator (written by first claim's PATCH); running totals accumulate correctly |
| PATCH returns `200`, then ledger write returns `5xx` | Claims Manager returns `503`; accumulator increment is not rolled back — no compensating write is possible; practicum limitation |

---

## Constraints

- Claims Manager only. No changes to Data Service API, Benefits Determiner, or
  Pricer. (Constitution: Spec Scope)
- No direct file access by Claims Manager. (Constitution: Data Layer)
- Claims Manager is the sole orchestrator; it initiates the PATCH call. Benefits
  Determiner and Pricer do not call the Data Service PATCH endpoint.
  (Constitution: Adjudication Flow)
- This spec depends on pod spec claim-runner/009 (Data Service — Live Accumulator
  Write-back). It must not merge until pod spec claim-runner/009 is merged.

---

## Acceptance Criteria

1. After a PAID claim where only a copay applies (no deductible), the member's OOP accumulator increases by the copay amount; the deductible accumulator is unchanged.

   ```
   Setup: MBR-10042 individual_deductible = { used: 125.00, limit: 500.00, met: false }
                    individual_oop_max    = { used: 155.00, limit: 4000.00, met: false }

   POST /claims/batch
   { claim for MBR-10042, procedure 99213, billed 250.00 }
   → PAID, deductible_applied: 0.00, copay_applied: 30.00, oop_delta: 30.00

   GET /members/MBR-10042  (via Claims Manager → Data Service)
   → individual_deductible = { used: 125.00, met: false }  ← unchanged
     individual_oop_max    = { used: 185.00, met: false }  ← 155.00 + 30.00
   ```

2. After a PAID claim where deductible is applied, the deductible accumulator increases correctly and `met` flips when the limit is reached.

   ```
   Setup: MBR-10043 individual_deductible = { used: 125.00, limit: 500.00, met: false }

   POST /claims/batch
   { claim for MBR-10043, procedure 42820 (surgical), billed 5000.00 }
   → deductible_applied: 375.00

   GET /members/MBR-10043  (via Data Service)
   → individual_deductible = { used: 500.00, met: true }
   ```

3. When the OOP max is reached, `met` flips to `true` and `used` is clamped to `limit`.

   ```
   Setup: MBR-10044 individual_oop_max = { used: 1950.00, limit: 2000.00, met: false }

   POST /claims/batch
   { claim for MBR-10044 with gross member_liability 75.00 }
   → Pricer caps member_liability to 50.00 (OOP ceiling); oop_delta: 50.00

   GET /members/MBR-10044  (via Data Service)
   → individual_oop_max = { used: 2000.00, met: true }
   ```

4. A DENIED claim does not update accumulators.

   ```
   Setup: record MBR-10042 accumulator values before submit.

   POST /claims/batch
   { claim for MBR-10042 for an excluded procedure }
   → DENIED

   GET /members/MBR-10042  (via Data Service)
   → accumulators identical to pre-submit values
   ```

5. A VALIDATION_ERROR claim does not update accumulators.

   ```
   POST /claims/batch  { claim missing required field }
   → per-claim status: VALIDATION_ERROR

   GET /members/{member_id}  (via Data Service)
   → accumulators unchanged
   ```

6. Two sequential claims for the same member accumulate correctly.

   ```
   Setup: MBR-10042 individual_oop_max.used = 155.00

   First claim:  copay 30.00  → oop.used = 185.00
   Second claim: copay 40.00  → oop.used = 225.00

   GET /members/MBR-10042 after both claims
   → individual_oop_max.used = 225.00
   ```

7. `GET /health` on Claims Manager returns `200 { "status": "UP" }`.
