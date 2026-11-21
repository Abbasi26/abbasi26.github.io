---
title: "patchorder: which CVEs actually hit the versions you run, and what to patch first"
date: 2026-10-29T08:20:00+01:00
description: "Start from a short YAML inventory, pin the right CPE, let a local version matcher decide affected or not, and rank the work with CISA KEV, EPSS, CVSS and exposure. No scanning, no agents, every score shows its arithmetic."
tags: ["vulnerability-management", "cve", "security", "python", "devops"]
repo: "patchorder"
image: "/images/patchorder.png"
---

A vulnerability feed can tell you that something called nginx is having a bad week. It can't tell you that your twelve internet-facing proxies on 1.25.2 are in the vulnerable range and the three you already moved to 1.27.2 aren't.

Good tools exist on either side of that gap. [Vulnerability-Lookup](https://vulnerability.circl.lu/) aggregates NVD, KEV, EPSS and CSAF and returns every CVE for a vendor and product, leaving the version filter to the client. [OpenCVE](https://www.opencve.io/) subscribes you to products. [CVE_Prioritizer](https://github.com/TURROKS/CVE_Prioritizer) scores a CVE list you already have. None of them starts from "Windows Server 2022, build 10.0.20348.2400, eight internal hosts". [patchorder](https://github.com/Abbasi26/patchorder) does exactly that step.

## What the output looks like

The offline demo (fictional "Harbor Workshop" inventory) opens with:

```text
Patch nginx 1.25.2 on 12 internet-facing hosts (owner platform). CVE-2023-44487 is listed in CISA KEV
(exploited in the wild). NVD marks versions through 1.25.2 as affected (versionEndIncluding).
Move to a release newer than 1.25.2. Priority 91.86.

Patch OpenSSH 9.6 on 2 internal hosts (owner ops). Highest issue: CVE-2024-6387. ... Priority 45.88.

Patch Windows 11 23H2 10.0.22631.3593 on 15 internal hosts (owner it). Highest issue: CVE-2024-38063.
Update to 10.0.22631.4037 or later (NVD versionEndExcluding). Priority 43.25.
```

```bash
python -m pip install -e .
python -m patchorder demo
```

The demo reads fixtures recorded on 2026-10-21 and uses no network.

## The inventory is yours

```yaml
schema: patchorder.inventory/v1
name: Harbor Workshop (fictional)
assets:
  - id: edge-proxy
    product: nginx
    vendor: F5
    version: "1.25.2"   # quote versions: YAML reads 1.10 as 1.1
    count: 12
    owner: platform
    exposure: internet
```

Most inventories already live in a spreadsheet, so `patchorder inventory from-csv hosts.csv --output inventory.yaml` converts one. It understands headers such as `Software`, `Version`, `Hosts` and `Team`, and quotes every version. `patchorder inventory check` then lints the inventory offline and flags rows a scan couldn't decide.

## Pin the CPE, then let the matcher decide

`patchorder resolve` proposes CPE candidates with a confidence score. You pin the one you mean, and the lock file stores vendor and product but not the version, which stays in the inventory and is compared on every run. Pinning is a write, so it needs `--write` and prints the proposed lock first.

The local matcher is the source of truth for *affected / not affected / not decided*. NVD and Vulnerability-Lookup only supply candidate records. It handles:

- NVD `versionStart*` / `versionEnd*` ranges and exact CPE versions, including update fields such as OpenSSH `p1`
- CVE 5 `versions` blocks when a record has no NVD configuration
- **Windows builds** compared as four numbers: `10.0.22631.3593` is before `10.0.22631.4037`
- OpenSSH `9.8p1` after `9.8`, and OpenSSL `1.1.1w` before `1.1.1x`

It's equally careful about what it *won't* decide. A row whose version is the marketing name `2022` is "not decided", because comparing the number 2022 with `10.0.20348.2655` would produce a false "not affected". Negated CPE nodes and AND configurations that need a product the row doesn't name are also not decided.

## Priority you can audit

Every finding prints its arithmetic. The maximum is 105 points:

| Signal | Points |
| --- | --- |
| In the CISA KEV catalog loaded for this run | 40 |
| EPSS probability | probability × 25 |
| CVSS base score | score / 10 × 20 |
| Exposure | 10 internet, 4 internal |
| Host count | up to 5, logarithmic |
| A loaded CSAF document marks this exact version `known_affected` | 5 |

CSAF documents (for example from CERT-Bund) are read from files you already have. A product name without a version counts as a mention, not an "affected" decision.

## Gates for CI and night runs

```bash
python -m patchorder scan -i inventory.yaml --fail-on kev --fail-on priority:70
python -m patchorder scan -i inventory.yaml --state .patchorder/state.json --save-state --fail-on new
```

`--fail-on kev|new|internet|priority:N` exits 4 when a matching finding appears, and the briefing is printed either way. `--format csv` gives one row per finding. Cells that start with `=`, `+`, `-` or `@` get a `'` prefix, so a spreadsheet won't execute them as formulas.

## Polite to public APIs

A live scan calls NVD CVE API 2.0, Vulnerability-Lookup, the CISA KEV catalog and the FIRST EPSS API, caches responses under `.patchorder/cache` and respects rate limits (6.5 s between NVD requests without a key). `--offline` uses only the cache. There's no agent, no credentials on the inventoried systems and no telemetry. An optional LLM summary is off by default. When enabled, it receives CVE ids and descriptions, never the inventory, and `--llm-preview` shows the exact body without sending it.

Code: [Abbasi26/patchorder](https://github.com/Abbasi26/patchorder) (MIT). Good first issues are in the ROADMAP.
