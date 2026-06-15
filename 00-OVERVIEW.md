# Suki Analytics Dashboard — Canonical Measure Algorithms

Operational algorithm specs that bind the **ACI canonical measures**
([pbiondich/aci](https://github.com/pbiondich/aci)) to **Epic's data model** (via the
[epic-ehi-kg](https://github.com/pbiondich/epic-ehi-kg) knowledge graph) and **Suki's**
ambient-documentation telemetry, for the Suki analytics dashboard.

This batch covers the **Clarity-computable** tier — measures derivable from Epic's
relational (Clarity / EHI-export) tables:

| Spec | Canonical measure | D&M dimension |
|---|---|---|
| [CM-06](CM-06-chart-closure.md) | Chart Closure Timeliness | Individual Impact |
| [CM-20](CM-20-financial-productivity.md) | Financial Productivity & Revenue | Organizational Impact |
| [CM-21](CM-21-coding-accuracy.md) | Coding Accuracy (ICD-10 / HCC / E&M) | Organizational Impact |
| [CM-22](CM-22-volume-throughput.md) | Patient Volume & Throughput | Organizational Impact |

## The four data tiers (why this scoping)

A measure's algorithm is a **binding**: canonical definition → data source → comparison.
The 25 canonical measures draw from four different wells, and that sorting is the
dashboard architecture:

1. **Epic Clarity-computable** — relational tables in the EHI/Clarity model (the KG we
   built). *This batch.* CM-06, CM-20, CM-21, CM-22.
2. **Epic Signal / UAL telemetry** — keystroke/active-time audit logs; a **separate
   feed**, not in the EHI Clarity export. CM-04 (documentation time), CM-05 (after-hours
   "pajama time"), CM-07 (total EHR time).
3. **Suki telemetry / API** — ambient sessions, per-note timing, `structured-data` code
   output. CM-13/14 (adoption), CM-21b (suggestion match-rate).
4. **Survey / qualitative** — no algorithm; needs instruments. CM-01 burnout, CM-02
   cognitive load, CM-15 satisfaction, CM-16 trust.

## Shared methods (applies to every spec in this batch)

**Attribution — the one hard dependency on Suki.** Epic tells you *what happened*; only
Suki tells you *which encounters were Suki-assisted* and *when each provider went live*.
Every pre/post comparison needs a per-provider **Suki go-live date** and, ideally, an
**encounter-level exposure flag** (was this specific encounter documented with Suki?),
both sourced from Suki ambient-session records keyed to the Epic provider and, where
available, the encounter (CSN). `⚠️ CONFIRM` the exact Suki fields with the team.

**Comparison design.** Prefer **within-provider pre vs. post** (each provider is their own
control) over cross-sectional adopter-vs-non-adopter. Where possible add a
**same-period-prior-year** pre-window to control for seasonality. An exposed-vs-unexposed
contemporaneous design (Suki vs non-Suki encounters in the same period) is the strongest
if encounter-level exposure is available.

**Onboarding ramp.** Exclude each provider's first **4–6 weeks** of Suki use from the
post window (learning curve).

**Normalization.** Compare within specialty (or risk-adjust); normalize to scheduled
clinical hours/FTE for part-time providers; exclude providers who changed specialty, site,
panel size, or employment status mid-window.

**Completed-encounter rule.** "Encounter" = a **completed, signed** encounter
(`PAT_ENC.ENC_CLOSED_YN = 'Y'`), not a scheduled appointment or check-in — unsigned notes
inflate counts and distort per-encounter denominators.

## Epic join paths used (verified in the KG)

| From | Key | To | KG confidence |
|---|---|---|---|
| `PAT_ENC_DX` | `PAT_ENC_CSN_ID` | `PAT_ENC` | high |
| `ARPB_TRANSACTIONS` | `PAT_ENC_CSN_ID` | `PAT_ENC` | high |
| `PAT_ENC_DX` | `DX_ID` | `CLARITY_EDG` (dx master) | n/a* |
| `ARPB_TRANSACTIONS` | `PROC_ID` | `CLARITY_EAP` (procedure master) | n/a* |

\* The master-table link is by Epic convention; the EHI export of this build does **not**
expose `EDG_CURRENT_ICD10` (ICD-10 string) or a CPT column on `CLARITY_EAP`. Consequences
are handled per-spec: coding depth uses **distinct `DX_ID`** as the export-native proxy,
and E/M-level classification needs the CPT code from the procedure master or a ref-bill
table (`⚠️ CONFIRM` the field).

## Conventions in each spec

- **`⚠️ CONFIRM`** — a Suki field or Epic detail to verify with the team before production.
- SQL is **illustrative pseudocode** (ANSI-ish), not tuned for SQL Server/Oracle Clarity.
- All dollar/RVU references cite the canonical measure's literature anchors.

*Built against epic-ehi-kg (Epic EHI export, May 2026 release) and the ACI canonical
measures (derived 2026-03/04). Not affiliated with or endorsed by Epic or Suki.*
