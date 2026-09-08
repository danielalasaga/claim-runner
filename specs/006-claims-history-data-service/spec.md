# Spec: Member Claims History — Data Service

**Pod spec number:** claim-runner/006
**Cpod spec:** claim-runner/cpod/006 (Member Claims History)
**Service scope:** Data Service only
**Status:** Allocated
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow
**Allocated:** 2026-09-08

---

## Goal

Add a query endpoint to the Data Service that returns all claim ledger records
for a given member. This is the upstream dependency for the Claims Manager route
introduced in pod spec claim-runner/007. The Claims Manager spec must not merge
until this spec is merged and the endpoint is available.

---

## Out of Scope

- Pagination or cursor-based scrolling
- Filtering by status, date range, or procedure code
- Aggregating or summarising claim history
- Any change to Claims Manager, Benefits Determiner, or Pricer
- Sorting — the Data Service returns records in store order; Claims Manager
  applies reverse-chronological sort (pod spec claim-runner/007)

---

## Functional Requirements

### FR-1 — Member claims query endpoint

`GET /claims?member_id={member_id}` scans the in-memory claims store and
returns every claim ledger record whose `member_id` matches the query parameter.

If no records match, the service returns `200` with an empty `records` array. A
`member_id` parameter that matches no member is not an error at this layer —
the store does not validate member existence for query requests.

The `member_id` query parameter is required. Omitting it returns `422`.

### FR-2 — Health check unchanged

`GET /health` continues to return `200`.

---

## Domain Model

### GET /claims?member_id={id} — Request

| Parameter | Type | Required | Description |
|---|---|---|---|
| `member_id` | query string | Yes | Member identifier to retrieve history for |

### GET /claims?member_id={id} — Response (records found)

```json
{
  "member_id": "MBR-10042",
  "records": [
    { "<full claim ledger record>" },
    { "<full claim ledger record>" }
  ]
}
```

Each entry in `records` is a full claim ledger record — the same structure
stored in the in-memory claims store, identical to what `GET /claims/{claim_id}`
returns for a single record.

### GET /claims?member_id={id} — Response (no records)

```json
{
  "member_id": "MBR-UNKNOWN",
  "records": []
}
```

### Status codes

| HTTP Status | Condition |
|---|---|
| `200` | Query succeeded; `records` array may be empty |
| `422` | `member_id` query parameter omitted |

---

## Integration

This endpoint is new. It is called by Claims Manager (pod spec claim-runner/007)
after that spec is merged. No other service calls it.

### Existing routes unchanged

`POST /claims`, `GET /claims/{claim_id}`, and `GET /health` are not modified.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Additive change only:** No existing Data Service routes are modified
- **In-memory only:** The query scans the in-memory claims store; no disk access
  (constitution: Data Layer, Decision 0010)
- **No sort requirement:** Records are returned in store insertion order; sorting
  is the caller's responsibility

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| `member_id` omitted from query | `422` |
| `member_id` matches no claims | `200` with `records: []` |
| `member_id` identifies a known member with no claims | `200` with `records: []` |
| Single claim on file for the member | `records` array with one entry |

---

## Constraints

- Additive endpoint only. No existing Data Service routes are modified.
- In-memory store only. No disk access. (Constitution: Data Layer, Decision 0010)
- The Claims Manager spec (pod spec claim-runner/007) depends on this endpoint
  and must not merge until this spec is merged.

---

## Acceptance Criteria

1. Two claims for the same member are returned in the `records` array.

   ```
   Setup: submit CLM-20250901-001 (MBR-10042) and CLM-20250901-002 (MBR-10042)
          via the normal adjudication flow.

   GET /claims?member_id=MBR-10042  (on Data Service directly)
   → 200
   {
     "member_id": "MBR-10042",
     "records": [
       { "claim_id": "CLM-20250901-001", ... },
       { "claim_id": "CLM-20250901-002", ... }
     ]
   }
   ```

2. An unknown `member_id` returns an empty `records` array, not `404`.

   ```
   GET /claims?member_id=MBR-UNKNOWN  (on Data Service directly)
   → 200 { "member_id": "MBR-UNKNOWN", "records": [] }
   ```

3. A known member with no adjudicated claims returns an empty `records` array.

   ```
   GET /claims?member_id=MBR-10001  (member in members.json; no claims this session)
   → 200 { "member_id": "MBR-10001", "records": [] }
   ```

4. Omitting the `member_id` query parameter returns `422`.

   ```
   GET /claims   (no query param, on Data Service directly)
   → 422
   ```

5. Each entry in `records` has the same structure as `GET /claims/{claim_id}`.

   ```
   GET /claims/CLM-20250901-002  (on Data Service directly)
   → 200 { "claim_id": "CLM-20250901-002", "status": "PAID", ... }

   GET /claims?member_id=MBR-10042
   → records contains an entry with identical fields and values
   ```

6. `GET /health` on Data Service returns `200` after this endpoint is added.
