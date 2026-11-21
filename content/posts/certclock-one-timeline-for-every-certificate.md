---
title: "CertClock: one timeline for every certificate, including the CA behind it"
date: 2026-10-29T08:00:00+01:00
description: "An agentless expiry radar that reads TLS endpoints, certificate files, Windows stores and ADCS exports, deduplicates by fingerprint, and tells you when an issuing CA will take its leaves down with it."
tags: ["certificates", "tls", "pki", "python", "devops"]
repo: "certclock"
image: "/images/certclock.png"
---

A gateway certificate expires overnight. The same certificate also sits in a file on a Linux host. It was issued by an internal CA that itself expires in 70 days, and a second server certificate from that CA still shows 200 days left. That second certificate stops being trusted when the CA expires, not after 200 days.

Tools that watch only TLS endpoints, only files or only the CA each see one piece of that story. [CertClock](https://github.com/Abbasi26/certclock) collects all of them, deduplicates by SHA-256 fingerprint, and records which certificate depends on which issuer.

This matters more every year. Public TLS lifetimes have been capped at 200 days since 15 March 2026 and are scheduled to reach 47 days in 2029 under the [CA/Browser Forum baseline requirements](https://cabforum.org/working-groups/server/baseline-requirements/requirements/). Renewals that used to happen once a year will happen every few weeks.

## Try it in a minute

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
certclock demo
```

That writes `certclock-demo/` with fictional data (`example.test`). Open `index.html` for the timeline. Add `--offline` to skip the loopback TLS servers the demo starts. Nothing is sent anywhere.

## Where the certificates come from

CertClock is agentless. It reads sources you already have:

- **TLS endpoints** from a `hosts.txt` (one `host:port [sni]` per line, IPv6 included). It keeps the leaf and every other certificate the handshake presented. On Python 3.13+ that uses `SSLSocket.get_unverified_chain()`. On 3.11 and 3.12 it falls back to `openssl s_client -showcerts`.
- **Files**: `.pem`, `.crt`, `.cer`, `.der`, `.pfx` and `.p12`. `PRIVATE KEY` blocks are ignored and called out in a warning. Password-protected PKCS#12 is skipped, and CertClock never tries a password.
- **Windows certificate stores** through a read-only PowerShell script (`Get-CertClockStore.ps1`). It opens stores with `ReadOnly` and `OpenExistingOnly` and writes metadata only, never the key.
- **ADCS**: a CSV produced with `certutil -view`. Revoked and failed requests are skipped, so they don't clutter the renewal timeline.
- **Linux hosts** through a read-only Ansible playbook that runs a small collector script and saves JSON on the controller.

`certclock export-collectors --dest ./collectors` copies the Windows script, the Linux collector and the playbook out of the installed package.

## Dependencies, not just dates

The interesting findings concern relationships between certificates. CertClock links issuers by authority key identifier first and issuer DN second, then reports:

| Finding | What it means |
| --- | --- |
| `issuer_expires_first` | The CA expires before the leaf. Clients stop trusting the leaf at the CA's date |
| `chain_omits_issuer` | The server didn't send an intermediate that you do have in the inventory |
| `missing_intermediate` | The server didn't send an issuer that you don't have |
| `issuer_not_in_inventory` | No chain was seen and the issuer wasn't collected, so the dependency can't be timed |

Alongside those come the usual hygiene checks: expiry windows of 7, 30 and 90 days, `san_mismatch`, `weak_key` (RSA under 2048 bits, EC under 256, any DSA), `sha1` and `self_signed`. A self-signed root isn't flagged just because a server didn't send it, and a leaf-only TLS read isn't reported as a missing intermediate, because the chain was never seen.

## Outputs that plug into what you run

One `certclock scan --config certclock.yml` writes:

- `index.html`: a self-contained timeline with expired, 7, 30 and 90-day buckets
- `calendar.ics`: one event per certificate with 30, 7 and 1-day reminders
- `checkmk.out`: Checkmk local-check lines (WARN at 30 days, CRIT at 7)
- `certclock.prom`: a Prometheus textfile for `node_exporter`
- `inventory.json`, `inventory.csv` and `report.md`
- `webhook-preview.json`: the notification body, **not sent**

Webhooks are off by default. `--send` is the only switch that transmits anything, and it prints the preview first.

An optional `owners.yml` maps certificates to teams. Rules are matched in order, and unknown match keys are an error, so a typo can't silently send everything to the catch-all.

## A gate for CI and cron

`certclock check` is the one-liner for pipelines:

```bash
certclock check --host vpn.example.test:443 --path /etc/ssl/certs/app.pem
certclock check --db certclock.db --fail-on warn --format json
```

| Exit code | Meaning |
| --- | --- |
| `0` | Nothing at or past the failing level |
| `1` | Something is at WARN (default 30 days), only with `--fail-on warn` |
| `2` | Something is at CRIT (default 7 days) or already expired |
| `3` | Usage or input error |

There's also a GitHub Action that runs `check` and writes the Markdown report to the job summary:

```yaml
- uses: Abbasi26/certclock@v0.2.0
  with:
    hosts: |
      www.example.test:443
    paths: deploy/certs
    fail-on: crit
```

## What it doesn't do

CertClock doesn't renew anything, doesn't open WinRM, doesn't run `certutil` and doesn't listen for scrapes. The only write outside its output directory and SQLite snapshot is a webhook POST, and only with `--send`. The database holds the latest snapshot, not a history.

The code is MIT-licensed and on GitHub: [Abbasi26/certclock](https://github.com/Abbasi26/certclock). Good first issues are in [ROADMAP.md](https://github.com/Abbasi26/certclock/blob/main/ROADMAP.md).
