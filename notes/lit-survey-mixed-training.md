# Literature survey — human + robot mixed-data training

> Workstream **B**. ← back to [overview](../README.md).
>
> Last updated: 2026-06-03

---

## Why this matters

We will have a **large amount of human (no-robot) data** plus robot teleop data. The plan is
**two-stage IL**: pretrain on human data → fine-tune on robot teleop (mixed / sequential).
So understanding how recent work combines human + robot data is a **direct need**, not optional.

## Goal

1. Build a list of recent works (academia + industry) that mix human (no-robot) + robot data.
2. For each, pin down the **method**: policy **backbone** · **action head** · **training schedule**
   (mixed vs. sequential vs. staged).
3. Produce **my own notes + a concrete list of options to try** (see [Options to try](#options-to-try)).

---

## Inbox — to read

> Dump candidate works here (title / link / one-line why). Promote to "Notes" once read.

> **Reading priority (set 2026-06-03):** (1) survey for the big picture → (2) GR00T (well-known
> baseline) → (3) hand / dexterous (our focus) → (4) browse the VLA / co-training / latent-action
> tier. Dropped the *humanoid whole-body* and *human-only / data-editing* groups as less relevant
> (if needed later, the survey in tier 1 covers them). ⭐ = top pick within a tier.
>
> URLs verified to exist; data-mixture *ratios* come from abstracts — confirm against the paper
> before quoting numbers.

### 1 · Overview — read first
- ⭐ **"From Human Videos to Robot Manipulation: A Survey"** (2026) — https://arxiv.org/abs/2606.00054 — organizes the field into 4 paradigms (latent actions / world models / explicit 2D / explicit 3D) + a companion GitHub paper list. Big picture; best single source to expand coverage later.

### 2 · Well-known baseline — read next
- ⭐ **GR00T N1** (NVIDIA) — https://arxiv.org/abs/2503.14734 — dual-system VLA (VLM System-2 + diffusion-transformer System-1), pretrained on a robot + human-video + synthetic mixture. Flagship, widely-compared co-training baseline; code available. Follow-ups **N1.5 / N1.7** are release artifacts (N1.7 reportedly adds EgoScale's 20K-hr human video).
- **Emergence of Human→Robot Transfer in VLAs** (PI) — https://www.pi.website/research/human_to_robot — *(mine)* embedding analysis of how human vs. robot data is encoded across pretraining ratios. Good analysis idea for our own development. Read in detail.

### 3 · Hand / dexterous — our focus
- ⭐ **DexWild** (RSS 2025) — https://arxiv.org/abs/2505.07813 — in-the-wild portable hand rig; **co-trains** human + robot; ~4× robot-only in novel envs.
- **EgoScale** (NVIDIA et al., 2026) — https://arxiv.org/abs/2602.16710 — 20.8K hrs action-labeled egocentric human pretrain; scaling law; reportedly feeds GR00T N1.7. Central to our direction.
- **DexUMI** — https://arxiv.org/abs/2505.21864 — hand exoskeleton + inpainting closes the kinematic gap; code available.
- **UniDex** (2026) — https://arxiv.org/abs/2603.22264 — universal dexterous 3D VLA from human video; supports human-robot co-training.

### 4 · VLA / co-training / latent-action — browse
*Pretrain-on-human → finetune-on-robot (sequential / staged):*
- ⭐ **EgoVLA** — https://arxiv.org/abs/2507.12440 — VLA on egocentric human video → IK/retarget → robot finetune. Clean two-stage recipe; matches our direction directly.
- **H-RDT** — https://arxiv.org/abs/2507.23523 — 2B RDT-family DiT w/ flow-matching head; EgoDex human pretrain → cross-embodiment robot finetune.
- **Being-H0** — https://arxiv.org/abs/2507.15597 — large-scale human-hand VLA pretrain w/ part-level motion tokens → robot post-training.
- **RynnVLA-001** (Alibaba, ICRA 2026) — https://arxiv.org/abs/2509.15212 — 12M egocentric video-gen pretrain → VLA finetune (ActionVAE head); claims > π0 / GR00T-N1.5. Code available.
- **VITRA** — https://arxiv.org/abs/2510.21571 — 1M-episode/26M-frame human-hand VLA pretrain → robot finetune.

*Co-training (human + robot as joint demos):*
- ⭐ **EgoMimic** (CoRL 2024) — https://arxiv.org/abs/2410.24221 — co-trains human egocentric (Project Aria) + robot data as equal embodied demos; ACT-style policy. Canonical co-training baseline.
- **MotionTrans** — https://arxiv.org/abs/2509.17759 — weighted human↔robot co-training, 30 tasks, zero-shot on 9.
- **EMMA** — https://arxiv.org/abs/2509.04443 — co-train human *mobile* manip + static robot data. Same group as EgoMimic.
- **EgoMI** — https://arxiv.org/abs/2511.00153 — active-vision + whole-body from egocentric human demos.

*Latent-action / latent-motion pretraining (action-free human video → robot):*
- ⭐ **LAPA** (ICLR 2025) — https://arxiv.org/abs/2410.11758 — VQ-VAE latent actions from action-free (incl. human) video → robot finetune. Seminal latent-action route; code available.
- **Moto** (ICCV 2025 Oral) — https://arxiv.org/abs/2412.04445 — latent motion tokens; GPT pretrain → co-finetune to robot actions. Code available.
- **ViSA-Flow** — https://arxiv.org/abs/2505.01288 — semantic-action-flow prior from human-object video → robot finetune; strong low-data.
- *(Genie — https://arxiv.org/abs/2402.15391 — world model, conceptual origin of unsupervised latent actions; not itself a policy.)*

*Egocentric data (misc):*
- **EgoVerse** — https://egoverse.ai/ — *(mine)* recent egocentric human data work; model part may be trivial, but worth a look.

### Reference
- *Unverified — confirm before citing:* DemoBot (2601.01651), DexMan/DexMachina (2510.08475), Object-centric 3D Motion Field (2506.04227), Gen2Act (2409.16283).
- *Source datasets to know:* EgoDex, PH2D, Ego4D/SSv2, Project Aria captures, EgoScale 20K-hr corpus.

---

## Notes (per paper)

> Template per work. Keep concise and formal.
>
> ### <Title> (<venue/year>) — [link]()
> - **Idea / problem:**
> - **Backbone:**
> - **Action head:**
> - **Training schedule:** (human→robot? mixed? staged? data ratios?)
> - **Force/tactile:** (used? how?)
> - **Key takeaway:**
> - **What to try / relevance to us:**

- _(no papers processed yet)_

---

## Options to try

> Candidate approaches surfaced by the survey — running list.

- _(add candidates as they surface)_

---

## Open questions

- _(park survey-specific uncertainties here)_
