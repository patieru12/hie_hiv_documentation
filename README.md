# HIV Data Push — API Contract

**Version:** 0.1 (draft for review)
**Date:** 2026-09-03
**Direction:** Ministry of Health central data platform → **HIE**. The HIE stores; EMRs read from the HIE.
**Patient key:** TRACnet ID.

This document specifies the requests we will send. §11 lists what we need from you before
implementation starts.

---

## 1. Overview

We operate a central platform that continuously receives clinical data from health facilities
nationwide. This integration emits an event whenever HIV-relevant clinical data is recorded, amended,
or retracted for a patient in HIV care.

- **Push, not pull.** We call you. You persist. EMRs query you.
- **Cohort:** patients holding a TRACnet ID.
- **Content:** the agreed HIV clinical variable set — laboratory results, vital signs, and visit
  observations. Scope options in §11.4.

## 2. Transport

| | |
|---|---|
| Method | `POST` |
| Path | `/v1/hiv/events` *(to be confirmed by you)* |
| Content-Type | `application/json; charset=utf-8` |
| Encoding | `Content-Encoding: gzip` (please confirm support) |
| Auth | to be specified by you — see §11.1 |
| Batch size | ≤ 500 events or ≤ 5 MB uncompressed, whichever is reached first |
| Retry | exponential backoff 1s → 5m, up to 24h, then dead-letter and alert |

**Delivery is at-least-once.** You **must** deduplicate on `idempotency_key` (§3.2). We will resend
after a network timeout even in cases where you actually persisted the batch successfully.

## 3. Request body

### 3.1 Envelope

```json
{
  "batch_id": "01J9Z7K2Q4X8N5V3B6M1T0Y7WD",
  "sent_at": "2026-09-03T14:22:07+02:00",
  "source": { "system": "moh-central", "version": "1.0.0", "environment": "production" },
  "event_count": 3,
  "events": []
}
```

### 3.2 Fields on every event

| Field | Notes |
|---|---|
| `event_id` | UUIDv4, unique per **transmission**. Changes on retry. For tracing only — **do not deduplicate on this.** |
| `idempotency_key` | **Stable across retries and across amendments.** Deduplicate on this. |
| `event_type` | see §3.4 |
| `event_version` | integer from `1`, incremented on amendment or retraction of the same `idempotency_key` |
| `emitted_at` | transmission timestamp |
| `sequence` | monotonically increasing per `source.system`; lets you detect gaps and request replay |

### 3.3 Example — a viral load result

```json
{
  "event_id": "9f2c1e40-6a3b-4c8e-9d21-0b7a5e3f1c88",
  "idempotency_key": "obs:HC-0142:48127733",
  "event_type": "observation.recorded",
  "event_version": 1,
  "emitted_at": "2026-09-03T14:22:07+02:00",
  "sequence": 90124771,

  "patient": {
    "source_patient_key": "HC-0142:14582",
    "identifiers": [
      { "system": "RW-TRACNET", "value": "1234567890" }
    ],
    "demographics": {
      "gender": "F",
      "birth_date": "1990-04-17",
      "birth_date_estimated": false
    }
  },

  "facility": {
    "code": "HC-0142",
    "name": "Example Health Centre",
    "type": "HEALTH_CENTRE"
  },

  "observation": {
    "code": { "system": "OPENMRS-CONCEPT", "code": "856", "display": "HIV VIRAL LOAD" },
    "category": "LABORATORY",
    "observed_at": "2026-08-26T10:14:00+02:00",
    "value": { "type": "numeric", "numeric": 40, "unit": "copies/mL" },
    "encounter": { "type": "HIV VISIT", "occurred_at": "2026-08-26T09:50:00+02:00" }
  },

  "provenance": {
    "source_record_id": "obs:HC-0142:48127733",
    "recorded_at": "2026-08-26T10:31:22+02:00"
  }
}
```

### 3.4 Event types

| `event_type` | Meaning | Your action |
|---|---|---|
| `observation.recorded` | New clinical fact | Store |
| `observation.amended` | Value corrected at source | **Replace** the record with the same `idempotency_key`. Do not append a second copy. |
| `observation.retracted` | Withdrawn at source | **Stop serving it.** See §6. |
| `patient.identifier_linked` | A TRACnet ID became known, or was corrected | See §5. |

### 3.5 Value shapes

