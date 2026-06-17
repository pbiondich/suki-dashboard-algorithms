# CM-04 — Documentation Time (Cerner Millennium proxy)

**Measure:** [CM-04](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-04%20Documentation%20Time.md) · Cerner counterpart of the [Epic CM-04](../epic/CM-04-documentation-time.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## The idea (and the key caveat)

Documentation time is canonically a keystroke-active telemetry metric (Cerner Advance /
"Lights On Network" — not in the EHI export). On Millennium we proxy it from clinical-event
timestamps. **Unlike Epic**, `CLINICAL_EVENT` does *not* expose a per-edit history we could
sessionize — it gives only an authored instant and a verified instant. So the Cerner CM-04
is the **author→verify span tier only** (analogous to Epic's coarse CM-04b fallback): a
wall-clock elapsed time, not active time. Use it as a relative within-provider signal and
calibrate against an efficiency-telemetry feed if one is available.

## Cerner source & join path

| Table | Fields | Role |
|---|---|---|
| `CLINICAL_EVENT` | `PERFORMED_DT_TM` (authored), `VERIFIED_DT_TM` (verified/signed), `PERFORMED_PRSNL_ID` (author), `PERFORMED_TZ`, `EVENT_CLASS_CD`, `EVENT_CD`, `RESULT_STATUS_CD`, `ENCNTR_ID`, `PERSON_ID` | the documentation event |
| `ENCOUNTER` | `ENCNTR_ID`, `ARRIVE_DT_TM`, `ENCNTR_TYPE_CD` | encounter filter |

**Join (published FK):** `CLINICAL_EVENT --ENCNTR_ID--> ENCOUNTER`. Attribute to the author
`PERFORMED_PRSNL_ID`. **Filter to documentation events** — `EVENT_CLASS_CD` = the
document/note class (`⚠️ CONFIRM` the `code_value`; clinical notes are stored as
`CLINICAL_EVENT` rows of the "DOC"/document class), otherwise you'll sweep in labs, vitals,
and every other clinical event.

## Algorithm (pseudocode)

```sql
WITH notes AS (
  SELECT ce.PERFORMED_PRSNL_ID AS author_id,
         ce.PERFORMED_DT_TM, ce.VERIFIED_DT_TM,
         DATEDIFF(minute, ce.PERFORMED_DT_TM, ce.VERIFIED_DT_TM) AS span_min
  FROM CLINICAL_EVENT ce
  JOIN ENCOUNTER e ON e.ENCNTR_ID = ce.ENCNTR_ID
  WHERE ce.EVENT_CLASS_CD = :doc_class_cd                 -- ⚠️ CONFIRM documentation class
    AND ce.RESULT_STATUS_CD = :authenticated_cd           -- signed/authenticated only ⚠️
    AND ce.VERIFIED_DT_TM IS NOT NULL
    AND ce.PERFORMED_DT_TM IS NOT NULL
    AND e.ENCNTR_TYPE_CD IN (:clinical_types)
)
SELECT author_id, phase(PERFORMED_DT_TM, suki_golive) AS phase,   -- ⚠️ CONFIRM go-live
       AVG(span_min) AS avg_author_to_verify_min
FROM notes JOIN suki_adoption s ON s.epic_prov_id = author_id
WHERE phase IS NOT NULL AND span_min BETWEEN 0 AND :cap_min  -- drop negatives/outliers
GROUP BY author_id, phase;
```

## Delta interpretation

Lower author→verify minutes after adoption → faster note turnaround. Objective telemetry
studies show modest reductions (≈−12%, Ma 2025 / Guo 2026); self-report overstates.

## Confounders & guardrails

- **Span, not active time.** Author→verify is wall-clock and includes everything the
  clinician did in between — it overstates and is noisier than Epic's sessionized proxy.
  Present it as a *relative trend*, never absolute minutes.
- **Document-class filter is essential** — without it the metric isn't about notes at all.
- **Authored vs verified semantics** vary by site/workflow (some auto-verify); confirm what
  `PERFORMED_DT_TM`/`VERIFIED_DT_TM` mean in this instance.
- **Cosign / scribe** — resident-authored + attending-verified notes split the timestamps
  across two people; decide whose time it is.
- **Calibrate** against Cerner Advance / "Lights On" efficiency data where available.

## Open questions for the team

1. The `EVENT_CLASS_CD` (and/or `EVENT_CD`) values that identify **clinical documents/notes**.
2. Does this instance auto-verify notes (collapsing the span to ~0)?
3. Any **efficiency-telemetry** (Advance/Lights On) extract for calibration?
