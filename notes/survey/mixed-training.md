# Literature survey — human + robot mixed-data training

> Workstream **B**. ← back to [overview](../../README.md).
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

## Reading lens (our stance)

> We are choosing a **SOTA model / training framework to *use*** on our human + robot mixture
> data, and collecting **engineering caveats / tricks** — *not* hunting for research novelty.
> Read to decide *what to adopt* and *what to watch out for*.

What to extract while reading:

- **Mental map** — which of the 4 paradigms (see survey note below) fits *our* setting:
  same-task human + robot, roughly matched viewpoint, dexterous hands. Use the taxonomy to
  **shortlist a framework**, not to catalogue everything.
- **Adoptability** — code / datasets available, maturity, and compute realism (4090
  prototyping → large-scale later). Filter for what we can actually run.
- **Representation × schedule** — *what crosses the gap* (→ **action head**) and *how training
  is staged*: co-train vs. pretrain→finetune (sequential) vs. action-free latent pretrain,
  incl. mixing ratios / loss weighting. These are the concrete **knobs we'll set**.
- **Our two watch-items** (from the survey's three challenges — see note): (a) handling
  **non-perfectly-aligned views** (a smaller gap than in-the-wild); (b) **early offline signals
  for policy quality** (loss / proxies), since real-world eval is costly and we have no sim.
- **Engineering tricks & caveats** — log these to the running ledger:
  [decisions-and-caveats.md](../decisions-and-caveats.md) (data-mixing ratios, augmentation, eval
  proxies, schedule, the action-vs-video representation fork, …).
- **Force / tactile fit** — our core interest; note where force could attach to a given
  framework (links to [D](force-impedance-control/index.md)). Likely sparse in human-video work.

---

## Inbox — to read

> Dump candidate works here (title / link / one-line why). Promote to "Notes" once read.

> **Reading priority (set 2026-06-03):** (1) survey for the big picture → (2) GR00T (well-known
> baseline) → (3) hand / dexterous (our focus) → (4) browse the VLA / co-training / latent-action
> tier. Dropped the *humanoid whole-body* and *human-only / data-editing* groups as less relevant
> (if needed later, the survey in tier 1 covers them). ⭐ = top pick within a tier.
>
> **Update (2026-06-03, post-survey):** bump **dexterous-hand** manipulation works (GR00T,
> DexMachina — tier 3 / reference) up the priority — hand motion is a harder problem than gripper
> open/close, and dex hands are our focus. Still scan **gripper** works for shared tricks (the two
> end-effectors overlap).
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
- ⭐ **DexMachina** — https://arxiv.org/abs/2505.24853 — *(bumped 2026-06-03; id corrected 2026-06-05)* **functional retargeting** of non-trivial human manipulation onto **dexterous (multi-finger) hands**, via **RL** with a virtual-object-control curriculum. Hits a likely **bottleneck for us** — crossing the human→robot-hand gap for *very* dexterous manipulation — and previews an **RL route** we may need. *(NB: `2510.08475` is a **different** paper — "DexMan" — don't confuse.)*

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

### 5 · World action models — separate line from VLAs
- **DreamZero** — *(deprioritized 2026-06-04 — see Notes)* world action model; supervised on frame-to-frame transformations rather than explicit actions. Representative of a line competing with VLAs for leveraging diverse pretraining video. Parked: we lean VLA route + DP/ACT first.
- **Cosmos (NVIDIA)** — *deprioritized.* Skimmed; sits more on the multimodal large model side than the policy / WAM side — less directly relevant for our backbone choice.

### 6 · Dexterous RL & human-skill transfer (from DexMachina's related work, queued 2026-06-05)
- ★ **Learning to Transfer Human Hand Skills for Robot Manipulations** (Park et al., 2025) —
  https://arxiv.org/abs/2501.04169 — learns a **joint motion manifold** (human-hand ↔ robot-hand ↔
  object); synthetic triplets → **functional retargeting** respecting robot-object interaction;
  **real-world, no sim**. *Relevance: sim-free, data-driven alternative to DexMachina's RL retargeting.*
- ★ **Dexterous Manipulation from Images: Autonomous Real-World RL via Substep Guidance** (Xu et al.,
  2022) — https://arxiv.org/abs/2212.09902 — vision-based **real-world** RL on a 4-finger hand; tasks
  defined by **image examples**; autonomous practice, **no sim, no manual reward engineering**.
  *Relevance: dexterous RL under our exact no-sim/no-privileged-reward constraint.*
- **DAPG — Learning Complex Dexterous Manipulation with Deep RL and Demonstrations** (Rajeswaran et
  al., RSS 2018) — https://arxiv.org/abs/1709.10087 — foundational **RL + demos**: human demos cut
  sample complexity to ~hours on a 24-DoF hand. *Sim.* *Relevance: the mechanism behind "bootstrap RL
  with our robot demos."* **Read 2026-06-08 → see Notes.**
- **RL-100** (2025) — https://arxiv.org/abs/2510.14830 — IL→RL within the diffusion denoising process
  (PPO at each denoising step); real-world, gripper-only, 100% success on 8 tasks. *Relevance: IL→RL
  recipe for diffusion-policy backbone; not dexterous.* **Pre-read 2026-06-08 → see Notes.**
- **VideoDex — Learning Dexterity from Internet Videos** (Shaw, Bahl, Pathak, CoRL 2022) —
  https://arxiv.org/abs/2212.04498 — extracts visual/action/physical **priors from human video** for a
  real dexterous robot. *Relevance: human-video → dexterous priors (internet video; lower priority).*
