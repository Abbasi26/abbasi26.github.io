---
title: "Secure App Factory: Win32 packaging for Intune where every byte is pinned"
date: 2026-10-29T08:30:00+01:00
description: "A PowerShell 7 module and GitHub Actions template that builds a Windows package from a pinned vendor installer: SHA-256 and signer-thumbprint pins, PSAppDeployToolkit 4.1, .intunewin, a CycloneDX SBOM and local provenance, with uploads off by default."
tags: ["intune", "powershell", "supply-chain", "sbom", "packaging"]
repo: "secure-app-factory"
image: "/images/secure-app-factory.png"
---

A packaging script that downloads "the latest 7-Zip" wraps a different file the next time the vendor ships a release. That's a reproducibility problem, and it's also a supply-chain problem. The official 7-Zip 26.04 x64 executable isn't Authenticode-signed, so a signature check can't tell a reviewed build from a swapped one. The control that still works is a hash pin.

[Secure App Factory](https://github.com/Abbasi26/secure-app-factory) is a PowerShell 7 module plus a GitHub Actions template that builds a Windows package from exactly one reviewed installer, and stops the build when anything about that installer changes.

## Two kinds of pins

**Unsigned installer, hash pin.** `apps/7zip/app.yaml` pins `7z2604-x64.exe` to the SHA-256 that GitHub publishes for that release asset. The build fails when the downloaded bytes don't match.

**Signed installer, signer pin.** A signature that's merely "valid" isn't enough, because the publisher renews its leaf certificate and a different certificate is a different trust decision. Notepad++ 8.9.8.1 is pinned to a valid Authenticode signature whose subject contains `NOTEPAD++` *and* whose leaf thumbprint is `1E8E0D13B608BA908572C1A129FAEC5D228DF8A2`. A new thumbprint fails the build until someone reviews it and updates the manifest. A new file hash must be updated in both the manifest and `lock.json`.

```yaml
trust:
  algorithm: SHA256
  value: 26f3bcced788fadbac4e7e577b430ea2ce6060cf3bcad993ef4612ecd7d743da
  signature:
    required: true
    subjectContains: NOTEPAD++
    thumbprint: 1E8E0D13B608BA908572C1A129FAEC5D228DF8A2
```

The manifest dialect is deliberately strict: unknown keys, tabs and duplicate keys are rejected. An optional Evergreen source has to match the pinned version exactly, so a newer release is a failure, not a silent upgrade.

## Try it offline

```powershell
pwsh -NoLogo -File ./demo/Invoke-Demo.ps1
```

The demo uses local fixtures. It doesn't download, install or call Intune, and it writes a `REPORT.md` per app.

## What a real build produces

```powershell
pwsh -NoLogo -File ./scripts/Invoke-Package.ps1 -App notepadpp -Destination ./out/notepadpp `
  -IncludeToolkit -DownloadContentPrepTool -SmokeTest -Force
```

Every switch that downloads or writes is opt-in:

- `-IncludeToolkit` downloads the pinned **PSAppDeployToolkit 4.1.8** template, checks its SHA-256 and generates a v4 deployment script.
- `-DownloadContentPrepTool` downloads the pinned **Win32 Content Prep Tool 1.8.7**, checks `IntuneWinAppUtil.exe` and writes the `.intunewin`.
- `-SmokeTest` installs silently, checks the detection rule, then uninstalls (on Windows, elevated).

The result folder contains `REPORT.md`, `trust.json`, a **CycloneDX 1.6** SBOM and a `provenance.json` recording the builder, the installer digest and the invocation. To be precise: that provenance file isn't an in-toto statement or a SLSA attestation, and the docs say so.

## Review the catalog on every pull request

`Test-SafAppCatalog` reads every `apps/<id>` folder offline and reports by severity:

- **Error**: a missing SHA-256 pin, a manifest and lock that disagree, schema errors
- **Warning**: a URL that doesn't contain the version, a signature rule without a leaf thumbprint, a fixed drive letter
- **Info**: the hash pin is the only control because the installer is unsigned

The catalog check job in the workflow writes these findings to the job summary on every pull request.

Adding an app doesn't require a download either. `New-SafAppManifest` takes the SHA-256 the vendor publishes (or a file you already reviewed), validates the manifest against the schema before writing, and supports `-WhatIf`.

## Upload is off, and never assigns

Intune upload happens only with `-Upload` (or the workflow's `upload` input) and the three `SAF_INTUNE_*` secrets. It creates a `win32LobApp`, uploads the encrypted content and commits it. It never assigns the app to users, groups or devices. If a secret is missing, no Graph call is made. Use a dedicated app registration, because `DeviceManagementApps.ReadWrite.All` is a powerful permission.

## Where it fits

This is a template for one reviewed installer at a time, not a catalog. Winget-AutoUpdate updates apps on the device. Evergreen finds current download URLs. IntuneWin32App manages the full Win32 app lifecycle, including assignments and supersedence. Commercial catalogs cover thousands of apps with support. Secure App Factory trades coverage for visibility: every pin, download and Graph call is in the repository.

Code: [Abbasi26/secure-app-factory](https://github.com/Abbasi26/secure-app-factory) (MIT. PSAppDeployToolkit remains LGPL-3.0-or-later and isn't stored in the repo).
