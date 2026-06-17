# Suki Dashboard Algorithms

Operational algorithm specs for an **Ambient Clinical Intelligence (ACI) analytics
dashboard** — each one binds a **canonical evaluation measure** to an **EHR data model**
and **Suki's** ambient-documentation telemetry, so a measure goes from a literature
definition to a computable dashboard metric. Both major EHRs are covered:

- **Epic (Clarity)** — [`epic/`](epic/00-OVERVIEW.md).
- **Cerner / Oracle Health (Millennium)** — [`cerner/`](cerner/00-OVERVIEW.md).

Three sources meet here:

- **What to measure** — the canonical measures from the ACI structured evidence review
  ([`pbiondich/aci`](https://github.com/pbiondich/aci)).
- **Where the data lives & how to join it** — Epic's EHI/Clarity schema
  ([`epic-ehi-kg`](https://github.com/pbiondich/epic-ehi-kg)) and Cerner's Millennium schema
  ([`cerner-ehi-kg`](https://github.com/pbiondich/cerner-ehi-kg)).
- **The intervention & its telemetry** — Suki ambient sessions and structured-data output
  ([developer.suki.ai](https://developer.suki.ai/documentation/ambient-documentation)).

## Specs

Each measure is bound twice — once per EHR — under the same `CM-` number. The two
folders mirror each other (`00-OVERVIEW.md` + `CM-04/05/06/20/21/22.md`).

### Epic (Clarity) — [`epic/`](epic/00-OVERVIEW.md)

| Spec | Measure | D&M dimension | Data basis |
|---|---|---|---|
| [CM-04](epic/CM-04-documentation-time.md) | Documentation Time | Individual Impact | Clarity proxy (note edit log) |
| [CM-05](epic/CM-05-after-hours.md) | After-Hours Documentation | Individual Impact | Clarity proxy (note edit log) |
| [CM-06](epic/CM-06-chart-closure.md) | Chart Closure Timeliness | Individual Impact | Clarity-native |
| [CM-20](epic/CM-20-financial-productivity.md) | Financial Productivity & Revenue | Organizational Impact | Clarity-native |
| [CM-21](epic/CM-21-coding-accuracy.md) | Coding Accuracy (ICD-10 / HCC / E&M) | Organizational Impact | Clarity-native + Suki API |
| [CM-22](epic/CM-22-volume-throughput.md) | Patient Volume & Throughput | Organizational Impact | Clarity-native |

**Start with [`epic/00-OVERVIEW.md`](epic/00-OVERVIEW.md)** — it lays out the four data
tiers, the shared comparison methodology (within-provider pre/post, onboarding-ramp
exclusion, specialty/FTE normalization), the Suki attribution dependency, and the verified
Epic join paths every spec reuses.

### Cerner / Oracle Health (Millennium) — [`cerner/`](cerner/00-OVERVIEW.md)

| Spec | Measure | D&M dimension | Data basis |
|---|---|---|---|
| [CM-04](cerner/CM-04-documentation-time.md) | Documentation Time | Individual Impact | Millennium proxy (`CLINICAL_EVENT` author→verify span) |
| [CM-05](cerner/CM-05-after-hours.md) | After-Hours Documentation | Individual Impact | Millennium proxy (`CLINICAL_EVENT` timestamps) |
| [CM-06](cerner/CM-06-chart-closure.md) | Chart Closure Timeliness | Individual Impact | Millennium-native (`CHART_COMPLETE_DT_TM`) |
| [CM-20](cerner/CM-20-financial-productivity.md) | Financial Productivity & Revenue | Organizational Impact | Millennium-native (`CHARGE`/`PFT_RVU_CONTENT`) |
| [CM-21](cerner/CM-21-coding-accuracy.md) | Coding Accuracy (ICD-10 / HCC / E&M) | Organizational Impact | Millennium-native (`DIAGNOSIS→NOMENCLATURE`) + Suki API |
| [CM-22](cerner/CM-22-volume-throughput.md) | Patient Volume & Throughput | Organizational Impact | Millennium-native (`ENCOUNTER`) |

Because Cerner publishes real keys and richer reference data, several bindings are *more
direct* than Epic's — real ICD-10 codes via `NOMENCLATURE` (vs Epic's `DX_ID` proxy), native
`PFT_RVU_CONTENT` for wRVU, and a true `CHART_COMPLETE_DT_TM` timestamp. The trade-off: CM-04
has no sessionizable edit log on Cerner, so it's a coarser author→verify span. The canonical
definition, Suki attribution, and comparison methodology are identical across vendors —
which is the point: one dashboard, two EHR bindings, neutral measures.

## Status & roadmap

Covered so far: all six **Individual- and Organizational-Impact** measures
(CM-04/05/06/20/21/22), bound for **both** Epic and Cerner. CM-06/20/21/22 are derived
natively from the relational record; CM-04/05 are documentation-time/after-hours proxies
from note edit timestamps. Planned next:

- **EHR efficiency-telemetry tier** — CM-07 total EHR time (needs Epic Signal/UAL or Cerner
  Advance / "Lights On"); also the gold-standard calibration source for the CM-04/05 proxies.
- **Suki / adoption tier** — CM-13 utilization, CM-14 long-term use.
- Survey-based measures (burnout, trust, satisfaction) need instruments, not algorithms.

## Conventions

Each spec gives the canonical definition, the EHR source + join path (verified against the
relevant knowledge graph — `epic-ehi-kg` or `cerner-ehi-kg`), the Suki field bindings,
illustrative SQL/pseudocode, the pre/post comparison logic, and confounders/guardrails. Each
vendor folder's `00-OVERVIEW.md` holds the shared methodology and the verified join paths its
specs reuse. Items needing verification with the Suki team (or vendor-specific code values)
are marked **`⚠️ CONFIRM`**. SQL is illustrative, not tuned for a specific Clarity or
Millennium backend.

## Provenance & disclaimer

Derived from the ACI canonical measures (2026-03/04) and the vendors' published EHI-export
schemas — Epic's Clarity model (`epic-ehi-kg`, May 2026 release) and Cerner's Millennium
model (`cerner-ehi-kg`, 2026.1.01). **Not affiliated with, endorsed by, or sponsored by
Epic, Oracle, Cerner, Oracle Health, or Suki.** Table/column names are factual schema
metadata from the vendors' public EHI Export specifications; Suki field/endpoint references
are from public documentation and the `aci` repo's hints, and should be confirmed against
Suki's actual data contract.

## License

[Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](LICENSE) — the
same license as [`pbiondich/aci`](https://github.com/pbiondich/aci), whose canonical
measures these specs adapt and extend (ShareAlike). If you build on this, contribute
improvements back.
