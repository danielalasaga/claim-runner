# Spec: Live Accumulator Write-back

**Intake ID:** 0008-accumulator-write-back  
**Pod:** claim-runner  
**Pod spec number:** 008  
**Status:** Draft  
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow  
**Source:** control pod session 2026-09-03

---

## Goal

After each successful adjudication, Claims Manager writes the updated accumulator
balances back to the Data Service so that subsequent claims for the same member
reflect live year-to-date spending rather than the static seeded snapshot.

Currently, the Pricer returns an `accumulator_snapshot` showing
`_used_before` and `_used_after` values. Claims Manager surfaces these in the
response but does not update the Data Service. This spec closes that gap.

Once accumulator values reflect actual accumulated spend, member record responses
from the Data Service become materially more useful to any caller that reads them.

---

## Out of Scope

- Writing accumulator state to `members.json` on disk — the data layer remains
  in-memory only (constitution: Data Layer, Decision 0010)
- Family deductible or family OOP max accumulators
- Accumulator reset at plan year rollover
- Benefits Determiner or Pricer

---

## Functional Requirements

### FR-1 — Accumulator delta endpoint on Data Service

`PATCH /members/{member_id}/accumulators` applies additive delta values to the
member's in-memory accumulator record.

Request body:

```json
{
  "individual_deductible_delta": 375.00,
  "individual_oop_delta": 405.00
}
```

Both fields are required. Both must be non-negative numbers. A negative delta
returns `422`. A delta of `0.00` is accepted and is a no-op for that
accumulator.

The service applies each delta atomically to the in-memory record:

```
individual_deductible.used = individual_deductible.used + individual_deductible_delta
individual_oop_max.used    = individual_oop_max.used    + individual_oop_delta
```

After applying, the service recomputes `met` for each accumulator:

```
individual_deductible.met = (individual_deductible.used >= individual_deductible.limit)
individual_oop_max.met    = (individual_oop_max.used    >= individual_oop_max.limit)
```

`used` is clamped to at most `limit` for `individual_oop_max` (the member
cannot exceed the OOP ceiling). `individual_deductible.used` may exceed
`individual_deductible.limit` by the delta amount if the deductible was nearly
met; it is not clamped.

Returns `200` with the full updated member record on success. Returns `404` if
the member is not found. Returns `422` if either delta is negative.

### FR-2 — Write-back in Claims Manager adjudication flow

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

### FR-3 — Write-back only on successful adjudications

Accumulator write-back occurs only when the claim produces a `PAID` or
`PARTIALLY_PAID` status. `DENIED`, `VALIDATION_ERROR`, and `CONFLICT` outcomes
do not trigger a PATCH call.

A `DENIED` claim has no Pricer response and no deltas to apply. A
`VALIDATION_ERROR` or `CONFLICT` claim never reaches the Pricer.

### FR-4 — Health check

`GET /health` on Claims Manager and Data Service continues to return `200`.

---

## Domain Model

### PATCH /members/{member_id}/accumulators — Request

```json
{
  "individual_deductible_delta": 375.00,
  "individual_oop_delta": 405.00
}
```

| Field | Type | Required | Constraint |
|---|---|---|---|
| `individual_deductible_delta` | number | Yes | >= 0 |
| `individual_oop_delta` | number | Yes | >= 0 |

### PATCH /members/{member_id}/accumulators — Response

```json
{
  "member_id": "MBR-10043",
  "first_name": "...",
  "last_name": "...",
  "accumulators": {
    "plan_year": "2025",
    "individual_deductible": { "limit": 500.00, "used": 500.00, "met": true },
    "family_deductible": null,
    "individual_oop_max": { "limit": 4000.00, "used": 530.00, "met": false },
    "family_oop_max": null
  },
  "..."
}
```

Full member record is returned so callers can verify the resulting accumulator
state.

### PATCH /members/{member_id}/accumulators — Status codes

