# CM-06 — Chart Closure Timeliness (Cerner Millennium binding)

**Measure:** [CM-06](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-06%20Chart%20Closure%20Timeliness.md) · Cerner counterpart of the [Epic CM-06](../epic/CM-06-chart-closure.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## What we compute

How quickly the chart is completed after the visit. On Millennium this is **native and
exact** — `ENCOUNTER.CHART_COMPLETE_DT_TM` is a true datetime (Epic's `ENC_CLOSE_DATE` was
date-only).

- **CM-06a — 24-hour closure rate**: % of encounters with chart complete within 24h.
- **CM-06b — same-day closure rate**: % complete by midnight of the visit date.
- **CM-06c — median hours-to-close**: `CHART_COMPLETE_DT_TM − ARRIVE_DT_TM`.

## Cerner source & join path

Single table — **`ENCOUNTER`** (PK `ENCNTR_ID`):

| Field | Role |
|---|---|
| `CHART_COMPLETE_DT_TM` | chart-complete instant (the closure event) |
| `ARRIVE_DT_TM` / `REG_DT_TM` | visit start (closure clock starts here) |
| `ENCNTR_COMPLETE_DT_TM` | encounter-completed instant (alt definition) |
| `ENCNTR_TYPE_CD`, `ENCNTR_STATUS_CD` | filters (drop non-clinical/pre-reg types) |

Provider for the cohort: `ENCNTR_PRSNL_RELTN` (`ENCNTR_ID ↔ PRSNL_PERSON_ID`, attending
relationship type — `⚠️ CONFIRM` the `ENCNTR_PRSNL_R_CD` value).

> **Note-level alternative.** For *note signing* specifically (rather than whole-chart
> completion), use `CLINICAL_EVENT.VERIFIED_DT_TM` vs `PERFORMED_DT_TM` for documentation
> events — see [CM-04](CM-04-documentation-time.md). `CHART_COMPLETE_DT_TM` is the
> institutional chart-closure metric and the right default here.

## Algorithm (pseudocode)

```sql
WITH enc AS (
  SELECT e.ENCNTR_ID, e.CHART_COMPLETE_DT_TM, e.ARRIVE_DT_TM,
         DATEDIFF(hour, e.ARRIVE_DT_TM, e.CHART_COMPLETE_DT_TM) AS hrs_to_close,
         r.PRSNL_PERSON_ID AS provider_id
  FROM ENCOUNTER e
  JOIN ENCNTR_PRSNL_RELTN r
    ON r.ENCNTR_ID = e.ENCNTR_ID AND r.ACTIVE_IND = 1
   AND r.ENCNTR_PRSNL_R_CD = :attending_reltn_cd        -- ⚠️ CONFIRM code_value
  WHERE e.CHART_COMPLETE_DT_TM IS NOT NULL
    AND e.ENCNTR_TYPE_CD IN (:clinical_types)            -- ⚠️ CONFIRM type set
)
SELECT provider_id, phase(ARRIVE_DT_TM, suki_golive) AS phase,   -- ⚠️ CONFIRM go-live
       AVG(CASE WHEN hrs_to_close <= 24 THEN 1.0 ELSE 0 END) AS rate_24h,   -- CM-06a
       AVG(CASE WHEN CAST(CHART_COMPLETE_DT_TM AS DATE) = CAST(ARRIVE_DT_TM AS DATE)
                THEN 1.0 ELSE 0 END)                          AS rate_same_day, -- CM-06b
       MEDIAN(hrs_to_close)                                   AS median_hrs    -- CM-06c
FROM enc JOIN suki_adoption s ON s.epic_prov_id = enc.provider_id
WHERE phase IS NOT NULL GROUP BY provider_id, phase;
```

## Delta interpretation

Higher 24h/same-day rates and lower median hours after adoption → charts close sooner.
Boyter/KLAS 2025 reported **−41%** chart-closure time.

## Confounders & guardrails

- **Convention** (24h vs same-day) materially changes the number — fix and report it.
- **Encounter-type filter** drives the denominator; hold constant across pre/post.
- Distinguish whole-chart completion from note signing (use `CLINICAL_EVENT` for the latter).
- Normalize within specialty; exclude providers who changed role/site mid-window.

## Open questions for the team

1. Attending-relationship `code_value` in `ENCNTR_PRSNL_RELTN`, and the clinical
   `ENCNTR_TYPE_CD` set.
2. `CHART_COMPLETE_DT_TM` vs `ENCNTR_COMPLETE_DT_TM` — which matches the site's "chart
   closed" definition?
3. Suki go-live / exposure fields.
