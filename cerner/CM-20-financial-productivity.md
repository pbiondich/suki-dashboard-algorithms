# CM-20 — Financial Productivity & Revenue (Cerner Millennium binding)

**Measure:** [CM-20](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-20%20Financial%20Productivity%20and%20Revenue%20Impact.md) · Cerner counterpart of the [Epic CM-20](../epic/CM-20-financial-productivity.md) · see [cerner/00-OVERVIEW](00-OVERVIEW.md).

## What we compute

- **CM-20a — wRVU output**: wRVUs per provider per week.
- **CM-20b — billing revenue**: collected revenue per provider per month.
- **CM-20c — E/M level mix**: % of E/M encounters billed Level 4–5.

> **Cerner ships RVU content.** `PFT_RVU_CONTENT` (+ `RCR_RVU_MODIFIER_FORMULA`) are in the
> model, so wRVU weights may come from the export itself rather than an external CMS file
> (which Epic required). `⚠️ CONFIRM` the wRVU column and effective-date handling.

## Cerner source & join paths

Professional billing on Millennium is spread across **`CHARGE`** / **`CHARGE_EVENT`**,
**`BILL_REC`**, **`ENCNTR_FINANCIAL`**, and the **`PFT_*`** professional-fee tables;
procedures/CPT live in **`PROCEDURE` → `NOMENCLATURE`**; RVU weights in **`PFT_RVU_CONTENT`**.

| Need | Tables / fields (`⚠️ CONFIRM` exact columns) |
|---|---|
| Charges, amounts, provider, date | `CHARGE` / `CHARGE_EVENT` (charge amount, `PERFORMED`/service date, performing-prsnl); `CHARGE_EVENT_ACT_PRSNL` for personnel |
| CPT / E/M code | `PROCEDURE.NOMENCLATURE_ID → NOMENCLATURE` (`SOURCE_IDENTIFIER` = CPT, `SOURCE_VOCABULARY_CD` = CPT-4) |
| wRVU weight | `PFT_RVU_CONTENT` (work-RVU per CPT) |
| Collected revenue | `BILL_REC` / remittance tables (payments/adjustments) |
| Encounter / provider | `ENCOUNTER`, `ENCNTR_PRSNL_RELTN`, `PRSNL` |

> This is the **least-uniform area** of the Millennium export (many billing-model variants:
> `PFT_*`, `SN_CHARGE_*`, `RC_*`). Confirm with the team which charge/bill tables this
> instance populates before wiring CM-20b.

## Algorithm sketches (pseudocode)

### CM-20a — wRVU per provider per week
```sql
SELECT c.performing_prsnl_id AS provider_id,
       DATETRUNC(week, c.service_dt_tm) AS wk,
       SUM(rvu.work_rvu * c.qty) / COUNT(DISTINCT DATETRUNC(week,c.service_dt_tm)) AS wrvu_per_wk
FROM CHARGE c                                                       -- ⚠️ CONFIRM table/cols
JOIN PROCEDURE p   ON p.<link> = c.<link>                           -- charge → procedure
JOIN NOMENCLATURE n ON n.NOMENCLATURE_ID = p.NOMENCLATURE_ID AND n.SOURCE_VOCABULARY_CD = :cpt_cd
JOIN PFT_RVU_CONTENT rvu ON rvu.<cpt_key> = n.SOURCE_IDENTIFIER     -- ⚠️ CONFIRM rvu join
GROUP BY c.performing_prsnl_id, DATETRUNC(week, c.service_dt_tm);
```

### CM-20c — % E/M at Level 4–5
```sql
WITH em AS (
  SELECT c.performing_prsnl_id AS provider_id, c.service_dt_tm, n.SOURCE_IDENTIFIER AS cpt
  FROM CHARGE c
  JOIN PROCEDURE p ON p.<link> = c.<link>
  JOIN NOMENCLATURE n ON n.NOMENCLATURE_ID = p.NOMENCLATURE_ID AND n.SOURCE_VOCABULARY_CD = :cpt_cd
  WHERE n.SOURCE_IDENTIFIER IN ('99202','99203','99204','99205','99211','99212','99213','99214','99215')
)
SELECT provider_id, phase(service_dt_tm, suki_golive) AS phase,
       AVG(CASE WHEN cpt IN ('99204','99205','99214','99215') THEN 1.0 ELSE 0 END) AS pct_l4_5
FROM em GROUP BY provider_id, phase;
```

CM-20b sums collected revenue per provider per month from the remittance/`BILL_REC` side
(payments + adjustments, not gross charges).

## Delta interpretation

Higher wRVU/week, collected revenue, and Level 4–5 share after adoption → fuller complexity
capture. Anchors: Holmgren 2026 **+1.81 wRVU/wk (~$3,044/yr)**, no denial rise; Boyter/KLAS
2025 **+$13,049/yr** (HCC-driven).

## Confounders & guardrails

- **Read CM-20c with [CM-21c denials](CM-21-coding-accuracy.md)** — rising E/M + rising
  denials = possible overcoding.
- **Collected, not gross** for CM-20b; control payer-mix shifts.
- **Specialty/FTE normalization**; same-period-prior-year as the pre window.
- **PHTI 2025 societal caveat** — accurate higher coding still raises system spend.

## Open questions for the team

1. Which **charge/bill** tables does this instance populate (`CHARGE` vs `PFT_*` vs `SN_*`)?
2. `PFT_RVU_CONTENT` wRVU column + how it keys to CPT; CPT `SOURCE_VOCABULARY_CD`.
3. Charge→procedure→encounter→provider link, and the collected-revenue source.
