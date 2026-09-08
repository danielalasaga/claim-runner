# Spec: Live Accumulator Write-back — Data Service

**Pod spec number:** claim-runner/009
**Cpod spec:** claim-runner/cpod/008 (Live Accumulator Write-back)
**Service scope:** Data Service only
**Status:** Allocated
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow
**Allocated:** 2026-09-08

---

## Goal

Add a PATCH endpoint to the Data Service that accepts additive accumulator
deltas and applies them to the member's in-memory accumulator record. This is
the upstream dependency for the Claims Manager adjudication flow update
introduced in pod spec claim-runner/010. The Claims Manager spec must not merge
until this spec is merged and the endpoint is available.

Currently, the Data Service has no write path for accumulators. After this spec
is delivered, Claims Manager (pod spec claim-runner/010) can call this endpoint
after each successful adjudication so that subsequent claims for the same member
reflect live year-to-date spending.

---

## Out of Scope

- Writing accumulator state to `members.json` on disk — the data layer remains
  in-memory only (constitution: Data Layer, Decision 0010)
- Family deductible or family OOP max accumulators
- Accumulator reset at plan year rollover
- Any change to Claims Manager, Benefits Determiner, or Pricer
- Rollback or compensating writes — the PATCH endpoint accepts non-negative
  deltas only; no undo path exists (practicum limitation)

---

## Functional Requirements

### FR-1 — Accumulator delta endpoint

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

### FR-2 — Health check unchanged

`GET /health` continues to return `200`.

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

### PATCH /members/{member_id}/accumulators — Response (success)

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

This endpoint is new. It is called by Claims Manager (pod spec claim-runner/010)
after that spec is merged. No other service calls it.

### Existing routes unchanged

All existing Data Service endpoints (`POST /claims`, `GET /claims/{claim_id}`,
`GET /members/{member_id}`, `GET /health`, and the new `GET /claims?member_id=`
from pod spec claim-runner/006) are not modified.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Additive change only:** No existing Data Service routes are modified
- **In-memory only:** Accumulator state lives in the in-memory member store; no
  disk writes (constitution: Data Layer, Decision 0010)
- **Atomicity:** The delta and `met` recomputation are applied in a single
  in-memory operation per PATCH request; concurrent requests to the same member
  are not expected in the practicum configuration

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| `member_id` not found | `404` |
| Either delta is negative | `422` |
| Both deltas are zero | `200`; no-op; member record returned unchanged |
| Deductible delta brings `used` above `limit` | `used` increases by full delta amount (not clamped); `met` flips to `true` |
| OOP delta brings `used` to or above `limit` | `used` is clamped to `limit`; `met` flips to `true` |

---

## Constraints

- Additive endpoint only. No existing Data Service routes are modified.
- In-memory store only. `members.json` is not written at runtime.
  (Constitution: Data Layer, Decision 0010)
- The Claims Manager spec (pod spec claim-runner/010) depends on this endpoint
  and must not merge until this spec is merged.

---

## Acceptance Criteria

1. Applying a deductible delta updates `individual_deductible.used` and flips `met` when the limit is reached.

   ```
   Setup: MBR-10043 individual_deductible = { used: 125.00, limit: 500.00, met: false }

   PATCH /members/MBR-10043/accumulators
   { "individual_deductible_delta": 375.00, "individual_oop_delta": 0.00 }
   → 200
   { "accumulators": { "individual_deductible": { "used": 500.00, "met": true }, ... } }
   ```

2. Applying an OOP delta clamps `used` at `limit` and flips `met` to `true`.

   ```
   Setup: MBR-10044 individual_oop_max = { used: 1950.00, limit: 2000.00, met: false }

   PATCH /members/MBR-10044/accumulators
   { "individual_deductible_delta": 0.00, "individual_oop_delta": 75.00 }
   → 200
   { "accumulators": { "individual_oop_max": { "used": 2000.00, "met": true }, ... } }
   ```

3. A zero delta is accepted and the accumulator is unchanged.

   ```
   PATCH /members/MBR-10042/accumulators
   { "individual_deductible_delta": 0.00, "individual_oop_delta": 0.00 }
   → 200  (accumulators identical to pre-call values)
   ```

4. A negative delta returns `422`.

   ```
   PATCH /members/MBR-10042/accumulators
   { "individual_deductible_delta": -10.00, "individual_oop_delta": 0.00 }
   → 422
   ```

5. An unknown member returns `404`.

   ```
   PATCH /members/MBR-UNKNOWN/accumulators
   { "individual_deductible_delta": 30.00, "individual_oop_delta": 30.00 }
   → 404
   ```

6. Two sequential PATCH calls to the same member accumulate correctly.

   ```
   Setup: MBR-10042 individual_oop_max.used = 155.00

   First PATCH:  { "individual_deductible_delta": 0.00, "individual_oop_delta": 30.00 }
   → individual_oop_max.used = 185.00

   Second PATCH: { "individual_deductible_delta": 0.00, "individual_oop_delta": 40.00 }
   → individual_oop_max.used = 225.00
   ```

7. `GET /health` on Data Service returns `200` after this endpoint is added.
