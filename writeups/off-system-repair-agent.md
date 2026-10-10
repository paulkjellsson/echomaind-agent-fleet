# The off-system repair agent

**Designing self-healing for a fleet whose failure modes include "the machine
that runs the healer is dead".**

## The problem

A fleet that runs itself on a single always-on box has an obvious, load-bearing
dependency: **that box**. If the VivoBook freezes, panics, or powers off, every
agent on it stops — including any agent whose job is to notice and fix problems.
You cannot put the doctor inside the patient.

The repair-agent ticket is exactly that: an agent that lives **off** the box it
repairs, so it can act when the box can't act for itself.

## The design

The repair agent runs on a **separate node** (a Chromebook) and, on a schedule,
reaches *into* the repaired host to check health and attempt recovery. It never
depends on the host's own agent stack to do its job. Three questions drove the
design.

### 1. Can the host recover *itself* from the failures that don't kill the OS?

Before building an external fixer, the host was given its own first line of
self-recovery, so the repair agent only has to handle what self-recovery can't:

- **Hardware watchdog.** A kernel watchdog device (the AMD `sp5100_tco` TCO
  timer) is bound and *fed by systemd* with a 1-minute timeout
  (`RuntimeWatchdogSec`). If systemd deadlocks and stops pinging, the hardware
  reboots the box. This catches a **total userspace freeze** — a hang that never
  produces a panic.
- **Panic-reboot.** `kernel.panic=10` and `kernel.panic_on_oops=1` are persisted
  via sysctl. A kernel panic — or an oops that would otherwise leave the box
  half-dead — reboots automatically after 10 seconds. This catches a **panic**,
  which the watchdog alone wouldn't.
- **Suspend-wake.** A Wi-Fi-only laptop has no wired NIC, so classic
  Wake-on-LAN is impossible. The design instead uses **Wake-on-Wireless-LAN**
  (magic packet over the Intel AX200) plus an **RTC alarm** to wake from
  suspend.

The two mechanisms cover **different failure modes** and deliberately overlap:
watchdog for freezes, panic-reboot for panics, and Wi-Fi wake for scheduled or
remote resume from suspend.

### 2. What happens after a full power-off — the honest limit

Here the design records a **negative result** as first-class:

> A full shutdown on this hardware **cannot** be reversed remotely. There is no
> wired NIC (so no Wake-on-LAN) and no USB-Ethernet, and the BIOS exposes no
> OS-visible power-on setting. WoWLAN works from **suspend (S3) only**, never
> from power-off.

Documenting the limit is the point. The repair agent's playbook now *knows* that
a powered-off host is a human visit, not a retry — so it doesn't burn cycles
sending magic packets into the void. A repair agent that doesn't know its own
reach is worse than none: it reports "recovery attempted" for a machine that
was never reachable.

### 3. What does the external agent actually do?

Given the boundary above, the off-system agent's job is precisely the set of
failures that survive on-box self-recovery:

- **Detect silence.** Poll the host over the mesh network (Tailscale). If it
  stops responding, that's the trigger — no reliance on the host's own agents.
- **Classify the failure.** Distinguish "suspend" (recoverable by WoWLAN/RTC)
  from "powered off" (not remotely recoverable) from "frozen but network-alive"
  (recoverable by `sysrq` / service restart) from "hung the kernel" (recoverable
  by the watchdog, so just wait — and *verify* it came back).
- **Intervene at the right layer.** Send a magic packet for suspend; wait out
  the watchdog for a freeze; SSH in and restart the agent stack for a
  service-level failure.
- **Escalate the unreachable.** When the only fix is physical, produce a
  complete human work order (what to do, why, where) rather than a retry.

## Design lessons

| Lesson | Why it matters |
|---|---|
| **Put the healer off the patient.** | A self-healing fleet whose healer shares the failure domain isn't self-healing. |
| **Layer the recovery to the failure mode.** | Watchdog (freeze) ≠ panic-reboot (panic) ≠ WoWLAN/RTC (suspend) ≠ human (power-off). Each is a different mechanism. |
| **Record negative results as design inputs.** | "Remote power-on is impossible here" is as valuable as any positive capability — it stops the agent from pretending. |
| **Recovery must be verified, not assumed.** | "I sent the magic packet" is not "the box came back". The agent confirms reachability after every intervention. |
| **Know when to hand off.** | The end state of an un-fixable failure is a *clean human handoff*, not another automated attempt. |

## Relationship to the rest of the fleet

The repair agent is the **outer loop** of the fleet's reliability model:

- **Inner loop:** the Maker/Checker protocol keeps *work* correct.
- **Middle loop:** the hosts self-recover from freezes and panics.
- **Outer loop:** the off-system repair agent watches the hosts themselves and,
  when the failure is beyond automation, produces a human work order.

Each loop only escalates what it genuinely can't handle — which is the same
discipline the Maker/Checker loop applies to tickets, raised one level to the
infrastructure.
