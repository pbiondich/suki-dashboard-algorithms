# CM-21 — Coding Accuracy: ICD-10 / HCC / E&M (algorithm binding)

**Canonical measure:** [CM-21 Coding Accuracy ICD-10 HCC and EM](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-21%20Coding%20Accuracy%20ICD-10%20HCC%20and%20EM.md) · **D&M:** Organizational Impact
**Data tier:** Epic Clarity-computable (21a, 21c) + Suki API (21b) · see [00-OVERVIEW](00-OVERVIEW.md).

## What we compute

- **CM-21a — ICD-10 coding depth**: distinct diagnoses per completed encounter.
- **CM-21b — Suki suggestion match-rate**: % of Suki-suggested codes that end up billed.
- **CM-21c — claim denial rate**: % of Level 4–5 claims denied (the no-upcoding check).

## Epic source & join paths

| Sub | Table(s) | Key fields | Join |
|---|---|---|---|
| 21a | `PAT_ENC_DX` → `PAT_ENC` | `DX_ID`, `PRIMARY_DX_YN`, `PAT_ENC_CSN_ID` | `PAT_ENC_CSN_ID` (KG-high) |
| 21a | `PAT_ENC_DX` → `CLARITY_EDG` | `DX_ID` → dx master | by convention |
| 21c | `ARPB_TRANSACTIONS` → denial records | `TX_ID`, denial reason | `ARPB_PMT_RELATED_DENIALS` |

> **⚠️ ICD-10 string not in this export.** `EDG_CURRENT_ICD10` is absent, so a literal
> ICD-10 list per encounter isn't directly available. **Use distinct `DX_ID` count** as the
> export-native **coding-depth proxy** (each `DX_ID` is one diagnosis concept). Resolve to
> actual ICD-10 only if the dx master's ICD field is added. This is a *depth/completeness*
> proxy, **not** accuracy — accuracy needs 21b + 21c.

## CM-21a — ICD-10 coding depth

```sql
WITH enc_dx AS (
  SELECT e.PAT_ENC_CSN_ID, e.VISIT_PROV_ID,
         CAST(e.EFFECTIVE_DATE_DTTM AS DATE) AS visit_date,
         COUNT(DISTINCT dx.DX_ID) AS dx_count          -- distinct diagnoses (proxy for ICD-10 depth)
  FROM PAT_ENC e
  JOIN PAT_ENC_DX dx ON dx.PAT_ENC_CSN_ID = e.PAT_ENC_CSN_ID   -- KG-high edge
  WHERE e.ENC_CLOSED_YN = 'Y' AND e.ENC_TYPE_C NOT IN (1,31)
  GROUP BY e.PAT_ENC_CSN_ID, e.VISIT_PROV_ID, CAST(e.EFFECTIVE_DATE_DTTM AS DATE)
)
SELECT VISIT_PROV_ID, phase(visit_date, golive) AS phase, AVG(dx_count) AS avg_dx_per_enc
FROM enc_dx JOIN suki_adoption s ON s.epic_prov_id = VISIT_PROV_ID   -- ⚠️ CONFIRM
GROUP BY VISIT_PROV_ID, phase;
```

**Interpretation:** more diagnoses/encounter after adoption → fuller capture of complexity
(a *completeness* proxy). More is not automatically better — confirm with 21c.

## CM-21b — Suki suggestion match-rate *(Suki API × Epic claim)*

For each Suki-assisted encounter, compare the **structured-data** diagnoses Suki produced
to the **final billed** codes, then `matched / total_billed`.

```text
for each suki_assisted_encounter (linked by CSN ⚠️ CONFIRM):
    suggested = suki.structured_data.diagnoses        # Suki API; may be IMO codes (see below)
    billed    = epic_billed_icd10(PAT_ENC_CSN_ID)     # from claim/account dx
    match_rate = |suggested ∩ billed| / |billed|
report avg(match_rate) over period
```

> **⚠️ Two pipeline dependencies (flagged in the canonical measure):**
> - **IMO → ICD-10**: Suki structured data may return **IMO** terms, not ICD-10 — add a
>   mapping step before comparison.
> - **Encounter→claim linkage** not in the standard Suki pull — `⚠️ CONFIRM` pipeline
>   availability (the repo names "Amita" as the contact).
> Suki's developer docs confirm structured-data output of *diagnoses/medications* per
> ambient session, organized by LOINC sections — that is the suggestion source.

**Interpretation:** higher match-rate → Suki's suggestions align with what's billed; target
~85–90% at maturity. Some divergence is legitimate (clinician/coder edits in final review).

## CM-21c — claim denial rate (Level 4–5)

```sql
SELECT phase(t.SERVICE_DATE, golive) AS phase,
       SUM(CASE WHEN d.RELATED_BDC_ID IS NOT NULL THEN 1 ELSE 0 END)*1.0 / COUNT(*) AS denial_rate_l45
FROM ARPB_TRANSACTIONS t
JOIN cpt_xref x ON x.PROC_ID = t.PROC_ID                       -- ⚠️ CONFIRM CPT resolution
LEFT JOIN ARPB_PMT_RELATED_DENIALS d ON d.<tx_link> = t.TX_ID  -- ⚠️ CONFIRM denial link
WHERE t.TX_TYPE_C_NAME = 'Charge'
  AND x.cpt IN ('99204','99205','99214','99215')               -- Level 4–5 only
  AND <denial_reason_is_coding_related>                        -- ⚠️ exclude auth/eligibility denials
GROUP BY phase;
```

**Interpretation (the key validation):** a **stable or falling** denial rate while E/M
levels rise ([CM-20c](CM-20-financial-productivity.md)) = higher coding is *legitimate*,
not upcoding. Holmgren 2026 found **no denial increase** — the central no-upcoding finding.

## Confounders & guardrails

- **Depth ≠ accuracy** — 21a alone can be gamed by overcoding; always pair with 21c.
- **Case mix** — normalize within specialty / use within-provider comparisons.
- **Denial reason codes required** — separate coding denials from authorization/eligibility.
- **Small samples** — use rolling/quarterly windows for stable Level 4–5 denial rates.

## Open questions for the team

1. Add the dx master's **ICD-10 field** to the export, or accept distinct-`DX_ID` depth?
2. Suki **structured-data** code system (IMO vs ICD-10) and the **encounter→claim** link
   (the "Amita" pipeline)?
3. Exact **denial** linkage from `ARPB_PMT_RELATED_DENIALS` to the charge `TX_ID`, plus the
   coding-related reason-code set.
