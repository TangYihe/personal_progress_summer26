# RL fine-tuning — options & decision guide

> Quick reference for when we reach the RL phase. **Not a paper survey** — one entry per decision
> point, with valid options, evidence, and caveats. Details live in the [lit survey](../survey/mixed-training.md).
>
> Last updated: 2026-06-09

---

## When to reach for RL at all

Trigger: IL plateaus below task-completion threshold on contact-rich / dexterous tasks.
This is expected to happen — ManipTrans and DexMachina both show pure kinematic retargeting + IL
is insufficient for high-dexterity contact tasks. Confirm empirically with our own data.

**Start scouting RL codebase in parallel with IL experiments** — setup time is significant.
Don't wait until IL visibly fails.

---

## 1. Bootstrap strategy — how to use demos

### Option A: BC init → on-policy RL with decaying demo gradient (DAPG)
- BC pretraining → Natural Policy Gradient (or PPO) with an additive demo-gradient term that
  decays geometrically across iterations (λ₀=0.1, λ₁=0.95 → ~zero by convergence).
- Key distinction from DDPGfD: demos stay in the gradient throughout training, not in a replay
  buffer. On-policy; scales to high-DoF.
- **Insight:** demos solve the exploration problem → you can then use sparse task-completion
  reward instead of dense shaped reward. This is DAPG's central finding.
- **Caveat:** sim-based in the original; the IL→RL recipe is hardware-agnostic but the policy is
  state-based.
- **For us:** robot teleop data = our demo set. BC on teleop → PPO with decaying demo loss.

### Option B: IL init → RL within diffusion denoising (RL-100)
- Train a diffusion policy with IL, then apply PPO at each denoising step (treating denoising
  steps as RL actions). Consistency distillation compresses to one-step real-time inference.
- **Insight:** works with sparse human-keypress reward on 5/8 real-world tasks. IL init provides
  sufficient exploration coverage; reward engineering largely dropped.
- **Caveat:** gripper-only (not dexterous). Integration into denoising is non-trivial to implement.
- **For us:** relevant if we adopt a diffusion policy backbone (DP/ACT-D). Watch as the
  cleanest real-world recipe for IL→RL on diffusion.

### Option C: Demo bootstrap → residual RL on top of base (ManipTrans / DexMachina pattern)
- Pretrain a base policy (generalist trajectory imitator, or kinematic retargeting), then learn
  a residual Δa = a - a_base constrained by interaction dynamics.
- **Insight:** decomposing motion imitation (base) from physics/contact correction (residual)
  shrinks the effective RL action space → tractable on high-DoF contact-rich tasks.
- **Caveat:** both works are sim-trained (Isaac Gym). Base policy in ManipTrans is a generalist
  imitator trained on hand-motion-only data (no objects); in DexMachina the base is kinematic
  retargeting. Neither uses an IL policy on robot teleop data as the base — that's our variant.
- **For us:** replace the sim-trained base with our IL policy trained on robot teleop data.
  Then residual RL on top. This is the most direct adaptation of their pattern to our no-sim setting.

**Default starting point:** Option A (DAPG) — simplest, best-evidenced for demo + sparse reward.
Option C (residual-on-IL) is the longer-term target for dexterous contact-rich tasks.

---

## 2. Reward selection

### Option A: Sim-privileged dense rewards — NOT our path
- Object pose + articulation state, hand-object contact distance, fingertip tracking error.
- Used by: ManipTrans (4-term dense: task + imitation + BC + contact), DexMachina (similar),
  Object-Centric `2411.04005` (hierarchical: high-level object-goal reward, low-level RL on fingers).
- **Blocker for us:** requires simulator with object state access and privileged contact info.
  We have no sim. Ruled out unless we later set up an IsaacGym/Genesis environment for specific tasks.

### Option B: Sparse task-completion + IL init — our most realistic path
- Binary +1/0 at episode end: success or failure. Source: human supervisor keypress once per
  successful episode (RL-100), or automated detector (object in goal zone, etc.).
- **Evidence:** DAPG — sparse reward + demos ≈ dense reward + demos on dexterous sim tasks.
  RL-100 — sparse human keypress works on 5/8 real-world gripper tasks; 1 task (continuous
  precision pushing) still needed dense automated reward.
- **Key condition:** IL init must be good enough to occasionally succeed, giving the sparse reward
  a learning signal. If IL never succeeds, sparse reward provides no gradient.
- **For us:** default reward design for real-world RL phase. Human keypress per successful episode
  is low-overhead supervision.
- **Caveat:** may be insufficient for very long-horizon or precision tasks (the Push-T case).
  Dense reward or reward shaping may be needed there.

### Option C: Joint torque as contact proxy (no sim, real hardware)
- Our robot has per-joint force feedback on all arm joints. Joint torques encode contact events
  without explicit contact sensors or sim.
- **Potential use:** reward shaping (penalize unexpected torque spikes, reward smooth contact),
  policy observation (force-conditioned policy), or a real-world substitute for sim contact reward.
- **Caveat:** joint torque ≠ fingertip force. The signal depends on robot configuration and
  reflects contact indirectly through arm dynamics. Needs calibration / modeling.
- **Status:** open — this is the thread the groupmates raised. Worth reading the force/impedance
  queue to understand how existing works use torque as a contact signal.
  → See [force-impedance-control/index.md](../survey/force-impedance-control/index.md).

### Option D: Classifier-based reward from goal images (Xu et al. `2212.09902`) — investigate later
- User provides K milestone image snapshots + a directed graph of milestone ordering. For each
  milestone, a binary classifier is trained: positive = goal images, negative = agent's own
  on-policy rollouts (updated live during training). Reward = log p(milestone | state).
- No demos, no manual reward formula, no human keypress during training. Only upfront input:
  goal images + milestone graph.
- The classifier is contrast-based: it sharpens as the agent improves and its rollouts start
  looking closer to the goal — keeping the reward signal informative throughout training.
- RL algorithm: SAC + DroQ. Policy is NOT bootstrapped by BC/demos — learns from scratch.
- **When to investigate:** if sparse task-completion (Option B) is insufficient and task admits
  a clean milestone decomposition.
- **Caveat:** requires upfront task decomposition into milestones — non-trivial for dexterous
  contact-rich tasks. More setup than a keypress; reward quality depends on milestone image quality.
- **Distinct from demo-based reward learning** (REWIND etc.) — no demos needed, just goal images.
  REWIND remains a separate lead if this is also insufficient.

---

## 3. Open question: contact without sim

The structural barrier from ManipTrans + DexMachina: both rely on sim-privileged contact info
in observations and rewards. For real-world deployment we need a substitute.

Leading options (to explore in order):
1. **Sparse task-completion reward** — sidestep contact reward entirely (see above)
2. **Joint torque as contact proxy** — real hardware, no extra sensors (Option C above)
3. **Visual contact inference** — infer contact state from RGB obs. Xu et al. `2212.09902`
   (vision-based real-world RL, no sim, no manual reward) is the key read here.
4. **Learned reward from demos** — REWIND and similar (Option D above; last resort)

---

## Reading still queued (relevant to this note)

- **Xu et al. `2212.09902`** — ✓ read 2026-06-09. Summarized as Option D above.
- **Park et al. `2501.04169`** — joint motion manifold, functional retargeting, real-world no-sim.
  Relevant if IL + sparse reward is insufficient and we pursue functional retargeting.
- **Force/impedance queue** — Adaptive Compliance, variable impedance, etc. Relevant to Option C
  (joint torque as contact proxy).
