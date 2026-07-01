# New Features in Lynis-Plus 4.0 / 4.0.1

## 1. CIS Benchmark Compliance Scoring

New opt-in flags on the existing `lynis audit system` command:

```sh
./lynis audit system --hardening        # score against CIS Level 1 controls
./lynis audit system --hardening-l2     # also score CIS Level 2 controls
```

Implementation:
- `include/consts` — new `HARDENING_MODE` / `HARDENING_INCLUDE_L2` state flags.
- `include/parameters` — parses `--hardening` / `--hardening-l2`.
- `lynis` — after the normal test run, conditionally sources `include/compliance` and `include/report_hardening`.

Requires root (or sudo) for a full scan, same as a normal Lynis audit. Any global flag (`--no-colors`, `--quiet`, ...) must be placed *before* `audit system`, same convention as `lynis show`/`lynis configure`.

## 2. Compliance Engine (`include/compliance`)

- Auto-detects OS/version and selects the matching `.rules` file from `include/rules_cis/`.
- Parses each CIS rule block (`test-no`, `title`, `section`, `severity`, `category`, `level`, `mapping`, `note`).
- Cross-references each rule's `test-no` against the Lynis test run:
  - `SKIP` — underlying test never ran for this system.
  - `FAIL` — test produced a `warning[]`/`suggestion[]` entry.
  - `PASS` — test ran clean.
- Computes an overall score and a per-category score, for L1 and L2 independently.
- Writes `finding[]=` and `cis_*=` entries back into the Lynis report file for downstream consumption.

## 3. CIS Rule Catalog (`include/rules_cis/`)

Declarative, versioned `.rules` files (10 total: 5 distros × L1/L2), each control mapped to a real Lynis `test-no`:

| Distro | L1 | L2 |
|---|---|---|
| Ubuntu 22.04 LTS | ✅ | ✅ |
| Ubuntu 24.04 LTS | ✅ | ✅ |
| Oracle Linux 9 | ✅ | ✅ |
| RHEL 9 | ✅ | ✅ |
| Rocky Linux 9 | ✅ | ✅ |

Every `test-no` referenced has been validated against the actual `Register --test-no` entries in `include/tests_*`.

## 4. Hardening Report Engine (`include/report_hardening`)

Generates two standalone, offline reports next to the standard Lynis report:

- `lynis-hardening-report.html` — dark-themed dashboard with score, per-category bars, and a findings table color-coded by status (pass/fail/skip).
- `lynis-hardening-report.md` — same data in Markdown table form for CI pipelines / PRs.

Level 2 findings are rendered in a separate section with a banner noting they are audit-only (no automated fix planned, given the higher blast radius of L2 controls).

## 5. Backup Engine (`include/backup`)

- `BackupFile <rule-id> <file>` — copies the file to `backups/<date>/<rule>.<flattened-path>.<timestamp>.bak` and appends a line to `backups/manifest.log` (`timestamp|rule|source|backup-path|sha256|hostname`).
- `BackupFindLatest <rule-id> <file>` — looks up the manifest for the most recent backup of a given file for a given rule.
- Fails safe: if the source file doesn't exist, it returns non-zero and no backup is written.

## 6. Remediation Engine (`include/remediation`)

Extends the `.rules` format (only for L1 rules) with three optional fields:

```
backup-file=/etc/ssh/sshd_config
fix-cmd=<shell command that makes the control compliant>
verify-cmd=<shell command that exits 0 when compliant>
post-cmd=<optional best-effort command, e.g. reload a service>
```

`RemediationRun <test-no-or-rule-id> <dry-run-flag>`:
1. Resolves the target rule (by CIS rule ID or Lynis `test-no`) from the OS-appropriate L1 `.rules` file. L2 files are never loaded here — Level 2 stays audit-only.
2. Runs `verify-cmd` first: if it already passes, prints green `[OK]` and stops — no prompt, no changes.
3. If no `fix-cmd` is defined, says so and stops.
4. Otherwise prints the fix/verify preview in red `[FAIL]`. In `--dry-run` mode, stops here.
5. Prompts `Apply this fix? [y/N]` — nothing happens without an explicit `y`.
6. Backs up the target file (if `backup-file` is set), runs `fix-cmd`, then re-runs `verify-cmd`.
7. If verification fails, automatically restores the backup via the Rollback Engine and reports the failure.
8. Logs every outcome to `history/remediation.log` (`timestamp|rule|test-no|status|hostname`).

