# we-teleop: recorded data — schema and semantics

> Detailed reference for the recorded Parquet dataset. Covers field semantics, shapes,
> data origin, action representation, and cross-stream synchronization.
> ← back to [code-structure](code-structure.md) · [decisions index](../decisions-and-caveats.md)
>
> Last updated: 2026-06-15

---

## State — origin and shape

**Column:** `observation.state` · **Shape:** `(qpos_dim,)` · **dtype:** float32

`qpos_dim` is the MuJoCo model's full qpos length. For Tianji Gento Luna this covers:
- Optional free-body DOFs for the base (position + quaternion if mobile base is enabled)
- 6 torso joints (`Gento Luna_Leg_J1–J6`)
- 2 head joints (`Head_J1`, `Head_J2`)
- 7 left arm joints (`Arm_L1–L7`)
- 7 right arm joints (`Arm_R1–R7`)

**Data origin — depends on hardware mode:**

- **With hardware connected (real robot recording):** `compose_state_vector()` overwrites the
  MuJoCo qpos with actual hardware-reported joint positions per section
  (`torso`, `head`, `left_arm`, `right_arm`) from `proprioceptive.joint_positions` in the
  driver snapshot. So `observation.state` is the **real robot's measured joint positions**,
  indexed into the full MuJoCo qpos address space.
- **Without hardware (sim only / preview):** `observation.state` is the MuJoCo sim state,
  which equals the IK-solved qpos sent to the sim.

Also recorded (same shape `(qvel_dim,)`, MuJoCo dof address space):
- `observation.joint_qvel` — actual joint velocities from hardware
- `observation.joint_cmd_pos` — last commanded joint position acknowledged by the robot
  (from `applied_action` in the driver snapshot — useful for measuring control lag)

Each of these has a paired `*_available` column (same shape, float32): elements are `1.0`
where the hardware reported a value, `0.0` otherwise. **Always check availability before using.**

---

## Force-related fields

All force/torque signals come from the robot hardware driver via the ZMQ snapshot
(`proprioceptive` dict). All have a paired `*_available` array.

### Per-joint signals (shape: `(qvel_dim,)`, indexed by MuJoCo dof address)

| Column | Hardware key | Meaning |
|--------|-------------|---------|
| `observation.joint_effort` | `joint_efforts` / `joint_torques` | Motor effort — raw motor torque τ_motor at each joint |
| `observation.joint_external_torque` | `joint_external_torques` / `joint_external_efforts` | **τ_ext** — external torque at each joint (physics-derived, contact proxy) |

**τ_ext detail:** τ_ext = τ_motor − τ_model, where τ_model is the model-predicted torque for the
current motion (gravity + inertia). What remains is the external contact contribution. The robot
exposes this directly via its API (no need to compute via dynamics model). This is the signal
used by FACTR and FACTR 2 for contact detection and phase labeling.

**Implication for design decisions:** the `joint_external_torque` column in the recorded data
directly provides the τ_ext signal from [force-feedback.md](../design/force-feedback.md) §4.
No offline computation needed — it is captured at collection time.

### Fixed-size wrench/IMU signals (shape: `(6,)`, per arm)

| Column | Shape | Meaning |
|--------|-------|---------|
| `observation.left_arm.base_force` | `(6,)` | 6D wrench [Fx, Fy, Fz, τx, τy, τz] at the **left arm base** (shoulder / proximal mounting point, `Arm_L1_Joint` region) |
| `observation.right_arm.base_force` | `(6,)` | Same for right arm |
| `observation.left_arm.base_gyro` | `(6,)` | **IMU at the left arm base (shoulder)** — [ωx, ωy, ωz, ax, ay, az]: 3-axis angular velocity (gyroscope) + 3-axis linear acceleration. Measures *motion* of the arm's proximal end; NOT a force sensor. |
| `observation.right_arm.base_gyro` | `(6,)` | Same for right arm |
| `observation.left_arm.flange_force` | `(6,)` | **6D EEF wrench [Fx, Fy, Fz, τx, τy, τz] at the left flange** (`Arm_L7_Flange` — tool mounting plate). Standard "EEF force/torque" in the literature. |
| `observation.right_arm.flange_force` | `(6,)` | Same for right arm |

**"Base" vs. "flange":** In this context, "base" is the *proximal* end of the arm (shoulder, where it mounts to the torso); "flange" is the *distal* end (tool plate, where the gripper/hand attaches). They are the two ends of the arm.

**`base_gyro` is not a force sensor.** It measures the kinematic state of the arm's shoulder — useful for detecting arm-base vibration, computing dynamic terms, or inertial navigation, but does not directly measure contact force.

**Which signal to use for force-feedback policies:**
- `flange_force` — directly at the tool center point; the canonical EEF wrench; most task-relevant.
  FILIC's contribution was computing this from joint torques via Jacobian inversion; here it's available as a direct measurement if the robot has a wrist F/T sensor (`flange_force_available` will be all-ones if fitted).
- `joint_external_torque` — per-joint τ_ext, arm joints only; what FACTR/TA-VLA use; broader joint coverage, available even without a dedicated wrist sensor.
- `base_force` — shoulder-mounted wrench; more distal from the task; captures whole-arm loading but less task-specific than flange.
- `base_gyro` — kinematic (not force); secondary signal for dynamic estimation, not primary contact proxy.

