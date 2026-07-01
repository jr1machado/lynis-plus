

[![Linux Security Expert badge](https://badges.linuxsecurity.expert/tools/ranking/lynis.svg)](https://linuxsecurity.expert/tools/lynis/)
[![Build Status](https://travis-ci.org/CISOfy/lynis.svg?branch=master)](https://travis-ci.org/CISOfy/lynis)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/96/badge)](https://bestpractices.coreinfrastructure.org/projects/96)
[Documentation]

[Documentation]: https://cisofy.com/documentation/lynis/

Do you like this software? **Star the project** and become a [stargazer](https://github.com/CISOfy/lynis/stargazers).

----

# lynis

> Lynis - Security auditing and hardening tool, for UNIX-based systems.

## lynis-plus (this fork)

This repository is **lynis-plus**, a fork of upstream Lynis, currently at **version 4.0**. It adds a CIS Benchmark compliance, hardening-report, and guided remediation layer on top of the standard Lynis audit engine. See [`RELEASE_NOTES_4_0.md`](RELEASE_NOTES_4_0.md) and [`NEW_FEATURES_4_0.MD`](NEW_FEATURES_4_0.MD) for details.

### Contributions in 4.0

- CIS Benchmark compliance scoring engine (`include/compliance`), scoring Lynis test results against CIS controls per category and per CIS Level (L1/L2).
- CIS rule catalog (`include/rules_cis/`) covering Ubuntu 22.04/24.04 LTS, Oracle Linux 9, RHEL 9, and Rocky Linux 9 (L1 + L2), with every mapped `test-no` validated against real Lynis tests.
- Offline HTML/Markdown hardening dashboard generator (`include/report_hardening`).
- New `--hardening` / `--hardening-l2` CLI flags, fully opt-in and backwards compatible with the standard `lynis audit system` flow.
- Backup Engine (`include/backup`) — timestamped snapshot of any file before it's modified, tracked in a plain-text manifest.
- Remediation Engine (`include/remediation`) — applies a single CIS control fix with mandatory human approval, then auto-verifies and auto-rolls-back on failure.
- Rollback Engine (`include/rollback`) — restores a file to its last backed-up state, on demand or automatically.
- New `lynis apply` / `lynis rollback` commands, with fix/verify metadata currently shipped for `SSH-001` (disable root login) and `AUTH-005` (default umask) across all 5 supported distros; every other control reports "no automated fix available yet" instead of guessing.

### Usage guide

**1. CIS compliance scoring + hardening report**

```sh
./lynis audit system --hardening        # score the audit against CIS Level 1 controls
./lynis audit system --hardening-l2     # also score CIS Level 2 controls (audit-only, separate score)
```

Requires root (or sudo) for a full scan, same as a normal Lynis audit. Output:
- On-screen: overall + per-category score, right after the normal Lynis summary.
- `/var/log/lynis-report.dat`: raw `finding[]=` / `cis_*=` entries.
- `/var/log/lynis-hardening-report.html`: standalone offline dashboard (dark theme, color-coded PASS/FAIL/SKIP).
- `/var/log/lynis-hardening-report.md`: same data as a Markdown table, good for CI/PRs.

Supported OS: Ubuntu 22.04/24.04 LTS, Oracle Linux 9, RHEL 9, Rocky Linux 9. Other OSes just skip the compliance step (audit still runs normally).

**2. Applying a fix for a CIS control**

```sh
./lynis apply --test-no SSH-7408              # disable root SSH login (with approval prompt)
./lynis apply --test-no AUTH-9328              # set a strict default umask (027)
./lynis apply --test-no <ANY-TEST-ID> --dry-run  # preview impact + commands, no changes made, no prompt
```

`<ID>` is the Lynis `test-no` shown in `finding[]=` entries or the hardening report table (e.g. `SSH-7408`); the CIS rule ID (e.g. `SSH-001`) also works. Flow, always in this order:

1. Shows the rule's title, CIS section, severity, target file, and the exact fix/verify commands.
2. If no fix is defined for that control yet, says so and stops — nothing is touched.
3. Prompts `Apply this fix? [y/N]` — nothing happens without an explicit `y`.
4. Backs up the target file (`backups/<date>/<rule>.<file>.<timestamp>.bak`, indexed in `backups/manifest.log`).
5. Runs the fix, then the verify command.
6. If verification fails, automatically restores the backup and reports the failure.
7. Logs the outcome to `history/remediation.log` (`timestamp|rule|test-no|status|hostname`).

Only CIS Level 1 rules can be applied — Level 2 controls are intentionally audit-only (higher risk of breaking things) and are never targeted by `apply`.

**3. Rolling back a fix**

```sh
./lynis rollback --test-no SSH-7408
```

Restores the target file from its most recent backup for that rule, regardless of whether the last `apply` succeeded, failed, or was applied a while ago. Safe to run even if nothing was ever applied — it just reports there's no backup to restore.

**4. Global flags still apply**

Any global Lynis flag (`--no-colors`, `--quiet`, `--verbose`, ...) must be placed *before* the subcommand, same as with `lynis show` or `lynis configure`:

```sh
./lynis --no-colors audit system --hardening
```

Lynis is a security auditing tool for systems based on UNIX like Linux, macOS, BSD, and others. It performs an **in-depth security scan** and runs on the system itself. The primary goal is to test security defenses and **provide tips for further system hardening**. It will also scan for general system information, vulnerable software packages, and possible configuration issues. Lynis was commonly used by system administrators and auditors to assess the security defenses of their systems. Besides the "blue team," nowadays penetration testers also have Lynis in their toolkit.

We believe software should be **simple**, **updated on a regular basis**, and **open**. You should be able to trust, understand, and have the option to change the software. Many agree with us, as the software is being used by thousands every day to protect their systems.

## Goals

The main goals of Lynis include:
- Automated security auditing
- Compliance testing (e.g. ISO27001, PCI-DSS, HIPAA)
- Vulnerability detection

The software (also) assists with:
- Configuration and asset management
- Software patch management
- System hardening
- Penetration testing (privilege escalation)
- Intrusion detection

### Audience

Typical users of the software:
- System administrators
- Auditors
- Security officers
- Penetration testers
- Security professionals

## Installation

There are multiple options available to install Lynis.

### Software package

For systems running Linux, BSD, and macOS, there is typically a package available. This is the preferred method of obtaining Lynis, as it is quick to install and easy to update. The Lynis project itself also provides [packages](https://packages.cisofy.com/) in RPM or DEB format suitable for systems systems running:
`CentOS`, `Debian`, `Fedora`, `OEL`, `openSUSE`, `RHEL`, `Ubuntu`, and others.

Some distributions may also have Lynis in their software repository: [![Repology](https://repology.org/badge/tiny-repos/lynis.svg)](https://repology.org/project/lynis/versions)

Note: Some distributions don't provide an up-to-date version. In that case it is better to use the CISOfy software repository, download the tarball from the website, or download the latest GitHub release.

### Git

The very latest developments can be obtained via git.

1. Clone or download the project files (**no compilation nor installation** is required) ;

        git clone https://github.com/CISOfy/lynis

2. Execute:

        cd lynis && ./lynis audit system

If you want to run the software as `root` (or sudo), we suggest changing the ownership of the files. Use `chown -R 0:0` to recursively alter the owner and group and set it to user ID `0` (`root`). Otherwise Lynis will warn you about the file permissions. After all, you are executing files owned by a non-privileged user.


## Documentation

Have a look at the [Lynis documentation](https://cisofy.com/documentation/lynis/) to learn more about the configuration and usage of Lynis. When you are interested in reading more articles about Linux security, then check out the [Linux security blog](https://linux-audit.com/) named Linux Audit. For some suggestions by Lynis, this is also the source used to learn more about specific findings.

## Customization

If you want to create your own tests, have a look at the [Lynis software development kit](https://github.com/CISOfy/lynis-sdk).

## Security

We participate in the [CII best practices](https://www.bestpractices.dev/en/projects/96) badge program of the Linux Foundation.

## Media and Awards

Lynis is collecting some awards along the way and we are proud of that.

* 2016
  * [Best of Open Source Software Awards 2016](http://www.infoworld.com/article/3121251/open-source-tools/bossie-awards-2016-the-best-open-source-networking-and-security-software.html#slide13).
  * Article by TechRepublic, considering Lynis a "must-have" tool: [How to quickly audit a Linux system from the command line](http://www.techrepublic.com/article/how-to-quickly-audit-a-linux-system-from-the-command-line/)
  * [![ToolsWatch Best Tools (top 10)](https://www.toolswatch.org/badges/toptools/2016.svg)](https://www.toolswatch.org/2017/02/2016-top-security-tools-as-voted-by-toolswatch-org-readers/)

* 2015
  * [![ToolsWatch Best Tools (second place)](https://www.toolswatch.org/badges/toptools/2015.svg)](https://www.toolswatch.org/2016/02/2015-top-security-tools-as-voted-by-toolswatch-org-readers/)
  * [Best of Open Source Software Awards 2015](http://www.idgenterprise.com/news/press-release/infoworld-announces-the-2015-best-of-open-source-software-awards/) ([mirror](https://web.archive.org/web/20210313082124/https://www.idg.com/news/infoworld-announces-the-2015-best-of-open-source-software-awards/)).

* 2014
  * [![ToolsWatch Best Tools (third place)](https://www.toolswatch.org/badges/toptools/2014.svg)](https://www.toolswatch.org/2015/01/2014-top-security-tools-as-voted-by-toolswatch-org-readers/)

* 2013
  * [![ToolsWatch Best Tools (sixth place)](https://www.toolswatch.org/badges/toptools/2013.svg)](https://www.toolswatch.org/2013/12/2013-top-security-tools-as-voted-by-toolswatch-org-readers/)

## Contribute

> We love contributors.

Do you have something to share? Want to help out with translating Lynis into your own language? Create an issue or pull request on GitHub, or send us an e-mail: lynis-dev@cisofy.com.

More details can be found in the [Contributors Guide](https://github.com/CISOfy/lynis/blob/master/CONTRIBUTING.md).

You can also simply contribute to the project by _starring_ the project and show your appreciation that way.

Thanks!

## License

> GPLv3

## Enterprise version for companies

This software component is also part of an enterprise solution and focuses on companies. Same quality, yet with more functionality.

Focus areas include compliance (`PCI DSS`, `HIPAA`, `ISO27001`, and others). The Enterprise version comes with:
* a web interface;
* dashboard and reporting;
* hardening snippets;
* improvement plan (based on risk);
* commercial support.