| HTTP Status | Condition |
|---|---|
| `200` | Deltas applied; full updated member record in body |
| `404` | Member not found |
| `422` | Either delta is negative |

---

## Integration

### New: Claims Manager → Data Service (accumulator write-back)

```
PATCH http://${DATA_SERVICE_URL}/members/{member_id}/accumulators
Content-Type: application/json

{
  "individual_deductible_delta": <number>,
  "individual_oop_delta": <number>
}
```

`DATA_SERVICE_URL` defaults to `http://localhost:8083`.

Called after the Pricer returns, before `POST /claims` writes the ledger entry.
If this call fails, the ledger write is aborted and the batch returns `503`.

### Updated: Claims Manager → Data Service (inter-service contract change)

The existing `architecture/inter-service-contracts.md` section for Claims
Manager → Data Service must be updated to document the new PATCH endpoint.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **In-memory only:** Accumulator state lives in the Data Service in-memory
  store; no disk writes (constitution: Data Layer, Decision 0010)
- **Atomicity:** The delta and `met` recomputation are applied in a single
  in-memory operation per PATCH request; concurrent requests to the same member
  are not expected in the practicum configuration
- **Additive to adjudication flow:** The PATCH call and delta computation are
  inserted after the Pricer returns. No existing adjudication behaviour changes.

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| Copay-only claim (no deductible applied) | `deductible_delta = 0`; PATCH still called; `individual_deductible.used` unchanged |
| Claim that fully meets the deductible | `used` equals `limit`; `met` flips to `true` |
| Claim where OOP max is reached mid-claim | Pricer caps member liability; delta reflects capped amount; `individual_oop_max.met` flips to `true` |
| DENIED claim | No PATCH call; accumulators unchanged |
| VALIDATION_ERROR claim | No PATCH call; accumulators unchanged |
| Negative delta submitted directly to Data Service PATCH | `422` |
| Data Service returns `404` on PATCH | Claims Manager returns `503`; claim not written to ledger |
| Data Service unreachable during PATCH | Claims Manager returns `503`; claim not written to ledger |
| Two claims for same member submitted sequentially | Second claim reads the updated accumulator (written by first claim's PATCH); running totals accumulate correctly |

---

## Constraints

- This spec touches both Data Service (new PATCH endpoint) and Claims Manager
  (adjudication flow update). Per the pod-local constitution (Spec Scope), it
  must be decomposed into two per-service specs at allocation time: one for Data
  Service and one for Claims Manager. The pod should merge the Data Service spec
  first.
- Accumulator state is in-memory only; `members.json` is not written at runtime.
  (Constitution: Data Layer, Decision 0010)
- No direct file access by Claims Manager. (Constitution: Data Layer)
- Claims Manager is the sole orchestrator; it initiates the PATCH call. Benefits
  Determiner and Pricer do not call the Data Service PATCH endpoint.
  (Constitution: Adjudication Flow)

---

## Acceptance Criteria

1. After a PAID claim where only a copay applies (no deductible), the member's OOP accumulator increases by the copay amount; the deductible accumulator is unchanged.

   ```
   Setup: MBR-10042 individual_deductible = { used: 125.00, limit: 500.00, met: false }
                    individual_oop_max    = { used: 155.00, limit: 4000.00, met: false }

   POST /claims/batch
   { claim for MBR-10042, procedure 99213, billed 250.00 }
   → PAID, deductible_applied: 0.00, copay_applied: 30.00, oop_delta: 30.00

   GET /members/MBR-10042  (via Data Service)
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

7. `PATCH /members/{member_id}/accumulators` with a negative delta returns `422`.

   ```
   PATCH /members/MBR-10042/accumulators
   { "individual_deductible_delta": -10.00, "individual_oop_delta": 0.00 }
   → 422
   ```

8. `GET /health` on Claims Manager returns `200 { "status": "UP" }`.

9. `GET /health` on Data Service returns `200` with accurate record counts.
