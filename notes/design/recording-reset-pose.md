# Recording reset logic — current mechanism + reset-to-grasping-pose design (plan 1.a)

> Read on `develop` 2026-07-15 (workstation tree, post-switch). Prep for the team discussion.
> ← back to [week-2026-07-13](../weekly-notes/week-2026-07-13.md)

## Current mechanism (as of develop, 07-15)

**The reset pose is a single hardcoded constant.** `_TIANJI_GENTO_LUNA_OPERATIONAL_JOINTS`
in `src/teleop/control/kinematics/init_pose.py:19` — a joint-name→angle dict (arms at
±90° configs = "arms at side pointing forward", legs 60/−70/10, head 0). Loaded ONCE at app
launch (`apply_tianji_operational_pose` → `RuntimeBundle.init_qpos` → `app._init_qpos`).

**Key flow (all in `src/teleop/runtime/app/main.py`):**
- `r` to start (`command_record_start:696`): pause → `app.reset()` (sim qpos ← `_init_qpos`,
  `reset():1596`) → `_send_hardware_reset(start_qpos=<physical pose>)` → re-align targets to
  robot → resume → record. Skip-ramp optimization: if `_recording_reset_pose_ready` (robot
  already holds the reset pose, e.g. right after a save), the ramp is skipped.
- `s` save / `r` discard (`_finish_recording_episode:808`): stop → pause → `reset()` →
  async hardware reset → robot ramps back to init pose after EVERY episode.
- The hardware reset (`_send_hardware_reset:1951` + `command_bridge.py`): sends a
  `state="reset"` packet; the bridge interpolates physical-pose → init-pose **in joint space
  over 2.0 s** (`SOFT_RESET_DURATION_S`, `command_bridge.py:25`).

**Hands: reset does NOT touch them.** `hand_targets` is set only in `__init__` (`main.py:1473`)
and every reset packet just re-sends the last commanded hand pose. (This is the known
"hands don't open on reset" behavior — for our purpose it's a FEATURE: a grasping left hand
stays closed through reset for free.)

**Deploy shares the same path.** Post-rewrite deploy runs inside TeleopApp; `Tab` reset uses
the same `reset()` + `_send_hardware_reset()` → same `_init_qpos`. So changing the pose at the
source automatically keeps teleop, collection, and deploy consistent — no train/deploy
start-pose mismatch by construction.

**Separate thing, don't confuse:** the driver's own `--reset-once` / `--move-to-init` home
pose is driver-side (Orin) and unrelated to this workstation-side reset target.

## Design options for reset-to-grasping-pose

**Option A — per-task reset pose in config.** Teleop profile (or recording-task config) gets
an optional `reset_qpos` / joint-map override; falls back to the operational pose. Explicit +
reproducible; but authoring a 25-joint grasping pose by hand is painful.

**Option B — "capture reset pose" hotkey (preferred?).** During teleop, operator grasps the
cube, presses a key → snapshot `data.qpos` (+ hand targets) as the session reset pose; persist
to a file so collection AND later deploy load the same pose. Ergonomic; pose is by-construction
reachable and IK-consistent. Persisted artifact should live with the task/recording metadata.

**Hybrid:** B captures → writes the config A reads. Capture once, reuse across sessions.

## Caveats to raise in the discussion

1. **Joint-space linear ramp (2 s) to a grasping pose can collide** — from arms-at-side to
   hand-around-cube may sweep through the table/cube. May need a via-point or ramping only
   after the operator confirms clearance.
2. **The cube itself can't be reset by software** — grasp pose is restored but the cube must
   physically be re-seated in the hand; operator procedure question (does the left hand stay
   closed while repositioning the cube? note `Tab`/reset keeps it closed today).
3. **Skip-ramp flag** (`_recording_reset_pose_ready`) assumes reset pose == the one it last
   ramped to; a per-task pose must invalidate it on pose change.
4. **Synergy: the "lower default torso pose" TODO is the same mechanism** — leg J2/J3/J4 in
   the same joint dict set robot height; a per-task/configurable pose solves both at once.
5. **Recorded episodes start at the next control action after reset** (boundary snapshot
   excluded, `_publish_recording_start_action`) — episode 0th frame will be the grasping
   pose; good, that's exactly the start-state distribution we want for IL.
6. Ask team: does the planned refactor touch `init_pose.py` / TeleopApp reset? Build this
   before or after?
