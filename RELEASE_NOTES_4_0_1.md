# Lynis-Plus 4.0 / 4.0.1 — Release Notes

Lynis-Plus is a fork of [Lynis](https://github.com/CISOfy/lynis), built on top of upstream release 3.1.7, adding a CIS Benchmark compliance, hardening-report, and guided remediation layer on top of the existing audit engine.

## 4.0 — CIS compliance scoring + hardening reports

- **CIS Compliance Engine** (`include/compliance`): cross-references Lynis test results against CIS Benchmark controls and computes an adherence score, per category and per CIS Level (L1 / L2).
- **CIS Rule Catalog** (`include/rules_cis/`): declarative `.rules` mappings covering Ubuntu 22.04/24.04 LTS, Oracle Linux 9, RHEL 9, and Rocky Linux 9 (L1 + L2). Every `test-no` validated against the real `Register --test-no` entries in `include/tests_*`.
- **Hardening Report Engine** (`include/report_hardening`): standalone offline HTML and Markdown dashboards (`lynis-hardening-report.html` / `.md`) with score, per-category breakdown, and per-control status.
- **New CLI flags**: `--hardening` (score against CIS Level 1), `--hardening-l2` (also score CIS Level 2, tracked separately).

## 4.0.1 — Guided remediation (backup, apply, rollback)

- **Backup Engine** (`include/backup`): timestamped snapshot of a file before it's modified, tracked in a plain-text manifest (`backups/manifest.log`).
- **Remediation Engine** (`include/remediation`): applies a single CIS control fix — checks compliance first (green `[OK]`/red `[FAIL]`), requires explicit `y/N` approval, backs up, runs the fix, verifies, and auto-rolls-back on verification failure.
- **Rollback Engine** (`include/rollback`): restores a file to its last backed-up state, on demand (`lynis rollback`) or automatically when a fix fails verification.
- **New CLI commands**: `lynis apply --test-no <ID> [--dry-run]`, `lynis rollback --test-no <ID>`.
- **Fix coverage** (CIS Level 1 only — Level 2 stays audit-only by design):

  | Distro | Controls with a fix | Total L1 controls |
  |---|---|---|
  | Ubuntu 22.04 | 15 | 21 |
  | Ubuntu 24.04 | 16 | 22 |
  | Oracle Linux 9 | 11 | 21 |
  | RHEL 9 | 11 | 21 |
  | Rocky Linux 9 | 11 | 21 |

  Covered: SSH root login, umask, PAM password quality, password aging, core dump storage, safe sysctl hardening (dmesg_restrict, kptr_restrict, tcp_syncookies), AppArmor/SELinux tooling install (Ubuntu also gets AppArmor enable), login/net banners, cron/at, rsyslog/journald/logrotate.

  Left manual on purpose, not for lack of effort: account/group duplicate cleanup (needs a human decision on which to touch), SSH user/group allowlists (same reason), firewall/nftables rules, disabling IP forwarding, and flipping SELinux enforcing mode — all high blast-radius changes that can lock out remote access, break routing/NAT, or need a reboot + filesystem relabel.

## Upgrade notes

- Fully backwards compatible: existing `lynis audit system` behavior is unchanged when `--hardening` is not passed, and `apply`/`rollback` are new commands that don't touch the normal audit path.
- No new external dependencies; everything runs in the same `/bin/sh`-compatible engine as upstream Lynis.
- `backups/` and `history/` are runtime state created by `lynis apply` (git-ignored, not shipped).

## Known limitations / roadmap

- History/snapshot comparison across audits (`lynis compare`) is not yet implemented.
- Fix coverage for the remaining L1 controls, and any L2 remediation, is intentionally out of scope until the associated risk can be scoped down control-by-control.
- See `NEW_FEATURES_4_0.MD` for a more detailed walkthrough of each engine and how to use it.
