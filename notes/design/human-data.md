# Human data & data pipeline — options & decisions

> Decisions for human data collection, mixing strategy, and training schedule.
> Part of the [decisions index](../decisions-and-caveats.md).
>
> **Entry format:** short claim · **Why it matters** · **Source** · **Status**
>
> Last updated: 2026-06-09

---

## Data mixing & ratios

- **Human : robot mixing ratio is a first-class hyperparameter we must tune.**
  - *Why:* directly controls transfer quality; no universal default to inherit.
  - *Source:* **DreamZero** publishes explicit **1:1** (human video-only loss co-trained with robot,
    action loss masked on human data). **GR00T N1** does not report any pretraining mixing ratio
    or weights (checked text + appendices) → reproducibility gap. PI's *Human→Robot Transfer*
    studies embedding effects across pretrain ratios — good analysis template for our own sweeps.
  - *Concrete starting point:* **DexWild** co-trains at fixed in-batch ratio, **1:2 robot:human
    optimal** (~2× human per batch; datasets: 1,395 robot vs 9,290 human demos; no curriculum,
    uniform from init). Best available concrete number.
  - *Status:* **open** — start at DexWild's 1:2 robot:human; sweep empirically and log as a
    tracked hyperparameter.

---

## Temporal alignment (human ↔ robot speed)

- **Human demos run faster than robot execution — largely unaddressed in the field.**
  - *Why:* speed mismatch between human and robot data could hurt co-training and policy timing;
    policy may learn human-paced dynamics that don't match the robot.
  - *Source:* **DexWild** — 30 Hz capture + action chunking, no time-normalization described.
    **ManipTrans** declines to enforce temporal alignment ("real robots cannot always operate as
    quickly as human hands") → field largely leaves this open.
  - *Status:* **open** — consider temporal resampling / time-normalization for our human data.

---

## Training schedule / staging

- **Field leans co-training (mixed) over sequential pretrain→finetune.**
  - *Why:* shapes our pipeline default; human-only pretrain → robot-only finetune is not what
    SOTA does.
  - *Source:* GR00T (mixed data-pyramid pretrain), DreamZero (1:1 co-train at transfer),
    EgoMimic (joint co-train). None do strict sequential.
  - *Status:* **leaning** co-training; confirm against EgoVLA / LAPA / H-RDT as we read.

- **The finetune stage is rarely pure robot-action — consider an auxiliary objective / data there.**
  - *Why:* both field leaders enrich finetune rather than doing action-only BC.
  - *Source:* **GR00T** post-train co-trains real robot + model-generated *neural* trajectories at
    1:1 (~10× aug via img→video model fine-tuned on 88 h teleop). **DreamZero** per-task post-train
    uses task robot data only but keeps the joint video+action (world-model) objective throughout.
  - *Status:* **open** — decide whether to keep an auxiliary loss during robot finetune and/or
    augment with synthetic data.

---

## Human-data collection & hand-pose tracking

- **Accurate 6-DoF hand-pose localization is a core open problem for us.**
  - *Why:* pose accuracy bounds the quality of purpose-collected human data; bimanual dexterous
    coordination raises the accuracy bar further.
  - *Options:*
    - **Vision-based hand tracking** — insufficient accuracy, especially bimanual.
    - **IMU-based** — drift-prone (keep refs for later).
    - **Cubical AprilTags on gloves + monocular exocentric camera** — DexWild's solution; leading
      candidate for us.
  - *Caveat (marker visibility):* AprilTags appear in egocentric obs (visible on human hand, absent
    on robot) → marker-induced visual gap. DexWild sidesteps this with palm-cam-only obs (tags out
    of frame). For our egocentric head-cam we'd need **inpainting / marker removal** (well-studied).
  - *Source:* DexWild. *Status:* **open** — evaluate AprilTag-glove approach for our setup.

- **Data-distribution stance: target purpose-collected high-quality human-embodiment data.**
  - *Why:* sweet spot between teleop (too costly) and YouTube ego video (too distant from task).
    DexWild's label for this: "purpose-collected high-quality human-embodiment data."
  - *Source:* DexWild. *Status:* **leaning** — our framing.
