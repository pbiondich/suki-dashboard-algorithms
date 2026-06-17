# CM-05 — After-Hours Documentation (Clarity-proxy variant)

**Canonical measure:** [CM-05 After-Hours Documentation](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-05%20After-Hours%20Documentation.md) · **D&M:** Individual Impact
**Data tier:** *Normally Epic Signal/UAL* ("pajama time") — this spec derives a **Clarity-only proxy** from note edit timestamps. See [00-OVERVIEW](00-OVERVIEW.md).

## The idea

"Pajama time" — documentation done outside clinic hours — is canonically a Signal metric.
But the same note **edit-event instants** used for [CM-04](CM-04-documentation-time.md)
carry a **time of day**, and `NOTES_HISTORY_LOG.EDIT_HX_INSTANT` is already in **local
time** — so we can classify each documentation event as in-hours vs after-hours and
compute an after-hours share. It's a proxy for note-authoring after-hours, a subset of
total pajama time, but it needs no extra data beyond CM-04's.

## Metrics

- **CM-05a — after-hours documentation share**: % of note edit events (or sessionized
  active minutes) occurring after-hours, per provider.
- **CM-05b — after-hours minutes per day**: sessionized active minutes (from CM-04a) that
  fall in after-hours windows, averaged per provider per day.

## Epic source & join path

Same as [CM-04](CM-04-documentation-time.md): `NOTES_HISTORY_LOG --NOTE_ID--> HNO_INFO
--PAT_ENC_CSN_ID--> PAT_ENC`, attributed to the **note author**. The only addition is a
**time-of-day classifier** on `EDIT_HX_INSTANT` (Local).

> **Defining "after-hours" (`⚠️ CONFIRM` with the team):**
> - **Fixed window** (simplest): before 07:00, after 18:00, or any weekend/holiday. No
>   extra data needed; transparent; but unfair to non-standard shifts.
> - **Schedule-relative** (better): outside the provider's scheduled clinic sessions —
>   needs a provider schedule/template source (not in this export; `⚠️ CONFIRM` a feed).
> Start with the fixed window; upgrade to schedule-relative if a schedule source exists.

## CM-05a — after-hours share (pseudocode)

```sql
WITH ev AS (
  SELECT h.ENTRY_USER_ID AS author_id,
         l.EDIT_HX_INSTANT AS ts,                            -- Local time
         e.EFFECTIVE_DATE_DTTM
  FROM HNO_INFO h
  JOIN NOTES_HISTORY_LOG l ON l.NOTE_ID = h.NOTE_ID
  JOIN PAT_ENC e ON e.PAT_ENC_CSN_ID = h.PAT_ENC_CSN_ID
  WHERE h.UNSIGNED_YN = 'N' AND e.ENC_CLOSED_YN = 'Y' AND e.ENC_TYPE_C NOT IN (1,31)
),
flagged AS (
  SELECT author_id, EFFECTIVE_DATE_DTTM,
         CASE WHEN DATEPART(weekday, ts) IN (1,7)            -- weekend
                OR DATEPART(hour, ts) < 7                    -- before 07:00
                OR DATEPART(hour, ts) >= 18                  -- after 18:00  ⚠️ window
              THEN 1 ELSE 0 END AS after_hours
  FROM ev
)
SELECT author_id, phase(EFFECTIVE_DATE_DTTM, golive) AS phase,   -- ⚠️ CONFIRM Suki go-live
       AVG(after_hours * 1.0) AS after_hours_share
FROM flagged GROUP BY author_id, phase;
```

CM-05b reuses CM-04a's sessionized minutes, bucketed by the same after-hours flag and
divided by active days per provider.

## Delta interpretation

A falling after-hours share / fewer after-hours minutes after adoption → Suki is moving
documentation back into the workday. This is one of the most clinically meaningful
burnout-adjacent outcomes (pairs with the survey-based CM-01 burnout measure).

## Confounders & guardrails

- **Note-authoring after-hours only** — excludes after-hours In Basket, orders, chart
  review that Signal's pajama-time captures. A **floor** on total after-hours work.
- **Window choice drives the number** — a 17:00 vs 18:00 cutoff shifts results; fix the
  convention and report it. Schedule-relative is fairer where a schedule feed exists.
- **Local-time integrity** — use the Local instant (`EDIT_HX_INSTANT`); if any source is
  UTC, convert with the site/provider timezone before bucketing, or DST/late-evening
  edits misclassify.
- **Calibrate against Signal** pajama-time where available, as with CM-04.
- **Night-shift / non-traditional schedules** misclassify under a fixed window — exclude
  or handle via schedule-relative.

## Open questions for the team

1. After-hours definition — fixed 7–18 weekday window, or schedule-relative (needs a
   provider-schedule source — is one available)?
2. Confirm `EDIT_HX_INSTANT` local-time handling across sites/timezones.
3. Report **share** (CM-05a) or **minutes/day** (CM-05b) as the headline, or both?
