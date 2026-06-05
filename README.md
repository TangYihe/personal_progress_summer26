# Model Training Track — Summer 2026

> **Entry point & living overview.** High-level plan, current status, and milestones for
> the model-training / policy-learning workstream. Kept concise on purpose — details live
> in `notes/` (created as we reach each milestone). For session-level "where we left off",
> see [HANDOFF.md](HANDOFF.md). For how this repo is organized, see [AGENTS.md](AGENTS.md).
>
> Last updated: 2026-06-05

---

## Context (read this first)

Summer research project (collaborating with the lab team). I **lead the
model-training / policy-learning** part; two labmates (师弟) own the teleoperation side
(hand retargeting + body/teleop on the real robot).

- **Mandate right now: "explore"** — map the design space and get foundations ready.
  *Not* shipping trained models against a deadline yet.
- **Research direction:** two-stage imitation learning — pretrain on **human data
  (no robot embodiment)** → fine-tune on **robot teleop data** (mixed / sequential).
  **Force/tactile sensing** is a core interest. Working hypothesis: the contact-rich final
  task may eventually need **RL post-training** on top of IL.
- **Hard constraints:** no data/hardware on my side yet; workstation has **no GPU**.
  Slow iteration is a *resource* bottleneck, not a *me* bottleneck. Leading solo, with the
  inherent uncertainty + slow loops of training work — that's the nature of the task.

---

## The Plan (workstreams)

Status legend: 🟢 unblocked now · 🟡 needs data/compute · ⚪ if time

| # | Workstream | Status |
|---|-----------|--------|
| C | **Compute conversation** — confirm GPU for workstation, source of training GPUs, infra help. *Gates everything downstream.* | 🟢 |
| B | **Mixed-data literature survey** — how recent work does human (no-robot) + robot teleop training: backbone · action head · training schedule. Produce my own notes + a list of options to try. ([notes](notes/lit-survey-mixed-training.md) · [decisions & caveats](notes/decisions-and-caveats.md)) | 🟢 |
| D | **Background reading** — force/impedance control, incl. building intuition + whether to put force into the policy ([notes](notes/force-impedance-control/index.md)); ⚪ large-scale/multi-GPU training; ⚪ Physical Intelligence series + RL post-training. | 🟢 / ⚪ |
| A | **Data-quality validation** — replay success-rate sanity check (cheap, on real robot) → then DP / ACT to validate data quality. | 🟡 |

**Sequencing:** do **C first** (it gates training/iteration) → **B** (highest-leverage prep
while waiting on data) → **D reading** in parallel. Once data exists: **A** (replay → DP/ACT).

> Reframe to hold onto: I am *not* "stuck with nothing to do until data arrives." C + B + D
> are fully unblocked now and set up everything for when data + compute land.

---

## Current Status

**Phase:** Setup / explore. Repo just created to track progress across lab + personal machines.
**Up next:** see [HANDOFF.md](HANDOFF.md).

---

## Compute & Resources

- [ ] **Local GPU for prototyping** — *in progress.* PI approved buying a card; leaning
  **RTX 4090 (24 GB)**. Compatibility check underway (case fit + PSU are the open gates).
  See [notes/gpu-procurement.md](notes/gpu-procurement.md).
- [ ] **Training-GPU source (large-scale)** — *looks resolved in principle.* Advisor says
  resources will be **abundant**; details coming in a couple of days (~2026-06-04/05).
- [ ] Infra help / collaborator — *TBD*

---

## Open Questions

*(Park uncertainties here so they don't loop in my head.)*

- _(none logged yet)_

---

## Milestones / Progress Log

Concise, dated, newest first. Big moments only — session detail lives in HANDOFF.

- **2026-06-05** — B (dexterous deep-dive): read **DexWild**, **DexMachina**, **ManipTrans**.
  Through-line: **kinematic retargeting fails on dexterous contact-rich tasks** → functional
  retargeting via **residual-RL-on-a-base + contact**, but strong real-world results all rely on
  **sim → sim-to-real**. Our crystallizing question: *can we do residual-on-base + contact **sim-free**,
  bootstrapped by robot data?* Logged many adopt-items + caveats to the ledger; fixed DexMachina's
  arxiv id (`2505.24853`, not `2510.08475`=DexMan). **Next:** dexterous + RL background list (RL-100).
  Force-control basics deferred.
- **2026-06-04** — B: GR00T/DreamZero human-data schedule clarified (both **co-train, not
  sequential**; GR00T omits its pretrain mixing ratio) → **parked DreamZero**, leaning **VLA route**
  (+ DP/ACT first). Started **DexWild** (closest-match work): AprilTag-glove hand tracking, relative
  wrist-pose actions, matched wrist cams. Created the [decisions & caveats ledger](notes/decisions-and-caveats.md);
  surfaced **egocentric main-obs alignment** as a likely challenge for us.
- **2026-06-03** — B (mixed-data survey): processed the overview survey ([2606.00054](https://arxiv.org/abs/2606.00054))
  — captured the 4-paradigm taxonomy, set our *adopter* reading lens + stance on the field's three
  challenges. Verdict: good map, shallow analysis → read specific architectures next (GR00T N1, then
  DexMachina — dexterous-hand works prioritized). Set the reading workflow: I read, agent organizes.
- **2026-06-02** — Compute conversation (C) with advisor: large-scale training resources
  will be **abundant** (details in a couple of days); local prototyping GPU to be sourced via
  ops/management colleagues. Kickoff sync; tasked to lead model training ("explore" phase).
  No data/hardware yet, workstation has no GPU. Drafted the plan and set up this repo.
