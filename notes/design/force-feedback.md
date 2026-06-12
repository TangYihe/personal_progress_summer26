# Force & tactile integration — options & decisions

> Technical decisions for incorporating force/torque feedback into the policy (as observations,
> reward proxies, or both). Part of the [decisions index](../decisions-and-caveats.md).
>
> Last updated: 2026-06-09

---

## Context

Our robot has per-joint force feedback on all arm joints — an advantage over most prior work
that uses vision-only policies. Two open questions:

1. **Reward signal:** can joint torque serve as a contact proxy for RL, replacing sim-privileged
   contact info? (Relevant to [rl-finetuning.md](rl-finetuning.md) §Contact-without-sim.)
2. **Policy observation:** should force/torque be an input to the policy? If so, at what level
   and with what representation?

**Theoretical grounding (TA-VLA §3):** joint torque is not a direct contact-force reading — it
satisfies τ_measured = τ_model + J^T(q) · F_ext. The configuration-dependent Jacobian J(q) means
the same contact produces different torque patterns at different arm poses. A wrist F/T sensor
gives F_ext directly (configuration-independent); joint torque encodes F_ext only indirectly
through the arm geometry. The practical implication: if feeding raw torque to a policy, joint
positions must accompany it so the network has the context (q) to learn the relationship.

---

## Reading queue (updated 2026-06-10)

Reprioritized: practical design-space papers first; impedance theory deferred until we have
real-robot experience to anchor it. Full reading notes in [survey/force-related-works.md](../survey/force-related-works.md).

- **Torque-aware VLA (`2509.07962`)** — *done 2026-06-09*
- **In-Hand Manipulation (`2509.23075`)** — *summarized 2026-06-10*; deferred (fingertip-tactile focus, not our setting)
- **`2502.17432`** — *done 2026-06-10*
- **FACTR 2 (`2606.12406`)** — *done 2026-06-11*
- **FILIC (`2509.17053`)** — queued (dual-loop IL + impedance; EEF force from Jacobian)
- **Slow-fast trio (`2605.27886`) + ImplicitRDP (`2512.10946`)** — queued next
- **UniTacHand (`2512.21233`)** — low priority; revisit if we add fingertip tactile
- **Adaptive Compliance (`2410.09309`)** — theory-heavy; deferred
- **Variable impedance (`2603.14068`)** — theory-heavy; deferred

---

## Technical decisions

### 1. Signal source

Where does the force/torque signal come from?

**Option A: Per-joint torque from motor current** (τ = k_t × i)
- No extra hardware; signal exists in the motor driver already.
- Configuration-dependent: same contact force → different torque pattern at different arm poses.
- TA-VLA's choice: 7D per arm, raw scaled values from motor current.

**Option B: Wrist F/T sensor**
- Measures F_ext directly in the end-effector frame — configuration-independent.
- Same contact always produces the same 6D reading regardless of arm pose.
- Cleaner signal but additional hardware cost and calibration.

**Option C: Fingertip / tactile sensors**
- Most direct for dexterous manipulation — contact at finger level.
- Very limited coverage in existing work; hardware fragility is a concern.
- Pending: UniTacHand, In-Hand Manipulation papers.

**Current stance:** Option A (what we have). Option B is worth considering if contact detection
quality proves insufficient and hardware is available.

---

### 2. Signal placement in policy

*Architecture-agnostic framing: include torque with exteroceptive observations (images, language)
or with proprioceptive state (joint positions, velocities)?*

**Option A: Proprioceptive stream** — alongside joint positions and velocities.
- Torque is an internal body-state signal: measures what the robot's joints are experiencing,
  not what the external world looks like.
- VLA framing: decoder / action expert side.
  ACT framing: concatenated with joint-state tokens in the observation encoder.
  DP framing: added to the proprioceptive conditioning vector.
- Theoretical support: τ_measured = τ_model + J^T(q) · F_ext — passing torque alongside joint
  positions (q) gives the network everything it needs to implicitly learn the contact relationship.
- TA-VLA evidence: HSIC analysis confirms torque correlates more strongly with joint angles than
  with visual or language tokens → proprio placement is correct.
- TA-VLA ablation: decoder-side (proprio) consistently outperforms encoder-side (exteroceptive).

