# Spec: Effective Date Eligibility Enforcement

**Intake ID:** 0007-effective-date-eligibility  
**Pod:** claim-runner  
**Pod spec number:** 007  
**Status:** Draft  
**Constitution references:** Technology Stack, Data Layer, Spec Scope, Adjudication Flow  
**Source:** control pod session 2026-09-03

---

## Goal

Close a known gap in the Benefits Determiner: claims submitted for a date of
service before a member's enrollment effective date are currently not caught and
pass through as eligible. This spec formalises the `effective_date` guard and
updates seed data to make the scenario exercisable.

The gap was identified in the spec 002 reconciliation note (run 2, 2026-09-02):
FR-1 specifies `effective_date <= date_of_service` but the pod's implementation
only checks `termination_date`. No existing acceptance criterion in spec 002
covers the pre-effective-date scenario because all seed members had an
`effective_date` of `2025-01-01`.

---

## Out of Scope

- Any change to Claims Manager, Pricer, or the Data Service API
- Family or group enrollment effective dates
- Retroactive enrollment corrections
- Any change to the `PLAN_TERMINATED` or auth-related determination logic

---

## Functional Requirements

### FR-1 — Effective date guard (Benefits Determiner)

In the eligibility evaluation in `POST /benefits/determine`, after confirming
the member exists (FR-1 in spec claim-runner/002), add the following check as
the first step:

If `date_of_service < enrollment.effective_date`:
- Set `eligible: false`
- Set `denial_reason: NOT_ELIGIBLE`
- Return immediately with empty `line_determinations`
- Do not proceed to the termination date check, network determination, or
  per-code evaluation

The effective date boundary is inclusive: `date_of_service == enrollment.effective_date`
is eligible and proceeds through the normal determination flow.

The check must be ordered before the termination date check. A member whose
`date_of_service` precedes `effective_date` returns `NOT_ELIGIBLE` regardless
of the termination date.

### FR-2 — Seed data update

At least two member records in `members.json` must have
`enrollment.effective_date` set to a date after `2025-01-01` (e.g. `2025-06-01`)
so that AC-1 is exercisable with a `date_of_service` in the same calendar year
without fabricating dates. Members MBR-10150 and MBR-10151 are reserved for this
purpose; they must be present in the seed data after this spec is delivered.

The seed data generation script must be updated to produce these members
consistently so that re-running the script does not remove them.

---

## Domain Model

### Endpoint

No new endpoint. The existing `POST /benefits/determine` endpoint is updated.

### Request (unchanged)

```json
{
  "member_id": "MBR-10150",
  "provider_id": "PRV-10001",
  "procedure_codes": ["99213"],
  "date_of_service": "2025-03-15"
}
```

### Response — date of service before effective date

```json
{
  "member_id": "MBR-10150",
  "plan_id": "PLN-GOLD-001",
  "eligible": false,
  "network_status": null,
  "overall_covered": false,
  "denial_reason": "NOT_ELIGIBLE",
  "line_determinations": []
}
```

This response shape is identical to the "member not found" case from spec 002.
The caller (Claims Manager) treats both as `NOT_ELIGIBLE`.

### Response — date of service on exactly the effective date

```json
{
  "member_id": "MBR-10150",
  "plan_id": "PLN-GOLD-001",
  "eligible": true,
  "network_status": "IN_NETWORK",
  "overall_covered": true,
  "line_determinations": [
    {
      "procedure_code": "99213",
      "covered": true,
      "requires_auth": false,
      "auth_on_file": null,
      "denial_reason": null
    }
  ]
}
```

### Denial reason code (updated table)

| Code | Scope | Trigger |
|---|---|---|
| `NOT_ELIGIBLE` | Claim | Member not found; no active enrollment on date of service; **or date of service before enrollment effective date** |
| `PLAN_TERMINATED` | Claim | Plan termination date precedes the date of service |
| `NOT_COVERED` | Line | Procedure excluded by or absent from the plan's covered list |
| `AUTH_REQUIRED_NOT_ON_FILE` | Line | Procedure requires auth; no valid auth found |

The bold addition documents the new trigger for `NOT_ELIGIBLE`.

---

## Integration

### Benefits Determiner → Data Service

