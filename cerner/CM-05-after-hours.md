# CM-05 — After-Hours Documentation (Cerner Millennium proxy)

**Measure:** [CM-05](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-05%20After-Hours%20Documentation.md) · Cerner counterpart of the [Epic CM-05](../epic/CM-05-after-hours.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## The idea

"Pajama time" — documentation outside clinic hours. Same source as
[CM-04](CM-04-documentation-time.md): the time-of-day of `CLINICAL_EVENT` documentation
timestamps. A proxy for note-authoring after-hours (a subset of total after-hours EHR work),
needing no data beyond CM-04's.

## Metrics

- **CM-05a — after-hours share**: % of documentation events authored/verified after-hours.
- **CM-05b — after-hours minutes per day**: author→verify span (CM-04) falling after-hours,
  per provider per active day.

## Cerner source

`CLINICAL_EVENT.PERFORMED_DT_TM` (and `VERIFIED_DT_TM`), attributed to
`PERFORMED_PRSNL_ID`, joined to `ENCOUNTER` (`ENCNTR_ID`), filtered to the documentation
class — exactly as in CM-04. Add a **time-of-day classifier**.

> **Timezone matters.** `CLINICAL_EVENT` stores `PERFORMED_TZ` alongside the instant; convert
> to the provider/site local time before bucketing, or evening edits misclassify. (`⚠️
> CONFIRM` whether stored instants are local or UTC in this instance.)

> **"After-hours" definition** (`⚠️ CONFIRM`): start with a fixed window (before 07:00,
> after 18:00, weekends); upgrade to schedule-relative (outside the provider's scheduled
> sessions) if a scheduling feed is available.

## Algorithm (pseudocode)

```sql
WITH ev AS (
  SELECT ce.PERFORMED_PRSNL_ID AS author_id,
         localize(ce.PERFORMED_DT_TM, ce.PERFORMED_TZ) AS ts,   -- ⚠️ tz handling
         ce.PERFORMED_DT_TM AS visit_anchor
  FROM CLINICAL_EVENT ce
  JOIN ENCOUNTER e ON e.ENCNTR_ID = ce.ENCNTR_ID
  WHERE ce.EVENT_CLASS_CD = :doc_class_cd                  -- ⚠️ CONFIRM doc class
    AND ce.RESULT_STATUS_CD = :authenticated_cd
),
flagged AS (
  SELECT author_id, visit_anchor,
         CASE WHEN DATEPART(weekday, ts) IN (1,7)
                OR DATEPART(hour, ts) < 7 OR DATEPART(hour, ts) >= 18   -- ⚠️ window
              THEN 1 ELSE 0 END AS after_hours
  FROM ev
)
SELECT author_id, phase(visit_anchor, suki_golive) AS phase,    -- ⚠️ CONFIRM go-live
       AVG(after_hours * 1.0) AS after_hours_share
FROM flagged JOIN suki_adoption s ON s.epic_prov_id = author_id
WHERE phase IS NOT NULL GROUP BY author_id, phase;
```

CM-05b reuses CM-04's span minutes, bucketed by the same after-hours flag, divided by active
days per provider.

## Delta interpretation

A falling after-hours share after adoption → documentation moving back into the workday — a
burnout-adjacent outcome that pairs with the survey-based CM-01.

## Confounders & guardrails

- **Note-authoring after-hours only** — a floor on total after-hours EHR work (excludes
  In Basket/orders/chart review).
- **Window/timezone choices drive the number** — fix and report; handle `PERFORMED_TZ`.
- **Non-traditional shifts** misclassify under a fixed window — use schedule-relative where
  possible.
- **Calibrate** against Cerner efficiency telemetry where available.

## Open questions for the team

1. Are `CLINICAL_EVENT` instants local or UTC here; how is `PERFORMED_TZ` populated?
2. After-hours definition — fixed window or schedule-relative (schedule feed available)?
3. Headline metric — share (CM-05a) or minutes/day (CM-05b)?
