---
title: "RunbookRehearsal: dry-run your runbooks before 03:00 does it for you"
date: 2026-10-29T08:25:00+01:00
description: "A non-destructive checker for written incident and operations runbooks: stale reviews, dead contacts, unreachable hosts, 404s, missing files, scripts that no longer parse and groups that no longer exist, with a freshness score and a diff."
tags: ["sre", "incident-response", "runbooks", "python", "nis2"]
repo: "runbook-rehearsal"
image: "/images/runbook-rehearsal.png"
---

The 03:00 procedure usually lives in a YAML file or a wiki page: who to call, which host to look at, which script to read, which share to restore. Those references rot quietly. The pager service gets replaced, the host gets renamed, the restore note gets moved and the script picks up a syntax error in a "quick fix".

[RunbookRehearsal](https://github.com/Abbasi26/runbook-rehearsal) reads runbooks as YAML, checks every reference without running a single step, and writes a maturity report that shows which procedures have gone stale.

## The demo: a cooperative with one rotten procedure

Harbor and Pine is a fictional library cooperative. Its finance-share restore procedure was last reviewed 507 days ago, past its 180-day maximum. It still points on-call to a retired pager and the number `12345`, expects TCP 445 on a file server that refuses it, links a restore note that's no longer in the folder, and tells you to read `invoke-share.sh`, which doesn't parse. The status URL returns 404, and the admin group named in the text isn't in the directory.

```bash
python -m pip install -e ".[dev]"
rehearse demo
```

```text
Runbook                 Score  Band        Pass  Warn  Fail  Skip
linux-disk-incident     100.0  Maintained  20    0     0     0
rebuild-backup-catalog  71.7   Drifting    14    1     3     0
restore-finance-share   36.3   Stale       11    1     10    1
Overall                 69.3   Drifting
```

The demo is fully offline. Hostnames are under `.example`, addresses are in `203.0.113.0/24`, and the directory is a stub list. It writes a self-contained `report.html` plus Markdown and JSON.

## What gets checked

Every check is non-destructive, and there's no apply command:

| Check | What it does |
| --- | --- |
| Freshness | `lastReviewed` against the review interval and maximum age |
| Contacts | Email shape, and `+` plus 8 to 15 digits for phone numbers |
| DNS / TCP / HTTP | Resolves the host, connects to the port, requests the URL |
| Files and links | Local paths relative to the runbook, plus one level of links inside them |
| Script syntax | Python via `compile()`, bash via `bash -n`, PowerShell via `Parser.ParseFile`. Parsed, never run |
| Groups | Only when a directory provider (stub, LDAP or Microsoft Graph) is configured |
| Completeness | High/critical procedures need preconditions, and every step needs an owner |

A missing `pwsh` or `bash` counts as a warning, not a pass. Skipped findings are listed and left out of the score.

## A score you can recompute

Weights are freshness 20, contacts 15, reachability 25, references 20, directory 10 and completeness 10. A pass counts 1, a warning 0.5 and a failure 0. The bands are Maintained (90+), Usable with gaps (75+), Drifting (50+), Stale (25+) and Not rehearsable. The JSON report repeats the weights, every finding and the SHA-256 of each runbook file, so a CI job can query it:

```bash
jq -r '.runbooks[].findings[] | select(.status=="fail") | "\(.runbookId) \(.check): \(.message)"' rehearse-out/report.json
```

## Adopt it on old procedures without drowning

```bash
rehearse check runbooks --config rehearse.yaml --output out --baseline last-week/report.json --fail-on-regression
```

`rehearse diff` and `--fail-on-regression` fail the run only when something got *worse*: a new failure or a band drop. Freshness findings are matched per runbook, so a new review date shows up as fixed. A team can start with a pile of old runbooks and still stop new rot. An example GitHub Actions workflow compares the base branch with the pull request offline and puts the diff in the job summary.

`rehearse init restore-erp --output runbooks/restore-erp.yaml` writes a skeleton that already passes validation.

## Tabletop exercises, recorded

`rehearse tabletop` walks the preconditions and steps in the terminal and records outcomes (`met`, `not_met`, `partial`, `skipped`), notes, prompt answers and time per step. An answers file works for transcribed exercises. The record is kept separate from the score, because it captures what people said, not proof that a restore works.

## Evidence, with its limits stated

Reports name the themes a reviewer would file them under: NIS2 Article 21(2)(b) and (c), ISO/IEC 27001:2022 Annex A controls A.5.24 to A.5.37, and BSI IT-Grundschutz DER.2.1 and DER.4. Next to each one, the report states the limit. A dry run shows that the written procedure and its references were checked. It doesn't show that a restore meets an RTO or that someone answers the phone.

## What it isn't

It isn't Rundeck or PagerDuty Runbook Automation, which execute jobs. It isn't a chaos tool, because it injects no faults. In live mode, the only traffic is DNS, a TCP handshake and an HTTP request to targets the procedure already names, so get permission before pointing it at a network. It's not on PyPI yet, so install from a checkout or the git URL.

Code: [Abbasi26/runbook-rehearsal](https://github.com/Abbasi26/runbook-rehearsal) (MIT).
