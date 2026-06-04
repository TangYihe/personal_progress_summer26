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
  - *Status:* **open** — plan to sweep this empirically and track it as a logged hyperparameter.

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

## Force / tactile integration

- *(placeholder — fill from [D](force-impedance-control/index.md): where/how force attaches to the
  chosen backbone, controller assumptions, etc.)*
