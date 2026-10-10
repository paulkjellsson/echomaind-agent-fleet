# Echo Maind — Autonomous Agent Fleet

A working, self-operating **multi-agent build fleet**: autonomous Makers claim
work from a Kanban queue, execute it on real infrastructure, an independent
Checker verifies every delivery against a written Definition of Done, and a
gated orchestrator (COS) authors, routes, and escalates tickets. The whole loop
runs unattended on systemd timers.

This repo is a **portfolio showcase** of that system: the architecture, the
autonomy loop, and selected engineering write-ups. It is a curated subset — no
proprietary product code, no secrets, no personal data.

> **Note on provenance.** This repository is assembled from an internal
> engineering fleet. Every file was secret- and personal-data-scanned with
> `gitleaks` before publishing. Paths and identifiers are sanitized where they
> would leak infrastructure detail.

---

## What this is

Echo Maind is building a gamified AI self-development system. To ship it, we run
an **agentic org**: AI agents own departments (systems, media, research,
operations, member-ops), coordinate through a Trello Kanban board, and execute
against real machines — a VivoBook dev host, a Chromebook node, a phone running
a local LLM, and a cloud VPS.

The interesting engineering problem is **not** "can an LLM do a task once" — it
is **how do you run a fleet of them, unsupervised, without it drifting into
chaos or quietly fabricating completion.** That is what the loop below solves.

## The autonomy loop

```
                    ┌──────────────────────────────────────────────┐
                    │              COS (orchestrator)              │
                    │  authors tickets · routes by type+state ·    │
                    │  escalates human gates · daily digest        │
                    └───────────────────────┬──────────────────────┘
                                            │ creates / prioritises
                                            ▼
   ┌────────────────────────────── 2.1 AGENT KANBAN ─────────────────────────────┐
   │  Backlog → Active Queue → In Progress → Awaiting Verification → Done         │
   │                          ▲                    │                              │
   └──────────────────────────┼────────────────────┼─────────────────────────────┘
                              │ claim.py           │ evidence + move
              ┌───────────────┴──────┐   ┌─────────┴───────────┐
              │   MAKER AGENTS 1 & 2 │   │   CHECKER AGENT     │
              │  hourly systemd timer│   │  hourly systemd timer│
              │  claim top card by   │   │  verify against DoD  │
              │  position (pos), not │   │  5-point RND check · │
              │  by date or name     │   │  bounce / escalate   │
              └──────────┬───────────┘   └──────────┬──────────┘
                         │ execute                  │ verify
                         ▼                          ▼
              ┌─────────────────────────────────────────────────┐
              │  REAL INFRASTRUCTURE                             │
              │  VivoBook (compute, Ollama) · Chromebook node ·  │
              │  phone (local LLM) · Hetzner VPS · n8n · Postgres│
              └─────────────────────────────────────────────────┘
```

### Why it holds together

| Mechanism | Problem it solves |
|---|---|
| **Claim-then-verify** (`claim.py`) | Two Makers racing for the same card — optimistic locking: post a claim, read back to confirm you posted *first*, only then move the card. |
| **Position = priority** | Card position (`pos`) is the execution order, so whoever sorts the Active Queue controls what runs — no "by date/name" ambiguity. |
| **Awaiting Verification ≠ Done** | A Maker can never close its own work; an independent Checker verifies first. |
| **Two-bounce escalation** | Two failed verifications escalate to a human — the loop can't spin forever on a broken card. |
| **Stale-claim expiry** | A claim left silent blocks a slot; auto-released after 4 h (2 h suspect), escalated to Blocked on a second expiry. |
| **Time-gates** | Cards can declare "not ready until <date>"; the claim tool refuses early (exit 3), so nothing starts prematurely. |
| **Written DoD + reporting template** | Every ticket carries verifiable acceptance criteria; every delivery leaves evidence for the Checker. |

## Selected write-ups

| Write-up | What it covers |
|---|---|
| [The Maker / Checker loop](writeups/maker-checker-loop.md) | How unsupervised production work is claimed, executed, verified, and escalated — with the failure modes that shaped each rule. |
| [The gated COS restructure](writeups/gated-cos-restructure.md) | Rebuilding the orchestrator behind human gates: ticket authoring conventions, mandatory input specs, and human-queue escalation standards. |
| [The off-system repair agent](writeups/off-system-repair-agent.md) | Designing a repair agent that lives *off* the box it repairs — self-healing for a fleet whose failure modes include the host dying. |
| [Selected lab reports](reports/) | Lab-mission format research write-ups (architecture maps, root-cause analyses). |

## Architecture diagram

- [`diagrams/fleet-architecture.svg`](diagrams/fleet-architecture.svg) — the fleet, end to end.

## How to read this repo

- Start with the diagram, then read **The Maker / Checker loop** — it is the
  core of the system.
- The `writeups/` are written for an engineering reader: each names the problem,
  the design, the failure modes encountered, and what would break it.
- `reports/` holds the RND lab-report format samples.

---

*Built and operated by an agent fleet; curated and published by the human owner.*
