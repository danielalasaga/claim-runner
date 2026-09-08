# Spec: Member Claims History — Claims Manager

**Pod spec number:** claim-runner/007
**Cpod spec:** claim-runner/cpod/006 (Member Claims History)
**Service scope:** Claims Manager only
**Status:** Allocated
**Sequencing:** Must not merge until pod spec claim-runner/006 (Data Service — Member Claims History) is merged
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow
**Allocated:** 2026-09-08

---

## Goal

Add a `GET /claims?member_id={member_id}` route to Claims Manager that delegates
to the Data Service query introduced in pod spec claim-runner/006, applies
reverse-chronological sorting, and returns the result to callers. Claims Manager
remains the sole external entry point for claim history.

---

## Out of Scope

- Pagination or cursor-based scrolling
- Filtering by status, date range, or procedure code
- Aggregating or summarising claim history
- Any change to the Data Service, Benefits Determiner, or Pricer
- Any change to the adjudication flow or the `POST /claims/batch` path

---

## Functional Requirements

### FR-1 — Member claims history route

`GET /claims?member_id={member_id}` on Claims Manager (port 8080) calls
`GET /claims?member_id={member_id}` on the Data Service and assembles the
response. It sorts the returned records in reverse chronological order by
`adjudicated_at` (most recent first) before returning.

The `member_id` query parameter is required. Omitting it returns `422`.

### FR-2 — Response shape

The top-level response is an object with `member_id` (echoed from the query
parameter) and `results` (array). Each entry in `results` has exactly the same
shape as a single item in `POST /claims/batch results` and as `GET
/claims/{claim_id}`.

Note: the Data Service response uses `records`; Claims Manager renames the array
to `results` in its own response.

An unknown `member_id` or a member with no adjudicated claims returns `200`
with an empty `results` array, not `404`.

### FR-3 — Error propagation

If the Data Service returns `5xx` or is unreachable, Claims Manager returns
`503`.

### FR-4 — Health check unchanged

`GET /health` continues to return `200`.

---

## Domain Model

### GET /claims?member_id={id} — Claims Manager request

| Parameter | Type | Required | Description |
|---|---|---|---|
| `member_id` | query string | Yes | Member identifier to retrieve history for |

### GET /claims?member_id={id} — Claims Manager response (records found)

```json
{
  "member_id": "MBR-10042",
  "results": [
    {
      "claim_id": "CLM-20250901-002",
      "status": "PAID",
      "adjudicated_at": "2025-09-01T14:31:00Z",
      "totals": {
        "billed_amount": 250.00,
        "allowed_amount": 115.00,
        "member_liability": 30.00,
        "payer_liability": 85.00
      },
      "denial_reasons": [],
      "errors": [],
      "line_detail": [
        {
          "line_number": 1,
          "procedure_code": "99213",
          "billed_amount": 250.00,
          "allowed_amount": 115.00,
          "contractual_adjustment": 135.00,
          "deductible_applied": 0.00,
          "copay_applied": 30.00,
          "coinsurance_applied": 0.00,
          "member_liability": 30.00,
          "payer_liability": 85.00,
          "adjustment_reason_code": "CO-45",
          "denial_reason": null,
          "line_status": "PAID"
        }
      ]
    },
    {
      "claim_id": "CLM-20250901-001",
      "status": "DENIED",
      "adjudicated_at": "2025-09-01T14:30:00Z",
      "totals": { "..." },
      "denial_reasons": [{ "code": "NOT_COVERED" }],
      "errors": [],
      "line_detail": [ { "..." } ]
    }
  ]
}
```

Results are sorted by `adjudicated_at` descending (most recent first). Each
entry in `results` is structurally identical to a `GET /claims/{claim_id}`
response.

### GET /claims?member_id={id} — Claims Manager response (no records)

```json
{
  "member_id": "MBR-UNKNOWN",
  "results": []
}
```

---

## Integration

### Claims Manager → Data Service (member query)

```
GET http://${DATA_SERVICE_URL}/claims?member_id={member_id}
```

`DATA_SERVICE_URL` defaults to `http://localhost:8083`.

This endpoint was introduced in pod spec claim-runner/006. This spec must not
merge until pod spec claim-runner/006 is merged.

If the Data Service returns `5xx` or is unreachable, Claims Manager returns
`503`.

### Existing routes unchanged

`POST /claims/batch`, `GET /claims/{claim_id}`, and `GET /health` on Claims
Manager are not modified. The new route is additive.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Additive change only:** No existing Claims Manager routes are modified; no
  adjudication behaviour changes
- **No direct data access:** Claims Manager does not access the claims store
  directly; all data flows through the Data Service (constitution: Data Layer)
- **Sort order:** Most-recently-adjudicated first; ties in `adjudicated_at` may
  be returned in any stable order

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| `member_id` omitted from query | `422` |
| `member_id` matches no claims | `200` with `results: []`; not `404` |
| `member_id` identifies a known member with no claims | `200` with `results: []` |
| Single claim on file for the member | `results` array with one entry |
| Data Service returns `5xx` | Claims Manager returns `503` |
| Data Service unreachable | Claims Manager returns `503` |

---

## Constraints

- Claims Manager only. No changes to Data Service, Benefits Determiner, or
  Pricer. (Constitution: Spec Scope)
- No direct file or store access by Claims Manager. All data flows through the
  Data Service. (Constitution: Data Layer)
- Claims Manager remains the sole external entry point. (Constitution:
  Adjudication Flow)
- This spec depends on pod spec claim-runner/006 (Data Service — Member Claims
  History). It must not merge until pod spec claim-runner/006 is merged.

---

## Acceptance Criteria

1. Two claims for the same member are retrievable together in reverse-chronological order.

   ```
   Setup: submit CLM-20250901-001 (for MBR-10042) at T=14:30, then CLM-20250901-002 (for MBR-10042) at T=14:31.

   GET /claims?member_id=MBR-10042  (on Claims Manager, port 8080)
   → 200
   {
     "member_id": "MBR-10042",
     "results": [
       { "claim_id": "CLM-20250901-002", "adjudicated_at": "...14:31...", ... },
       { "claim_id": "CLM-20250901-001", "adjudicated_at": "...14:30...", ... }
     ]
   }
   ```

2. An unknown `member_id` returns an empty `results` array, not `404`.

   ```
   GET /claims?member_id=MBR-UNKNOWN  (on Claims Manager)
   → 200 { "member_id": "MBR-UNKNOWN", "results": [] }
   ```

3. A known member with no adjudicated claims returns an empty `results` array.

   ```
   GET /claims?member_id=MBR-10001  (member in members.json; no claims this session)
   → 200 { "member_id": "MBR-10001", "results": [] }
   ```

4. Each entry in `results` has the same shape as `GET /claims/{claim_id}`.

   ```
   GET /claims/CLM-20250901-002  (on Claims Manager)
   → 200 { "claim_id": "CLM-20250901-002", "status": "PAID", "totals": {...}, "line_detail": [...], ... }

   GET /claims?member_id=MBR-10042  (on Claims Manager)
   → results[0] has identical fields and values to the single-claim response above
   ```

5. Omitting the `member_id` query parameter returns `422`.

   ```
   GET /claims   (on Claims Manager, no query param)
   → 422
   ```

6. `GET /health` on Claims Manager returns `200 { "status": "UP" }` after the new route is deployed.
