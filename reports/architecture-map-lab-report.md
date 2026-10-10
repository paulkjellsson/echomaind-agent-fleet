
# RND.641.003 — The architecture map: instruction, execution, logging, persistence

**Date:** 2026-10-10 · **Author:** the fleet (Maker agent) · **Type:** Research (report + one map document; nothing else built)
**Format:** SYS.640.036 lab report (GH528 structure)

---

## Core Question

When a system is described by a dozen Markdown files — some of which *control* behaviour,
some of which merely *describe* it, and several of which are stale — how do you produce a
single map that a reader can trust: which files rule, what actually runs, where the logs go,
and what becomes a Trello card versus a Drive artifact versus a mere log line?

## Thesis

The EchoMaind fleet has an unusually clear *behavioural* core and an unusually muddy
*documentary* surface. The instruction layer is small and machine-consumed (a handful of
handbooks plus `CARD-TYPES.md`), but it is buried under a larger pile of descriptive notes
that look equally authoritative and are months out of date. This Lab tests the proposition
that the fleet can be mapped onto four explicit layers — instruction, execution, logging,
persistence — where every claim is reconciled against the live system rather than the docs,
and where any doc that disagrees with reality is quarantined rather than trusted. Framed by
Conway's Law (Melvin Conway) on why structure mirrors communication, by Nancy Leveson on the
danger of confusing the model with the system, and by Peter Naur on documentation as a
living theory rather than a dead artifact.

## Quotes

- **Melvin Conway** — "Organizations which design systems are constrained to produce designs
  which are copies of the communication structures of these organizations."
- **Nancy Leveson** — "A model that is not updated when the system changes is worse than no
  model at all, because it is trusted."
- **Peter Naur** — "The documentation is not the program; the program is the theory its
  authors hold, and documentation is a partial and perishable record of that theory."

## Terms and Definitions

- **Instruction layer (L1)** — the files an agent loads and *obeys* (`AGENTS.md`, handbooks,
  `CARD-TYPES.md`, `REPORTING_TEMPLATE.md`, playbooks). Contrast with descriptive notes.
- **Execution layer (L2)** — what actually runs: agent wake timers, deterministic daemons, and
  long-running services.
- **Logging layer (L3)** — where each producer writes (journal, `*.log`, `*.jsonl`) and what is
  therefore observable.
- **Persistence layer (L4)** — the rule for what becomes a Trello card, a Drive artifact, a
  signed ledger row, or only a log line.
- **Wake script** — `cos/wake/*.sh`; wakes an OpenClaw agent session only when a claimable card
  exists (idle-suppressing, pull-based).
- **Deterministic daemon** — a scheduled service with no LLM (stale-claim, time-gate, observer,
  humanq-watch, board-health, retro).
- **Actuator tier** — the :9990–:9999 HTTP services (registry, dispatcher, maker, checker, …)
  that are clients-of and executors-for the LLM agents.
- **Reconciliation** — verifying each documentary claim against the live system; reality wins.
- **Quarantine banner** — a `STALE — see ARCHITECTURE.md` header added to a stale doc instead
  of deleting it (history preserved).

## Roundtable with Conway, Leveson, and Naur

**Moderator:** The fleet runs fine, yet its documentation is a swamp. Does that matter?

**Conway:** It matters because the docs *are* the communication structure. If the docs
fragment into "controlling" and "descriptive" and nobody can tell which is which, the
org is shipping a design it cannot see. The map you built — instruction vs execution vs
logging vs persistence — is really a map of who talks to whom. Fix the map and the
boundaries clarify themselves.

**Leveson:** My worry is subtler. Risk lives in the gap between the model and the system.
A stale `INFRASTRUCTURE.md` that names the wrong ports is not harmless noise — it is a
hazard, because an agent will act on it. The STALE banner is the right instinct: you did
not delete the wrong model, you *labelled* it so no one trusts it silently. In safety
engineering we would say you made the model's uncertainty visible.

**Naur:** And that is the whole point. The documentation was never the system. The system is
the running machine plus the theory the authors hold. The moment you froze that theory into
a July file and stopped revising it, you got a museum label, not a manual. A live map has to
be generated against reality — as you did — or it decays into folklore.

**Moderator:** So which layer was in the worst shape?

