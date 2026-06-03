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

**Active focus = D (force & impedance control)** — chosen first for uninterrupted conceptual learning.

1. **D — Force & impedance control** → [notes/force-impedance-control/](notes/force-impedance-control/)
   (`index.md` + `concepts.md`). Papers inbox seeded (5 works). **Suggested read order:** Adaptive
   Compliance Policy (`2410.09309`) → variable impedance (`2603.14068`) → slow-fast trio (`2605.27886`,
   ImplicitRDP `2512.10946`) → Torque-aware VLA (`2509.07962`). Dump rough understanding into
   `concepts.md` as I go. Motivation: our robot data records joint torques/forces + supports impedance
   control → want to try putting force into the policy. I'm less familiar here — budget time.
2. **B — Mixed-data survey** (parked, ready) → [notes/lit-survey-mixed-training.md](notes/lit-survey-mixed-training.md).
   Inbox prioritized into 4 tiers: survey (`2606.00054`) → GR00T N1 → hand/dexterous → browse VLA.
   Good for lighter, modular reading sessions between D blocks.

## Watch out / open threads

- GPU: awaiting colleague's case-clearance + PSU-label report.
- Awaiting large-scale training-resource details from advisor (~6.4–6.5).
