# Lynis-Plus 4.0 — Release Notes

Lynis-Plus is a fork of [Lynis](https://github.com/CISOfy/lynis), built on top of upstream release 3.1.7, adding a CIS Benchmark compliance and hardening-report layer on top of the existing audit engine.

## Highlights

- **CIS Compliance Engine** (`include/compliance`): cross-references Lynis test results against CIS Benchmark controls and computes an adherence score, per category and per CIS Level (L1 / L2).
- **CIS Rule Catalog** (`include/rules_cis/`): declarative `.rules` mappings covering:
  - Ubuntu 22.04 LTS (L1 + L2)
  - Ubuntu 24.04 LTS (L1 + L2)
  - Oracle Linux 9 (L1 + L2)
  - RHEL 9 (L1 + L2)
  - Rocky Linux 9 (L1 + L2)
- **Hardening Report Engine** (`include/report_hardening`): generates standalone offline HTML and Markdown dashboards (`lynis-hardening-report.html` / `.md`) with score, per-category breakdown, and per-control status (PASS/FAIL/SKIP).
- **New CLI flags**:
  - `--hardening` — run the audit and score it against CIS Level 1 controls.
  - `--hardening-l2` — also load and score CIS Level 2 controls (audit-only, scored separately from L1, no automated remediation).

## Upgrade notes

- Fully backwards compatible: existing `lynis audit system` behavior is unchanged when `--hardening` is not passed.
- No new external dependencies; everything runs in the same `/bin/sh`-compatible engine as upstream Lynis.

## Known limitations / roadmap

- Remediation, backup, and rollback engines are not yet implemented (planned for a future release).
- History/snapshot comparison across audits is not yet implemented.
- See `NEW_FEATURES_4_0.MD` for a full breakdown of what shipped in this release.