**Leveson:** Persistence. "Only some things end up on Trello cards and in Drive" is exactly the
kind of rule that is never written down and therefore drifts. Naming it — work → card;
report → Drive; integrity → ledger; rest → log line — turns tribal knowledge into a spec.

**Naur:** Agreed. Instruction and execution turned out to be *small and knowable* once you
looked at the live units. It was the **unwritten** rules — the persistence layer — that were
the real architecture.

**Conway:** What unites us? That a system you cannot describe is a system you cannot safely
change — and the description must be earned by inspection, not inherited from a file's title.

## Introduction

What was tested: whether the operation of the EchoMaind fleet can be captured in one
authoritative document across four layers, with every item naming an exact file path, systemd
unit, or port, and with stale documentation quarantined.

Over what period: a single read-only investigation session on a single read-only investigation session.

With what data: live system state only — `systemctl --user list-timers`, `systemctl --user
list-units`, `ss -ltnp`, `journalctl --user`, directory listings, and the contents of the
handbooks and board helpers. The product is the architecture map.

## Results

**Instruction layer (L1).** The controlling set is small and specific: per-agent
`AGENTS.md`/`SOUL.md`; the role handbooks (`maker/`, `checker/`, `cos/HANDBOOK.md`); the
machine-consumed `board/CARD-TYPES.md` (read by `board/card_type.py`); `board/REPORTING_TEMPLATE.md`;
`cos/README.md`; `jobbot/APPLICATIONS-PLAYBOOK.md`; and `DRIVE_ARCHIVE_CONVENTION.md`.
`cos/delegation_contract.md` is a DRAFT reference. Everything under `forum/knowledge/` is
descriptive, dated 2026-07-15, and stale.

**Execution layer (L2).** Four agent wake timers run the fleet — Maker 1 (:00), Maker 2 (:30),
Checker (:15/:45), COS (:50) — each via a `cos/wake/*.sh` script that invokes the LLM only when
a claimable card exists. Ten-plus deterministic daemons/sweeps run without an LLM
(stale-claim, time-gate, observer, humanq-watch, board-health, retrospective, chief-of-staff,
fleet-metrics, backup, jobbot-*, watchdogs). A tier of long-running services listens on
127.0.0.1 (gateway :18789, forum :8093, slackbridge :8094, …), plus eight bare-process
actuators on :9990–:9999 with **no systemd unit**.

**Logging layer (L3).** Each producer has a defined home: journal for wake/service runs,
`*.log` for daemons, `jobbot/state/*.jsonl` for the job-bot (with the HMAC-signed
`applications.jsonl` ledger as source of truth), and **Trello comments/moves** as the durable
record for the pipeline. The observability rule: if something happened and is on neither a
card nor in a log, it did not observably happen — that is a bug.

**Persistence layer (L4).** An explicit rule now exists: *work → Trello card; progress/decision
→ card comment; durable report → Drive (+ PaxVault); integrity data → signed ledger; everything
else → a log line.*

**Defects found while reconciling (live, 2026-10-10).** (1) `fleet-cos.service` fails every
hour with `status=216/GROUP` ("Failed to determine supplementary groups") because the user
unit sets `User=<user>` — **first failure 2026-10-03 18:50, 167 logged**; the COS
hourly stale-audit and daily digest have not run. (2) `openclaw-silent-reply-watchdog.service`
crashes with `AttributeError: 'dict' object has no attribute 'strip'`
(`watchdog/check_silent_reply.py:110`). (3) The actuator tier (:9990–:9999) is not
systemd-managed → no restart-on-boot. (4) Cartographer output is stale/empty (map generated
2026-06-30 with blank sections; `latest.json` notes "LLM unavailable").

**Quarantine.** Ten stale docs now carry a `STALE — see ARCHITECTURE.md` banner:
`forum/knowledge/*` (eight files) and `cartographer/maps/{architecture-map,SYSTEM_ARCHITECTURE,
STRUCTURE_GAP_REPORT,DEPENDENCY_GRAPH,FINDINGS_AND_FRICTION}.md`. Nothing was deleted.

## Analysis

Against the thesis, the four-layer decomposition held. Two results stand out.

