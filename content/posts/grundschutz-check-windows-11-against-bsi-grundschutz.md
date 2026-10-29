---
title: "GrundschutzCheck: checking Windows 11 against BSI IT-Grundschutz without guessing"
date: 2026-10-29T08:10:00+01:00
description: "A read-only PowerShell checker with 63 Windows 11 client checks, each tied to a Kompendium requirement or a Grundschutz++ control, with mapping confidence, documented exceptions and baseline comparison."
tags: ["bsi", "grundschutz", "windows-11", "powershell", "compliance"]
repo: "grundschutz-check"
image: "/images/grundschutz-check.png"
---

A Windows 11 client often has diagnostic data left at "Required" (`AllowTelemetry = 1`). The BSI Hilfsmittel for Windows clients, under SYS.2.2.3.A4, recommends `0 - Security` where that value is honored, and Grundschutz++ KONF.2.4 says to turn off telemetry services that aren't needed.

So the check is easy. Being honest about the citation is the hard part. BSI hasn't published a Windows 11 settings matrix, the Hilfsmittel values come from Windows 10, and Microsoft doesn't honor value 0 on every edition. [GrundschutzCheck](https://github.com/Abbasi26/grundschutz-check) scores the setting as `GS-W11-010`, cites both documents, and marks the Windows 11 carry-over as `unverified` instead of pretending otherwise.

## Try it, even on Linux

The demo uses bundled fictional evidence and doesn't read the local machine:

```bash
pwsh -NoProfile -File ./Invoke-GrundschutzCheck.ps1 -Demo -OutputDirectory ./grundschutzcheck-out
```

Exit code 1 is expected, because the fictional host fails checks on purpose. You get a self-contained HTML report plus Markdown, JSON and CSV.

On a real Windows 11 client (Windows PowerShell 5.1 or PowerShell 7):

```powershell
Import-Module .\GrundschutzCheck.psd1
Invoke-GrundschutzCheck -OutputDirectory .\grundschutzcheck-out
```

## Two separate kinds of uncertainty

Each of the 63 checks in `catalog/windows11-client.json` carries two independent flags:

- **`mapping`** is `verified` or `unverified` for the link between a published requirement and the setting. For example, BER.6.5 (password age) conflicts with the Hilfsmittel, so it stays unverified.
- **`windows11_carryover`** is `unverified` on every check, because the values come from the Hilfsmittel for Windows 10 21H2 and later, not from a Windows 11 matrix.

The sources are named and dated: Hilfsmittel Windows-Clients 1.0 (20 February 2025), IT-Grundschutz-Kompendium Edition 2023, and the Grundschutz++ resolved catalog of 2026-09-24. The contributing guide has one hard rule: don't invent requirement ids.

Some Hilfsmittel rows are deliberately *not* scored. SmartScreen, for example, is a data-leaves-the-organisation trade-off, not a single safe value. A pass is an indicator for that setting, not proof that a whole BSI requirement is met. The report says so.

## Exceptions are decisions, not silence

Grundschutz expects a deviation to be a documented decision. Put them in a JSON file:

```json
{ "exceptions": [
  { "check_id": "GS-W11-004",
    "reason": "Passwords do not expire by policy; length and breach screening are enforced instead.",
    "owner": "IT security officer", "expires": "2027-06-30", "ticket": "RISK-0001" } ] }
```

With `-ExceptionPath`, a failing check covered by a current exception shows up as `accepted` with its reason, owner and expiry, and it doesn't set exit code 1. An expired exception doesn't hide anything: the check stays `fail`. An exception never hides an `error`, and one for a check that now passes is listed as no longer needed.

## Ratchet instead of big-bang

Keep last month's JSON and pass it as a baseline:

```bash
pwsh -NoProfile -File ./Invoke-GrundschutzCheck.ps1 -BaselinePath ./last-month/grundschutzcheck-report.json -FailOnRegression
```

The report gains a "Changes since baseline" section: newly failing, fixed and still failing checks. It warns when the baseline comes from a different host fingerprint. With `-FailOnRegression`, the exit code is 1 only when something *newly* fails, so you can roll the catalog out to an existing fleet and tighten from there.

## A detail for German Windows

Audit policy checks run `auditpol /get /subcategory:{GUID} /r` and parse the CSV by column. A German display name such as "Anmelden" or an inclusion setting such as "Erfolg und Fehler" therefore doesn't turn into a false error. If the GUID query fails, a de-DE fallback table takes over, and a subcategory that is still missing becomes `error`, not `fail`.

## Read-only, and honest about it

Live collection reads HKLM, runs `secedit /export` and `auditpol /get` as a local administrator, and calls cmdlets such as `Get-MpPreference` and `Get-BitLockerVolume`. The secedit export is deleted after parsing. The host name is stored only as a SHA-256 fingerprint. Remediation is text (a Group Policy path, an Intune Settings Catalog note and a registry snippet), and nothing is applied. If a cmdlet throws, that check becomes `error`, not `fail`.

## How it relates to HardeningKitty

[HardeningKitty](https://github.com/scipag/HardeningKitty) is a general Windows hardening engine, and its BSI SiSyPHuS lists target Windows 10 1809 / LTSC 2019. GrundschutzCheck doesn't replace it. It's a smaller checker whose catalog is tied to Kompendium requirement ids and confirmed Grundschutz++ control ids, with an evidence report.

There's also an Ansible example for remote, read-only collection. Code and catalog are at [Abbasi26/grundschutz-check](https://github.com/Abbasi26/grundschutz-check) (MIT).
