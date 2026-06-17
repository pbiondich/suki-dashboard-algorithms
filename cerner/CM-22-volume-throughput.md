# CM-22 — Patient Volume & Throughput (Cerner Millennium binding)

**Measure:** [CM-22](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-22%20Patient%20Volume%20and%20Throughput.md) · Cerner counterpart of the [Epic CM-22](../epic/CM-22-volume-throughput.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## What we compute

- **CM-22a — completed encounters per provider per week** *(primary)*.
- **CM-22b — visit cycle time / ED throughput** `ARRIVE_DT_TM → DEPART_DT_TM` *(secondary)*.

## Cerner source & join path

**`ENCOUNTER`** (PK `ENCNTR_ID`) provides everything; provider via `ENCNTR_PRSNL_RELTN`.

| Field | Role |
|---|---|
| `ENCNTR_COMPLETE_DT_TM` | **completed** filter (not null) — the strict denominator |
| `ARRIVE_DT_TM`, `DEPART_DT_TM` | cycle-time / throughput (CM-22b) |
| `ENCNTR_TYPE_CD`, `ENCNTR_STATUS_CD` | filters |
| `ENCNTR_PRSNL_RELTN` (`PRSNL_PERSON_ID`, `ENCNTR_ID`) | provider attribution (attending reltn) |

> **Completed = signed/closed**, mirroring the canonical "total closed encounter count":
> require `ENCNTR_COMPLETE_DT_TM` populated (and/or `ENCNTR_STATUS_CD` = completed), not a
> scheduled or merely arrived encounter — unsigned/incomplete rows inflate the count and the
> per-encounter denominators used in CM-20/CM-21.

## Algorithm (pseudocode) — CM-22a

```sql
WITH wk AS (
  SELECT r.PRSNL_PERSON_ID AS provider_id,
         DATETRUNC(week, e.ARRIVE_DT_TM) AS wk,
         COUNT(*) AS completed_encounters
  FROM ENCOUNTER e
  JOIN ENCNTR_PRSNL_RELTN r ON r.ENCNTR_ID = e.ENCNTR_ID
        AND r.ACTIVE_IND = 1 AND r.ENCNTR_PRSNL_R_CD = :attending_reltn_cd   -- ⚠️ CONFIRM
  WHERE e.ENCNTR_COMPLETE_DT_TM IS NOT NULL
    AND e.ENCNTR_TYPE_CD IN (:clinical_types)
  GROUP BY r.PRSNL_PERSON_ID, DATETRUNC(week, e.ARRIVE_DT_TM)
)
SELECT provider_id, phase(wk, suki_golive) AS phase,                          -- ⚠️ go-live
       AVG(completed_encounters) AS avg_encounters_per_week,
       AVG(completed_encounters / NULLIF(h.scheduled_clinical_hours,0)) AS enc_per_hour  -- ⚠️ hours source
FROM wk JOIN suki_adoption s ON s.epic_prov_id = wk.provider_id
LEFT JOIN provider_hours h ON h.prov_id = wk.provider_id AND h.wk = wk.wk     -- ⚠️ CONFIRM
WHERE phase IS NOT NULL GROUP BY provider_id, phase;
```

CM-22b: `AVG(DATEDIFF(minute, ARRIVE_DT_TM, DEPART_DT_TM))` per provider/period (ED-style
throughput); for door-to-doc you'd need the ED tracking events (`⚠️ CONFIRM` ED event
tables present in the export).

## Delta interpretation

More completed encounters/week after adoption → freed capacity. Holmgren 2026: **+0.80
encounters/week** — real but modest. Higher volume isn't automatically good (workload).

## Confounders & guardrails

- **Completed-only** (`ENCNTR_COMPLETE_DT_TM`); never count scheduled/arrived rows.
- **Normalize to scheduled hours / FTE** — needs an hours source.
- **Exclude** providers who changed specialty/site/panel mid-window.
- **Throughput ≠ financial productivity** — patient *count*, not wRVUs ([CM-20](CM-20-financial-productivity.md)).

## Open questions for the team

1. Attending-relationship `code_value`; completed-status definition (`ENCNTR_COMPLETE_DT_TM`
   vs `ENCNTR_STATUS_CD`).
2. Scheduled clinical hours / FTE source for hours-normalized throughput.
3. ED throughput in scope? (door-to-doc needs ED tracking tables — confirm presence.)