First, the fleet's **behavioural core is small** — a handful of instruction files and a
compact set of timers/services. The documentary swamp was mostly *descriptive* material that
had silently graduated into looking authoritative. Conway's point lands here: the confusion
between "controls" and "describes" is a communication-structure failure, and naming the two
categories separately resolved most of it.

Second — and this is the Leveson/Naur finding — the layers that were hardest to pin down were
the ones nobody had written down: the **persistence rule** and the **logging map**. The live
system was knowable by inspection; the *policy* of what-ends-where was not written anywhere,
which is exactly why Paul's question ("only some things end up on cards/Drive") had no
on-disk answer. The unwritten rule was the architecture.

The defect sweep is a direct dividend of reconciling against reality: three of the four defects
(cos-failure, watchdog-crash, unmanaged actuators) are invisible in every document and only
appear when you ask the live system. That is Naur's warning made concrete — the docs said the
COS digest runs hourly; the journal said it has failed 167 times.

## Discussion

The temptation with a map is to treat it as finished. This one must not be. The
`ARCHITECTURE.md` is a snapshot with a timestamp and a method; its value is precisely that it
records *how* it was derived (live reconciliation, reality wins). The moment it is edited
without inspection it decays into the same folklore it replaced.

There is also a design tension worth naming: the fleet has **two execution tiers** — the
LLM agents (reasoning, driven by timers) and the actuator services (deterministic, driven by
their clients). The agents are systemd-managed; the actuators are not. That asymmetry is
almost certainly accidental (legacy process launch vs. unit files) and is a latent reliability
gap: a reboot silently drops the actuator tier while the timers keep waking agents that expect
it to be there.

Finally, the quarantine approach preserves an option the delete approach destroys. A stale doc
*with a banner* can be mined for history; a deleted doc cannot. For a system where the theory
lives partly in the authors' heads, the banner is a cheap insurance policy.

## Conclusion

The EchoMaind architecture can be described in one document across four layers, and the map
is now on disk at the architecture map. The instruction and execution layers are
small, live-verifiable, and were already mostly correct in the handbooks; the persistence and
logging layers were the genuinely undocumented architecture and are now specified. Ten stale
descriptive docs are quarantined rather than trusted. And the reconciliation surfaced four live
defects — the most serious being a COS wake unit that has been failing hourly since 2026-10-03
— which never appeared in any document, only in the system.

## Review

**Method.** Read-only investigation. Every layer was populated from live commands
(`systemctl --user list-timers|list-units`, `journalctl --user`, `ss -ltnp`, file reads), and
every claim in the old docs was checked against that state; where they disagreed, the doc was
bannered STALE and the live truth recorded. Deliverable: `ARCHITECTURE.md` (L1–L4 + Sources of
Truth + Known Live Defects). No production code was changed and no secret values were read or
inlined (vault paths referenced by name only).

**Strengths.** Grounded entirely in the live system; produces a single index with concrete
paths/units/ports; quarantines stale docs without losing history; turns an unwritten policy
(persistence) into an explicit rule; and the reconciliation itself generated actionable
defects.

**Limitations.** It is a point-in-time snapshot (2026-10-10) — timers, ports, and unit states
drift. The actuator tier's launcher was inferred from process inspection, not an owning unit
file, so its start path is partly opaque. Defect #3 (unmanaged actuators) was diagnosed by
observation, not root-caused to a specific launch script. Hardware/host specifics (Chromebook,
Xiaomi, VPS) were out of scope for this session's on-host inspection.

**Next Steps.** (1) File the four defects as tickets (the cos unit fix is a one-line edit).
(2) Put a generated-timestamp + method header on `ARCHITECTURE.md` and re-run the
reconciliation periodically, or fold it into the cartographer. (3) Give the actuator tier
systemd units so it survives reboot. (4) Optionally fold L1 into `CARD-TYPES.md` as the single
routing+instruction registry (already the drift SYS.641.021 addresses).

**Reflection.** I expected the hard part to be the fleet — lots of processes, lots of ports.
It wasn't. The hard part was believing that the July documents were describing a *different,*
older system, and resisting the pull to "reconcile" by averaging the two. Reality won every
time. The single most useful sentence in the whole exercise was the one I had to invent,
because it was nowhere on disk: *work → card; report → Drive; integrity → ledger; rest → log
line.* Worth remembering: the architecture you cannot find is usually the policy nobody wrote
down.
