# Technical decisions & caveats — running ledger

> Cross-cutting accumulator: design decisions we'll have to make + engineering caveats / tricks to
> carry into implementation. Sourced from papers (B, [survey](lit-survey-mixed-training.md)),
> control reading (D, [force/impedance](force-impedance-control/index.md)), and our own constraints.
> ← back to [overview](../README.md).
>
> **Entry format:** short claim · **Why it matters** · **Source** · **Status** (open / leaning / decided).
> Keep concise and formal. Newest evidence appended within each item.
>
> Last updated: 2026-06-04

---

## Data mixing & ratios

- **Human : robot (: synthetic) mixing ratio is a first-class hyperparameter we must set ourselves.**
  - *Why:* it directly controls transfer; there's no default to inherit.
  - *Source:* **DreamZero** publishes an explicit **1:1** (human video-only co-trained with robot,
    action loss masked on human). **GR00T N1** does **not** report any pretraining mixing
    ratio/weights/strategy (checked text + appendices) → reproducibility gap. PI's *Human→Robot
    Transfer* studies embedding effects **across** pretrain ratios — good analysis template.
  - *Concrete data point:* **DexWild** co-trains at a **fixed in-batch ratio**, **1:2 robot:human
    optimal** (~2× human per batch; datasets 1,395 robot vs 9,290 human demos; no curriculum, uniform
    from init) → fills the gap GR00T left.
  - *Status:* **open** — plan to sweep this empirically and track it as a logged hyperparameter
    (DexWild's 1:2 robot:human is a sensible starting point).

## Temporal alignment (human ↔ robot speed)

- **Human demos run faster than robot execution — DexWild does *not* explicitly handle this.**
  - *Why it matters:* a speed/tempo mismatch between human and robot data could hurt co-training and
    policy timing; the policy may learn human-paced dynamics that don't match the robot.
  - *Source:* DexWild — 30 Hz capture + action chunking + relative state-action rep, but **no
    time-normalization / resampling / speed-matching** described (our observation from their demo
    videos; unaddressed in paper).
  - *Corroboration:* **ManipTrans** also **declines to enforce temporal alignment** ("real robots
    cannot always operate as quickly as human hands") → the field largely leaves this **unaddressed**.
  - *Status:* **open** — consider temporal resampling / time-normalization / speed-matching for our
    human data; check whether DexMachina or others handle it.

## Training schedule / staging

- **Field leans co-training (mixed) over sequential pretrain→finetune.**
  - *Why:* shapes our pipeline default; "human-only pretrain → robot-only finetune" is *not* what
    SOTA does.
  - *Source:* GR00T (mixed data-pyramid pretrain), DreamZero (1:1 co-train at transfer), EgoMimic
    (joint co-train). None do strict sequential.
  - *Status:* **leaning** co-training; confirm against EgoVLA / LAPA / H-RDT as we read.
- **The finetune stage is rarely pure robot-action — consider an auxiliary objective / data there.**
  - *Why:* both leaders enrich finetune rather than doing action-only BC.
  - *Source:* **GR00T** post-train co-trains real robot + **model-generated *neural* (synthetic)
    trajectories at 1:1** (synthetic video from an img→video model fine-tuned on 88 h teleop → 827 h,
    ~10× aug). **DreamZero** per-task post-train uses task **robot data only**, but **keeps the joint
    video+action (world-model) objective** ("as in pretraining"), not action-only.
  - *Status:* **open** — decide whether to (a) keep a world-model / video-prediction auxiliary loss
    during our robot finetune, and/or (b) augment finetune with synthetic data.

## Architecture / representation

- **Key fork: does human data become *action-shaped* or stay *perception/video*?** — tied to backbone.
  - *Why:* this, more than schedule, is where SOTA actually diverges; it gates the backbone choice
    (action-head VLA vs. video-diffusion world-action model).
  - *Source:* **GR00T** forces human video into action-like targets (VQ-VAE latent actions / IDM
    pseudo-actions → FLARE latent alignment → shared relative-EEF space). **DreamZero** keeps human
    data as a pure **video-denoising target** (action loss masked), letting the world-model objective
    absorb it natively — retains world-change signal a narrow action-prediction target may drop.
  - *Status:* **leaning VLA / action-head route** (2026-06-04) — our setting (same-task human +
    robot) favors it over the world-action-model line; DreamZero deep-read parked. **Not urgent:**
    we start with **train-from-scratch DP / ACT** before committing to any foundation-model backbone.

## Observation & action space

- **Relative action representation** (record **relative wrist poses**, not an absolute world frame).
  - *Why:* removes the need for an absolute world frame / **calibration**; also a candidate answer to
    our open **action-space view-robustness** question (abs vs. rel).
  - *Source:* DexWild (full action space = wrist pose + hand-joint angles).
  - *Caveat:* relative actions can **accumulate error** over a rollout; DexWild's data is likely OK
    (exo-camera pose estimation), but watch this at **policy-learning / inference** time.
  - *Status:* **leaning** adopt (with the error-accumulation caveat in mind).
- **Hand action-space alignment via fingertip retargeting** to cross the human→robot-hand
  kinematic gap.
  - *Why:* hands differ kinematically → copying joint angles breaks grasps. Instead **solve for robot
    finger-joint angles whose fingertips match the human's fingertip geometry** (relative
    fingertip-to-fingertip / -to-palm vectors preserve pinch/grasp intent). Yields a **shared
    fingertip action space** that transfers across embodiments.
  - *Source:* DexWild (builds on **DexPilot** `1910.03135` optimization-based, **Robotic Telekinesis**
    `2202.10448` learned-net retargeting). **DexMachina** = the **RL** route for the harder case.
  - *Ours:* we already retarget human hand → robot via **fingertips** — same family as this line.
  - *Status:* **leaning** — default retargeting approach; revisit RL (DexMachina) if pure retargeting
    underperforms on our dexterous tasks.
- **Matched wrist-camera mounting (human glove ↔ robot hand)** to close the visual embodiment gap.
  - *Why:* identical camera placement → no visual domain gap on that input stream.
  - *Source:* DexWild.
  - *Status:* **leaning** adopt for any wrist-cam stream we use.
- **Bimanual: append inter-hand (relative) pose to the observation.**
  - *Why:* gives the policy explicit relative-pose info between the two hands → easier bimanual
    coordination.
  - *Source:* DexWild.
  - *Status:* **leaning** adopt (bimanual).
- **Main visual-observation alignment is an OPEN challenge for us — especially with egocentric obs.**
  - *Why:* DexWild leans on **wrist-cam-only** input plus a carefully-positioned external cam; we plan
    **egocentric (head-camera) obs**, where the human↔robot visual gap on the *main* view is larger
    and must be deliberately aligned. Misalignment here could undercut transfer.
  - *Also:* no third-view/head cam limits **environmental context** → DexWild likely relies on a
    **controlled desktop layout + initial pose** (objects in view). We can't assume that with
    egocentric obs.
  - *Source:* DexWild (wrist-cam only, no head cam; external cam useful "when carefully positioned").
  - *Action:* **queue a targeted B search** — mitigating the human↔robot visual gap for
    egocentric/head-cam obs, incl. **inpainting / marker removal** (AprilTags) and view alignment.
  - *Status:* **open — watch.**

## Vision encoder

- **Prefer a pretrained ViT whose pretraining mix includes human-hand imagery (over a ResNet).**
  - *Why:* DexWild inits from **Soup 1M (`_DH`)** (Dasari et al. 2023) — pretraining-data
    *distribution* matters more than scale; a hand-inclusive mix (100 Days of Hands) suits
    manipulation; ViT > ResNet in-the-wild.
  - *Source:* DexWild ([11] Data4Robotics, [16]).
  - *Status:* **leaning** — candidate encoder init for our dexterous tasks.

## Augmentation

- **Random image cropping** to stop the policy memorizing **absolute positions**.
  - *Why:* cheap, standard regularizer for visuomotor policies; improves spatial generalization.
  - *Source:* survey §3.1 (catalogue of training tricks — **revisit §3.1** for more once we've fixed
    a concrete architecture).
  - *Status:* **leaning** adopt.

## Evaluation

- **Need early *offline* proxies for policy quality (loss / other signals).**
  - *Why:* real-world eval is cumbersome and we have **no sim** → slow iteration without a proxy.
  - *Source:* survey challenge #3 (deployment-predictive evaluation) — flagged high-priority for us.
  - *Status:* **open** — identify which offline signals correlate with deployment success.

## Human-data collection & hand-pose tracking

- **Accurate 6-DoF hand-pose localization for human data is a core open problem for us.**
  - *Why:* pose accuracy bounds the quality of purpose-collected human-embodiment data; **bimanual
    dexterous** coordination raises the accuracy bar further.
  - *Options seen:* **vision-based hand tracking** — insufficient accuracy (esp. bimanual);
    **IMU-based** — drift-prone (keep refs for later); **cubical AprilTags on gloves + monocular
    exocentric camera** — DexWild's solution, leading candidate to evaluate.
  - *Source:* **DexWild** (+ its prior-work analysis of tracking methods).
  - *Caveat (marker visibility):* AprilTags appear in **egocentric** obs (visible on human, absent on
    robot) → a marker-induced visual gap. DexWild dodges it with **palm-cam-only**; for our egocentric
    cams we'd likely need **inpainting / marker removal** (well-studied) — fold into the visual-gap search.
  - *Status:* **open** — evaluate the AprilTag-glove approach for our bimanual setup.
- **Data-distribution stance:** target **purpose-collected high-quality human-embodiment data**
  (humans doing the *same* tasks) — the sweet spot between teleop (too costly) and YouTube ego video
  (too distant). Matches our intended strategy. *Source:* DexWild. *Status:* **leaning** — our framing.

## RL & simulation dependency

- **Functional (object-centric) retargeting via RL typically needs a simulator — we don't have one.**
  - *Why it matters:* DexMachina's RL curriculum depends on **virtual object control** (object forced
    along the desired trajectory early, so the hand policy mimics motion without early-failure
    derailment) + **privileged sim contact** for rewards. We have **no sim**.
  - *Our substitutes (to explore):* (a) **bootstrap from real robot data on the same task** (incl.
    teleoperated **recovery behaviors**) instead of a sim curriculum; (b) get **contact without
    privileged info** — infer from vision, read **hand-hardware joint torque**, or borrow reward
    designs from **real-world RL**.
  - *Source:* DexMachina (intro).
  - *Sim-free leads (DexMachina refs):* **data-driven functional retargeting** via a human-robot-object
    motion manifold (Park et al. 2025, `2501.04169`, no sim); **real-world** dexterous RL w/o
    sim/privileged rewards (Xu et al. 2022, `2212.09902`); RL **+ demos** (DAPG, `1709.10087`).
  - *Status:* **open** — core design question if we pursue RL for dexterous functional retargeting.

## Force / tactile integration

- **Contact-state sensing for RL rewards / policy input** — we lack sim's privileged contact, so we
  need a real source: **infer from vision** vs. **hand-hardware per-joint torque** readings.
  - *Source:* DexMachina (uses privileged sim contact we don't have). *Status:* **open** → resolve via [D](force-impedance-control/index.md).
- *(More to fill from [D](force-impedance-control/index.md): where/how force attaches to the chosen
  backbone, controller assumptions, etc.)*
