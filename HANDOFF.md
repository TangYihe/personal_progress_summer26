# Handoff — where we left off

> Session bookmark. **Current state only** — no history (that lives in the README progress
> log). Update at the end of every working session: what's done, where to pick up next.
>
> Last session: 2026-06-05 (week wrap)

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

1. **B — Mixed-data survey** → [notes/lit-survey-mixed-training.md](notes/lit-survey-mixed-training.md).
   **Done this week:** survey (`2606.00054`), **DexWild** (`2505.07813`), **DexMachina** (`2505.24853`),
   **ManipTrans** (`2503.21860`). **Through-line:** kinematic retargeting fails on dexterous,
   contact-rich tasks → functional retargeting via **residual-RL-on-a-base + contact** — but the strong
   real-world results all lean on **sim → sim-to-real**. **Crystallizing question for us:** *can we get
   the residual-on-base + contact structure working **sim-free**, bootstrapped by our robot data?*
   **Pick up next:** **compile the dexterous-manipulation + RL background reading list** (expand from
   survey §6). Groups: RL + demos (DAPG), **real-world / sim-free RL** (Xu `2212.09902`), human→dex
   transfer (DexMachina / ManipTrans / Park `2501.04169`), sim-to-real dexterity, **RL-post-IL
   (RL-100)**. Then read the sim-free picks first (Park `2501.04169`, Xu `2212.09902`) + RL-100.
   Open threads (tracked in [ledger](notes/decisions-and-caveats.md)): action-space abs-vs-rel;
   training-MSE-as-eval-proxy (C3); egocentric main-obs alignment (+ AprilTag inpainting); human↔robot
   **speed mismatch**; **mixing ratio** (DexWild's **1:2 robot:human** = our starting point).
2. **D — Force & impedance control** — **DEFERRED** (pushed all week; not started) →
   [notes/force-impedance-control/](notes/force-impedance-control/). Inbox 5 works; read order: Adaptive
   Compliance Policy (`2410.09309`) → variable impedance (`2603.14068`) → slow-fast trio (`2605.27886`,
   ImplicitRDP `2512.10946`) → Torque-aware VLA (`2509.07962`). Now also ties to **contact-without-sim**
   (can hand-hardware **joint torque** supply contact signal?). Deserves its own focused block.
3. **A — Next-week experiment plan** — **pending** (for when robot + teleop data arrive): **ACT/DP
   overfit-on-robot-data** check + **human-data replay** check → then derive remaining lit gaps.

## Watch out / open threads

- GPU: awaiting colleague's case-clearance + PSU-label report.
- Awaiting large-scale training-resource details from advisor (~6.4–6.5).
- Design threads accumulating in [decisions-and-caveats.md](notes/decisions-and-caveats.md):
  hand-pose tracking, relative-action space, egocentric main-obs alignment, speed mismatch,
  mixing ratio, RL & sim dependency, vision encoder, contact sensing.
- Queued: dexterous + RL background list (incl. **RL-100**); force-control basics; A experiment plan.
