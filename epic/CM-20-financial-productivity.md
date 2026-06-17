# CM-20 — Financial Productivity & Revenue Impact (algorithm binding)

**Canonical measure:** [CM-20 Financial Productivity and Revenue Impact](https://github.com/pbiondich/aci/blob/main/Canonical%20Measures/CM-20%20Financial%20Productivity%20and%20Revenue%20Impact.md) · **D&M:** Organizational Impact
**Data tier:** Epic Clarity-computable · see [00-OVERVIEW](00-OVERVIEW.md) for shared methods.

## What we compute

Three sub-metrics (matching the canonical measure's CM-20a/b/c):

- **CM-20a — wRVU output**: wRVUs per provider per week.
- **CM-20b — billing revenue**: adjusted net (collected) revenue per provider per month.
- **CM-20c — E/M level mix**: % of E/M encounters billed at Level 4–5.

All three live in professional billing — **`ARPB_TRANSACTIONS`** (PB transactions, PK `TX_ID`).

## Epic source & join path

| Field (`ARPB_TRANSACTIONS`) | Role |
|---|---|
| `TX_TYPE_C_NAME` | `'Charge'` / `'Payment'` / `'Adjustment'` — filter per sub-metric |
| `SERVICE_DATE` | service date (period bucketing) |
| `SERV_PROVIDER_ID` | provider (cohort + within-provider) |
| `PROV_SPECIALTY_C_NAME` | specialty (normalization) |
| `PROC_ID` (→ `CLARITY_EAP`) | procedure → **CPT** (E/M classification, wRVU lookup) |
| `MODIFIER_ONE..FOUR` | E/M modifiers (e.g. 25) |
| `AMOUNT` | charge amount (gross) |
| `TOTAL_MATCH_AMT` | matched/collected amount (adjusted net basis) |
| `PAT_ENC_CSN_ID` | → `PAT_ENC` (KG-high), if encounter-grain rollups are needed |

Provider-level metrics (CM-20a/b) aggregate on `SERV_PROVIDER_ID` + `SERVICE_DATE` — **no
encounter join required**. CM-20c needs the **CPT code**, via `PROC_ID → CLARITY_EAP`.

> **⚠️ Two external/reference dependencies.**
> 1. **wRVU weights** are not in Clarity — map each charge's **CPT → CMS PFS wRVU** from
>    the CMS RVU file (or Epic's fee-schedule/`CL_RVU` reference). `⚠️ CONFIRM` the source.
> 2. The **CPT code string** is not exposed on `CLARITY_EAP` in this EHI export build —
>    resolve CPT via the procedure master's billing-code field or a ref-bill table.
>    `⚠️ CONFIRM` the column. (E/M set: new 99202–99205, established 99211–99215;
>    Level 4–5 = 99204/99205, 99214/99215.)

## Suki binding (attribution only)

Epic-derived metrics; Suki supplies the **per-provider go-live date** (pre/post split) and
optional **encounter exposure flag**. The repo also references a pre-aggregated Suki EHR
data-pull (`wrvu_total`, `revenue_paid`, `em_*_level*_count`, `em_total_count`) — treat
these as a **convenience layer**; the Clarity derivations below are the source of truth.
`⚠️ CONFIRM` all Suki fields.

## Algorithm (pseudocode)

### CM-20a — wRVU per provider per week
```sql
WITH charges AS (
  SELECT t.SERV_PROVIDER_ID,
         DATETRUNC(week, t.SERVICE_DATE)       AS wk,
         t.PROC_ID, t.PROCEDURE_QUANTITY
  FROM ARPB_TRANSACTIONS t
  WHERE t.TX_TYPE_C_NAME = 'Charge'
)
SELECT c.SERV_PROVIDER_ID, phase(c.wk, s.suki_golive) AS phase,
       SUM(rvu.work_rvu * c.PROCEDURE_QUANTITY) / COUNT(DISTINCT c.wk) AS wrvu_per_week
FROM charges c
JOIN cpt_xref  x  ON x.PROC_ID = c.PROC_ID                 -- ⚠️ CONFIRM CPT resolution
JOIN cms_wrvu  rvu ON rvu.cpt  = x.cpt                       -- ⚠️ CMS PFS wRVU reference
JOIN suki_adoption s ON s.epic_prov_id = c.SERV_PROVIDER_ID -- ⚠️ CONFIRM Suki field
GROUP BY c.SERV_PROVIDER_ID, phase;
```

### CM-20b — collected revenue per provider per month
```sql
SELECT t.SERV_PROVIDER_ID,
       DATETRUNC(month, t.SERVICE_DATE) AS mo,
       SUM(t.TOTAL_MATCH_AMT) AS adjusted_net_revenue   -- collected, not gross AMOUNT
FROM ARPB_TRANSACTIONS t
WHERE t.TX_TYPE_C_NAME IN ('Payment','Adjustment')       -- ⚠️ CONFIRM net-revenue convention
GROUP BY t.SERV_PROVIDER_ID, DATETRUNC(month, t.SERVICE_DATE);
-- annualized alt: wRVU_delta * 52 * CMS conversion factor (~$33.29/wRVU, 2026)
```

### CM-20c — % E/M at Level 4–5
```sql
WITH em AS (
  SELECT t.SERV_PROVIDER_ID, t.SERVICE_DATE, x.cpt
  FROM ARPB_TRANSACTIONS t
  JOIN cpt_xref x ON x.PROC_ID = t.PROC_ID                 -- ⚠️ CONFIRM CPT resolution
  WHERE t.TX_TYPE_C_NAME = 'Charge'
    AND x.cpt IN ('99202','99203','99204','99205','99211','99212','99213','99214','99215')
)
SELECT SERV_PROVIDER_ID, phase(SERVICE_DATE, golive) AS phase,
       AVG(CASE WHEN cpt IN ('99204','99205','99214','99215') THEN 1.0 ELSE 0 END) AS pct_level_4_5
FROM em GROUP BY SERV_PROVIDER_ID, phase;
```

## Delta interpretation

Higher wRVU/week, higher collected revenue, and a higher Level 4–5 share after adoption →
documentation is capturing complexity more completely. Anchors: Holmgren 2026 **+1.81
wRVU/week (~$3,044/yr)**, no denial increase; Boyter/KLAS 2025 **+$13,049/yr** (HCC-driven).

## Confounders & guardrails

- **Read CM-20c alongside [CM-21c denial rate](CM-21-coding-accuracy.md)** — rising E/M
  *with* rising denials = possible overcoding, not legitimate complexity capture.
- **Collected, not gross** — use `TOTAL_MATCH_AMT`/payments, not `AMOUNT`; gross misleads
  when payer mix shifts.
- **Seasonality** — use same-period-prior-year as the pre window.
- **Specialty/FTE normalization** — dollars and wRVUs vary hugely by specialty; normalize
  to scheduled hours for part-timers.
- **Societal caveat (PHTI 2025)** — even *accurate* higher coding raises system spend;
  health-system ROI ≠ payer/societal interest. Worth a dashboard footnote.

## Open questions for the team

1. Where does the **CPT code** resolve from (`CLARITY_EAP` field vs. ref-bill table)?
2. Which **wRVU reference** (CMS PFS year? Epic fee schedule)?
3. Net-revenue definition — payments only, or charges net of contractual adjustments?
4. Confirm Suki go-live / exposure fields and whether to use the pre-aggregated pull.
