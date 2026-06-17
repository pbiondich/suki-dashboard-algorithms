# Cerner (Oracle Health Millennium) — Canonical Measure Algorithm Bindings

The **Cerner Millennium** counterpart to the Epic specs in the parent folder: the same six
ACI canonical measures, bound to Millennium tables/columns via the
[`cerner-ehi-kg`](https://github.com/pbiondich/cerner-ehi-kg) knowledge graph (and Suki
telemetry for attribution). Read alongside the Epic specs — same measures, same comparison
methodology, different schema.

| Spec | Measure | Basis on Millennium |
|---|---|---|
| [CM-04](CM-04-documentation-time.md) | Documentation Time | proxy — `CLINICAL_EVENT` author→verify span |
| [CM-05](CM-05-after-hours.md) | After-Hours Documentation | proxy — `CLINICAL_EVENT` timestamps' time-of-day |
| [CM-06](CM-06-chart-closure.md) | Chart Closure Timeliness | native — `ENCOUNTER.CHART_COMPLETE_DT_TM` |
| [CM-20](CM-20-financial-productivity.md) | Financial Productivity & Revenue | native — `CHARGE` / `PROCEDURE` / `PFT_RVU_CONTENT` |
| [CM-21](CM-21-coding-accuracy.md) | Coding Accuracy | native — `DIAGNOSIS → NOMENCLATURE` (real ICD-10) |
| [CM-22](CM-22-volume-throughput.md) | Patient Volume & Throughput | native — `ENCOUNTER` completed-encounter timestamps |

## Where Cerner is *easier* than Epic

The Millennium export publishes real keys and richer reference data, so several bindings are
**more direct** than their Epic counterparts:

- **Real ICD-10 codes.** `DIAGNOSIS.NOMENCLATURE_ID → NOMENCLATURE`, where
  `SOURCE_IDENTIFIER` is the actual code and `SOURCE_VOCABULARY_CD` says which vocabulary
  (ICD-10-CM, SNOMED). Epic's export didn't expose the ICD-10 string, so Epic CM-21a used a
  distinct-`DX_ID` *proxy*; on Cerner it's the real thing.
- **Native RVU content.** `PFT_RVU_CONTENT` / `RCR_RVU_MODIFIER_FORMULA` are in the model,
  so wRVU mapping may not need an external CMS file (Epic did).
- **A true chart-closure timestamp.** `ENCOUNTER.CHART_COMPLETE_DT_TM` is a datetime, so
  CM-06 hours-to-close is exact — Epic's `ENC_CLOSE_DATE` was date-only.
- **Authoritative joins.** Every join below is a *published* FK (verified in the KG), not an
  inferred edge.

## Where Cerner is *harder*

- **CM-04 has no sessionizable edit log.** Epic's `NOTES_HISTORY_LOG` gives per-edit
  instants we could sessionize into active time. `CLINICAL_EVENT` exposes only
  `PERFORMED_DT_TM` (authored) and `VERIFIED_DT_TM` (verified) — so the Cerner CM-04 is the
  **span-only tier** (analogous to Epic's CM-04b fallback), an even looser proxy. Calibrate
  against an efficiency-telemetry feed (Cerner Advance / "Lights On") where available.

## Shared methods (same as the Epic specs)

- **Attribution is the one hard Suki dependency.** Millennium tells you what happened; Suki
  supplies the per-provider **go-live date** and (ideally) an **encounter-level exposure
  flag**. `⚠️ CONFIRM` the Suki fields.
- **Provider identity (Cerner):** personnel are `PRSNL.PERSON_ID`. On an **encounter**,
  providers attach via `ENCNTR_PRSNL_RELTN` (`PRSNL_PERSON_ID ↔ ENCNTR_ID`, filtered to the
  attending/provider relationship type). On **documentation**, the author is
  `CLINICAL_EVENT.PERFORMED_PRSNL_ID` directly.
- **Completed encounter:** `ENCOUNTER.ENCNTR_COMPLETE_DT_TM` populated (and/or
  `ENCNTR_STATUS_CD` = a completed value) — not a scheduled/arrived row.
- **Comparison design:** within-provider pre/post around Suki go-live; exclude the 4–6 week
  onboarding ramp; normalize within specialty / to scheduled hours.
- **Coded values:** nearly every `*_CD` FKs to `CODE_VALUE`; a code's *meaning* depends on
  its **`CODE_SET`**, a per-instance, site-governed dimension — the Cerner analog of Epic's
  organization-specific values, and the thing that breaks cross-site comparability without
  local mapping.

## Verified join paths (published FKs in the KG)

| From | To | Via |
|---|---|---|
| `ENCOUNTER` | `PERSON` | `PERSON_ID` |
| `ENCNTR_PRSNL_RELTN` | `ENCOUNTER` / `PRSNL` | `ENCNTR_ID` / `PRSNL_PERSON_ID` |
| `CLINICAL_EVENT` | `ENCOUNTER` | `ENCNTR_ID` |
| `DIAGNOSIS` | `ENCOUNTER` | `ENCNTR_ID` |
| `DIAGNOSIS` | `NOMENCLATURE` | `NOMENCLATURE_ID` |

`⚠️ CONFIRM` markers flag Suki fields or Cerner specifics to verify with the team. SQL is
illustrative pseudocode.

*Built against cerner-ehi-kg (Cerner Millennium Data Model Reports, 2026.1.01). Not
affiliated with or endorsed by Oracle, Cerner, or Suki.*