**Option B: Exteroceptive stream** — alongside images and language.
- TA-VLA ablation: consistently worse than Option A across all tasks.

**Decision: Option A.** Well-supported by both evidence and theory. Torque must be passed
alongside joint positions — not in isolation.

---

### 3. Modality encoding / adapter design

Each input modality needs to be projected into the policy's token space before it can be
processed. The question here is whether joint positions and torque share a single adapter or
each get their own.

**Option A: Shared adapter (pre-concat)**
- Concatenate joint positions and torque into one vector first, then pass the combined vector
  through a single linear projection (the joint position adapter).
- Simpler; one fewer learned module.
- **Problem when finetuning a pretrained model:** the joint position adapter was pretrained on a
  fixed-dimensionality input. Appending torque values changes that input vector, corrupting the
  pretrained representation. The adapter no longer receives what it was trained on.
- TA-VLA (DePre variant): worse than Option B in ablations.

**Option B: Separate adapters (post-concat)** (TA-VLA's choice)
- Joint positions pass through the joint position adapter as normal — unchanged.
- Torque passes through its *own* MLP adapter, producing a separate token in the same embedding
  space.
- Both tokens end up in the decoder's input sequence (torque token is prepended); they are
  separate tokens, not merged into one.
- TA-VLA (DePost variant): best-performing across tasks and both backbone architectures.
- CATFA (`2509.23075`) independently encodes 6D motor torque with a 2-layer MLP → 64D
  in a different architecture entirely (sim-to-real dexterous hand).
- FACTR (`2502.17432`) encodes external joint torque with a dedicated MLP → single token d,
  concatenated with vision tokens in an ACT encoder. Third independent confirmation.
- **Settled: MLP is sufficient for low-D joint torque encoding across architectures and tasks.**

**Reasoning (TA-VLA):** The decoder is a pretrained component with established expectations
about its input tokens. Option A disrupts the joint position token — changing what the adapter
receives invalidates the pretrained weights. Option B leaves the joint position token intact and
introduces torque as a new, cleanly-projected token the decoder can learn to attend to.

**Intuition:** treat torque as a new modality with its own voice, rather than forcing it to share
the joint position's slot. This follows the same principle used for every other modality (images
have their own visual encoder; language has its own embedding — you wouldn't merge them).

**Caveat:** the disruption concern (Option A's main drawback) is specific to *finetuning a
pretrained model*. When training from scratch, the adapter has no pretrained weights to protect,
so a shared adapter may work fine. Option B still follows better inductive structure regardless.

**Decision: Option B** for finetuning pretrained backbones. For scratch training, either may
work; Option B is still the cleaner inductive prior.

---

### 4. Signal preprocessing

How to handle the raw torque reading before feeding it to the policy?

**Option A: Raw torque** (TA-VLA's choice)
- Scale from motor current via τ = k_t × i; feed directly.
- Configuration-dependent, but passing joints alongside torque (§2) gives the network
  the context (q) to learn the relationship.
- Works if training data covers sufficient arm-configuration diversity.

**Option B: External torque / torque residual** (FACTR's choice)
- τ_ext = τ_measured − τ_model: subtract the robot's own predicted dynamics (gravity,
  inertia, Coriolis) to isolate the contact-induced component.
- Cleaner contact signal; less configuration-noise.
- Many robot arms (Franka Panda, KUKA iiwa) expose τ_ext directly via their API — no need
  to implement dynamics subtraction manually.
- **Action item: check which signal(s) our robot exposes.** Options: raw motor torque
  only / external torque only / both. This gates the choice between A and B.

**Option C: Explicit F_ext via Jacobian inversion**
- Recover the contact force: F_ext ≈ (J^T(q))⁻¹ · τ_residual.
- Configuration-independent; equivalent to a virtual wrist F/T sensor.
- Requires full kinematic model access and is numerically sensitive near singular configurations.
  Check with hardware team whether the kinematic model is available.

**Current stance:** Check hardware API first. Use Option B (external torque) if exposed
directly — cleaner signal with no extra work. Fall back to Option A (raw torque) if only
raw motor current is available; TA-VLA validates this path. Option C requires kinematic
model access — investigate later.

---

### 5. Temporal structure

Whether to include a history window for torque, and if so how to encode it.

**Background: torque vs. joint position history**

In TA-VLA (and most IL policies), joint positions use current frame only. Torque gets a
dedicated history window. The asymmetry reflects a signal-character difference:
- **Joint positions** are a slow-changing state signal — the current value fully characterizes
  the robot's configuration. History adds little.
- **Torque** is a transient event signal — contact spikes are brief. A single frame may catch
  the peak but miss the onset and decay. History is needed to see the event shape, not just
  a snapshot.

This distinction likely generalizes: any transient contact signal (wrist F/T, tactile) probably
also warrants a history window; slow-changing state signals generally do not.

---

**Decision A: whether to include history**

**Option A: Current frame only** (FACTR's choice)
- FACTR uses no history window for external torque and achieves strong results.
- May be sufficient if the signal is already clean (external torque with dynamics removed)
  or if the task doesn't require contact event shape — only current contact state.

**Option B: History window** (TA-VLA's choice) — 10 frames spanning ~2 seconds, uniformly
sampled. Captures contact onset, peak, and decay.
- Motivated by raw torque being noisy and transient — a single frame may miss the event shape.
- More important when using raw motor torque (Option A preprocessing) than external torque.

**Status: Open — try both empirically.** TA-VLA (raw torque, pretrained VLA) uses history;
FACTR (external torque, ACT from scratch) does not. The signal preprocessing choice likely
interacts with this: external torque may need less history because it's already cleaner.
Test once data is available.

---

**Decision B: how to encode the history**

**Option B1: Per-frame tokenization** (one token per history frame)
- Preserves temporal resolution.
- Adds many tokens to the decoder's input sequence (e.g. 10 frames → 10 extra tokens),
  substantially disrupting the pretrained sequence structure and attention patterns.
- TA-VLA validated the harm with a noise-token ablation: adding random dummy tokens (no real
  information, just extra length) sharply drops performance. The disruption comes from sequence
  structure change, not signal quality.
- **Note:** this concern is specific to finetuning a pretrained model. Training from scratch,
  per-frame tokenization might work fine since there are no pretrained patterns to protect.

**Option B2: Single aggregated token** (TA-VLA's choice)
- Compress all history frames into one token via a learned MLP, then add that one token.
- Sequence length changes by exactly one entry — minimal structural disruption.
- TA-VLA ablation: single token — 15/20 and 16/20; per-frame — 9/20 and 7/20.

**On aggregation method:** MLP > RNN > attention (TA-VLA ablation). With limited finetuning
data, RNN and attention have more parameters and underfit/overfit. A flat MLP is
parameter-efficient and sufficient for compressing a short torque history window.

**Decision: Option B2 (single aggregated token via MLP)** for finetuning pretrained backbones.
For scratch training, the comparison is an open question.

---

### 6. Role in the policy

Both options B and C below share the same high-level goal: **force the policy to actually
build a representation of force signals**, rather than ignoring them in favour of the dominant
visual input.

**Option A: Observation input only**
- Torque conditions the policy; no explicit mechanism to ensure it is used.
- Risk: the policy may learn to ignore force if visual input is sufficient for most training cases.

**Option B: Observation input + auxiliary torque prediction** (TA-VLA's choice — preferred)
- Policy also predicts torque as an auxiliary output during training. Forces the model to
  build a physically grounded internal representation — it cannot ignore a signal it also
  has to predict.
- TA-VLA: consistent improvement over input-only across tasks and both backbone architectures.
- **FACTR 2 cross-validation (2026-06-11):** in a non-VLA ACT-based setting, TA-VLA's auxiliary
  torque forecasting still outperforms plain torque-as-input. Confirms the strategy is
  **backbone-agnostic** — worth trying whether we start with ACT, DP, or a VLA.
- At inference, the torque prediction is discarded; only the action output is used.

  **Sub-decision: how to implement the auxiliary prediction**

  *Option B1: Separate prediction head*
  - Dedicated MLP head branching off the latent; separate loss term with tunable weight.
  - Works with any policy architecture, including regression-based ones like ACT.

  *Option B2: Expand the action/denoising target* (TA-VLA's choice)
  - Concatenate torque to the action vector so the policy denoises [action | torque] jointly.
  - No new module — just expand the output dimension. Architecturally simplest.
  - **Constraint:** requires a denoising-based policy (DP, flow matching).
    For ACT (regression), Option B1 is the natural equivalent.
  - Loss weight β tunable (TA-VLA: β=1.0 or 0.1 depending on variant).

  **Current stance:** Option B2 for diffusion/flow-matching backbones; Option B1 for ACT.

**Option C: Curriculum training — progressive visual degradation** (FACTR's approach)
- During training, progressively blur/degrade the visual input, forcing the policy to rely
  on force signals as vision becomes uninformative.
- No architectural change needed; purely a training schedule.
- FACTR result: force+no-curriculum → 61%; force+curriculum → 88% (+27 pp).
- Less intuitive than Option B: rather than explicitly predicting force, you penalise vision
  reliance indirectly. The policy learns to use force, but not necessarily to predict it.
- **Interaction with VLA finetuning:** degrading vision during finetuning risks corrupting the
  pretrained visual representations. More natural for ACT (training from scratch) than for
  finetuning a pretrained VLA backbone.

**Current stance:** Prioritise Option B (auxiliary prediction) — more principled, directly
validated on VLA backbones, architecture-agnostic in spirit. Option C is a useful fallback
or complementary technique, especially for ACT-based scratch training.

**Option C: Reward proxy for RL** (distinct from IL use above)
- Torque as a contact-detection signal for reward shaping or success detection in RL.
- Open — no clear evidence yet. See [rl-finetuning.md](rl-finetuning.md) §Option C.

**Option D: Contact-phase data reweighting** (FACTR 2 — FIRST)
- Label trajectory frames by phase (free-space / pre-contact / contact) using τ_ext threshold.
  Pre-contact = 1 s window before contact entry. Batch sampling probability ∝ phase weight.
- Orthogonal to A/B/C — a training-data intervention, not an architectural one. Can be combined
  with any of the above.
- Empirical motivation: contact phase has the highest validation loss → upweighting it directly
  targets the failure mode.
- Fully self-supervised from τ_ext; no manual annotation.

**Current stance (IL):** Option B (auxiliary prediction) as primary; Option D (FIRST) as a
complementary training trick — low-cost to add and empirically motivated.
**Current stance (RL):** open — pending further reading.

---

### 7. Which joints

**Option A: All arm joints** (TA-VLA's choice — 7D per arm, 14D bimanual)
- Simple; no design effort. Captures full kinematic chain.

**Option B: Distal joints only** (wrist + fingers)
- Higher signal-to-noise for contact events; smaller input.
- Untested — no ablation in any paper so far.

**Status: Open.** TA-VLA uses all joints; no ablation on this. Test empirically once data arrives.

---

### 8. Slow-fast structure

Whether torque should be processed at a different (faster) frequency than vision/language.

- TA-VLA: no slow-fast structure; single unified policy at one frequency.
- Pending: slow-fast trio (`2605.27886`) — likely the key read on this question.

**Status: Open.** Defer until slow-fast papers are read.

---

### 9. Low-level controller compatibility

What low-level controller mode (position / velocity / torque / impedance) is most compatible
with force-aware IL policies, and what does the policy output?

**Action space (what the policy predicts):**
- TA-VLA: joint position targets (implicit — not stated explicitly in the paper, but standard
  for π₀ on ALOHA with teleoperation). Torque is an auxiliary *training* signal only; it is
  not part of the control output and is discarded at inference.
- Implication: force-aware IL via joint torque input is fully compatible with standard
  position-controlled robots. No torque control mode required — torque is read-only from the
  controller's perspective.

**Controller mode (what runs underneath):**
- TA-VLA: not specified. Likely position control given the teleoperation setup, but implicit.
- All other papers so far are also silent on this.
- This question is best answered with real robot experience + targeted reading of impedance
  control papers (Adaptive Compliance, Variable impedance — currently deferred).

**Status: Partially answered.** Action space = joint positions (TA-VLA). Controller mode
remains open — revisit after first real-robot experiments.

---

## Open questions summary

- Does joint torque as reward proxy work for contact-rich dexterous tasks? (RL reading queue)
- What does tactile sensing add over joint torque for fingertip-level contact? (UniTacHand)
- Does a slow-fast structure add value for torque signals? (`2605.27886`)
- Which joints' torques are most informative — all vs. distal only? (empirical, needs data)
- Is the kinematic model available from the hardware team? (gates Option C preprocessing)
