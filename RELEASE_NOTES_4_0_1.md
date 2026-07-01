lynis-plus (this fork)
This repository is lynis-plus, a fork of upstream Lynis, currently at version 4.0. It adds a CIS Benchmark compliance, hardening-report, and guided remediation layer on top of the standard Lynis audit engine. See RELEASE_NOTES_4_0.md and NEW_FEATURES_4_0.MD for details.

Contributions in 4.0
CIS Benchmark compliance scoring engine (include/compliance), scoring Lynis test results against CIS controls per category and per CIS Level (L1/L2).
CIS rule catalog (include/rules_cis/) covering Ubuntu 22.04/24.04 LTS, Oracle Linux 9, RHEL 9, and Rocky Linux 9 (L1 + L2), with every mapped test-no validated against real Lynis tests.
Offline HTML/Markdown hardening dashboard generator (include/report_hardening).
New --hardening / --hardening-l2 CLI flags, fully opt-in and backwards compatible with the standard lynis audit system flow.
Backup Engine (include/backup) — timestamped snapshot of any file before it's modified, tracked in a plain-text manifest.
Remediation Engine (include/remediation) — applies a single CIS control fix with mandatory human approval, then auto-verifies and auto-rolls-back on failure.
Rollback Engine (include/rollback) — restores a file to its last backed-up state, on demand or automatically.
New lynis apply / lynis rollback commands, with fix/verify metadata currently shipped for SSH-001 (disable root login) and AUTH-005 (default umask) across all 5 supported distros; every other control reports "no automated fix available yet" instead of guessing.
Usage guide
1. CIS compliance scoring + hardening report

./lynis audit system --hardening        # score the audit against CIS Level 1 controls
./lynis audit system --hardening-l2     # also score CIS Level 2 controls (audit-only, separate score)
Requires root (or sudo) for a full scan, same as a normal Lynis audit. Output:

On-screen: overall + per-category score, right after the normal Lynis summary.
/var/log/lynis-report.dat: raw finding[]= / cis_*= entries.
/var/log/lynis-hardening-report.html: standalone offline dashboard (dark theme, color-coded PASS/FAIL/SKIP).
/var/log/lynis-hardening-report.md: same data as a Markdown table, good for CI/PRs.
Supported OS: Ubuntu 22.04/24.04 LTS, Oracle Linux 9, RHEL 9, Rocky Linux 9. Other OSes just skip the compliance step (audit still runs normally).

2. Applying a fix for a CIS control

./lynis apply --test-no SSH-7408              # disable root SSH login (with approval prompt)
./lynis apply --test-no AUTH-9328              # set a strict default umask (027)
./lynis apply --test-no <ANY-TEST-ID> --dry-run  # preview impact + commands, no changes made, no prompt
<ID> is the Lynis test-no shown in finding[]= entries or the hardening report table (e.g. SSH-7408); the CIS rule ID (e.g. SSH-001) also works. Flow, always in this order:

Shows the rule's title, CIS section, severity, target file, and the exact fix/verify commands.
If no fix is defined for that control yet, says so and stops — nothing is touched.
Prompts Apply this fix? [y/N] — nothing happens without an explicit y.
Backs up the target file (backups/<date>/<rule>.<file>.<timestamp>.bak, indexed in backups/manifest.log).
Runs the fix, then the verify command.
If verification fails, automatically restores the backup and reports the failure.
Logs the outcome to history/remediation.log (timestamp|rule|test-no|status|hostname).
Only CIS Level 1 rules can be applied — Level 2 controls are intentionally audit-only (higher risk of breaking things) and are never targeted by apply.

3. Rolling back a fix

./lynis rollback --test-no SSH-7408
Restores the target file from its most recent backup for that rule, regardless of whether the last apply succeeded, failed, or was applied a while ago. Safe to run even if nothing was ever applied — it just reports there's no backup to restore.

4. Global flags still apply

Any global Lynis flag (--no-colors, --quiet, --verbose, ...) must be placed before the subcommand, same as with lynis show or lynis configure:

./lynis --no-colors audit system --hardening
Lynis is a security auditing tool for systems based on UNIX like Linux, macOS, BSD, and others. It performs an in-depth security scan and runs on the system itself. The primary goal is to test security defenses and provide tips for further system hardening. It will also scan for general system information, vulnerable software packages, and possible configuration issues. Lynis was commonly used by system administrators and auditors to assess the security defenses of their systems. Besides the "blue team," nowadays penetration testers also have Lynis in their toolkit.

We believe software should be simple, updated on a regular basis, and open. You should be able to trust, understand, and have the option to change the software. Many agree with us, as the software is being used by thousands every day to protect their systems.
