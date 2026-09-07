# Spec: Claim Correction and Re-adjudication

**Intake ID:** 0009-claim-re-adjudication  
**Pod:** claim-runner  
**Pod spec number:** 009  
**Status:** Draft  
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow  
**Source:** control pod session 2026-09-03

---

## Goal

Allow a previously adjudicated claim to be corrected and re-adjudicated. A
caller submits a claim via `POST /claims/batch` with `allow_resubmit: true`; if
the claim ID already exists in the ledger, the claim is re-run through the full
adjudication pipeline and the existing ledger entry is replaced with the new
result.

Currently, re-submitting a known claim ID always returns `CONFLICT` and no
re-adjudication occurs. This spec adds an opt-in re-submission path without
changing default behaviour.

This feature builds on spec claim-runner/007 (effective date eligibility
enforcement): the effective-date eligibility guard must run on re-submitted
claims exactly as it does on new claims, which is satisfied by having
claim-runner/007 in place.

---

## Out of Scope

- Bulk correction workflows
- Reversal of accumulator state from the original adjudication
- Appeals or administrative override of denial reasons
- Audit trail or history of superseded versions
- Benefits Determiner or Pricer (their behaviour is unchanged)

---

## Functional Requirements

### FR-1 — Per-claim re-submission flag

`POST /claims/batch` accepts a per-claim `allow_resubmit` boolean field
(default `false`). The existing CONFLICT behaviour is unchanged when
`allow_resubmit` is absent or `false`.

### FR-2 — Re-adjudication flow

When a claim is submitted with `allow_resubmit: true` and the claim ID already
exists in the Data Service ledger:

1. Re-adjudicate the claim from the beginning: validate fields, check member
   existence (GET /members), call Benefits Determiner, call Pricer (if
   applicable).
2. The effective-date eligibility check (claim-runner/007) runs as part of the
   normal Benefits Determiner call. A re-submitted claim is not exempt from
   eligibility rules.
3. After adjudication completes, call `PUT /claims/{claim_id}` on the Data
   Service to replace the existing ledger entry.
4. Return the new adjudication result to the caller.

If `allow_resubmit: true` and the claim ID does not exist in the ledger,
adjudicate normally and write via `POST /claims` (not `PUT`).

### FR-3 — PUT endpoint on Data Service

`PUT /claims/{claim_id}` replaces the stored claim record for the given ID with
the request body. The existing in-memory entry is overwritten atomically.

Returns `200` with the replaced record on success. Returns `404` if
`claim_id` is not found — callers should use `POST /claims` to create new
records.

### FR-4 — No change to default CONFLICT behaviour

When `allow_resubmit` is absent or `false` and the claim ID already exists, the
response is unchanged from the current implementation: per-claim
`status: "CONFLICT"`, `errors: ["claim_id already exists"]`.

### FR-5 — Health check

`GET /health` on Claims Manager and Data Service continues to return `200`.

---

## Domain Model

### POST /claims/batch — Updated request shape

```json
{
  "claims": [
    {
      "claim_id": "CLM-20250901-001",
      "member_id": "MBR-10042",
      "provider_id": "PRV-90210",
      "date_of_service": "2025-09-01",
      "allow_resubmit": true,
      "claim_lines": [
        {
          "line_number": 1,
          "procedure_code": "99213",
          "diagnosis_codes": ["Z00.00"],
          "units": 1,
          "billed_amount": 275.00
        }
      ]
    }
  ]
}
```

`allow_resubmit` is optional and defaults to `false`. All other fields are
unchanged from spec claim-runner/001.

### POST /claims/batch — CONFLICT response (allow_resubmit: false, unchanged)

```json
{
  "results": [
    {
      "claim_id": "CLM-20250901-001",
      "status": "CONFLICT",
      "errors": ["claim_id already exists"],
      "denial_reasons": [],
      "line_detail": []
    }
  ]
}
```

### POST /claims/batch — Re-adjudication response (allow_resubmit: true)

```json
{
  "results": [
    {
      "claim_id": "CLM-20250901-001",
      "status": "PAID",
      "adjudicated_at": "2025-09-02T10:15:00Z",
      "totals": {
        "billed_amount": 275.00,
        "allowed_amount": 115.00,
        "member_liability": 30.00,
        "payer_liability": 85.00
      },
      "denial_reasons": [],
      "errors": [],
      "line_detail": [ { "..." } ]
    }
  ]
}
```

The `adjudicated_at` timestamp reflects the re-adjudication time, not the
original submission time.

### PUT /claims/{claim_id} — Request

Full claim ledger record (same schema as `POST /claims`). The `claim_id` in the
request body must match the path parameter.

### PUT /claims/{claim_id} — Response

| HTTP Status | Condition |
|---|---|
| `200` | Record replaced; body is the new stored record |
| `404` | `claim_id` not found in the in-memory store |
| `422` | Request body is missing required fields |

---

## Integration

### New: Claims Manager → Data Service (claim replace)

```
PUT http://${DATA_SERVICE_URL}/claims/{claim_id}
Content-Type: application/json

{ <full claim ledger record> }
```

`DATA_SERVICE_URL` defaults to `http://localhost:8083`.

Called after re-adjudication when `allow_resubmit: true` and the claim ID
already exists. If the PUT returns `5xx` or the Data Service is unreachable,
Claims Manager returns `503`.

### Updated: Claims Manager → Data Service (inter-service contract change)

`architecture/inter-service-contracts.md` must be updated to document the new
PUT endpoint in the Claims Manager → Data Service section.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Backward compatible:** `allow_resubmit` defaults to `false`. All existing
  ACs from spec claim-runner/001 continue to pass without modification.