`value.type` is one of `numeric`, `coded`, `text`, `datetime`, `boolean`. Exactly one matching field
is populated. Values are typed — you should never need to parse a number out of a string.

```json
{ "type": "numeric",  "numeric": 40, "unit": "copies/mL" }
{ "type": "coded",    "coded": { "system": "OPENMRS-CONCEPT", "code": "703", "display": "POSITIVE" } }
{ "type": "text",     "text": "Patient reports good adherence" }
{ "type": "datetime", "datetime": "2026-08-26T00:00:00+02:00" }
{ "type": "boolean",  "boolean": true }
```

---

## 4. Timestamps — please read

**Every clinical timestamp carries an explicit `+02:00` offset. We will never send a bare `Z` on a
clinical field.**

Source timestamps are Kigali local wall-clock time. Interpreting them as UTC shifts every reading two
hours earlier, which crosses a date boundary for anything recorded before 02:00 — a result recorded at
01:30 on the 26th would be stored as the 25th.

If you ever receive a bare `Z` on `observed_at`, `occurred_at`, `recorded_at`, `assigned_at`,
`retracted_at`, or a `datetime` value, treat it as a defect on our side and tell us.

`sent_at` and `emitted_at` are true instants and may carry any offset.

---

## 5. Patient identity and late linking

### 5.1 Two keys

| Key | Always present? | Purpose |
|---|---|---|
| `source_patient_key` | **Yes** | Stable, opaque, pseudonymous. Treat as an opaque string — do not parse it. |
| `RW-TRACNET` identifier | Usually | The national HIV programme identifier. Your primary index. |

TRACnet is present for the large majority of this cohort (§10), but a patient can have clinical data
recorded before their TRACnet ID is captured. Rather than withhold that data, we send it immediately
keyed on `source_patient_key`, with the `identifiers` array empty or partial.

### 5.2 When the identifier arrives

```json
{
  "event_id": "3d81b0aa-7e55-4f19-b2c6-9a4d8e70f512",
  "idempotency_key": "identity:HC-0142:14582:RW-TRACNET:1234567890",
  "event_type": "patient.identifier_linked",
  "event_version": 1,
  "emitted_at": "2026-09-03T14:22:07+02:00",
  "sequence": 90124772,

  "patient": {
    "source_patient_key": "HC-0142:14582",
    "identifiers": [
      { "system": "RW-TRACNET", "value": "1234567890", "assigned_at": "2026-09-01T08:12:00+02:00" }
    ],
    "demographics": { "gender": "F", "birth_date": "1990-04-17", "birth_date_estimated": false }
  },

  "supersedes": null
}
```

**On receipt you must retroactively attach every previously received event bearing that
`source_patient_key` to the TRACnet identity.** We will not replay those events.

### 5.3 Corrections

If a TRACnet ID was recorded incorrectly and later fixed, we send the same event type with
`supersedes` populated:

```json
"identifiers": [ { "system": "RW-TRACNET", "value": "1234567890" } ],
"supersedes":  { "system": "RW-TRACNET", "value": "1234567899" }
```

**You must relink the affected records from the superseded value to the new one — not duplicate
them.** Identifier corrections do occur in practice, and a stale link would merge two unrelated
patients' HIV records. This is the most consequential failure mode in the integration and it is worth
explicit test coverage on both sides.

### 5.4 Decision needed: unlinked records

This design means you will hold HIV clinical records that are not yet associated with any national
identifier, keyed only on `source_patient_key`, until a linking event arrives.

The alternative is that we withhold such records until a TRACnet ID exists — which keeps your store
free of unlinked data but delays clinical information indefinitely.

**This is a policy decision for both parties. See §11.3.**

---

## 6. Retractions

When a record is withdrawn at source we send:

```json
{
  "idempotency_key": "obs:HC-0142:48127733",
  "event_type": "observation.retracted",
  "event_version": 2,
  "retraction": {
    "reason": "Entered on the wrong patient",
    "retracted_at": "2026-09-02T11:04:00+02:00"
  }
}
```

**You must stop serving the record identified by that `idempotency_key`.** Whether you tombstone or
hard-delete is your choice, but it must no longer be retrievable by any consuming EMR.

Retractions are roughly 1.3% of event volume (§10). Handling them correctly is not optional: an
incorrect clinical result that remains queryable will be acted on by clinicians at other facilities.