- ★ **Object-Centric Dexterous Manipulation from Human Motion Data** (Chen et al., 2024) —
  https://arxiv.org/abs/2411.04005 — **hierarchical**: high-level generative model (human hand mocap)
  synthesizes **wrist trajectories** to object goals; low-level **RL** drives fingers. **Sim→real,
  bimanual.** *Relevance: object-centric (functional) + high/low split echoing slow-fast; bimanual
  sim→real — needs sim.*
- **ManipTrans: Efficient Dexterous Bimanual Manipulation Transfer via Residual Learning** (Li et al.,
  CVPR 2025) — https://arxiv.org/abs/2503.21860 — pretrain a generalist **trajectory imitator**, then a
  **residual module** constrained by interaction dynamics; releases **DexManipNet** (3.3K episodes:
  pen-capping, bottle-unscrewing). **Sim-trained → sim-to-real** (real bimanual demos, qualitative).
  **Read 2026-06-05 → see Notes.** *Relevance: residual-on-imitator + contact; not sim-free.*

### Reference
- *Unverified — confirm before citing:* DemoBot (2601.01651), DexMan (2510.08475 — a **separate** paper; DexMachina is `2505.24853`. Not yet read), Object-centric 3D Motion Field (2506.04227), Gen2Act (2409.16283).
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

### Ego-Pi (`2606.08107`) — [link](https://arxiv.org/abs/2606.08107)  *(in progress — reading 2026-06-11)*

- **Idea / problem:** Learn manipulation from co-training on egocentric human demos + robot
  teleop data, using a dexterous five-finger humanoid robot.
- **Backbone:** π₀.₅ (Physical Intelligence). Fine-tuned, not trained from scratch.
- **Action head:** _(to fill)_
- **Training schedule:** **co-training at 50-50** human:robot per batch (not sequential).
  Authors report the policy is **not sensitive to this ratio** — but it remains a design knob
  to tune. Compare: DexWild uses 1:2 robot:human. Worth tracking across works.
- **Wrist camera dropout (40%):** during robot data training, the wrist camera input is dropped
  with **40% probability**. Purpose: human demos have no wrist camera; dropout forces the policy
  to operate without it, bridging the human↔robot observation gap. Without this, the policy
  could over-rely on wrist imagery and fail to transfer from human data. Same principle as
  FACTR's visual blurring — deliberate input masking to prevent modality collapse.
- **Action space — two-token design:** 58D total (29D per hand: 3 pos + 6 rot + 20 fingers).
  Split into two tokens (L/R, interleaved temporally) because **π₀.₅ action tokens are capped
  at 32D** — concatenating both hands (58D) would exceed the limit and break the pretrained
  projection layer. This is an architectural constraint, not an independence argument.
  Effect: action horizon halved from 50 → 25 steps.
- **Cross-embodiment hand representation — joints, not fingertips:** they retarget human hand
  data to **robot joint space** directly (via Manus gloves or Hamer, a vision-based hand
  skeleton estimator). Intermediate representation = **joint angles**, not fingertip positions
  (contrast with DexWild/DexPilot fingertip retargeting). This aligns with our Manus-glove
  setting. Hardcoded offsets are applied post-retargeting — most significant on the **thumb**
  and **MCP of pinky** — likely task-calibrated for their specific hand hardware.
  *For us: the joint-space representation choice is relevant and portable; the specific offsets
  are probably not — use our own calibration.*
- **Visual alignment strategy — hand skeleton overlay:** they overlay a per-finger-colored hand
  skeleton on both human and robot video frames. Stated goal: improve visual interpretability
  of hand pose across embodiments. No explicit viewpoint alignment or egocentric-gap correction
  described. Implied assumption: π₀.₅'s diverse pretraining absorbs background/perspective
  variance; the skeleton overlay is the primary cross-embodiment visual bridge.
- **Camera setup — resolved:** both human and robot use a **Zed Mini**. Robot: head-mounted.
  Human: table-mounted during demo collection. The Quest in the figure is likely for hand
  tracking / Manus integration, not visual capture. From the paper's visualizations the two
  viewpoints appear **well-aligned in angle** — this is almost certainly a deliberate setup
  choice, not incidental. *Our read: careful viewpoint matching at data-collection time is
  likely a key enabler of transfer, not something π₀.₅ pretraining absorbs for free.
  Directly relevant to how we design our human data collection setup.*
- **Scope (verified):** focus is on **high-level task semantics and skill composition** — correct
  subtask ordering, instruction following — not low-level dexterity. Experiments and analysis
  confirm this: grasping and placement are treated as easy/solved; success/failure analysis is
  at the semantic level. Tasks appear designed so that low-level motor skill is not the bottleneck.
  *For us: our hard problem is the opposite — low-level dexterous contact skill is the bottleneck.
  Ego-Pi's human-data benefit likely doesn't transfer directly to our setting.*
- **Force / tactile:** _(to fill)_
- **Results:** tomato sorting 37/40 (vs. 16/40 baseline); packaging 9/10 (vs. 1/10); boxing
  14/15 with subtask generation (vs. 4/15).
