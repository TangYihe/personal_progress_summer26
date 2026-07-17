# Single-side teleop (sides argument) — spec + implementation plan

> Written 2026-07-16 evening for the 07-17 morning session. Goal: build BEFORE cube
> collection (est. ~2 h incl. tests). Base: `wip/collection-integration-20260716`
> (feature work goes on `feat/task-reset-pose` or a new branch off it).
> ← context: [week-2026-07-13](../weekly-notes/week-2026-07-13.md) · HANDOFF runbook
>
> **STATUS 07-17: BUILT (`ac81a41`) + ROBOT-VALIDATED same day** — full test plan incl.
> recording loop; parquet proof: left-hand action = bit-identical non-zero grasp
> constants, left-arm action deviation 0.0 (`rubik_test_v0__20260717T105348`). One spec correction: the idle hand must NOT stay `None` — a `None`
> hand action is *recorded as zeros* (`encode.py:_hand`) and fails the required-
> modality check; instead the control loop fills the idle side each tick from the
> active task pose's captured hand targets (recorded action = applied command =
> grasp constants). Both hand modalities stay required — open question 1 resolved
> (modality source = driver-applied command + measured sensor), question 2
> confirmed (IK freezes inactive joints at seed; regression test added).

## What the user wants (in their words, confirmed)

- During **data collection** for the cube-twist task: teleop ONLY the right arm + right
  hand. The left arm/hand must still be **driver-controlled** (PD-held — no droop!) but
  must NOT listen to PICO/IK: it only moves during resets, to the task init pose
  (where the left hand grasps the cube via the staged arms→fingers reset).
- During the **pose-capture session**: normal bimanual teleop (existing profiles) to
  set up the grasp, `setpose rubik_twist` once.
- Switching behavior via **profile** is the accepted UX: capture with
  `pico_world_wuji_hands`, collect with a new single-side profile.

## Design

New optional profile key `sides` (default `[left, right]`), e.g. new profile
`config/teleop/pico_world_wuji_right.yaml`:

```yaml
schema_version: 1
arms: pico_world
head: hold
torso: hold        # matches our local convention (upstream default is ik/pico — see note 4)
hands: wuji
sides: [right]
required_recording_modalities: [image, right_hand]   # ← see open question 1
```

Mechanism (all operator-side, no driver/wire changes):

1. **IK**: only the selected side's wrist stays a target; the other arm's joints leave
   `active_joints`. After any reset, `reset_inputs_to_measured()` re-seeds the solver
   from measured feedback (= task pose), so the inactive arm's joints stay frozen at
   the task pose in every solution and the driver PD-holds them there.
2. **Hands**: glove input restricted to the selected side (receiver retargeters +
   `prepare_wuji_network(sides=...)` → only ONE USB adapter needed). The idle side's
   slot in `HandAction` stays `None` → driver holds the last commanded left hand =
   the captured grasp from the staged reset. The cube stays held.
3. **PICO**: the idle controller is simply never read as a target — can stay on the rack.
4. **Resets unchanged**: `usepose rubik_twist` → `R` ramps BOTH arms (reset is a
   whole-body driver ramp; sides only gate *teleop following*), fingers close last,
   recording starts, and only the right side tracks the operator. Exactly the wanted flow.

## Data-format answer (user asked explicitly)

**Record the full canonical action vector, unchanged — do NOT record only the right
side.** The strict schema v10 keeps `action.joint_qpos` at the full 22-joint canonical
order + both hands; the left side simply appears as constants (frozen task pose /
held grasp). This is desirable for IL: the policy learns to output those constants,
and deploy (same full-body command path) reproduces them by construction — no schema
change, no train/deploy mismatch. Precedent: the 07-02 state-OOD investigation
established near-constant dims are fine; the bug back then was mis-indexing, not
constancy. ⚠️ Do not add per-side schema variants (CLAUDE.md forbids schema forks).

## Files to touch

1. `src/teleop/runtime/launcher.py` — `LaunchProfile.sides: tuple[str, ...] = ("left", "right")`
   + validation (nonempty unique subset of left/right); parse from profile yaml.
2. `config/teleop/pico_world_wuji_right.yaml` — new profile (above).
3. `src/teleop/runtime/control_process.py` (~lines 188–280) — build
   `enabled_target_names`, `target_bodies/offsets`, `active_joints` from
   `profile.sides`; pass sides into the hands input construction; keyboard path too.
4. `src/we/cli.py` + supervisor — the wuji network preflight call must pass the
   profile's sides (currently preflights both gloves).
5. Tests: launcher validation; `tests/contracts/test_public_cli.py`
   `test_teleop_profiles_are_discoverable...` pins the EXACT profile list → add the
   new profile there; a control-construction test if cheap.
6. `docs/teleoperation.md` — short paragraph (single-side teleop + task poses).

## Open questions to resolve FIRST (≤15 min of reading)

1. **`left_hand` recording modality source**: is it the operator glove stream or the
   driver's measured hand positions? If operator-side, the single-side profile must
   require only `[image, right_hand]` (else every episode fails on missing left hand).
   If driver-measured, keep requiring both. Check `src/teleop/data/recording/` +
   where modalities are validated. NOTE: launcher validation currently REQUIRES both
   hand modalities when `hands != "hold"` ("hand profiles must require both measured
   hands") — this rule must learn about sides.
2. **IK inactive-joint semantics**: verify `MujocoIKSolver` holds inactive joints at
   the re-anchored seed (not at `rest_joint_positions` = default pose). If it pulls
   toward rest, the frozen arm would drift command-wise — must pin to seed.
3. Whether `hands check` / preflight helpers need a sides parameter too (nice-to-have).

## Test plan

1. Unit: launcher sides validation + profile list contract test.
2. Sim (`keyboard` variant or new profile w/o driver): activate a task pose, reset,
   verify left-arm command stays exactly at pose while driving the right arm around.
3. Gloves: right-glove-only preflight passes with one adapter.
4. Robot: capture grasp pose bimanually → switch profile → `usepose` → `R` →
   verify left arm+hand rock-solid while right teleops; then first cube episodes.

## Explicitly deferred (user said "at least for now")

- Teleoping the left side while right is frozen (symmetric case) — design supports it
  via `sides: [left]`, but don't spend time validating it.
- Mid-session side switching without restarting teleop.