- **In-memory only:** The PUT operation replaces the in-memory record; no disk
  write occurs (constitution: Data Layer, Decision 0010)
- **Eligibility not bypassed:** Re-adjudicated claims go through the full
  Benefits Determiner call; there is no fast-path that skips eligibility checks

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| `allow_resubmit: false` (or absent), claim ID exists | `CONFLICT` — existing behaviour unchanged |
| `allow_resubmit: true`, claim ID exists | Re-adjudicate; replace ledger entry via PUT |
| `allow_resubmit: true`, claim ID does not exist | Adjudicate normally; write via POST |
| Re-submitted claim fails validation | `VALIDATION_ERROR`; existing ledger entry is not replaced |
| Re-submitted claim for a date before enrollment effective date (claim-runner/007) | Benefits Determiner returns `NOT_ELIGIBLE`; per-claim result is `DENIED`; existing ledger entry is replaced with the DENIED result |
| Re-submitted claim for a member with a terminated plan | Benefits Determiner returns `PLAN_TERMINATED`; existing ledger entry is replaced with the DENIED result |
| Batch mixing re-submit and new claims | Both processed independently; re-submit path and new-claim path applied per claim |
| `PUT /claims/{claim_id}` — ID not in store | `404`; caller should use POST for new records |
| Data Service unreachable during PUT | Claims Manager returns `503`; original ledger entry unchanged |

---

## Constraints

- This spec touches both Data Service (new PUT endpoint) and Claims Manager
  (CONFLICT branch updated). Per the pod-local constitution (Spec Scope), it
  must be decomposed into two per-service specs at allocation time: one for Data
  Service and one for Claims Manager. The pod should merge the Data Service spec
  first.
- This spec depends on claim-runner/007 (effective date eligibility enforcement).
  claim-runner/007 must be implemented before this spec enters the clarify phase.
- The `allow_resubmit` flag is per-claim, not per-batch. A batch may contain
  both re-submitted and new claims.
- Eligibility rules are not waived on re-submission. Benefits Determiner is
  called with the re-submitted claim's fields, and all determinations —
  including effective date enforcement — apply.
- The full adjudication pipeline runs on re-submission. There is no partial
  re-adjudication (e.g., skipping Benefits Determiner and repeating only Pricer).
- No direct file access by Claims Manager. (Constitution: Data Layer)
- Claims Manager is the sole orchestrator. (Constitution: Adjudication Flow)

---

## Acceptance Criteria

1. Re-submitting an existing claim without the flag returns `CONFLICT` — no regression.

   ```
   Setup: CLM-20250901-001 already in the ledger (PAID).

   POST /claims/batch
   { "claims": [{ "claim_id": "CLM-20250901-001", "allow_resubmit": false, ... }] }
   → 200 { "results": [{ "status": "CONFLICT", "errors": ["claim_id already exists"] }] }
   ```

2. Re-submitting with the flag re-adjudicates and returns a fresh result.

   ```
   POST /claims/batch
   { "claims": [{ "claim_id": "CLM-20250901-001", "allow_resubmit": true,
                  "billed_amount": 275.00, ... }] }
   → 200 { "results": [{ "status": "PAID", "adjudicated_at": "<new timestamp>", ... }] }
   ```

3. After re-adjudication, `GET /claims/{claim_id}` returns the new result, not the original.

   ```
   GET /claims/CLM-20250901-001
   → 200 { "adjudicated_at": "<new timestamp>", "totals": { "billed_amount": 275.00, ... }, ... }
   ```

4. A batch with `allow_resubmit: true` on a new claim ID adjudicates normally and creates a new ledger entry via POST (not PUT).

   ```
   POST /claims/batch
   { "claims": [{ "claim_id": "CLM-20250902-999", "allow_resubmit": true, ... }] }
   → 200 { "results": [{ "status": "PAID", ... }] }

   GET /claims/CLM-20250902-999  → 200  (new record created)
   ```

5. A re-submitted claim for a DOS before the member's enrollment effective date (claim-runner/007) returns `NOT_ELIGIBLE`.

   ```
   Setup: MBR-10150 has enrollment.effective_date = "2025-06-01" (per claim-runner/007 seed data).
          CLM-EARLY is already in the ledger (originally submitted with a valid date).

   POST /claims/batch
   { "claims": [{ "claim_id": "CLM-EARLY", "allow_resubmit": true,
                  "member_id": "MBR-10150", "date_of_service": "2025-03-01", ... }] }
   → 200 { "results": [{ "status": "DENIED",
                          "denial_reasons": [{ "code": "NOT_ELIGIBLE" }] }] }

   GET /claims/CLM-EARLY  → DENIED result (ledger entry replaced)
   ```

6. A batch mixing a re-submitted claim and a new claim processes both correctly in one request.

   ```
   POST /claims/batch
   { "claims": [
       { "claim_id": "CLM-EXISTING", "allow_resubmit": true, ... },
       { "claim_id": "CLM-NEW-001",  "allow_resubmit": false, ... }
   ]}
   → 200 { "results": [
       { "claim_id": "CLM-EXISTING", "status": "PAID", ... },
       { "claim_id": "CLM-NEW-001",  "status": "PAID", ... }
   ]}
   ```

7. `PUT /claims/{claim_id}` returns `404` when the claim ID is not in the store.

   ```
   PUT /claims/CLM-DOES-NOT-EXIST  { <valid claim body> }
   → 404
   ```

8. `GET /health` on Claims Manager returns `200 { "status": "UP" }`.

9. `GET /health` on Data Service returns `200` with accurate record counts.