- **What to try / relevance to us:**
  - 50-50 co-train ratio — compare to our current 1:2 robot:human default (DexWild).
  - Egocentric camera without wrist cam on human side: partially validates our setup. But
    execution-time wrist cam dependence confirms the egocentric alignment concern in our ledger.
  - Two-token action design is worth watching if we adopt π₀/π₀.₅ as backbone.
  - π₀.₅ is the current SOTA in the π₀ lineage — relevant if we go the VLA route.
- **Status:** done (2026-06-11). Limited direct relevance — engineering tricks (viewpoint
  matching, wrist cam dropout, joint-space retargeting) are portable; the core human-data
  benefit (semantic compositionality) is not our bottleneck.

---

### DexMachina — [link](https://arxiv.org/abs/2505.24853)  *(in progress — method read 2026-06-05)*

- **Core distinction — functional vs. kinematic retargeting:**
  - *Kinematic:* mimic the **human hand motion** — doable **offline**, no object interaction.
  - *Functional:* make the **manipulated object follow the same state change** — requires a **learned
    policy interacting with the object**. Harder, but it's what actually matters for manipulation.
- **Bottlenecks (dexterous functional retargeting):**
  1. **High hand DoF** → hard exploration for RL.
  2. **Contact-rich** → needs stable, precise hand motion.
  3. **Embodiment gap** → direct retargeting **fails functionally** (motion copied, task not achieved).
- **Relies heavily on a simulator.** Their RL curriculum needs:
  1. **Virtual object control** — early on the object is *forced* to follow the desired trajectory, so
     the hand policy can focus on motion-mimicking without being derailed by early failures.
  2. **Privileged sim info** (e.g. **contact** detection) feeds the contact reward.
- **Reward — 4 dense terms** (verified 2026-06-05): `r = λ_task·r_task + λ_imi·r_imi + λ_bc·r_bc +
  λ_con·r_con` —
  1. **task** `r_task`: object **pose + articulation** tracks the demo.
  2. **imitation** `r_imi`: fingertip / hand-link **keypoint** tracking.
  3. **behavior-cloning** `r_bc`: **joint-angle** tracking of the retargeted joints. *(the term easy to
     miss — motion is split into keypoint-space `r_imi` **and** joint-space `r_bc`.)*
  4. **contact** `r_con`: hand-object contact matches the demo (distance-based, validity-checked).
  *(The **virtual object controller** is a **dynamics / curriculum** mechanism — object has 6+1 virtual
  PD-controlled joints whose gains **decay** as the policy improves — **not** a reward term.)*
- **Hybrid action outputs** (verified 2026-06-05): base `Q` = **offline kinematic retargeting** (robot
  hand ↔ human MANO; replayed in sim to remove penetration). **Wrist (6-DoF) = base `Q` + scaled policy
  residual**; **fingers = absolute** (policy mapped to full joint range `[ℓ,u]`; `Q` *unused* for
  fingers). Constrains the action space → better learning efficiency. *(Intuition: wrist gross
  trajectory is well-retargeted → just refine; fingers need full freedom to find functional contact.)*
- **Force / tactile:** contact is central (via sim-privileged info). _(detail on full read)_
- **Our thoughts — how this maps to us** (we have **no sim**, no virtual-object-control, no privileged info):
  1. *Bootstrap substitute:* our **privilege = robot data on the same task** → could bootstrap the
     policy better than cold-start RL. Could even ask teleoperators to record **recovery behaviors** in
     robot data to aid RL.
  2. *Contact without sim:* explore (a) **infer contact from visual obs** (vision-based RL?),
     (b) how **real-world RL works design / check rewards**, (c) **our hand hardware signals** — e.g.
     **per-joint torque** readings (→ ties to [D / force](force-impedance-control/index.md)).
  3. *Action parameterization for us:* reuse DexMachina's **hybrid** split but **swap the base** —
     **wrist = IL-policy prediction + RL residual**, fingers = absolute joints. (= residual RL on an
     **IL base**; cf. DAPG / RL-100.) Promising option to try.
- **Follow-ups to read (next batch — real-world RL):** **RL-100** (IL-bootstrapped policy that
  improves during deployment) + **hand / dexterous RL** works. → see HANDOFF.
- **Key takeaway:** **direct kinematic retargeting fails on dexterous, contact-rich tasks** → this is
  the core motivation for **functional retargeting via RL** + the virtual-object curriculum. Experiments
  are light. Their close baseline **ManipTrans** is near our setting (real-world dexterous) → read next.
- **Status:** **done (2026-06-05).**

### ManipTrans (CVPR 2025) — [link](https://arxiv.org/abs/2503.21860)  *(read 2026-06-05; DexMachina's baseline)*

