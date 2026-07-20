# PICO right-controller position snap → arm lurch (2026-07-20)

**Status:** root-caused from recorded data · to report to PICO/teleop owner
**Severity:** ⚠️ safety — arm lurched at ~2.7 rad/s (~155°/s) with a person's hand
inside the workspace (visible in the egocentric frames).

## Symptom

During the bimanual pose-capture session (`pico_world_wuji_hands`, merged
`wip/collection-integration-20260720` tree, 50 Hz), the right arm twice moved fast
and unexpectedly while the operator held the controller still. Controller had
line-of-sight to the operator ("definitely visible") — but see root cause.

**Recording (44 s, whole event captured):**
`we-teleop/artifacts/data/robot/2026-07-20-10-24-48__unexpected_movement_0720`

## Evidence (all from the parquet — commanded vs applied vs measured + raw IK targets)

**Pipeline health first (`we data report`): CLEAN.** 0 missing actions/control ticks/
skipped driver ticks/lookahead misses/buffer overflows/state-alignment misses; latency
state→action median 2.3 ms. This is NOT the 07-17 driver-fault class. Driver exonerated:
command jumped FIRST, state followed (opposite ordering of a driver fault).

**Two spike events, both right side, both mid-`human` source (no pause/resume
transitions in the whole episode → NOT a re-anchor lunge):**

| t | joint | commanded peak | measured peak | wrist tgt step | rotation step |
|---|---|---|---|---|---|
| 34.46 s | Arm_R4 (elbow) | 4.9 rad/s | ~2.7 rad/s | 3.1 cm/frame (~7 cm total) | **0.3°** |
| 39.44 s | Arm_R4 (elbow) | **10.8 rad/s** | ~2.7 rad/s | 4.3 cm/frame (~10 cm total) | **0.0°** |

Key discriminators:
- **Translation-only steps (0° rotation)** — physically implausible for hand motion,
  classic for VIO position correction (orientation rides the IMU and stays right;
  position snaps on visual reacquisition).
- **Step-and-hold shape** (target near-flat before, teleports, stays) — not a
  spike-and-return glitch, not smooth human motion. Operator was holding still.
- **Left controller target unaffected at both events** → NOT a global SLAM/world-frame
  re-localization; right-controller-specific tracking loss.
- Targets not bit-identical during the flat stretch → not a stream freeze; upstream
  smoothing spread the snap over 2–3 frames (still 3–4 cm/frame into IK).
- Amplification chain: ~cm task-space step → IK swings the elbow (10.8 rad/s
  commanded) → driver shaper clamps to ~2.7 rad/s real motion = the visible lurch.
  Driver and host safety behaved as designed; nothing in the chain BOUNDS a
  task-space target step.

Plots + before/after egocentric frames: [evidence-2026-07-20/](evidence-2026-07-20/)
(`unexpected_movement_0720.png`, `ev2_before_f1947.jpg`, `ev2_after_f2002.jpg`).

## Interpretation

Right controller briefly lost visual tracking (visible to the *operator* ≠ visible to
the *headset cameras* — third-person rig, occlusion likely), pose drifted on IMU, then
snapped on reacquisition. Known-shaky third-person tracking (see 07-06/07-15 notes)
makes this a recurring risk, not a one-off.

## Suggested fixes (for the teleop owner — their call)

1. **Task-space step/velocity clamp on IK targets** (e.g. cap cm/tick with catch-up)
   so tracking snaps become brief smooth catch-ups instead of lurches — safety feature,
   operator-side, independent of which headset is used.
2. Feeds the **PICO → Quest switch** decision with quantified evidence.
3. Interim ops: controller reset before session (their earlier tip), mind headset
   placement/occlusion, **keep hands out of the workspace during teleop**.

## Repro/analysis snippets

Spike attribution: diff `action.joint_qpos` vs `action.human.joint_qpos` vs
`observation.state` per frame over `recording.action_time_s`; raw targets in
`action.ik_target.wrist.{left,right}.{position,quaternion}`;
`recording.action_source` for engagement transitions. Native rerun viewer works on
this tree: `we data rerun <recording> --episode 0`.
