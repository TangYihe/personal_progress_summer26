# Technical decisions & caveats — index

> Entry point for all technical design decisions across the training pipeline. Large topics have
> dedicated files (linked in the table); smaller topics are inline below.
> ← back to [overview](../README.md).
>
> Last updated: 2026-06-09

---

## Index

| Topic | Covers | Status | Detail |
|-------|--------|--------|--------|
| **Human data & data pipeline** | mixing ratios · temporal alignment · training schedule · data collection | open / leaning | [human-data.md](design/human-data.md) |
| **Training pipeline** | augmentation · evaluation proxies | open / leaning | inline §1 |
| **Model architecture & obs/action** | backbone fork · action space · vision encoder | open / leaning | inline §2 |
| **RL fine-tuning** | bootstrap strategy · reward selection · contact-without-sim | open | [rl-finetuning.md](design/rl-finetuning.md) |
| **Force & tactile integration** | torque as contact proxy · force in policy obs | open | [force-feedback.md](design/force-feedback.md) |

---

## §1 · Training pipeline

- **Random image cropping** to stop the policy memorizing absolute positions.
  - *Why:* cheap standard regularizer; improves spatial generalization.
  - *Source:* survey §3.1. *Status:* **leaning** adopt.

- **Need early offline proxies for policy quality.**
  - *Why:* real-world eval is cumbersome, no sim → slow iteration without a cheap proxy signal.
  - *Lead:* does lower training MSE reliably track deployed-policy performance? Verify in EgoVerse
    and others.
  - *Source:* survey challenge #3. *Status:* **open.**

---

## §2 · Model architecture & obs/action space

**Backbone fork — does human data become action-shaped or stay perception/video?**
- *Why:* gates the backbone choice (action-head VLA vs. video-diffusion WAM).
- *VLA route:* GR00T forces human video into action-like targets (latent actions / IDM / FLARE /
  shared relative-EEF action space).
- *WAM route:* DreamZero keeps human data as pure video-denoising target — retains world-change
  signal a narrow action-prediction target may drop; but advantage is smaller in our same-task
  setting than in in-the-wild scenarios.
- *Status:* **leaning VLA / action-head route.** Start with train-from-scratch DP / ACT first;
  foundation-model backbone choice is not urgent yet.

**Relative action representation** (relative wrist poses, not absolute world frame).
- *Why:* removes calibration dependency; candidate for view-robustness.
- *Caveat:* error accumulation over long rollouts.
- *Source:* DexWild. *Status:* **leaning** adopt.

**Hand action-space alignment via fingertip retargeting.**
- *Why:* hands differ kinematically; copying joint angles breaks grasps. Solve for robot
  finger-joint angles whose fingertips match the human's (relative fingertip-to-fingertip /
  fingertip-to-palm vectors — DexPilot / Robotic Telekinesis family).
- *Source:* DexWild + DexMachina (RL route for harder cases).
- *Status:* **leaning** as default; revisit DexMachina RL route if retargeting underperforms
  on dexterous tasks.

**Matched wrist-camera mounting (human glove ↔ robot hand).**
- *Why:* identical placement → no visual domain gap on that input stream.
- *Source:* DexWild. *Status:* **leaning** adopt.

**Bimanual: append inter-hand (relative) pose to the observation.**
- *Why:* gives policy explicit relative-pose info between hands → easier bimanual coordination.
- *Source:* DexWild. *Status:* **leaning** adopt.

**Main visual-observation alignment is OPEN — especially with egocentric obs.**
- *Why:* DexWild uses wrist-cam-only; we plan egocentric (head-camera) obs where the human↔robot
  visual gap on the main view is larger and must be deliberately handled.
- *Action:* queue a targeted B search — mitigating human↔robot visual gap for head-cam obs,
  incl. inpainting / marker removal and view alignment.
- *Status:* **open — watch.**

**Vision encoder: prefer a pretrained ViT with hand-inclusive pretraining mix.**
- *Why:* pretraining-data distribution > scale; hand-inclusive mix suits dexterous tasks.
- *Lead:* Soup 1M (`_DH`) — Dasari et al. 2023 (*Data4Robotics*), MAE-style ViT pretrained on
  ImageNet + 100 Days of Hands dataset.
- *Source:* DexWild. *Status:* **leaning** — candidate encoder init.