- **How it achieves dexterous bimanual manipulation — two RL/PPO stages (in sim):**
  1. **Generalist trajectory imitator** — pretrained on **hand-only** data (hand-motion datasets +
     synthetic interpolation, **no objects**); imitates human hand motion (wrist 6-DoF + MANO finger
     keypoints + velocities) on the robot hand. Reward = wrist + finger + smoothness → **mitigates the
     morphology gap**.
  2. **Residual module** — learns **Δa on top of the imitator's action** (`a = a_imitator + Δa_residual`),
     **constrained by interaction dynamics** (contact force, object tracking, gravity/friction).
  - *Why decompose:* separates **motion imitation** (morphology) from **physics/contact** → shrinks the
    RL action space → tractable on high-DoF contact-rich tasks. *(Same residual-on-base pattern as
    DexMachina's hybrid actions and our residual-on-IL idea.)*
- **Contact/force:** **sim contact force** used in **observation + reward + termination**; real hands add
  **tactile sensors**. *(Training-time contact is sim-privileged.)*
- **Sim → real:** trained in **Isaac Gym** (4096 envs, PPO; curriculum gravity 0→on, friction high→normal).
  **Real deploy** on **2× Realman arms + Inspire Hands (w/ tactile)** via **fingertip-alignment fitting**
  (12-DoF sim → 6-DoF real). Real results are **qualitative** (toothpaste-cap opening; "more on website")
  — **no quantitative real-world success rates**; marker/bottle tasks live in the **sim** dataset.
- **Human data:** MANO from video (OakInk-V2, FAVOR, GRAB, ARCTIC); retarget via **21 manual keypoints**.
  **DexManipNet**: 3.3K episodes / 1.34M frames / 61 tasks (~600 bimanual).
- **Speed mismatch:** explicitly **does NOT enforce temporal alignment** — "real robots cannot always
  operate as quickly as human hands." *(Corroborates our temporal-alignment caveat.)*
- **Action space — sim vs. real:**
  - *In sim:* wrist = **6-DoF wrench** (force + torque applied to rigid body in Isaac Gym, same as
    UniDexGrasp [121]/UniDexGrasp++ [111]); fingers = **joint position targets** (PD control). No
    explicit conversion from reference MoCap poses to wrenches — the RL policy **learns** what wrench
    achieves the desired pose tracking (reward = SE(3) pose error). A pose-target wrist baseline exists
    in their appendix and is explicitly contrasted ("PD control for wrist poses rather than 6-DoF force,
    as is done in ManipTrans").
  - *On real hardware:* **no wrench control**. Sim trajectory replayed via (a) **fingertip-alignment
    fitting** — optimize 6-DoF real joint angles to match 12-DoF sim fingertip positions; (b) **IK** to
    align arm flange with wrist pose `w_d`. Effectively position-control replay, no strict temporal
    alignment.
- **State-based policy — key sim-dependence caveat (2026-06-08):** both stages use **fully privileged
  sim state** as observations: explicit object pose + velocity, object mesh (BPS encoding), hand-object
  keypoint distances, contact force `C`. None of these are available in the real world without extra
  instrumentation. This is **another supporting reason why existing works are challenging to adapt
  sim-free**: the policy architecture itself is designed around sim-privileged inputs. Real-world
  deployment requires replacing these with visual observations (e.g. end-to-end visuomotor), which is
  a non-trivial design shift. → ties to our vision-based policy direction and the open contact-from-vision
  thread.
- **Why RL for Stage 1 rather than IL:** Stage 1 reference = MoCap **pose** sequences, not **action**
  sequences (no wrench labels exist). RL with pose-tracking reward bypasses the need for action labels.
  If wrist were position-controlled (their appendix baseline), direct BC on retargeted joint angles would
  be feasible — DexWild does exactly this (fingertip retargeting → IK → BC/ACT).
- **Relevance / reality check:** the **residual-on-imitator + contact** structure is directly relevant
  (matches our residual-on-IL idea). **BUT sim-trained** → its real-world dexterity is **sim-to-real, not
  sim-free**; like DexMachina it does **not** escape our **no-sim** constraint. Real demos are limited.
- **Cross-paper conclusions (read alongside DexMachina, 2026-06-08):**
  1. **Kinematic retargeting is insufficient for high-dexterity contact-rich bimanual tasks.** Both
     ManipTrans and DexMachina show that pure kinematic retargeting yields very low success rates; the
     Retarget+Residual baseline is competitive only on simple tasks. *We need to verify this empirically
     with our own data when available.*
  2. **Shared pattern: demo bootstrap → RL refinement.** Both works use demonstrations to seed a base
     policy (imitator or kinematic), then RL to add contact-precision. This is the core structural idea
     we are adapting (with our robot teleop data as the demo source and real-world RL replacing sim RL).
  3. **State-based policy + sim-privileged contact rewards is the bottleneck for sim-free adaptation.**
     Both works depend on explicit object state, mesh, hand-object distances, and contact force as policy
     inputs and reward signals — none available without sim or heavy instrumentation. For real-world
     deployment we need to extract task-progress / contact signals from visual observations instead.
     *Promising finding from DAPG + RL-100 (see entries below): with demos, sparse task-completion
     rewards may suffice — reducing or eliminating the need for dense contact-based reward engineering.*
- **Status:** fully read 2026-06-08.

### DAPG (RSS 2018) — [link](https://arxiv.org/abs/1709.10087)  *(read 2026-06-08)*

- **Problem:** model-free RL alone cannot practically solve high-DoF dexterous manipulation (24-DoF
  ADROIT hand) — pure RL needs ~100 hours equivalent; converged policies are brittle and unnatural.
- **Method — two-stage IL → RL with decaying demo gradient:**
  1. **BC pretraining:** maximize `Σ log π(a|s)` over demo data → gives a good RL initialization.
     BC policy alone usually fails (distribution shift), but bootstraps exploration efficiently.
  2. **Augmented NPG fine-tuning:** the RL gradient is augmented with a **decaying demo term**:
     `g_aug = Σ_{ρπ} ∇θ log π A^π  +  Σ_{ρD} ∇θ log π · w(s,a)`
     where `w(s,a) = λ0 · λ1^k · max_{ρπ} A^π` — scales demo gradient by current max RL advantage,
     decays geometrically with iteration `k` (λ0=0.1, λ1=0.95 → near-zero by convergence). Demos remain
     in the gradient throughout training, not just at init.
  - *Why the decaying demo term (not just BC init):* different demo segments are useful at different RL
    stages. E.g. hammering: BC teaches reach+grasp but rarely nail-strike. Once RL masters grasping, the
    nail-strike portion of the demo should guide — but only if it is still in the gradient.
- **Backbone:** Natural Policy Gradient (NPG); **on-policy — no replay buffer.** DDPGfD (a competing
  method) puts demos in a replay buffer; DAPG explicitly avoids this, using on-policy NPG for stability
  and high-DoF scalability. The demos enter as a gradient term, not stored transitions.
- **Demos:** few VR demos (MuJoCo + HTC Vive); no hardware deployment in this paper.
- **Tasks:** object relocation, in-hand pen spinning, door opening, hammer striking (24-DoF sim hand).
- **Reward — dense vs. sparse (key finding):** both shaped (dense) and sparse task-completion rewards
  are tested. Pure RL with sparse reward fails on most tasks (no exploration signal). **DAPG with sparse
  reward succeeds** — demos solve the exploration problem, making manual reward shaping unnecessary. This
  is a central result: *with a good demo set, you can train on sparse rewards and skip dense reward
  engineering.* Pure RL still needs dense shaped rewards to get off the ground.
- **Results:** converges in hours-equivalent; BC alone fails; pure RL needs ~100× more samples.
- **Relevance to us:** canonical mechanism for "bootstrap RL with our robot demos." Our robot teleop
  data = our demo set; BC on teleop data → NPG/PPO fine-tuning with decaying demo loss is a direct
  adaptation. Sim-based, but the IL→RL recipe is hardware-agnostic. **Reward implication: we may not
  need dense contact rewards if we have enough demos — sparse task-completion labels could suffice.**

### RL-100 (2025) — [link](https://arxiv.org/abs/2510.14830)  *(pre-read 2026-06-08)*

- **Full title:** *RL-100: Performant Robotic Manipulation with Real-World Reinforcement Learning.*
- **Method:** unifies IL pretraining and RL fine-tuning **within the diffusion denoising process** under
  a single clipped PPO surrogate objective — PPO is applied at each denoising step, treating denoising
  as an RL action sequence, rather than as a separate outer RL layer on a frozen policy. Then
  **consistency distillation** compresses multi-step diffusion → one-step inference for real-time
  high-frequency control.
- **Real-world, gripper-based.** 8 tasks (dynamic pushing, bowling, pouring, cloth folding, unscrewing,
  juicing, box folding). 1000/1000 success episodes (hence "RL-100"). ~90% zero-shot under env shifts.
  Deployed in a real shopping mall (7 hours, continuous).
- **Not dexterous hands.**
- **Reward — mostly sparse, human-labeled (verified from paper):**
  - *5 of 8 tasks:* **binary +1/0 sparse reward.** Human supervisor presses a key **once per episode**
    upon observing task completion. +1 at that timestep, 0 everywhere else. No per-step input.
  - *Dynamic Push-T (1 task):* **dense, fully automated** (no human input). Per-timestep:
    `r_t = r_pose + r_static + r_smooth` where `r_pose = exp(−3e)−1` (SE(3) discrepancy `e` between
    T-block and target pose); `r_static = −1` if block movement below threshold, else 0; 
    `r_smooth = −5‖a_t − a_{t−1}‖²` (jerk penalty). Dense needed here because terminal signal
    carries no gradient for continuous precision pushing.
  - *Practical note:* for sparse tasks, failed episodes cost zero human effort — only successful ones
    get labeled. Supervision budget is minimal.
- **Relevance to us:** gripper-only, so not directly applicable to our dexterous setting. The design
  insight — that you can fine-tune a **diffusion policy backbone with RL by applying PPO within the
  denoising process** — is worth tracking if we adopt a diffusion policy (DP/ACT-D) as our backbone.
  **Reward implication: human-labeled sparse rewards are a practical real-world substitute for
  sim-privileged dense rewards** (when combined with a good IL init). Directly relevant to our
  no-sim constraint.

### DexWild (RSS 2025) — [link](https://arxiv.org/abs/2505.07813)  *(in progress)*

- **Why it's our closest match (so far):** (1) targets **bimanual dexterous** manipulation;
  (2) their **ideal data distribution = ours** — not expensive teleop, not too-distant YouTube ego
  video, but **human data performing the *same* tasks** as the sweet spot between accessibility /
  portability (→ generalization) and quality. They name it **"purpose-collected high-quality human
  embodiment data"** — a nice descriptive label we may borrow.
- **Key challenge — accurate 6-DoF hand-pose localization for human data** (systematically analyzed
  in their prior work; this is one of *our* core open problems too):
  - **Vision-based hand tracking** — not accurate enough for our needs, esp. for **bimanual
    coordination**. *(We agree.)*
  - **IMU-based** — mentioned in related work, but **prone to drift**. *Worth keeping the refs so we
    know where to look later.*
  - **DexWild's solution:** **cubical AprilTag markers** mounted on the gloves, localized with
    **monocular exocentric camera(s)**. ← note as a candidate approach for us.
- **Action space — relative actions.** Records **relative wrist poses** (no absolute world frame)
  → **eliminates calibration**. Full action space = **wrist pose + hand-joint angles** *(full method
  to read tomorrow)*.
- **Hand action-space alignment — fingertip retargeting** (builds on **DexPilot** [17] & **Robotic
  Telekinesis** [44]). Human ≠ robot hand kinematically, so they **don't copy joint angles**; instead
  they **solve for the robot's finger-joint angles whose fingertips match the fingertip positions in
  the human demo**. (DexPilot matches *relative* fingertip-to-fingertip / fingertip-to-palm vectors →
  encodes pinch/grasp intent, robust to tracking noise; Telekinesis = a *learned* retargeting net from
  single-RGB / YouTube hand video.) Net: puts human demo + robot action in a **shared fingertip space**
  → transferable across the hand gap. *(Imitation-based answer to the human→robot-hand gap; DexMachina
  = the RL route for the harder case.)*
- **Closing the visual gap — matched wrist cameras.** Wrist/palm cameras mounted the **same way on
  the human glove and the robot hand** → **no visual gap** on that input stream.
- **Visual obs — flag for us.** Appears **palm/wrist-camera only (no head camera)**. Their external
  tracking cam, *"when carefully positioned,"* can also capture **supplementary environmental
  context** for robust policies → suggests **aligning the main visual obs still takes effort**.
  **Our challenge:** we plan **egocentric (head-cam) obs**, so the human↔robot visual gap on the main
  view will need deliberate attention → see [ledger](../decisions-and-caveats.md).
- **Our critique / open risks:**
  - **Relative actions may accumulate error** over a rollout. DexWild's *data* is likely fine (poses
    estimated from exo cameras), but **error accumulation could bite at policy-learning / inference**
    time — watch.
  - **No third-view / head camera** limits **environmental context** → likely relies on a
    **well-designed desktop layout + initial pose** to keep objects in view. We'll probably still want
    an **egocentric head camera**, so we need to **find works on mitigating the human↔robot visual
    gap** (→ ledger; queue a targeted B search).
  - **Speed mismatch — human demos run faster than robot rollouts, and DexWild doesn't address it**
    (30 Hz capture + action chunking only; no time-normalization / resampling). A **temporal/speed
    mismatch** we may need to handle for our human data → [ledger](../decisions-and-caveats.md).
- **Bimanual trick — append inter-hand pose to the observation** so the policy can reason about the
  **relative pose between the two hands**. Cheap; **consider adopting** for our bimanual setup.
- **Vision encoder:** pretrained **ViT** (not ResNet), initialized from the **"Soup 1M" model**
  (Dasari et al. 2023, *Data4Robotics* [11]; checkpoint `SOUP_1M_DH`). That ViT is pretrained
  **MAE-style** on a **~1M-image "soup"** combining standard vision datasets (ImageNet, …) **+ 100
  Days of Hands (DoH)** — a human-hands dataset (hence `_DH`). Their thesis: **pretraining-data
  *distribution* > scale**; ViT > ResNet for in-the-wild manip [16]. *Relevance: a **hand-inclusive**
  pretraining mix likely suits our dexterous task → candidate encoder init for us.*
- **Backbone:** _(to fill)_
- **Training schedule:** **co-training** with a **fixed in-batch ratio** `(w_h, w_r)` — no
  curriculum/staging, uniform from init. **Optimal 1:2 robot:human** (~2× human per batch) → 79.8%
  in-domain / 75.1% in-the-wild / 62.7% extreme. Datasets: **1,395 robot** vs **9,290 human** demos
  (5 tasks). *(Concrete mixing ratio — fills the gap GR00T left → ledger.)*
- **Force / tactile:** _(to fill — does the rig capture any contact signal?)_
- **Key takeaways (read 2026-06-05):**
  1. **Tasks aren't very dexterous** — mostly simple grasping (e.g. bottles). So **how to retarget +
     learn from human data on genuinely dexterous tasks remains open for us** (→ DexMachina next).
  2. **Human data mainly buys generalization** — to new environments / visual appearances (sensible),
     which is the headline empirical result.
  3. **Marker-visibility trick:** palm-cam-only **hides the AprilTags from the visual obs** (tags out
     of view) — a neat sidestep. **But egocentric cameras *would* see the tags** → a marker-induced
     human↔robot visual gap; we'd likely need **inpainting** (well-studied) to remove them. → [ledger](../decisions-and-caveats.md).
- **Status:** **done (2026-06-05).** Next: DexMachina.

### DreamZero — World Action Models are Zero-shot Policies (2026) — [link](https://arxiv.org/abs/2602.15922)  *(parked 2026-06-04 — deprioritized; see decision below)*

- **Idea / objective:** a **World Action Model (WAM)** on a **video-diffusion backbone**
  (Wan2.1-I2V-14B); **jointly denoises video + action** via a shared flow-matching objective —
  *not* action-only prediction.
- **Contrast w/ VLAs:** VLAs optimize a **dedicated target (action prediction)**, so they may
  **ignore other useful signals** — e.g. how the world changes in response to the action.
  DreamZero's video objective keeps that world-dynamics signal. *(Confirms our prior view that
  VLAs' narrow target can drop information.)*
- **Key takeaway (theirs):** **diverse** pretraining data works better than repetitive /
  narrowly-targeted data.
- **Human-data schedule (agent research 2026-06-04):** **mixed / co-training, NOT sequential.**
  - *Base pretrain:* robot **video+action only** (~500 h AgiBot).
  - *Cross-embodiment transfer (where human data enters):* **co-train 1:1** human *video-only* demos
    with pretraining data, 10K steps — human carries **video-only loss** (action loss masked); robot
    keeps the joint video+action loss.
  - *Per-task post-train (separate stage, Sec 4.2):* on that task's **robot teleop data only**
    (33/12/40 h), 50K steps/task; keeps the joint video+action objective ("as in pretraining") → a
    *world-model* finetune, not action-only. (So this finetune ≈ robot-data-only on the data axis.)
  - Web-scale human video also enters *implicitly* via the frozen **Wan2.1** backbone (upstream of
    DreamZero's own training).
- **Human signal form:** pure **video-denoising target** — no latent/pseudo-actions, no retargeting.
- **Decision (2026-06-04): stop deep-reading.** Our setting (same-task human + robot data) likely
  favors the **VLA / action-head route** over the world-action-model line; and we start with
  **train-from-scratch DP / ACT** first, so the foundation-model backbone choice isn't urgent.
  Revisit if we later pursue a world-model / video-prediction objective.

### GR00T N1 family (N1 / N1.5 / N1.7) — human-data schedule  *(agent research 2026-06-04, pre-read)*

> Focused finding answering the sequential-vs-mixed question; full read still pending (tier 2).

- **Verdict: co-training, NOT sequential.** Human/diverse data is mixed into the **same
  pretraining batches** as robot + synthetic data (the "data pyramid"), followed by a separate
  **per-embodiment robot finetune**. Robot data is present *throughout* pretraining → this is
  not "human-only pretrain → robot finetune."
- **Human signal form (evolves across versions):**
  - **N1:** human video → **VQ-VAE latent actions** / IDM **pseudo-actions** as flow-matching targets.
  - **N1.5:** **FLARE** future-latent-alignment loss (coeff 0.2) "unlocks learning from human
    video" — co-trained, no action labels.
  - **N1.7:** **20K hrs EgoScale** egocentric human video co-trained "alongside" robot demos;
    FLARE + a **shared relative-EEF action space**. 1k→20k hrs human video >2× avg task completion.
- **Pretrain mixing ratio — NOT stated.** The paper lists the data-pyramid sources/sizes but gives
  **no sampling ratio, per-source weights, or sampling strategy** for pretraining (checked text +
  appendices). Notable **asymmetry vs. DreamZero**, which *does* publish its 1:1. (N1.5 shows a
  data-distribution pie chart, but no numbers in the page text.)
- **Finetune (post-training) is NOT robot-only.** Co-trains **real robot demos + model-generated
  *neural* trajectories at 1:1** (Sec 2.3). "Neural" = synthetic video from an image-to-video model
  fine-tuned on 88 h teleop → 827 h generated (~10× aug); *not* human video, *not* real teleop.
  So no *human* data in post-training, but not pure real-robot either.
- Sources: [N1 paper](https://arxiv.org/abs/2503.14734) · [N1.5](https://research.nvidia.com/labs/gear/gr00t-n1_5/) · [FLARE](https://arxiv.org/abs/2505.15659) · [N1.7 blog](https://huggingface.co/blog/nvidia/gr00t-n1-7).

### From Human Videos to Robot Manipulation: A Survey (2026) — [link](https://arxiv.org/abs/2606.00054)

*Type: survey / overview. Read to build the mental map and set our stance, not for one method.*

- **Core question (the survey's framing):** when human videos enter a VLA training pipeline,
  *what type of information do they become*, and *how does that information interface with
  policy learning*? The taxonomy is organized by the **form the human-video signal takes**.
- **Four paradigms (by what the human signal becomes):**
  1. **Latent action** — policy carries a latent encoding implicit action / intent
     *(guess: paired with embodiment-specific action decoders — confirm in paper).*
  2. **World model** — human video trains a predictive / generative model (usually
     video-generation).
  3. **Explicit 2D cues** — non-action signals such as keypoints / traces in image space.
  4. **Explicit 3D cues** — same, lifted to 3D.
- **Three key challenges (survey) + our stance:**
  1. *Scalable episodization of unstructured video* — **low priority for us.** Our human +
     robot data is purposeful and task-aligned (both perform the *same* tasks), not in-the-wild
     / random play — so segmenting unstructured video is largely a non-issue.
  2. *Heterogeneity-aware grounding under embodiment / viewpoint mismatch* — **relevant but
     bounded.** We will match camera & viewpoint as far as possible, so our visual gap is far
     smaller than YouTube-scale sources. Focus = handling *non-perfectly-aligned* views (not
     large in-the-wild gaps), plus the hand / embodiment gap.
  3. *Deployment-predictive evaluation of transfer efficiency* — **high priority.** Real-world
     eval is cumbersome and we have no sim; we want **early offline proxies for policy quality**
     (loss / other signals) to predict deployment performance and speed iteration.
- **Relevance to us:** sets the map for choosing which paradigm / framework to adopt; sharpens
  that our two real watch-items are **(2) imperfect view alignment** and **(3) cheap early eval
  signals**.
- **Pointer — revisit later (§3.1):** good catalogue of **training tricks / technical solutions**
  (e.g. random cropping to stop the policy memorizing absolute positions). *Not* deep-read now —
  hard to place these without first understanding a concrete VLA architecture. **Refer back here**
  when we're hunting for tricks to improve training.
- **§5 critique (analysis is thin):** states what *needs* to happen without concrete model-level
  examples. Our resulting watch-items:
  - *Embodiment gap:* survey advocates "embodiment-invariant supervision (interaction intent,
    object-centric effects) over literal trajectory imitation" but cites no concrete design →
    **watch for concrete model-wise designs** that realize this in specific works.
  - *Viewpoint variance:* no solid solution offered. **Lead: action space** (absolute vs.
    relative joint/eef trade-off) — tracked in [Open questions](#open-questions).
  - *Eval:* survey only discusses **sim** → not useful for us (no sim). Better lead: an
    EgoVerse-style claim that **lower training MSE still tracks better policy performance** — a
    candidate cheap offline proxy (our challenge C3) — tracked in [Open questions](#open-questions).
- **Overall verdict:** decent **mental map / taxonomy**, but shallow and light on concrete
  designs → **low direct yield**. Next: **dive into specific works' architectures & technical
  choices**, prioritizing **dexterous-hand** manipulation (see priority update above).

---

### GR00T N1 (NVIDIA, 2025) — [link](https://arxiv.org/abs/2503.14734)

- **Idea / problem:** foundation VLA pretrained on heterogeneous data (robot + human-video +
  synthetic) for generalist robot manipulation.
- **Backbone:** dual-system — **System 2** (slow): VLM for language/visual reasoning; **System 1**
  (fast): diffusion transformer for action generation. Structurally similar in spirit to π0's
  slow-fast design.
- **Action head:** diffusion transformer (System 1).
- **Training schedule:** mixed pretraining on robot + human-video + synthetic data combined.
  Human data retargeted via **vision-based hand retargeting only** (no contact/force sensors).
- **Force/tactile:** not used.
- **Key takeaway:** broad architecture is not dramatically novel vs. peers (similar to π0). The
  **"additional training details"** section is the most informative part: uses **object detection
  as an auxiliary loss** (see [Cross-cutting tricks](#cross-cutting-tricks-and-insights)). Baselines
  are basic DP; tasks are standard pick-and-place. **Vision-only retargeting** is used for human
  data — likely adequate for easy gripper tasks, probably insufficient for dexterous manipulation.
- **What to try / relevance to us:** reference baseline architecture. The main flag for us: their
  human-data pipeline is **vision-only retargeting**, which is a reasonable shortcut for grippers
  and simple tasks — but for **complex dexterous manipulation**, this gap is likely a real
  bottleneck. Need to track how other works handle this (see [Open questions](#open-questions)).

---

## Cross-cutting tricks and insights

> Patterns that show up across multiple works — **not buried in per-paper notes**.
> Return here when designing our training pipeline.

- **WAM vs. VLA: pretraining data efficiency trade-off.** World action models (WAMs) are
  supervised on **frame-to-frame transformations** (not explicit actions), so they can absorb
  diverse, non-action-labeled pretraining video more naturally than VLAs (which require action
  labels). This makes WAMs more **teleop-data efficient** when the pretraining pool is
  heterogeneous / dissimilar from robot data (e.g. YouTube-scale video).
  **Caveat for us:** this advantage is largely context-dependent. Most WAM papers assume
  pretraining data that is far *less* similar to robot demonstrations than ours — our human data
  is **same-task, roughly-matched viewpoint**, not in-the-wild. So WAM's data-efficiency edge
  is less compelling in our setting and does not, on its own, argue for switching from VLA to WAM
  as our backbone. *(Insight from NV conversation, 2026-06-04.)*

- **Input dropout to bridge human↔robot observation gap.** Ego-Pi drops the wrist camera with
  40% probability during robot data training, because human demos have no wrist camera. Forces
  the policy not to over-rely on wrist imagery — same principle as FACTR's visual blurring.
  General pattern: **when human and robot observations differ in a modality, randomly mask that
  modality during robot training** so the policy learns to function without it.

- **Auxiliary loss to steer attention.** GR00T N1 uses **object detection as an auxiliary training
  loss** to force the model to attend to task-relevant objects. The same pattern appears in
  force-aware works: **force forecasting as auxiliary loss** to prevent mode collapse (policy
  ignoring force signals and attending only to vision). General principle: **if there is a signal
  or feature you want the policy to respect, attaching it as an auxiliary prediction target is a
  reliable forcing function.** → Directly relevant when we add force/torque to our policy
  (see [D](force-impedance-control/index.md)).

---

## Options to try

> Candidate approaches surfaced by the survey — running list.

- _(add candidates as they surface)_

---

## Open questions

- **Action space vs. viewpoint robustness.** Hypothesis (from experience; partly a guess):
  *absolute* joint actions overfit / fit better but are more **fragile to viewpoint change**;
  *relative* joint / eef actions are **harder to learn** but may be **more robust**. The survey
  gave no answer — resolve by watching the **action space** each specific work uses and how it
  fares under view shift.
- **Cheap offline eval proxy.** Does **lower training MSE reliably track better deployed-policy
  performance** (EgoVerse-style claim)? If so it's our main early signal given no sim. Verify in
  EgoVerse and others; this is our concrete answer to challenge **C3**.
- **Human→robot retargeting for dexterous hands.** GR00T uses vision-only retargeting —
  acceptable for grippers / easy tasks, likely insufficient for complex dexterous manipulation.
  **Watch for:** how works doing non-trivial dex manipulation handle human-hand → robot-hand
  alignment (functional retargeting, RL-based, sensor-augmented, etc.). DexMachina is the
  primary lead here.
