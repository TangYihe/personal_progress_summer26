# Model Training Track — Summer 2026

> **Entry point & living overview.** High-level plan, current status, and milestones for
> the model-training / policy-learning workstream. Kept concise on purpose — details live
> in `notes/` (created as we reach each milestone). For session-level "where we left off",
> see [HANDOFF.md](HANDOFF.md). For how this repo is organized, see [AGENTS.md](AGENTS.md).
>
> Last updated: 2026-06-11

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
| B | **Mixed-data literature survey** — how recent work does human (no-robot) + robot teleop training: backbone · action head · training schedule. Produce my own notes + a list of options to try. ([lit survey](notes/survey/mixed-training.md) · [decisions index](notes/decisions-and-caveats.md) · [human data](notes/design/human-data.md) · [RL options](notes/design/rl-finetuning.md) · [force options](notes/design/force-feedback.md)) | 🟢 |
| D | **Background reading** — force/impedance control, incl. building intuition + whether to put force into the policy ([notes](notes/survey/force-impedance-control/index.md)); ⚪ large-scale/multi-GPU training; ⚪ Physical Intelligence series + RL post-training. | 🟢 / ⚪ |
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

- **2026-06-11** — D (force survey) + VLA architecture. Read **Ego-Pi (`2606.08107`)**
  (PI dexterous humanoid, π₀.₅ backbone, egocentric human co-training): limited relevance to
  our hard problem (focus is semantic compositionality, not dexterous motor skill); portable
  tricks: viewpoint-matched Zed Mini setup, wrist-cam dropout, joint-space retargeting.
  Understood **π₀ / π₀.₅ architecture** (VLM + action expert split; FAST-token pre-training
  in π₀.₅; subtask prediction on VLM side) — confirms TA-VLA's torque → action expert
  placement. Read **FACTR 2 (`2606.12406`)**: NEXT (neural τ_ext estimation for sensorless
  arms), FIRST (contact-phase batch reweighting, self-supervised from τ_ext, directly
  portable); cross-paper confirmation that TA-VLA auxiliary prediction is backbone-agnostic.
  Summarized **FILIC (`2509.17053`)**: EEF wrench via Jacobian inversion > raw joint torque
  (+10–17 pp). [force-feedback.md](notes/design/force-feedback.md) §6 updated; FIRST added
  as Option D. **Next:** slow-fast papers → §8.

- **2026-06-10** — D (force survey continued). Summarized **CATFA (`2509.23075`)** (in-hand
  manipulation, fingertip tactile + torque): not deeply relevant to our setting (sim-to-real,
  dexterous fingertip focus) — deferred; one actionable note: MLP confirmed sufficient for
  low-D torque encoding. Read **FACTR (`2502.17432`)** (ACT-based, external joint torque):
  key findings — raw vs. external torque distinction (many arms expose τ_ext directly via API;
  **action item: check our robot**); force token in ACT encoder alongside vision + joints;
  no history window (contrast with TA-VLA's 10-frame window — now open, try both); curriculum
  training (progressive visual blurring) as an alternative to TA-VLA's auxiliary prediction
  for forcing force-signal utilization — both documented together, TA-VLA approach preferred.
  Created [survey/force-related-works.md](notes/survey/force-related-works.md) as dedicated
  per-paper reading notes file. Updated [design/force-feedback.md](notes/design/force-feedback.md):
  MLP encoding settled; preprocessing §4 restructured; history window re-opened; §6 role-in-policy
  expanded with curriculum option. Deep-dived ACT architecture (CVAE encoder, transformer
  encoder+decoder in the policy, fixed sinusoidal decoder queries, learnable W_Q/W_K/W_V).
  **Next:** TA-VLA backbone (π₀) architecture → slow-fast policy papers → adaptive compliance.

- **2026-06-09** — Planning + housekeeping. Finalized data-arrival plan (replay as pipeline check,
  ACT-first IL baseline, 1:2 co-train default, RL codebase scouting in parallel with IL).
  Surveyed **Xu et al. `2212.09902`**: classifier-based reward from goal images — no demos, no
  manual reward formula; per-milestone binary classifier (goal images vs. live rollouts) as reward
  signal. Reorganized notes into `survey/` + `design/` folders;
  [decisions-and-caveats.md](notes/decisions-and-caveats.md) is now a clean index.
  — D (force survey): reprioritized reading order to practical design-space papers before theory.
  Read **TA-VLA (`2509.07962`)** in full. Built out [design/force-feedback.md](notes/design/force-feedback.md)
  with 9 decision dimensions for integrating joint torque into IL policy (signal source, placement,
  adapter design, preprocessing, temporal structure, auxiliary prediction, which joints, slow-fast,
  controller compat). Key finding: torque belongs in the proprio stream alongside joint positions;
  separate adapter; 10-frame history aggregated to a single MLP token; auxiliary prediction via
  expanding action space (not separate head); compatible with standard position control.
  **Next:** skim remaining force papers for additional design choices → fill force-feedback.md.

- **2026-06-08** — B (RL + rewards deep-dive): fully read **ManipTrans** (method + experiments);
  read **DAPG**; pre-read **RL-100**. Crystallized the sim-free reward path: **demos + sparse
  task-completion labels suffice** (DAPG result); human keypress per episode is a practical real-world
  substitute for sim-privileged dense rewards (RL-100). Documented cross-paper conclusion: both
  ManipTrans and DexMachina require state-based policy + sim-privileged contact info — the main
  structural barrier to sim-free adaptation. Next: plan for when human + robot teleop data arrives.

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
