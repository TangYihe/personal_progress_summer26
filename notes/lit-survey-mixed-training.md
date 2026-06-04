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
  [decisions-and-caveats.md](decisions-and-caveats.md) (data-mixing ratios, augmentation, eval
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
- ⭐ **DexMachina** — https://arxiv.org/abs/2510.08475 — *(bumped 2026-06-03)* **functional retargeting** of non-trivial human manipulation onto **dexterous (multi-finger) hands**, via **RL**. Hits a likely **bottleneck for us** — crossing the human→robot-hand gap for *very* dexterous manipulation — and previews an **RL route** we may need if collected human data can't be replayed directly. *(id was listed jointly with DexMan; verify which is which before citing.)*

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

### Reference
- *Unverified — confirm before citing:* DemoBot (2601.01651), DexMan (2510.08475 — id shared w/ DexMachina, now promoted to tier 3; verify attribution), Object-centric 3D Motion Field (2506.04227), Gen2Act (2409.16283).
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
