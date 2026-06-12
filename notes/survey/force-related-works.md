# Reading notes — force & tactile sensing works

> Per-paper reading notes for workstream D. Design conclusions extracted from these notes
> live in [design/force-feedback.md](../design/force-feedback.md).
>
> Last updated: 2026-06-10

---

## TA-VLA (`2509.07962`) — read in full, 2026-06-09

**What it is:** Systematic ablation study of design choices for integrating joint torque into
a VLA-based IL policy (π₀ backbone, ALOHA robot, contact-rich bimanual tasks).

**Task / robot:** Bimanual manipulation (peg insertion, cloth folding, etc.) on ALOHA.
7D per-joint torque from motor current sensing.

**Method:** Finetune a pretrained VLA (π₀) with an additional torque adapter. Ablate:
signal placement (proprio vs. exteroceptive), adapter design (shared vs. separate),
history encoding (per-frame tokens vs. single aggregated token, MLP vs. RNN vs. attention),
auxiliary prediction (none vs. separate head vs. expand action space).

**Key findings (design-relevant):**
- Torque → proprio/decoder stream (HSIC analysis confirms correlation with joint angles, not vision/language).
- Separate MLP adapter for torque — preserves pretrained joint position adapter.
- 10-frame history (~2 sec) compressed to a single token via flat MLP — per-frame tokenization hurts by disrupting pretrained decoder's input structure.
- MLP outperforms RNN and attention for history aggregation (parameter efficiency, limited data).
- Auxiliary prediction via expanding action/denoising space to include torque — consistent improvement; at inference, torque prediction is discarded.
- Action space: joint position targets. Torque is sensing only, not a control output.

**What makes it distinctive:** The most complete ablation of torque-integration design choices
in a real-robot IL setting. High confidence in its conclusions as a starting point.

---

## In-Hand Manipulation / CATFA (`2509.23075`) — summarized, not read in full, 2026-06-10

**What it is:** Sim-to-real pipeline for in-hand articulated tool manipulation (scissors,
pliers, surgical tools) using a dexterous 6-DoF hand with fingertip tactile arrays + motor torque.

**Task / robot:** Dexterous hand, 6 active DoF. Signals: 36×44 tactile array per finger
(1,584 taxels) + motor torque (6D from current sensing).

**Method:** PPO in sim with privileged obs → distill to base policy (joints + command only)
→ BC-train cross-attention adapter (CATFA) on <50 real rollouts. Base policy frozen.

**Key findings (design-relevant):**
- Motor torque (6D) encoded with a 2-layer MLP → 64D. Independent confirmation that MLP
  is sufficient for low-D torque encoding.
- Spatial tactile arrays (1,584 taxels) encoded with a CNN → 64D. The encoder choice
  follows signal dimensionality and structure (MLP for low-D, CNN for spatial).
- No history window for either signal — cross-attention on current frame only at 30 Hz.
  Possible explanation: high spatial dimensionality of tactile may substitute for temporal
  context in their task.

**Relevance to us:** Limited — this is primarily a dexterous fingertip-tactile paper with
sim-to-real transfer, not aligned with our setting (real-robot IL from teleop data, arm-level
torque, VLA backbone). The torque encoding detail is the one directly applicable finding.

**Status:** Deferred — detailed reading deprioritized; summary sufficient for now.
Revisit if we add fingertip tactile sensing.

---

## FACTR (`2502.17432`) — summarized, 2026-06-10

**What it is:** Force-aware ACT-based IL policy for contact-rich manipulation. Key contribution
is a curriculum training strategy that forces the policy to learn from force signals.

**Task / robot:** Contact-rich manipulation tasks on Franka Panda / KUKA iiwa class arms.
Signal: external joint torque τ_ext (dynamics-subtracted), exposed directly by the robot API.
Gripper force from motor current with EMA filter (α=0.1).

**Method:** ACT encoder-decoder transformer (CVAE component dropped; plain MSE loss).
ViT vision encoder (pretrained) + MLP force encoder → tokens concatenated → fed into
transformer encoder. Progressive visual blurring curriculum during training.

**Key findings (design-relevant):**
- External torque (τ_ext = τ_measured − τ_model) used, not raw motor torque. Many robot arms
  expose this directly via API. Check which signal(s) our robot provides.
