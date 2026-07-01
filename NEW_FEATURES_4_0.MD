# New Features in Lynis-Plus 4.0

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

## 5. Verified end-to-end

Ran `./lynis audit system --hardening-l2` on a live Ubuntu 24.04 host:
- Compliance scoring: L1 68%, L2 37% (real scan, not synthetic).
- HTML/MD reports rendered correctly with per-control status.
- No regressions in the base (non-hardening) Lynis audit flow.