## 7. Rollback Engine (`include/rollback`)

- `RollbackFile <rule-id> <file>` — looks up the latest backup for that rule/file pair and restores it.
- Used automatically by the Remediation Engine on verify failure, and on demand via `lynis rollback --test-no <ID>`.
- Controls whose fix only creates a new file (e.g. a `/etc/sysctl.d/` drop-in) have no `backup-file` by design — there is nothing to "restore", `rollback` reports this instead of failing.

## 8. New CLI commands

```sh
lynis apply --test-no <ID> [--dry-run]
lynis rollback --test-no <ID>
```

`<ID>` accepts either the Lynis `test-no` or the CIS rule ID (e.g. `SSH-7408` or `SSH-001`). Use the **rule ID** when two CIS controls share the same underlying Lynis test (e.g. `KRNL-002`/`KRNL-003` both map to `KRNL-6000`) — a `test-no` lookup always resolves to the first rule using it in the file.

Wired via `include/helper_apply` / `include/helper_rollback`, dispatched from `include/parameters`, following the same `HELPER`/`HELPER_PARAMS` convention as `lynis show`/`lynis configure`.

## 9. Fix coverage (CIS Level 1 only)

| Distro | Controls with a fix | Total L1 controls |
|---|---|---|
| Ubuntu 22.04 | 15 | 21 |
| Ubuntu 24.04 | 16 | 22 |
| Oracle Linux 9 | 11 | 21 |
| RHEL 9 | 11 | 21 |
| Rocky Linux 9 | 11 | 21 |

Covered: SSH root login (`SSH-001`), default umask (`AUTH-005`), PAM password quality (`AUTH-006`), password aging (`AUTH-007`), core dump storage (`KRNL-001`), safe sysctl hardening (`KRNL-002`/`KRNL-003` — `dmesg_restrict`/`kptr_restrict` on Ubuntu, `tcp_syncookies` on RHEL family), AppArmor/SELinux tooling install (`MACF-001`, plus AppArmor enable `MACF-002` on Ubuntu), login/net banners (`BANN-001`/`BANN-002`, Ubuntu only), cron daemon and `at` restriction (`SCHD-001`/`SCHD-002`), rsyslog/journald/logrotate (`LOGG-001`/`LOGG-002`/`LOGG-003`).

Left manual on purpose:
- **`SSH-002`, `AUTH-001`-`AUTH-004`** (SSH user/group allowlist, duplicate UID/GID/username/group cleanup) — fixing these means picking which account or group to touch; that's a human decision.
- **`FIRE-001`/`FIRE-002`/`FIRE-003`** (firewall/nftables), **RHEL family's `KRNL-002`** (disable IP forwarding — would break routing/NAT, e.g. Docker hosts), **RHEL family's `MACF-002`** (SELinux enforcing mode — needs a relabel + reboot) — high blast-radius changes, deliberately left for manual action.

## 10. Verified end-to-end

- `./lynis audit system --hardening-l2` scoring tested on a live Ubuntu 24.04 host and, via Docker, on Ubuntu 22.04, Rocky Linux 9, Oracle Linux 9, and RHEL 9 (via the `redhat/ubi9` image) — correct OS detection, correct `.rules` file selection, correct L1/L2 scores on all 5.
- `lynis apply`/`lynis rollback` tested across all 5 distros: dry-run preview renders fix/verify commands correctly (including sed backreferences like `\1`), green `[OK]` vs red `[FAIL]` status is accurate, and a real apply → verify → rollback cycle restores the modified file byte-for-byte identical to its pre-fix state.
- No regressions in the base (non-hardening) Lynis audit flow.