No change to the Data Service API. Benefits Determiner already calls
`GET /members/{member_id}` and reads `enrollment.effective_date` from the
response. This spec adds logic that uses the field already present in the
member record.

---

## Non-functional Requirements

- **Language / framework:** Python 3.11+, FastAPI (constitution: Technology Stack)
- **Pure read:** No writes to any data file during normal operation
  (constitution: Data Layer)
- **Benefits Determiner only:** This spec changes no other service's API or
  behaviour (constitution: Spec Scope)
- **Backward compatible:** All eleven existing ACs from spec claim-runner/002
  must continue to pass

---

## Edge Cases

| Case | Expected behaviour |
|---|---|
| `date_of_service < enrollment.effective_date` | `eligible: false`, `denial_reason: NOT_ELIGIBLE`, empty `line_determinations` |
| `date_of_service == enrollment.effective_date` | Proceeds to normal determination; eligible if all other checks pass |
| `date_of_service > enrollment.effective_date` | Normal determination flow; no change to existing behaviour |
| `date_of_service < enrollment.effective_date` AND `termination_date` is in the past | `NOT_ELIGIBLE` returned; `PLAN_TERMINATED` is not evaluated because effective date check runs first |
| Member not found in Data Service | `NOT_ELIGIBLE` returned (existing behaviour, FR-1 in spec 002 — runs before effective date check) |

---

## Constraints

- This spec covers Benefits Determiner only. No change to Claims Manager, Pricer,
  or Data Service API contracts. (Constitution: Spec Scope)
- The effective date check is inserted before — not after — the termination date
  check. Sequencing is part of the acceptance criteria.
- This spec also requires a seed data update (FR-2). Seed data lives in the
  engineering pod repository. The spec documents the requirement; implementation
  is the pod's responsibility.
- No direct file access by Benefits Determiner. All data flows through the Data
  Service. (Constitution: Data Layer)

---

## Acceptance Criteria

1. A claim submitted for a date of service before the member's enrollment effective date returns `NOT_ELIGIBLE`.

   ```
   Setup: MBR-10150 has enrollment.effective_date = "2025-06-01"

   POST /benefits/determine
   {
     "member_id": "MBR-10150",
     "provider_id": "PRV-10001",
     "procedure_codes": ["99213"],
     "date_of_service": "2025-03-15"
   }
   → 200
   {
     "member_id": "MBR-10150",
     "plan_id": "PLN-GOLD-001",
     "eligible": false,
     "network_status": null,
     "overall_covered": false,
     "denial_reason": "NOT_ELIGIBLE",
     "line_determinations": []
   }
   ```

2. A claim submitted on exactly the effective date is eligible (boundary is inclusive).

   ```
   POST /benefits/determine
   {
     "member_id": "MBR-10150",
     "provider_id": "PRV-10001",
     "procedure_codes": ["99213"],
     "date_of_service": "2025-06-01"
   }
   → 200 { "eligible": true, "denial_reason": null, ... }
   ```

3. A claim submitted after the effective date proceeds through the normal determination flow.

   ```
   POST /benefits/determine
   {
     "member_id": "MBR-10150",
     "provider_id": "PRV-10001",
     "procedure_codes": ["99213"],
     "date_of_service": "2025-09-01"
   }
   → 200  (determination proceeds normally; result depends on plan coverage)
   ```

4. When both conditions apply — date of service before effective date AND enrollment terminated — `NOT_ELIGIBLE` is returned (effective date check precedes termination date check).

   ```
   Setup: MBR-10151 has effective_date = "2025-06-01" AND termination_date = "2025-08-31"

   POST /benefits/determine
   {
     "member_id": "MBR-10151",
     "procedure_codes": ["99213"],
     "date_of_service": "2025-03-01"
   }
   → 200 { "eligible": false, "denial_reason": "NOT_ELIGIBLE", "line_determinations": [] }
     (NOT "PLAN_TERMINATED")
   ```

5. Seed data contains at least two members (MBR-10150 and MBR-10151) with `enrollment.effective_date` after `2025-01-01`. These members are present after re-running the seed data generation script.

6. All eleven acceptance criteria from spec claim-runner/002 continue to pass without modification.

7. `GET /health` returns `200` when Benefits Determiner is running.