- MLP → single force token, concatenated with vision tokens in the encoder. Third confirmation
  that MLP is sufficient for low-D torque encoding.
- No explicit joint positions as policy input — image + τ_ext only. (Possible because
  τ_ext already has dynamics removed; or visual input implicitly encodes pose.)
- No history window — current frame only. Works despite FACTR using raw/single-frame signal.
- No auxiliary prediction loss.
- Curriculum: progressively blur vision during training → forces policy to use force signal.
  Without curriculum: 61% success; with: 88% (+27 pp). Force alone without curriculum adds
  only +40 pp over vision-only baseline (21%).
- The curriculum and TA-VLA's auxiliary prediction serve the same goal: preventing the policy
  from ignoring force signals. Preferred approach: auxiliary prediction (TA-VLA) — more
  principled and compatible with pretrained VLA backbones.

**Relevance to us:** High for ACT-stage experiments (our IL baseline). Force token placement
in ACT encoder is confirmed. Preprocessing choice (raw vs. external torque) is a hardware
question to resolve. Curriculum is a useful backup technique.

---

## FACTR 2 (`2606.12406`) — read 2026-06-11

**What it is:** Follow-up to FACTR. Two contributions for arms without dedicated force sensors:
NEXT (neural τ_ext estimation) and FIRST (contact-phase-weighted batch sampling).

**Task / robot:** Contact-rich manipulation on commodity arms (no dedicated force sensor).

**NEXT — neural external torque estimation:**
- τ_ext = τ_measured − τ_model. Goal: approximate τ_model (free-space dynamics) from data.
- Train a small neural net on ~10 min of unstructured free-space motion to predict τ_model
  from a short proprioceptive history (q, q̇, q̈). τ_ext = τ_measured − τ_net output.
- Training takes ~1 min. Estimates comparable to hardware torque sensors.
- *Relevance to us:* our arm exposes τ_ext directly via API, so NEXT is not needed. But this
  confirms what the API is computing under the hood — useful for understanding preprocessing.

**FIRST — force-informed re-sampling:**
- **Contact labeling:** τ_ext L1 norm vs. two thresholds (hysteresis). Pre-contact = 1 s window
  before contact entry (based on control frequency). Three phases: free-space / pre-contact / contact.
- **Batch reweighting:** sampling probability ∝ w(s_t) / Σ w(s_j) per phase label. Contact and
  pre-contact are upweighted; free-space is downweighted.
- Labeling is fully self-supervised from τ_ext — no manual annotation.
- *Relevance to us:* directly portable. The phase weights are a tunable hyperparameter; their
  values are a starting point.

**Key findings from experiments:**
- TA-VLA's auxiliary torque forecasting (predicting future τ in the action space) still
  outperforms plain torque-as-input even in non-VLA settings. Backbone-agnostic confirmation —
  the auxiliary prediction strategy is worth trying regardless of whether we use ACT, DP, or VLA.
- Contact phase has the highest validation loss in training curves → FIRST's reweighting is a
  principled intervention, not a heuristic.

**Relevance to us:** FIRST is a concrete training trick to adopt. The loss-curve finding gives
us a diagnostic: if contact phase validation loss is high, reweighting is the right lever.

---

## Reading queue

| Paper | arxiv | Priority | Notes |
|-------|-------|----------|-------|
| FACTR 2 | `2606.12406` | **now** | NEXT (neural τ_ext estimation, no sensor needed) + FIRST (contact-phase oversampling); directly resolves our τ_ext open action item |
| FILIC | `2509.17053` | high | dual-loop IL + impedance control; EEF force from joint torque via Jacobian inversion |
| Slow-fast trio | `2605.27886` | next | slow-fast structure question for §8 |
| ImplicitRDP | `2512.10946` | next | slow-fast structure question |
| FACTR | `2502.17432` | *done 2026-06-10* | ACT-based force integration |
| UniTacHand | `2512.21233` | low | tactile, dexterous, sim-free; revisit with tactile |
| Adaptive Compliance | `2410.09309` | deferred | theory-heavy; revisit after real-robot experience |
| Variable impedance | `2603.14068` | deferred | theory-heavy; revisit after real-robot experience |
