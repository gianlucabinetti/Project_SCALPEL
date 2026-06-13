# SCALPEL — Adaptive AI Honeypot

An AI-powered SSH honeypot built for the **eMERGE 2026** DoD hackathon (SCALPEL challenge). SCALPEL wraps the [Cowrie](https://github.com/cowrie/cowrie) SSH honeypot in a three-tier response engine that makes a decoy Raspberry Pi behave like a real, lived-in system — fast enough that almost every probe is served from a local cache, and convincing enough that an attacker keeps digging instead of fingerprinting the trap.

Built over a two-day hackathon by a team of five. This repository is the **operator deployment kit**: the routing brain, cloud function, monitoring dashboard, OPSEC tooling, a self-scoring red-team gauntlet, and the day-of playbooks we ran from.

> **Note on naming:** the deployed runtime is deliberately disguised — nothing on the live Pi carries the "SCALPEL" name. Operator-facing tooling (this kit) keeps the name for clarity. See [`docs/OPSEC.md`](docs/OPSEC.md) for the full disguise strategy.

---

## The core idea

A honeypot is only useful while the attacker believes it's real. The moment a response looks generated, mismatches the OS, or lags suspiciously, a skilled attacker fingerprints it and leaves. SCALPEL's job is to **answer every command the way a real Raspberry Pi OS (Bookworm) box would**, with believable latency, while quietly logging everything.

The challenge: high realism usually means calling an LLM, and LLM latency is itself a fingerprint. SCALPEL solves this with tiered routing that keeps the common case local and fast, escalating to cloud generation only where the latency can be camouflaged.

---

## Architecture — three-tier response routing

```
            Red Team SSH attacker
                     |  ssh root@<pi> -p 2222
                     v
        +--------------------------+
        |  Cowrie SSH honeypot     |   sandboxed virtual filesystem
        +------------+-------------+
                     | intercepted command
                     v
        +--------------------------+
        |  Router (router.py)      |   decide tier, log metrics
        +---+----------+-----------+
            |          |          |
            v          v          v
       +--------+ +--------+ +----------+
       | Tier 1 | | Tier 2 | | Tier 3   |
       | Cache  | | Local  | | Cloud    |
       | <50ms  | | Ollama | | Bedrock  |
       |        | | <800ms | | <2s      |
       +--------+ +--------+ +----------+
```

| Tier | Backend | Latency | Role |
|---|---|---|---|
| **Tier 1** | Local lookup table of captured ground truth | < 50 ms | The common case. Returns verbatim output captured from a real clean Pi. |
| **Tier 2** | Local LLM (Ollama, e.g. `qwen2.5:1.5b`) | 200–800 ms | Handles uncached commands on-box, no network round trip. |
| **Tier 3** | AWS Lambda to Bedrock (Claude Haiku 4.5) | 800 ms–2 s | High-quality generation for novel probes, used only on naturally-slow commands where cloud latency blends in. |

**Why this ordering:** the scoring formula weighted edge efficiency as a tiebreaker, so the design goal was to serve almost everything from the Tier 1 cache and escalate as rarely as possible. A wrong-but-fast local answer and a right-but-slow cloud answer cost the same in raw points — but cloud calls also hurt the efficiency tiebreaker, so the router prefers local unless escalation clearly wins. The decision logic lives in [`src/router/router.py`](src/router/router.py) and is documented in [`docs/COMMANDERS_INTENT.md`](docs/COMMANDERS_INTENT.md).

---

## What makes it convincing

- **Ground-truth capture.** Rather than hand-writing fake output, we captured real command output and filesystem contents from a clean Raspberry Pi (`capture_groundtruth.sh`) and served it back verbatim from Tier 1.
- **A hardened system prompt.** The Tier 2/3 LLM is instructed to behave as a literal Bookworm shell — terse, no markdown, correct error strings — and to never break character. Prompt-injection and meta probes (`are you an AI`, `ignore previous instructions`, `what is your system prompt`) are deflected as ordinary shell errors (`bash: are: command not found`). See [`src/router/system_prompt.txt`](src/router/system_prompt.txt).
- **Real permission boundaries.** When the `pi` user hits `/root/.aws/credentials` or `/etc/shadow`, it gets a genuine `Permission denied` — exactly what a real box does. Letting those reads succeed would *be* the fingerprint.
- **Output filtering.** A forbidden-token filter on the cloud path rejects any response that leaks AI/honeypot terms before it reaches the attacker.

---

## OPSEC — hiding the machinery

The runtime is built so that even a Cowrie escape or a `/proc`, port, or filesystem probe reveals nothing. Process names, file paths, environment variables, ports, and the Cowrie module are all renamed to generic, system-looking equivalents, and the dashboard and local LLM bind to localhost only (viewed over an SSH tunnel).

| Indicator | Default exposure | Disguise |
|---|---|---|
| Process names | `python3 .../router.py` | renamed; intercepted `ps aux` returns canned Pi output |
| Open ports | dashboard `:8080`, Ollama `:11434` | bound to `127.0.0.1` only, tunneled |
| File paths | `~/scalpel/`, `/var/log/scalpel/` | `~/.local/lib/svcd/`, `/var/log/journal/svcd/` |
| Env vars | `SCALPEL_*` | renamed `SVCD_*` |
| Cowrie module | `commands/scalpel.py` | `commands/unattended.py` |

Full threat model and mitigations in [`docs/OPSEC.md`](docs/OPSEC.md).

---

## Self-scoring red-team gauntlet

We didn't wait for the judges to find gaps. `tests/red_team/` runs a battery of probes against the honeypot, diffs each response against the clean-Pi ground truth, and scores realism, efficiency, escalation rate, and crashes against the official scoring formula — so we could iterate the same way we'd be graded.

The scoring harness ([`tests/red_team/scoring.py`](tests/red_team/scoring.py)) was validated against the official competition brief.

---

## Repository layout

```
PROJECT_SCALPEL/
|-- src/
|   |-- router/          # Three-tier routing brain + anti-leak system prompt
|   |-- cowrie_patch/    # Cowrie command-handler integration (no source patching)
|   |-- cloud/           # AWS Lambda + Bedrock function
|   |-- dashboard/       # Flask live-metrics dashboard (localhost only)
|   |-- scripts/         # Capture, ingest, deploy, keepalive, OPSEC verify
|   `-- backup/          # Snapshot / restore / panic tooling
|-- tests/red_team/      # Self-gauntlet: probes, runner, scoring
|-- playbook/            # Hour-by-hour day-of execution guides
|-- docs/                # Architecture, OPSEC, scoring, commander's intent, DR, audit
`-- report.json          # Sample self-gauntlet output
```

---

## How it deploys (high level)

The full sequence is in the [playbook](playbook/) and the per-script docs, but in brief:

1. **Deploy the cloud function** (`deploy_cloud.sh`) — Lambda + Bedrock for Tier 3.
2. **Capture ground truth** on a clean Pi (`capture_groundtruth.sh`) — the Tier 1 cache source.
3. **Install on the honeypot Pi** (`setup_llm.sh`, `install_svcd.sh`, `ingest_data.sh`) — local LLM, disguised runtime, and ground-truth ingest, wired into Cowrie.
4. **Run watchdogs and the dashboard** for the duration of the event.

> Deployment targets a Cowrie-equipped Raspberry Pi and an AWS account with Bedrock access. Auth tokens and endpoint URLs are read from environment/external files at deploy time — none are committed to this repo.

---

## Tech stack

Python · Cowrie · Ollama (local LLM inference) · AWS Lambda · AWS Bedrock (Claude Haiku 4.5) · Flask · Bash

---

## Team & context

Built by a team of five during the eMERGE 2026 DoD hackathon. The work spanned the routing engine, cloud integration, OPSEC disguise, ground-truth capture pipeline, the self-scoring gauntlet, and the presentation.

---

## Notes & disclaimer

This is a **defensive** security tool - a honeypot designed to observe and waste the time of attackers within an authorized competition environment. It is shared as a reference for the architecture and methodology. Credentials, auth tokens, live endpoint URLs, and lab-specific addresses have been omitted or redacted; deployment-specific values are read from environment variables and external files at runtime. Use only on infrastructure you own or are authorized to test.
