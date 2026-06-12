# Ideas parking lot

> Novel ideas that emerged from discussion or reading — not yet validated, not yet design
> decisions. Keep here so they don't get lost. Promote to the relevant design doc if/when
> we decide to pursue.
>
> Last updated: 2026-06-11

---

## q_IK as virtual target for implicit force encoding

**Origin:** discussion 2026-06-11, from thinking about wiping/sustained-force tasks under
position control.

**The problem it solves:** with standard position-controlled IL, the policy learns to predict
q_actual — the joint positions the robot actually reached. When in contact, q_actual is pinned
at the surface. The policy never learns to command positions "into" the surface, so it can't
implicitly encode how much force to apply.

**The idea:** in our VR teleop setup, the control chain is:

> VR controller → desired EEF pose → IK → **q_IK** → position controller → q_actual

When the robot is in free space, q_IK ≈ q_actual. When in contact, q_IK is the joint solution
for where the operator *intended* the EEF to be — the position controller can't reach it because
the surface is in the way. **q_IK is therefore already the virtual compliance target**, encoding
the operator's force intent as a position offset beyond the surface.

**How to use it:** record q_IK (the IK solution) as the IL demonstration target instead of
q_actual. The policy learns to predict targets that are "inside" the surface during contact
phases. At inference, commanding q_IK to the position controller generates a contact force
proportional to K × (q_IK − q_actual), recreating the intended force.

**What's appealing:**
- No controller change at inference — still standard position control.
- No explicit force output from the policy.
- The virtual target is available for free from the existing teleop pipeline; no torque
  computation needed.

**Caveat:** if the operator commands the EEF far inside the surface (large overshoot), q_IK
becomes physically unreliable as a target. Requires competent operators applying reasonable
force. May need a clipping/sanity check on the q_IK − q_actual offset during data collection.

**Consistency requirement:** the position controller stiffness K must be the same between
data collection and deployment. The force generated at inference is K × offset, so if K changes,
the implicit force mapping breaks.

**Related idea (EEF space):** if we operate directly in EEF space (policy outputs EEF poses,
not joint angles), the same split exists naturally: desired EEF pose (virtual target) vs. actual
EEF pose (what was reached). Same principle applies without needing IK at all.

**Connection to existing literature:** related in spirit to compliance/admittance learning papers
that record the "unconstrained intended trajectory" as the IL target. The torque-based virtual
target computation (q_virtual = q_actual + K⁻¹ · J⁻ᵀ · τ_ext) is a more principled version
of the same idea when the IK solution is not directly available (e.g. leader-follower teleop).

**Status:** untested. Worth keeping in mind when designing the data collection pipeline and
deciding what to record. Most relevant if our tasks require sustained contact force (wiping,
pressing, inserting with compliance).
