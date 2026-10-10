# The gated COS restructure

**Rebuilding the orchestrator behind human gates: turning a free-form agent into
a convention-enforcing ticket author, and making "escalate to a human" a
first-class, structured outcome.**

## The problem

The Chief-of-Staff (COS) agent is the orchestrator: it turns goals into tickets,
prioritises the queue, routes work to the right list, and escalates what it
can't resolve. Early on, it was doing the *reasoning* reasonably well and the
*bookkeeping* badly:

- **Under-specified tickets.** Tickets arrived with a title and a vague body.
  Makers then spent their first ten minutes hunting for paths, IDs, and links
  the author already knew — or worse, guessed, and produced work against the
  wrong artifact.
- **Ambiguous escalation.** "Needs a decision" cards landed in the human's lap
  with no statement of *why* a human was required, no proposed fix, and no
  links. The human had to reconstruct intent from the comment thread.
- **Silent scope drift.** The orchestrator would sometimes resolve a genuine
  human decision (a policy call, an account ownership question) by *making up*
  an answer and creating an execution ticket — instead of routing it to the
  human who actually held the authority.

The restructure's goal: make the orchestrator **rigorously conventional at the
authoring boundary**, and make every human touching-point a **complete,
self-contained work order**.

## The design

### 1. Mandatory ticket structure

Every ticket the orchestrator authors must contain these sections, in order:

1. **Context** — why the ticket exists (the problem, symptom, or goal).
2. **Input specification** — a required block listing the exact artifacts a
   Maker needs: file paths, URIs, IDs, document links, credential *references*.
3. **Task** — what to do, concretely.
4. **Scope** — what's in and out (optional).
5. **DONE WHEN** — verifiable acceptance criteria.

The **input specification** is the load-bearing change. It is *always* present —
`n/a` is valid only for genuinely generative tickets ("write a new README",
"enable a timer") and must be written explicitly as `n/a` with a reason. The
rule is **fail-closed for the Maker**: a ticket with no input-spec block and no
obvious generative nature is treated as a blocker to *ask about*, not a puzzle
to guess at.

> **Why a separate mandatory block rather than "be specific"?** Observed
> pattern: authors omit concrete artifacts because at authoring time they know
> them implicitly. Making the block a required, always-present section moves the
> cost to the cheapest place — the moment of authoring — and makes its absence
> mechanically detectable.

### 2. Deterministic ticket IDs

Ticket IDs follow `AREA.YWWW.SEQ` (e.g. `SYS.641.027` = systems, ISO year 2026,
week 41, ticket 27). The rule that matters: **never hand-format the ID**. A
deterministic helper recomputes the ISO week from the *clock on every call* and
returns the next free ID for the current week.

This fixed a real defect where a session crossing a week boundary carried the
old week's prefix into new tickets, so the ID said week 40 on a week-41 ticket.
Recomputing from the clock — never from memory or context — makes the ID
self-correcting at the boundary.

### 3. Prefix = type, list = state

Routing separates **what a thing is** from **where it is**:

- The **prefix** encodes the *area/type* of work (systems, media, research,
  operations, member-ops).
- The **list** encodes its *state* (queued, in progress, awaiting verification,
  human-blocked, done).

The subtlety: some rules are *state* rules, not *type* rules — e.g. "cards
awaiting a human go to the Human Queue" applies to *any* prefix, because it's
about state, not area. Conflating the two produced mis-routed cards.

To keep it honest, a **consistency lint** checks that every declared type/area
appears consistently across *four* sources: the router's prefix table, the
ticket-ID area map, the machine-readable card-type registry, and the
human-readable card-types table. A prefix missing from any one of the four fails
the lint, so the registry can't silently drift.

### 4. Human-queue cards as structured work orders

A card moved to the Human Queue must be openable *cold* — no thread reading —
and be fully actionable. The standard:

- A **`## Human checklist`** on the card description, using GitHub task-list
  syntax (`- [ ]`), each item actionable in **under 60 seconds**, one action per
  line, naming the exact page/field/file to click.
- A comment with four mandatory sections, in order:
  1. **Why is this blocked by human?** — the *specific* missing thing (a
     credential, an approval, an account, a policy call), not "needs a decision".
  2. **Long-term solution** — the systemic fix that removes this *class* of card
     permanently, not a one-off.
  3. **Bigger problem / cluster** — a one-line cluster label
     (`missing credentials`, `no OAuth token`, `manual approval gate`, …).
  4. **Links** — every URL needed to resolve it, one per line, each annotated
     with why it's needed.

### 5. Route human decisions directly — don't bounce

The orchestrator follows a routing rule: if the human reports a problem that is
*clearly a human decision* rather than a Maker execution task, move it to the
Human Queue **directly**. Do not create a bounce cycle to discover that a human
was needed. Bounces are reserved for genuine Maker *execution* failures.

This is the inverse of the common failure where an orchestrator, lacking
authority, retries a thing that structurally cannot be retried — burning cycles
and burying the real blocker.

## What it changed

- **Makers stopped hunting.** Tickets became self-contained; the first minutes
  of every pickup go to work, not archaeology.
- **Human touching-points became cheap.** A Human Queue card is now a checklist
  someone can complete in under a minute per step, with the *why* and the links
  already attached.
- **The registry stopped drifting.** The four-source lint makes an inconsistency
  a failing check, not a surprise weeks later.
- **Fewer invented answers.** Human decisions route to humans; the orchestrator
  authors and prioritises but does not adjudicate.

## Why "gated"

The whole restructure is built around **gates** — points where authority is
explicitly held by a human and the system *stops* rather than improvises:
credentials only the human can mint, approvals only the human can give, policy
only the human can set. The orchestrator's job isn't to remove those gates; it's
to make arriving at them **clean, complete, and unambiguous** — so the human's
decision is a click, not an investigation.