---

## 7. Expected response

Please return per-event status, so that one malformed record does not cost the whole batch:

```json
{
  "batch_id": "01J9Z7K2Q4X8N5V3B6M1T0Y7WD",
  "received_at": "2026-09-03T12:22:09Z",
  "accepted": 2,
  "rejected": 1,
  "results": [
    { "idempotency_key": "obs:HC-0142:48127733", "status": "accepted" },
    { "idempotency_key": "identity:HC-0142:14582:RW-TRACNET:1234567890", "status": "duplicate" },
    { "idempotency_key": "obs:HC-0311:9912",
      "status": "rejected",
      "error": { "code": "UNKNOWN_CONCEPT", "message": "concept 999999 not recognised" } }
  ]
}
```

| Response | Our behaviour |
|---|---|
| `2xx` with per-event `accepted` / `duplicate` | Complete |
| Per-event `rejected` | Logged, dead-lettered, alerted. Not retried blindly — a rejection indicates a contract mismatch to resolve between us |
| `4xx` on the whole batch | Not retried. Alerted |
| `429` | We honour `Retry-After` |
| `5xx` or timeout | Retried with backoff |

**A `2xx` with no `results` array is treated as full acceptance of every event in the batch.** If you
partially fail without telling us, that data is silently lost. Please always return `results`.

---

## 8. What we do not send

- No patient names, phone numbers, or addresses.
- No national ID number — TRACnet only, unless you require another identifier for matching.
- Demographics are limited to gender and birth date, included solely to support identity matching.
- `source_patient_key` is an opaque pseudonym with no meaning outside our platform.

Every transmission is recorded in an append-only audit log on our side.

---

## 9. Ordering, gaps and replay

- `sequence` increases monotonically per source. A gap means events are in flight or were lost. You
  may report the highest contiguous sequence you hold and we will replay from that point.
- **Events may arrive out of clinical order.** Facilities synchronise on different schedules and a
  facility that has been offline will deliver a backlog. **Order by `observation.observed_at`, never
  by arrival order or `sequence`.**
- Historical backfill uses this same endpoint with the same `idempotency_key` values, so it is safe to
  run more than once, and safe to interleave with live traffic.

---

## 10. Volumes for capacity planning

Measured across all participating facilities, September 2026.

| | |
|---|---|
| Patients in the cohort | ~64,600 |
| TRACnet coverage within the cohort | ~98.6% |
| **Historical backfill (one-off)** | **~7.8M events** |
| **Steady state** | **~1,900 events/day** (~81/hour) |
| Peak hour observed | ~734 events |
| Peak day observed | ~4,970 events |
| Weekday vs weekend | ~2,700/day vs ~125/day — a 20× swing |
| Retraction events | ~26/day (1.3% of volume) |

**Size for the peak, not the mean.** The weekday/weekend swing is the dominant pattern; clinics are
largely closed at weekends.

The backfill is the only large number, and we will pace it to whatever rate you specify.

---

## 11. What we need from you

1. **Authentication.** mTLS, OAuth2 client credentials, or a static bearer token? Who issues and
   rotates credentials, and how?
2. **Endpoints and environments.** Confirmed path, plus a sandbox we can integrate against before
   production.
3. **Unlinked records (§5.4).** May you hold HIV records keyed only on `source_patient_key` until a
   TRACnet ID arrives — yes or no?
4. **Scope.** Which set do you want?

   | Option | Backfill | Steady state |
   |---|---|---|
   | **A — full HIV clinical variable set** (results + vitals + visit observations) | ~7.8M | ~1,900/day |
   | B — HIV-specific results only (viral load, CD4, ARV) | ~780K | ~160/day |
   | C — viral load and CD4 only | ~290K | subset of B |

   We recommend **A**. The volume difference is immaterial at this scale, and vital signs alongside
   laboratory results are what make the record clinically useful to a receiving clinician.

5. **Terminology.** Can you accept OpenMRS concept identifiers as sent, or do you require a mapped
   terminology such as LOINC or SNOMED CT? If mapping is required, who owns and maintains the map?
6. **Backfill.** Do you want the full history, or should we start from go-live? If history, over what
   period and at what rate limit?
7. **Throughput and burst tolerance** on your side, so we can configure batching and pacing.
8. **Contact and escalation path** for rejected events and integration incidents.
