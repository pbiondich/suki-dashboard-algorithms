# CM-06 — Chart Closure Timeliness (algorithm binding)

**Canonical measure:** [CM-06 Chart Closure Timeliness](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-06%20Chart%20Closure%20Timeliness.md) · **D&M:** Individual Impact
**Data tier:** Epic Clarity-computable · see [00-OVERVIEW](00-OVERVIEW.md) for shared methods.

## What we compute

Completion *behavior* — how quickly the chart is closed after the visit — not time-spent
(that's CM-04). Three sub-metrics, in priority order:

- **CM-06a — 24-hour closure rate**: % of encounters closed within 24h of the visit. *(Preferred for pre/post; avoids penalizing late-day visits.)*
- **CM-06b — Same-day closure rate**: % closed by midnight of the visit date. *(Stricter; matches many institutional dashboards.)*
- **CM-06c — Median time-to-close (hours)**: median(close − visit) across encounters.

## Epic source & join path

Single table — **`PAT_ENC`** (one row per encounter, PK `PAT_ENC_CSN_ID`):

| Field | Role |
|---|---|
| `ENC_CLOSE_DATE` | when the encounter/chart was closed |
| `ENC_CLOSED_YN` | closed flag — filter to `'Y'` |
| `CONTACT_DATE` / `EFFECTIVE_DATE_DTTM` | visit date / visit start datetime |
| `VISIT_PROV_ID` | provider (cohort + within-provider comparison) |
| `DEPARTMENT_ID`, `ENC_TYPE_C` | filters (specialty, drop reg/PCP-change types 1/31) |

> **⚠️ Granularity caveat.** `ENC_CLOSE_DATE` is a **date**, not a timestamp. For a true
> *same-day* or *hours-to-close* metric you need a close **instant**. Two options, both to
> `⚠️ CONFIRM`: (a) the note-level signing instant in the notes table (`HNO_INFO` / note
> audit), or (b) a Clarity audit/`*_UTC_DTTM` close-instant field. With date-only data,
> CM-06a/b degrade to **day-count** rules (close date − visit date ≤ 1 day / = 0 days),
> which is the standard Clarity approximation — document the convention used.

> **Closure vs. signing.** `ENC_CLOSE_DATE` is *encounter* closure (note signed + charges
> dropped, per site config). If the measure must isolate *note signing*, use the note
> table's signed instant instead. For most institutional "chart closure" dashboards,
> encounter close is the right and standard field.

## Suki binding (attribution only)

CM-06 is fully Epic-derived; Suki supplies only the **per-provider go-live date** to split
pre/post, and optionally an **encounter-level exposure flag** for an exposed-vs-unexposed
design. `⚠️ CONFIRM` both fields from Suki ambient-session records.

## Algorithm (pseudocode)

```sql
WITH enc AS (
  SELECT e.PAT_ENC_CSN_ID,
         e.VISIT_PROV_ID,
         CAST(e.EFFECTIVE_DATE_DTTM AS DATE)            AS visit_date,
         e.ENC_CLOSE_DATE                                AS close_date,
         DATEDIFF(day, CAST(e.EFFECTIVE_DATE_DTTM AS DATE), e.ENC_CLOSE_DATE) AS days_to_close
  FROM PAT_ENC e
  WHERE e.ENC_CLOSED_YN = 'Y'
    AND e.ENC_TYPE_C NOT IN (1, 31)            -- drop registration / PCP-change contacts
)
SELECT
  enc.VISIT_PROV_ID,
  CASE WHEN visit_date <  s.suki_golive                      THEN 'pre'
       WHEN visit_date >= DATEADD(week, 6, s.suki_golive)    THEN 'post' END AS phase,
  AVG(CASE WHEN days_to_close <= 1 THEN 1.0 ELSE 0 END)      AS rate_24h,   -- CM-06a
  AVG(CASE WHEN days_to_close =  0 THEN 1.0 ELSE 0 END)      AS rate_same_day, -- CM-06b
  -- CM-06c: median(hours_to_close) once a close-instant field is available
  COUNT(*)                                                  AS n_encounters
FROM enc
JOIN suki_adoption s ON s.epic_prov_id = enc.VISIT_PROV_ID   -- ⚠️ CONFIRM Suki field
WHERE phase IS NOT NULL
GROUP BY enc.VISIT_PROV_ID, phase;
```

## Delta interpretation

Higher 24h/same-day closure rates and lower median time-to-close after adoption →
documentation is being *completed* sooner. Boyter/KLAS 2025 reported **−41%** chart
closure time.

## Confounders & guardrails

- **Convention matters**: report whether 24h or same-day is used; a 5 PM visit closed 8 AM
  next day fails same-day but passes 24h.
- **Date-only `ENC_CLOSE_DATE`** can't distinguish a 1-hour close from a 23-hour one —
  upgrade to a close-instant field before claiming "hours-to-close."
- **Not note-writing time** — keep distinct from CM-04; a note can be *written* fast but
  *signed/closed* late.
- **Encounter-type filter** drives the denominator; hold it constant across pre/post.

## Open questions for the team

1. Is a close/sign **instant** available (note table or audit)? Determines whether CM-06c
   and true same-day are possible vs. day-count approximations.
2. Does the dashboard want **encounter** closure or strict **note-signing**?
3. Confirm the Suki **go-live** field (and exposure flag, if any).
