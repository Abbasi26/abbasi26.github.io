---
title: "Quorum: finding devices that Intune, Entra ID and Defender disagree about"
date: 2026-10-29T08:05:00+01:00
description: "A read-only PowerShell module that matches device records across Intune, Entra ID and Defender for Endpoint and reports the contradictions between them, the gaps no single console shows."
tags: ["intune", "entra-id", "defender", "powershell", "endpoint-management"]
repo: "endpoint-quorum"
image: "/images/endpoint-quorum.png"
---

Take a laptop called `win11-stale-sensor`. It synced to Intune yesterday and Intune says it's compliant. Defender for Endpoint last saw it 41 days ago. Entra ID still has the device enabled, so a device-based access rule keeps trusting an object that nobody has actually checked in six weeks.

Each console is consistent on its own terms. The signal is in the gap between them. [Quorum](https://github.com/Abbasi26/endpoint-quorum) (repo `endpoint-quorum`, PowerShell module `Quorum`) reads all three sources, matches the records for each device and reports the contradictions, with the values from each source side by side.

## Try it offline

```powershell
Import-Module ./Quorum.psd1
Invoke-Quorum -Demo
```

`-Demo` uses no network. It evaluates a fictional inventory of about 50 devices (`lab.example`, documentation IP ranges) and writes a single-file HTML report with no CDN, plus a Markdown summary, CSV and JSON. It runs on PowerShell 7, and the same commands work on Windows PowerShell 5.1.

## How three records become one device

Matching is where cross-source tools usually go wrong, so Quorum is explicit about it and shows the method and confidence on every device:

1. Records that share an Entra device id (`azureADDeviceId`, `deviceId`, `aadDeviceId`) form one device. Confidence is **high** (0.95).
2. A record without a device id can join the one cluster with the same hostname that doesn't already have that source. An exact hostname match is **medium** (0.70). A short name matched to an FQDN is **low** (0.45).
3. Two records with *different* device ids are never merged, even when the hostnames match. They stay separate and are marked as duplicates.
4. A device seen in only one source is **uncorroborated**.

## Rules are data, not code

The twelve starter rules are YAML files. A condition is a tree of `all` / `any` / `not` over fact predicates:

```yaml
- fact: defender.lastSeenAgeDays
  op: gt
  valueFrom: defenderStaleDays
```

The facts cover things like `intune.complianceState`, `intune.lastSyncAgeDays`, `entra.enabled`, `entra.lastSignInAgeDays`, `defender.onboardingStatus`, `defender.riskScore`, `os.buildMismatch`, `compliance.disagrees` and `hostname.duplicate`. Thresholds live in each file under `parameters`, so you tune a number without touching the engine.

Some of the shipped rules:

| Id | Severity | Contradiction |
| --- | --- | --- |
| Q001 | high | Intune says compliant, Defender hasn't reported for more than 14 days |
| Q003 | high | Intune has the device, Defender has no record or it isn't onboarded |
| Q007 | high | Defender risk is `high` while Intune says compliant |
| Q009 | high | Entra device is disabled while Intune says compliant |
| Q012 | high | Entra device is disabled, but Defender still has an onboarded machine that reported recently |
| Q004 | medium | Two sources report different OS build numbers |

`Test-QuorumRule` lints rule files without touching any device. It flags missing fields, duplicate ids, unknown operators and, most usefully, unknown facts. A typo in a fact name would otherwise just never match, and you'd never know. CI runs it against `rules/`.

## Live collection stays read-only

Live mode needs one app registration with three read permissions:

| Source | Permission |
| --- | --- |
| Intune managed devices | `DeviceManagementManagedDevices.Read.All` |
| Entra ID devices | `Device.Read.All` |
| Defender for Endpoint machines | `Machine.Read` (delegated) or `Machine.Read.All` (application) |

Sign-in uses a device code or a certificate. There is no client-secret flow, and Quorum never requests a write scope. If one collector fails, you still get a report from the sources that worked, with the failure stated.

No app registration yet? Export the three inventories once (Graph Explorer or an existing script) and evaluate the files offline:

```powershell
Invoke-Quorum -IntunePath .\managedDevices.json -EntraPath .\devices.json -DefenderPath .\machines.json -Redact
```

## Sharing reports safely

`-Redact` replaces hostnames, display names, UPNs and IP addresses with tokens that stay stable within a run (`host-…`, `user-…`, `ip-…`) and writes a `Quorum-RedactionMap.json`. Share the report and keep the map local, because anyone with the map can reverse the report.

For scheduled runs, `-FailOnSeverity high` writes the report first and then throws if anything is at that severity or worse.

## What it deliberately doesn't do

Microsoft365DSC, Maester, CIPP, the built-in reports and Conditional Access What-If already cover configuration drift, policy tests and CA simulation. Quorum only cross-checks device state. It doesn't change devices or the tenant, and it doesn't phone home.

The roadmap adds MECM as a fourth read-only source through the AdminService. Code, rules and the roadmap are at [Abbasi26/endpoint-quorum](https://github.com/Abbasi26/endpoint-quorum) (MIT).
