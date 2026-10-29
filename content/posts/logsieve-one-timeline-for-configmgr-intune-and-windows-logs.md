---
title: "logsieve: one timeline for ConfigMgr, Intune and Windows logs, with the names taken out"
date: 2026-10-29T08:15:00+01:00
description: "A local CLI that merges CMTrace, IME, EVTX and journald logs onto one timeline, explains known error codes with sources, and pseudonymises hostnames, users and addresses reversibly before anything leaves the machine."
tags: ["intune", "configmgr", "windows", "privacy", "python"]
repo: "logsieve"
image: "/images/logsieve.png"
---

A client can't install an update. The useful lines are spread across `WUAHandler.log`, `AppEnforce.log`, `IntuneManagementExtension.log` and a Windows Update event, and those same lines name the laptop, the user and the WSUS server. Pasting them into a ticket, a forum or a chat model means pasting all of that too.

[logsieve](https://github.com/Abbasi26/logsieve) puts those lines on one timeline, matches the error codes it knows, and writes a report in which `laptop-07.northwind.example` has become `LS_HOST_2`. The mapping stays in an encrypted file on your machine, and `logsieve reveal` puts the names back into whatever answer you get.

## Try it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
logsieve demo
```

The demo needs no network and no real logs. It writes a report, a generated `.evtx` and a vault into `./logsieve-demo-out/`. The output starts like this:

```text
suspected cause: Windows Update cannot resolve the server name
pattern: wu-name-not-resolved  confidence: high
timeline:
  2026-10-21 08:01:05 UTC  error  notes.log  ERROR Windows Update failed with error 0x8024402C while contacting LS_HOST_1 from LS_HOST_2 at LS_IPV4_1.
  2026-10-21 08:01:40 UTC  error  collector.jsonl  lookup LS_HOST_1: no such host
report leak check: clean
llm: off
```

## What it reads

- **CMTrace / ConfigMgr** logs (`<![LOG[…]LOG]!>`), including UTF-16, with severity from the type attribute
- **IME** logs: `IntuneManagementExtension.log`, `AgentExecutor.log` and friends
- **`.evtx`** through python-evtx, plus exported event XML and CSV
- **journald** JSON lines
- Anything else as plain text, one line per event

Files are only read. When you point it at a directory, it skips its own reports, vault and key files, so a second run doesn't ingest the first run's output.

```bash
logsieve diagnose C:\Windows\CCM\Logs I:\Intune\Logs --output C:\Temp\logsieve-out --vault C:\Temp\logsieve-vault.json
```

## Error codes, with sources and honesty

```bash
logsieve explain 0x8024402C
logsieve explain -2016345060     # the signed decimal form IME likes to log
```

`explain` prints the code in hex, signed and unsigned form with its HRESULT facility, then every matching pattern with an explanation, what to check, public sources, and whether the meaning is marked uncertain. For `0x8007xxxx` it also tries the bare Win32 code. Exit code 1 means "not in the library", which isn't the same as "harmless".

The pattern library is YAML, and every entry needs a public `https://` source and a fictional `self_test` line. `logsieve patterns --check ./my-patterns` validates your own patterns (id collisions, missing sources, failing self-tests) and exits 1 on problems, so it fits in CI.

## Pseudonymisation you can reverse

Hostnames, UPNs and emails, SIDs, IP addresses, GUIDs and profile paths become stable tokens such as `LS_HOST_2` or `LS_EMAIL_1`. By default the mapping lives only in memory. `--vault` writes it encrypted to disk. If `LOGSIEVE_VAULT_PASSPHRASE` is set, the key is derived with PBKDF2-HMAC-SHA256. Otherwise a key file with mode `0600` is written next to the vault. The passphrase is never accepted as a command-line flag.

```bash
logsieve reveal --vault C:\Temp\logsieve-vault.json --input answer.md
```

## A model, if you want one, still local

Nothing is sent unless you pass `--llm`. The default endpoint is Ollama's OpenAI-compatible API on `127.0.0.1:11434`, and any `/chat/completions` URL works.

```bash
logsieve diagnose .\Logs --show-payload > payload.json
```

`--show-payload` prints the exact request body without opening a socket. Before anything is sent, logsieve checks that the original identifiers are absent from the payload. If one slipped through, it refuses, doesn't print the identifier, and exits 2. The suspected cause in the report is a ranking hint, and where one code may be a symptom of another, the report says that the link isn't proof.

## Where it stops

logsieve doesn't tail live channels, doesn't call `wevtutil` and doesn't change the endpoint. Its HRESULT library is a curated subset, not a complete catalogue. It's not a log viewer either. [CMTrace Open](https://github.com/adamgell/cmtraceopen) is a good one, and logsieve adds the cross-log timeline, reversible tokens and the optional explanation on top.

Code and pattern library: [Abbasi26/logsieve](https://github.com/Abbasi26/logsieve) (MIT).
