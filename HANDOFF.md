# Handoff — where we left off

> Session bookmark. **Current state only** — no history (that lives in the README progress
> log). Update at the end of every working session: what's done, where to pick up next.
>
> Last session: 2026-06-03

---

## Where we are

Repo structure set up: `README.md` (overview/entry point), `HANDOFF.md` (this file),
`AGENTS.md` (working rules). First `notes/` entry created: [gpu-procurement.md](notes/gpu-procurement.md).

**GPU procurement (workstream C):** PI approved buying a card; leaning **RTX 4090 (24 GB)**
(40-series > 50-series for stable CUDA/library support; 24 GB avoids OOM; matches 师姐's setup).
Probed the workstation via CLI — confirmed fine: **PCIe x16 slot free**, no existing discrete
GPU, i5-14600K + ASUS B760M-AYW support a 4090. Two open gates, both physical:
- **PSU** — needs 850 W+ with 12VHPWR or 3–4× 8-pin. Not a real bottleneck (cheap swap; buy an
  850 W+ ATX 3.0 unit if unsure). Check label + that a new one physically fits the case.
- **Case clearance** — the deciding factor; case looks compact. **Colleague is checking**
  length/width clearance + free slots + PSU label/size.

**Large-scale training compute:** abundant per advisor; details expected ~6.4–6.5 (awaiting).

## Pick up next

GPU parked pending the colleague's physical check — no action needed until that report comes back.

**Reading mode (set 2026-06-03):** *I* read the papers; the agent assists — clarifies questions,
verifies claims, and organizes my insights into the right notes doc. Collaborative, paper-by-paper.

1. **B — Mixed-data survey** (active this session) → [notes/lit-survey-mixed-training.md](notes/lit-survey-mixed-training.md).
   **Survey (`2606.00054`) done** — captured the 4-paradigm taxonomy, set our *adopter* reading
   lens + stance on the field's 3 challenges (C1 low-priority, C2 bounded view-gap, C3 cheap-eval =
   high-priority). Verdict: good map, shallow analysis → **read specific architectures next.**
   **Now reading (2026-06-04):** **DexWild** (`2505.07813`) → **DexMachina** (`2510.08475`) —
   dexterous-hand works, closest to our setting (read to surface the real technical challenges).
   GR00T's human-data *schedule* already answered via agent (co-train, not sequential; pretrain
   ratio unreported) → full GR00T deep-read deferred. **DreamZero parked** (lean VLA route + DP/ACT
   first; see survey note). Two open threads to resolve while reading: **action-space** (abs vs.
   rel) view-robustness, and **training-MSE-as-eval-proxy** (our C3 answer; EgoVerse lead).
2. **D — Force & impedance control** → [notes/force-impedance-control/](notes/force-impedance-control/)
   (`index.md` + `concepts.md`). **Not started.** Inbox seeded (5 works). **Suggested read order:**
   Adaptive Compliance Policy (`2410.09309`) → variable impedance (`2603.14068`) → slow-fast trio
   (`2605.27886`, ImplicitRDP `2512.10946`) → Torque-aware VLA (`2509.07962`). Motivation: our robot
   data records joint torques/forces + supports impedance control → want to try putting force into
   the policy. Less familiar here — budget time.

## Watch out / open threads

- GPU: awaiting colleague's case-clearance + PSU-label report.
- Awaiting large-scale training-resource details from advisor (~6.4–6.5).