The `*_available` flags are critical here: if a sensor is not fitted (e.g. no flange F/T sensor),
the `_available` vector will be all zeros. Check before assuming data exists.

---

## Actions — what is recorded and where it comes from

The data records **two levels of action representation simultaneously:**

### Level 1 — Joint-space command (IK solve output)

**Column:** `action.joint_qpos` · **Shape:** `(qpos_dim,)` · **dtype:** float32

This is a copy of the MuJoCo qpos at snapshot time — the IK-solved joint positions that are
**published to the robot hardware** via `_publish_hardware_packet()`. It is the motor command.

In IL terms: yes, this is the action — the joint-space command resulting from the IK solve on
the operator's PICO controller poses. Training a policy on `action.joint_qpos` gives a
joint-space policy with absolute joint position targets.

**Note:** `observation.state` and `action.joint_qpos` start from the same MuJoCo qpos, but
`observation.state` is then overwritten with *actual* robot positions from hardware. The two
will differ by the robot's tracking error and the ZMQ round-trip latency (see §Sync).

### Level 2 — Cartesian-space IK targets (IK solve inputs)

| Column | Shape | Meaning |
|--------|-------|---------|
| `action.ik_target.wrist.left.position` | `(3,)` | Left wrist target position (IK input) |
| `action.ik_target.wrist.left.quaternion` | `(4,)` | Left wrist target orientation (wxyz) |
| `action.ik_target.wrist.right.position` | `(3,)` | Right wrist target position |
| `action.ik_target.wrist.right.quaternion` | `(4,)` | Right wrist target orientation (wxyz) |
| `action.ik_target.headset.position` | `(3,)` | Headset (head) target position |
| `action.ik_target.headset.quaternion` | `(4,)` | Headset target orientation (wxyz) |

These come directly from `RecordingSnapshot.target_*`, which are the PICO-derived wrist/headset
poses after calibration transforms. They are the *inputs* to the pyroki IK solver. Training on
these gives a Cartesian-space / EEF-space policy, which then requires IK at inference time.

### Hand targets

| Column | Shape | Meaning |
|--------|-------|---------|
| `action.hand.left` | `(20,)` | Left hand finger DOF targets (5 fingers × 4 DOF) |
| `action.hand.right` | `(20,)` | Right hand finger DOF targets |
| `observation.hand.left` | `(20,)` | Left hand actual joint positions from hardware |
| `observation.hand.right` | `(20,)` | Right hand actual joint positions from hardware |

### Chassis command

**Column:** `action.chassis_command` · **Shape:** `(3,)` — [v_x, v_y, ω_z] base velocity command.

---

## Cross-stream synchronization

The recording has two conceptually separate streams that must be merged per frame:

**Host stream** (operator-side, in-process):
- IK solve → MuJoCo qpos → `action.joint_qpos` + `action.ik_target.*`
- PICO session → `source.pico.*`
- Produced by the control thread at `fps` (default 30 Hz)

**Remote stream** (robot-side, over ZMQ):
- Hardware sensors → actual joint positions, velocities, efforts, τ_ext, forces, gyros
- Camera → `observation.images.egocentric` (JPEG, decoded to RGB)
- Applied action echo → last command the robot received + its timestamp
- Fetched by a dedicated **hardware recorder thread** via `DriverSnapshotClient`

### How they are aligned

The hardware recorder thread runs at the target fps, scaled by the remote sample rate:

1. Reads the latest `_latest_host_snapshot` (locked copy from control thread) — the current host IK state
2. Issues a ZMQ request to the robot-side driver (`DriverSnapshotClient`, 250 ms timeout)
3. Driver returns the latest available sample (joint sensors + camera + applied_action echo)
4. Merges both into a single frame dict via `build_frame_payload()`

**The merge is not perfectly time-locked.** There is a ZMQ round-trip delay. The host snapshot
and the remote sample are captured at slightly different instants.

### Timestamp columns for diagnosing alignment

| Column | Meaning |
|--------|---------|
| `recording.camera_time_s` | Relative time of the camera frame (from frame_timestamp_ns), origin = first frame |
| `recording.applied_time_s` | Relative time when the robot received/applied the command (applied_timestamp_ns) |
| `recording.sample_time_s` | Relative time of the hardware sample (wall clock) |
| `recording.sample_monotonic_time_s` | Same but monotonic clock |
| `recording.applied_seq` | Sequence number of the last applied command on the robot — compare across frames to detect stale commands |
| `recording.camera_frame_id` | Camera frame counter — compare across frames to detect dropped camera frames |
| `recording.sample_missing` | **1.0 if the hardware sample was unavailable**; the previous good sample is repeated as a placeholder |

**Using these columns:** `recording.applied_time_s` vs. `recording.camera_time_s` gives the
camera-to-command latency per frame. `recording.applied_seq` lets you check whether the robot
received a new command between consecutive recorded frames. `recording.sample_missing` flags
frames where the robot was unreachable — these should typically be excluded from training.

### State vs. action temporal offset

Because host IK state and hardware sensor state come from different moments:
- `action.joint_qpos` ≈ command sent ~1–2 frames *before* `observation.state` reflects it
- `observation.joint_cmd_pos` (from `applied_action`) is the most recent command the robot
  *acknowledged receiving* — use this to verify the command was actually applied

For training: the standard IL convention (action at t predicts what to do at t, applied to
state at t) holds approximately. For fine-grained temporal analysis or for force prediction
(where contact state matters), use the timestamp columns to align precisely.
