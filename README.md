# Suki Dashboard Algorithms

Operational algorithm specs for an **Ambient Clinical Intelligence (ACI) analytics
dashboard** — each one binds a **canonical evaluation measure** to **Epic's data model**
and **Suki's** ambient-documentation telemetry, so a measure goes from a literature
definition to a computable dashboard metric.

Three sources meet here:

- **What to measure** — the canonical measures from the ACI structured evidence review
  ([`pbiondich/aci`](https://github.com/pbiondich/aci)).
- **Where the data lives & how to join it** — Epic's EHI/Clarity schema, via the
  knowledge graph in [`pbiondich/epic-ehi-kg`](https://github.com/pbiondich/epic-ehi-kg).
- **The intervention & its telemetry** — Suki ambient sessions and structured-data output
  ([developer.suki.ai](https://developer.suki.ai/documentation/ambient-documentation)).

## Specs

| Spec | Measure | D&M dimension | Data basis |
|---|---|---|---|
| [CM-04](CM-04-documentation-time.md) | Documentation Time | Individual Impact | Clarity proxy (note edit log) |
| [CM-05](CM-05-after-hours.md) | After-Hours Documentation | Individual Impact | Clarity proxy (note edit log) |
| [CM-06](CM-06-chart-closure.md) | Chart Closure Timeliness | Individual Impact | Clarity-native |
| [CM-20](CM-20-financial-productivity.md) | Financial Productivity & Revenue | Organizational Impact | Clarity-native |
| [CM-21](CM-21-coding-accuracy.md) | Coding Accuracy (ICD-10 / HCC / E&M) | Organizational Impact | Clarity-native + Suki API |
| [CM-22](CM-22-volume-throughput.md) | Patient Volume & Throughput | Organizational Impact | Clarity-native |

**Start with [`00-OVERVIEW.md`](00-OVERVIEW.md)** — it lays out the four data tiers, the
shared comparison methodology (within-provider pre/post, onboarding-ramp exclusion,
specialty/FTE normalization), the Suki attribution dependency, and the verified Epic join
paths every spec reuses.

## Status & roadmap

Covered so far: the **Clarity-native** measures (CM-06/20/21/22) plus **Clarity proxies**
for CM-04 (documentation time) and CM-05 (after-hours), derived from note edit-event
timestamps. Planned next:

- **Epic Signal / UAL tier** — CM-07 total EHR time (needs the Signal feed); also the
  gold-standard calibration source for the CM-04/05 proxies.
- **Suki / adoption tier** — CM-13 utilization, CM-14 long-term use.
- Survey-based measures (burnout, trust, satisfaction) need instruments, not algorithms.

## Conventions

Each spec gives the canonical definition, the Epic source + join path (verified against the
`epic-ehi-kg` graph), the Suki field bindings, illustrative SQL/pseudocode, the pre/post
comparison logic, and confounders/guardrails. Items needing verification with the Suki team
are marked **`⚠️ CONFIRM`**. SQL is illustrative, not tuned for a specific Clarity backend.

## Provenance & disclaimer

Derived from the ACI canonical measures (2026-03/04) and the Epic EHI export (May 2026
release) knowledge graph. **Not affiliated with, endorsed by, or sponsored by Epic or
Suki.** Epic table/column names are factual schema metadata from Epic's public EHI Export
Specification; Suki field/endpoint references are from public documentation and the `aci`
repo's hints, and should be confirmed against Suki's actual data contract.

## License

[Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](LICENSE) — the
same license as [`pbiondich/aci`](https://github.com/pbiondich/aci), whose canonical
measures these specs adapt and extend (ShareAlike). If you build on this, contribute
improvements back.
