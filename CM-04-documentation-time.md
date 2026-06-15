# CM-04 — Documentation Time (Clarity-proxy variant)

**Canonical measure:** [CM-04 Documentation Time](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-04%20Documentation%20Time.md) · **D&M:** Individual Impact
**Data tier:** *Normally Epic Signal/UAL telemetry* — this spec derives a **Clarity-only proxy** from note edit timestamps. See [00-OVERVIEW](00-OVERVIEW.md).

## The idea

CM-04 (time from note initiation to signature) is canonically measured with Epic
**Signal/UAL** keystroke-active telemetry, which is **not** in the EHI/Clarity export.
But the export *does* carry per-note **edit-event timestamps**, so we can reconstruct a
documentation-time proxy from them — directionally useful for within-provider pre/post
Suki comparisons, even though it is not active keystroke time.

Two tiers, in priority order:

- **CM-04a — sessionized active authoring time** *(preferred proxy)*: collapse a note's
  edit-event instants into sessions, sum within-session durations.
- **CM-04b — create→sign elapsed span** *(coarse fallback)*: a single subtraction; wall-clock.

## Epic source & join path

| Table | Fields | Role |
|---|---|---|
| `NOTES_HISTORY_LOG` | `NOTE_ID`, `EDIT_HX_INSTANT` (Local), `EDIT_HX_ACTION_C_NAME` | per-edit event stream (sessionize) |
| `HNO_INFO` | `NOTE_ID` (PK), `CREATE_INSTANT_DTTM`, `LST_FILED_INST_DTTM`, `ENTRY_USER_ID`, `CURRENT_AUTHOR_ID`, `PAT_ENC_CSN_ID`, `UNSIGNED_YN` | note master: author, encounter, create/sign |
| `PAT_ENC` | `PAT_ENC_CSN_ID`, `VISIT_PROV_ID`, `EFFECTIVE_DATE_DTTM` | encounter / specialty filters |

**Join path (verified in KG):**
`NOTES_HISTORY_LOG --NOTE_ID--> HNO_INFO --PAT_ENC_CSN_ID--> PAT_ENC`.
**Attribute documentation time to the note author** (`HNO_INFO.ENTRY_USER_ID` /
`CURRENT_AUTHOR_ID`), *not* the visit provider — they differ for resident/attending cosign
and scribe-entered notes.

## CM-04a — sessionized active authoring time (pseudocode)

```sql
-- 1. per-note edit events, attributed to the note author
WITH ev AS (
  SELECT h.NOTE_ID, h.ENTRY_USER_ID AS author_id, h.PAT_ENC_CSN_ID,
         l.EDIT_HX_INSTANT AS ts
  FROM HNO_INFO h
  JOIN NOTES_HISTORY_LOG l ON l.NOTE_ID = h.NOTE_ID          -- KG: NOTE_ID link
  WHERE h.UNSIGNED_YN = 'N'                                  -- signed notes only
),
-- 2. gap to previous edit; a gap > 10 min starts a new "session"
gaps AS (
  SELECT *, DATEDIFF(minute,
              LAG(ts) OVER (PARTITION BY NOTE_ID ORDER BY ts), ts) AS gap_min
  FROM ev
),
seg AS (   -- sum durations only within sessions (ignore the idle gaps)
  SELECT NOTE_ID, author_id, PAT_ENC_CSN_ID,
         SUM(CASE WHEN gap_min IS NULL THEN 0
                  WHEN gap_min <= 10  THEN gap_min      -- ⚠️ tunable session threshold
                  ELSE 0 END) AS active_min
  FROM gaps GROUP BY NOTE_ID, author_id, PAT_ENC_CSN_ID
)
SELECT s.author_id,
       phase(e.EFFECTIVE_DATE_DTTM, k.suki_golive) AS phase,   -- ⚠️ CONFIRM Suki go-live
       AVG(s.active_min) AS avg_active_doc_min_per_note
FROM seg s
JOIN PAT_ENC e   ON e.PAT_ENC_CSN_ID = s.PAT_ENC_CSN_ID
JOIN suki_adoption k ON k.epic_prov_id = s.author_id
WHERE e.ENC_CLOSED_YN = 'Y' AND e.ENC_TYPE_C NOT IN (1,31)
GROUP BY s.author_id, phase;
```

## CM-04b — create→sign elapsed span (fallback)

```sql
SELECT h.ENTRY_USER_ID AS author_id,
       AVG(DATEDIFF(minute, h.CREATE_INSTANT_DTTM, h.LST_FILED_INST_DTTM)) AS avg_span_min
FROM HNO_INFO h
WHERE h.UNSIGNED_YN = 'N'
GROUP BY h.ENTRY_USER_ID;   -- wall-clock; includes idle time → overstates
```

## Delta interpretation

Lower active authoring minutes per note after adoption → Suki is reducing note-writing
effort. Expect modest objective effects: the literature's reliable (telemetry) estimates
run **−12%** (Ma 2025, Guo 2026), versus self-report's inflated −60–72%.

## Confounders & guardrails

- **Event-gated, not keystroke-gated.** This is wall-clock *between logged edit events*;
  the session threshold (10 min default) is a tunable assumption. Sensitivity-test it.
- **Note-authoring only** — a *subset* of Signal documentation time (which also counts
  orders, In Basket, chart review). Report it as "note-authoring time," not "total
  documentation time"; it is a **floor**.
- **Calibrate against Signal** where any overlapping provider-period exists — derive a
  correction factor; never present proxy minutes as if they were Signal minutes.
- **Multi-author notes** — attribute to the editing user where the edit log carries one;
  otherwise to `ENTRY_USER_ID`. Resident edits shouldn't be charged to the attending.
- **Few-event notes** bound the span loosely (a single long typing burst between two
  events). Pair CM-04a with CM-04b and flag notes with < 2 events.
- **Per-note vs per-day** — this is per-note; don't compare to per-day Signal figures
  without converting via encounters/day.

## Open questions for the team

1. Session threshold — 5, 10, or 15 min? (Run the sensitivity analysis.)
2. Is any **Signal/UAL** extract available for **calibration** on a provider subset?
3. Does `NOTES_HISTORY_LOG` log enough edit actions per note to sessionize, or mostly
   create/sign endpoints? (Determines whether CM-04a beats CM-04b in practice — `⚠️ CONFIRM`
   against real data volume.)
