# CM-21 — Coding Accuracy (Cerner Millennium binding)

**Measure:** [CM-21](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-21%20Coding%20Accuracy%20ICD-10%20HCC%20and%20EM.md) · Cerner counterpart of the [Epic CM-21](../epic/CM-21-coding-accuracy.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## What we compute

- **CM-21a — ICD-10 coding depth**: distinct ICD-10 codes per completed encounter.
- **CM-21b — Suki suggestion match-rate**: % of Suki-suggested codes that end up recorded.
- **CM-21c — claim denial rate** (Level 4–5).

> **Better than Epic here.** Epic's export didn't expose the ICD-10 string, so Epic CM-21a
> counted distinct `DX_ID` as a *proxy*. Cerner's `NOMENCLATURE` table carries the **real
> code** (`SOURCE_IDENTIFIER`) and the **vocabulary** (`SOURCE_VOCABULARY_CD`), so CM-21a is
> true distinct-ICD-10, not a proxy.

## Cerner source & join paths

| Table | Fields | Role |
|---|---|---|
| `DIAGNOSIS` | `ENCNTR_ID`, `PERSON_ID`, `NOMENCLATURE_ID`, `ACTIVE_IND`, `CONFIRMATION_STATUS_CD` | encounter diagnoses |
| `NOMENCLATURE` | `NOMENCLATURE_ID`, `SOURCE_IDENTIFIER` (the code), `SOURCE_VOCABULARY_CD` (which vocab) | code resolution |
| `ENCOUNTER` | `ENCNTR_ID`, `ENCNTR_COMPLETE_DT_TM`, `ENCNTR_TYPE_CD` | completed-encounter filter |

**Joins (published FKs):** `DIAGNOSIS --ENCNTR_ID--> ENCOUNTER` and
`DIAGNOSIS --NOMENCLATURE_ID--> NOMENCLATURE`. Filter `NOMENCLATURE.SOURCE_VOCABULARY_CD`
to the **ICD-10-CM** code_value (`⚠️ CONFIRM` the value).

## CM-21a — ICD-10 coding depth

```sql
WITH enc_dx AS (
  SELECT e.ENCNTR_ID, r.PRSNL_PERSON_ID AS provider_id, e.ENCNTR_COMPLETE_DT_TM,
         COUNT(DISTINCT n.SOURCE_IDENTIFIER) AS icd10_codes
  FROM ENCOUNTER e
  JOIN ENCNTR_PRSNL_RELTN r ON r.ENCNTR_ID = e.ENCNTR_ID
        AND r.ACTIVE_IND = 1 AND r.ENCNTR_PRSNL_R_CD = :attending_reltn_cd   -- ⚠️ CONFIRM
  JOIN DIAGNOSIS d  ON d.ENCNTR_ID = e.ENCNTR_ID AND d.ACTIVE_IND = 1
  JOIN NOMENCLATURE n ON n.NOMENCLATURE_ID = d.NOMENCLATURE_ID
        AND n.SOURCE_VOCABULARY_CD = :icd10cm_cd                              -- ⚠️ CONFIRM
  WHERE e.ENCNTR_COMPLETE_DT_TM IS NOT NULL AND e.ENCNTR_TYPE_CD IN (:clinical_types)
  GROUP BY e.ENCNTR_ID, r.PRSNL_PERSON_ID, e.ENCNTR_COMPLETE_DT_TM
)
SELECT provider_id, phase(ENCNTR_COMPLETE_DT_TM, suki_golive) AS phase,        -- ⚠️ go-live
       AVG(icd10_codes) AS avg_icd10_per_enc
FROM enc_dx JOIN suki_adoption s ON s.epic_prov_id = provider_id
WHERE phase IS NOT NULL GROUP BY provider_id, phase;
```

**Interpretation:** more distinct ICD-10 per encounter after adoption → fuller capture of
complexity (completeness, not accuracy — confirm with 21c).

## CM-21b — Suki suggestion match-rate

For each Suki-assisted encounter, compare Suki `structured-data` diagnoses to the recorded
`DIAGNOSIS`/`NOMENCLATURE` ICD-10 set; `matched / recorded`.

```text
suggested = suki.structured_data.diagnoses     # ⚠️ IMO→ICD-10 mapping may be needed
recorded  = {NOMENCLATURE.SOURCE_IDENTIFIER for DIAGNOSIS in encounter, vocab=ICD-10-CM}
match_rate = |suggested ∩ recorded| / |recorded|
```
`⚠️ CONFIRM` the Suki encounter↔Cerner-encounter linkage (the "Amita" pipeline analog).

## CM-21c — claim denial rate (Level 4–5)

Cerner professional billing lives in `CHARGE` / `BILL_REC` / `ENCNTR_FINANCIAL` (and the
`PFT_*` professional-fee tables). Denials and remittance are in the billing/remit tables —
`⚠️ CONFIRM` the exact denial table + reason-code columns (this is the least-mapped area of
the export). Compute denied Level 4–5 E/M claims ÷ total Level 4–5, pre vs post.

**Interpretation (key validation):** stable/falling denials while E/M levels rise
([CM-20c](CM-20-financial-productivity.md)) ⇒ legitimate complexity capture, not upcoding
(Holmgren 2026: no denial increase).

## Confounders & guardrails

- **Depth ≠ accuracy** — pair 21a with 21c.
- **Vocabulary filter** — `SOURCE_VOCABULARY_CD` must isolate ICD-10-CM; diagnoses may also
  carry SNOMED rows in `NOMENCLATURE` (don't double-count).
- **Confirmation status** — consider filtering `CONFIRMATION_STATUS_CD` to confirmed dx.
- **Case mix** — normalize within specialty / within-provider.

## Open questions for the team

1. `SOURCE_VOCABULARY_CD` code_value for **ICD-10-CM** (and whether to include SNOMED).
2. Cerner **billing/denial** tables + coding-related reason codes for 21c.
3. Suki structured-data code system + the encounter→claim linkage.
