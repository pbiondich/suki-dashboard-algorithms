# CM-22 — Patient Volume & Throughput (algorithm binding)

**Canonical measure:** [CM-22 Patient Volume and Throughput](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-22%20Patient%20Volume%20and%20Throughput.md) · **D&M:** Organizational Impact
**Data tier:** Epic Clarity-computable · see [00-OVERVIEW](00-OVERVIEW.md) for shared methods.

## What we compute

- **CM-22a — completed encounters per provider per week** *(primary; outpatient productivity).*
- **CM-22b — ED throughput** (door-to-doc, length of stay) *(secondary; ED only).*

## Epic source & join path

**CM-22a** — single table **`PAT_ENC`** (PK `PAT_ENC_CSN_ID`):

| Field | Role |
|---|---|
| `ENC_CLOSED_YN` | **completed/signed** filter (`'Y'`) — the strict, correct denominator |
| `EFFECTIVE_DATE_DTTM` / `CONTACT_DATE` | visit date → weekly buckets |
| `VISIT_PROV_ID` | provider |
| `DEPARTMENT_ID`, `ENC_TYPE_C` | filters (specialty; drop types 1/31) |

> **Why `ENC_CLOSED_YN = 'Y'` and not appointment count?** The canonical measure (and
> Suki's own "total closed encounter count") define a completed encounter as one with a
> **signed note** — stricter than scheduled/checked-in counts. Unsigned notes inflate the
> count and the per-encounter denominators used in CM-20/CM-21.

**CM-22b (ED)** — door-to-doc and LOS come from **ED tracking / ADT**, not `PAT_ENC`
(`ED_IEV_EVENT_INFO` / ADT event tables; `⚠️ CONFIRM` which are in the export). Treat as a
separate ED-only panel; the strongest corpus data (Shuaib 2021) is a **human-scribe ED
baseline**, not ambient AI — label accordingly.

## Suki binding (attribution only)

Epic-derived; Suki supplies **go-live date** (pre/post) and optional **exposure flag**.
Repo convenience fields: `total_encounters` (monthly), `encounters_per_day` — `⚠️ CONFIRM`;
the `ENC_CLOSED_YN='Y'` Clarity count is the source of truth.

## Algorithm (pseudocode) — CM-22a

```sql
WITH wk AS (
  SELECT e.VISIT_PROV_ID,
         DATETRUNC(week, e.EFFECTIVE_DATE_DTTM) AS wk,
         COUNT(*) AS completed_encounters
  FROM PAT_ENC e
  WHERE e.ENC_CLOSED_YN = 'Y'
    AND e.ENC_TYPE_C NOT IN (1, 31)
  GROUP BY e.VISIT_PROV_ID, DATETRUNC(week, e.EFFECTIVE_DATE_DTTM)
)
SELECT wk.VISIT_PROV_ID,
       CASE WHEN wk.wk <  s.suki_golive                   THEN 'pre'
            WHEN wk.wk >= DATEADD(week, 6, s.suki_golive)  THEN 'post' END AS phase,
       AVG(wk.completed_encounters) AS avg_encounters_per_week,
       AVG(wk.completed_encounters
           / NULLIF(h.scheduled_clinical_hours, 0)) AS enc_per_clinical_hour  -- ⚠️ FTE/hours source
FROM wk
JOIN suki_adoption s ON s.epic_prov_id = wk.VISIT_PROV_ID          -- ⚠️ CONFIRM
LEFT JOIN provider_hours h ON h.prov_id = wk.VISIT_PROV_ID AND h.wk = wk.wk  -- ⚠️ CONFIRM
WHERE phase IS NOT NULL
GROUP BY wk.VISIT_PROV_ID, phase;
```

## Delta interpretation

More completed encounters/week after adoption → freed capacity (plausibly from lower
per-encounter documentation time). Holmgren 2026 found **+0.80 encounters/week** —
real but modest. **Effect direction is not automatically good**: more volume without
staffing/compensation adjustment can raise workload (note this on the dashboard).

## Confounders & guardrails

- **Normalize to scheduled clinical hours / FTE** — providers working more hours see more
  patients independent of Suki (hence the `enc_per_clinical_hour` line; needs an hours source).
- **Completed-only** — `ENC_CLOSED_YN='Y'`; never count unsigned notes.
- **Exclude** providers who changed specialty, site, or panel size mid-window.
- **Don't conflate** with financial productivity — throughput is patient *count*, not wRVUs
  ([CM-20](CM-20-financial-productivity.md)); the Shuaib "Productivity (%)" label is
  throughput, not RVU productivity.

## Open questions for the team

1. Source for **scheduled clinical hours / FTE** (for hours-normalized throughput)?
2. Is **ED throughput** (CM-22b) in scope, and are `ED_IEV_*`/ADT tables in the export?
3. Confirm Suki go-live / exposure fields and whether to use `total_encounters` pull.
