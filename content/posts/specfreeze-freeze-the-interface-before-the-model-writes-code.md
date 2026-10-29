---
title: "specfreeze: decide the design first, freeze the interface, then let the model write code"
date: 2026-10-29T08:35:00+01:00
description: "A local Python CLI that asks design decisions as closed questions, freezes them as a signed interface contract, and rejects generated code and repairs that drift from it, plus a static verify gate for any later edit."
tags: ["ai", "llm", "python", "testing", "code-generation"]
repo: "specfreeze"
image: "/images/specfreeze.png"
---

Ask a code model to "split the bill fairly" and it has to invent the policy. For 100 cents and 3 people it might return `[33, 33, 33]` and lose a cent, put the leftover on the last share, or add a `mode` parameter nobody asked for. Every one of those is plausible code, and every one is a design decision you didn't make.

[specfreeze](https://github.com/Abbasi26/specfreeze) makes you make it. It asks the decisions up front as closed questions, freezes the answers as a contract, and refuses implementations and repairs that drift from that contract.

## Closed questions, not a chat

For the bill-splitting example, the questions look like this:

- Where do leftover cents go, the front of the list or the back?
- Does a negative total raise `ValueError`?
- Does `people < 1` raise `ValueError`?

Questions come from a YAML catalog and have three types: `yes_no`, `choice` (2 to 6 options) and `score` (2 to 5 levels). Follow-ups fire on specific answers. A low-confidence answer falls back to the catalog's `conservative` value when one is set. The model's confidence is self-reported, not a calibrated probability, and the docs say that plainly.

The answers freeze `split_cents(total: int, people: int) -> list[int]`, its error behaviour and its examples into `contract.json`, protected by a digest over the module name, constants, signatures, errors, examples and calls.

## A repair loop that can't move the goalposts

Generation is followed by a test-driven repair loop with strict acceptance rules:

- Pytest runs in a temporary directory with `shell=False`.
- A candidate is kept only if the number of failures goes **down** and the number of passes doesn't drop.
- Signature changes, new public functions and reassigned frozen constants are rejected **before** the tests run.

The offline demo replays that story with a recorded provider. The first implementation drops the remainder. The first repair adds `*, mode: str = "front"` and is discarded because the signature moved. The second repair uses integer division, puts the extra cents on the first shares and passes all 10 tests.

```bash
python -m pip install -e ".[dev]"
python -m specfreeze demo --out .specfreeze/demo
```

No API key, no network.

## Gate any edit, from any tool

The contract is useful after generation too. `specfreeze verify` checks any later change to the module, whether it came from you, a teammate, Aider, Copilot or another specfreeze run:

```text
$ specfreeze verify portion.py --contract .specfreeze/demo
3 problem(s):
  [signature] signature deviates from the frozen contract: expected `def format_shares(shares: list[int]) -> str:`, got `def format_shares(shares: list[int], sep: str=',') -> str:`
  [constant] frozen constant `SEPARATOR` has a different value than the contract
  [extra_public] public function `helper` is not in the contract; make it private (`_helper`) or add it to the spec
```

`verify` parses with `ast` and never imports or runs the file. Exit codes are 0 for clean, 1 for drift and 2 for a usage or contract error, and a tampered contract is refused. Commit `contract.json` next to the module and the example GitHub Actions workflow turns it into a pull-request gate.

## Bring your own endpoint

Live runs talk to any OpenAI-compatible `/chat/completions` endpoint. There's no default URL, key or model. Decision, spec and code roles can use different models and endpoints. Keys are redacted from the generation log and the saved prompts. Without `--apply`, an accepted repair lands in `preview/` and `preview.diff`, so nothing outside the run directory changes until you say so.

## When it's worth the overhead

For a one-line change, specfreeze is overhead. It earns its keep when the failure mode is "the model picked a reasonable design you didn't ask for". It sits next to tools such as Aider (in-place edits with a test loop) and GitHub Spec Kit (a spec-driven document process). specfreeze is the checker: once the signature is frozen, a repair that moves it is discarded.

Code and catalogs: [Abbasi26/specfreeze](https://github.com/Abbasi26/specfreeze) (MIT).
