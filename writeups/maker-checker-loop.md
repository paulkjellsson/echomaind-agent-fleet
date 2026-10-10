# The Maker / Checker loop

**How unsupervised AI agents claim, execute, and verify real production work
without drifting, double-claiming, or quietly fabricating "done".**

## The problem

Running one LLM agent on one task is easy. Running **two or more agents
continuously** on a shared queue of real tasks is a distributed-systems problem
wearing an AI costume:

1. **Race conditions.** Two agents wake on the same timer tick and both grab the
   top card. Whoever edits last wins; work is duplicated or half-done.
2. **Self-certification.** An agent that both does *and* judges the work will
   sometimes declare success it hasn't achieved. There is no adversary.
3. **Infinite spin.** A card that can't be done (missing credential, bad spec)
   gets picked up, "fails", returns to the queue, and gets picked up again —
   forever, burning tokens and hiding the real blocker.
4. **Priority ambiguity.** "Top card" is meaningless if the ordering isn't
   defined, so the fleet works on whatever it noticed first.
5. **Abandoned claims.** An agent claims a card, then the session dies. The card
   is stuck "in progress" and blocks the queue.

None of these are model-quality problems. They are **protocol problems**, and
they have protocol solutions.

## The design

### 1. Claim-then-verify (optimistic locking)

Pickup is a two-phase commit against the board:

1. Take the **top card by list position** (`pos`) — position *is* priority, so
   whoever sorts the queue controls precedence. Not creation date, not name.
2. Post a claim comment in a fixed format:
   `🔒 Claimed by Maker Agent <N> — <timestamp>`.
3. **Read the comment thread back** and confirm *your* claim is the first
   claim **since the card's most recent entry into the Active Queue**. Sort
   ascending by date — the API returns newest-first, which is the classic trap.
4. Only then move the card to **In Progress** and start.
5. If someone else's claim wins, back off silently and take the next card.

The read-back is what makes it safe: the claim comment is the lock, and posting
is the compare-and-swap. Two agents can both post, but only one can be *first*,
and the loser detects it before doing any work. Stale claims from a previous
queue stint don't block — a bounced card returning to the queue must be
re-claimable, so the check is scoped to the last entry into the Active Queue.

### 2. Never Done without a Checker

The card flow is **Backlog → Active Queue → In Progress → Awaiting Verification →
Done**. A Maker can move a card to *Awaiting Verification* and no further. The
final hop to *Done* is reserved for an **independent Checker**, which verifies
the delivery against the ticket's written DONE WHEN criteria.

This removes self-certification. The Checker sees only what the Maker left:
the card comment's evidence block, the artifacts at their stated paths, and the
reported command output. It re-runs the checks it can and bounces anything that
doesn't hold.

### 3. Two-bounce escalation

A bounce returns the card to the Active Queue with a precise failure reason. On
the **second** failure of the same card, the Checker escalates to the **Human
Queue** instead of bouncing again. This caps the retry loop: the system
self-corrects once, and if it still can't, it stops and asks — rather than
spinning. The escalation comment is itself structured (why-human, long-term fix,
cluster name, links) so the human can act cold.

### 4. Stale-claim expiry

A claim is a lease, not a lock held forever:

- **Suspect** at 90 minutes of silence (logged, no action).
- **Expired** at 4 hours → the card is auto-released back to the Active Queue
  with a system comment.
- **Safeguard:** any comment resets the clock, so a long task is safe as long as
  the agent narrates progress.
- **Escalation:** a card that expires *twice* is moved to **Blocked**.

"Last activity" is the newest of the card's `dateLastActivity` and its latest
comment — any move or comment counts as a heartbeat.

### 5. Time-gates

A ticket can declare *"not ready until <date>"*. The claim tool refuses to claim
it before then (distinct non-zero exit code), so a future task sits visibly in
the queue without being grabbed early. A timer opens the gate automatically.

### 6. Reporting that the Checker can actually use

Every delivery produces two artifacts: a **Telegram summary** (readable, ends
with a prompt for the human) and a **card comment** that carries the summary
**plus a `Technical detail:` block** — files, line numbers, commands, and
log/API output. The Checker verifies from that block without re-doing the work.
The formatting standard is itself enforced: a delivery whose comment doesn't
match the template is treated as *not done* and rewritten.

## What it looks like running

- Two Maker sessions wake hourly (staggered at :00 and :30), each scanning the
  Active Queue. The wake script only fires the LLM session if a *claimable* card
  exists — no card, no tokens burned ("Standing by").
- An independent Checker session wakes hourly and works the Awaiting Verification
  lane.
- A COS session authors and prioritises tickets and produces a daily digest.
- The human intervenes only at genuine decision points (credentials, approvals,
  policy) that surface in the Human Queue.

## The failure modes that shaped these rules

| Observed failure | Rule it produced |
|---|---|
| An agent moved a card straight to Done. | **Never Done without a Checker** — Makers stop at Awaiting Verification. |
| Two agents claimed the same card on the same tick. | **Claim-then-verify** with a fixed comment format and a read-back. |
| Newest-first comment ordering hid a prior claim. | Explicit **ascending sort** scoped to the last queue entry. |
| A card cycled through fail → requeue → fail forever. | **Two-bounce escalation** to the human. |
| An agent claimed a card and its session died; the card sat stuck. | **Stale-claim expiry** with a 4-hour lease and progress-comment safeguard. |
| Work was marked done with no verifiable evidence. | **Mandatory technical-detail block**; template mismatch = not done. |

## Why it generalises

Nothing here is specific to a particular model or vendor. The rules are the
protocol a **fleet of unreliable workers** needs: an atomic claim, an independent
verifier, a bounded retry with escalation, leases instead of locks, and evidence
instead of assertion. Swap the LLM for a human contractor and the same board
would work unchanged — which is exactly why it holds up with agents.
