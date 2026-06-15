# Labmate sync — codebase investigation & open questions

> Prep doc for the meeting with 师弟 (teleop/hardware side).
> Covers what we found from code inspection, what's already documented, and what needs syncing.
> ← back to [code-structure](code-structure.md) · [data-schema](data-schema.md)
>
> Investigation date: 2026-06-15 · Meeting: TBD · Answers: TBD

---

## Investigation status

Repo investigated: `~/we-teleop/`. Full structural map in [code-structure.md](code-structure.md);
data schema detail in [data-schema.md](data-schema.md). Summary of what's confirmed from code:

- **Recording format:** Parquet, per-episode, in `data/robot/<task>__<timestamp>/`. Schema version 7.
  All sensor streams recorded (joint positions, velocities, effort, τ_ext, base force/gyro,
  flange force), each with an `*_available` availability mask.
- **Training:** ACT and DP backends are original submodules (`third_party/policy/{act,dp}`),
  connected via a Parquet→HDF5 adapter layer in `src/train/policies/act.py`. Z-score
  normalization per dimension (std clipped to 1e-2 min). Isolated uv venvs per backend.
- **Action spaces available:** joint-space (`action.joint_qpos`, full qpos) or EE-space
  (21D IK target concat: left pos+quat, right pos+quat, headset pos+quat). Hand DOFs are
  recorded but not wired into training.
- **Open-loop eval:** MuJoCo only — policy gets ground-truth recorded images + states as
  input, never its own rollout. No real-robot closed-loop eval available yet (`plan` command
  raises `NotImplementedError`).
- **Control modes confirmed in driver:** `position`, `impedance` (joint spring-damper),
  `imp_cart` (Cartesian spring-damper), `force_impedance` (hybrid position + contact force).
- **Force signals confirmed available:** τ_ext (`joint_external_torques`) for arm joints only
  (torso/head return None). `flange_force` column exists; actual availability depends on
  whether the wrist F/T sensor is fitted — need hardware confirmation.

---

## 1 · Cameras

**What we know from code:**
- Exactly one camera in the recording schema: `observation.images.egocentric` — head-mounted
  RealSense D405 at `Head_J2_Link`, 640×480 RGB.
- Wrist camera server scripts exist (`scripts/servers/wrist_cam_server.py`,
  `wrist_cam_viewer.py`) as standalone tools, but wrist streams are **not in the recording
  schema** and not integrated into training.

**To sync:**

**Q1.1** Are wrist cameras planned for recording integration? Timeline?

**Q1.2** When added: as new `observation.images.*` columns in the same Parquet, or a
separate dataset? Who drives the schema change?

**Q1.3** Is the current head camera setup (D405, 640×480, fixed at `Head_J2_Link`) the
final configuration, or will placement / resolution change?

**Answers:**

---

## 2 · Demo data

**What we know from code:**
- `we data rerun <recording_root>` visualizes saved datasets using the Rerun SDK.
- Schema is fully understood (see [data-schema.md](data-schema.md)). We know what fields
  exist and their shapes, but have not seen real values.

**To sync:**

**Q2.1** Is there an existing recorded episode we can get access to — even a short one —
to inspect real image streams, force signal magnitudes, and availability flags?

**Q2.2** What data is currently on the robot-side machine, and how do we rsync it over?
(`scripts/rsync.sh` exists but we haven't verified the paths.)

**Answers:**

---

## 3 · Replay

**What we know from code:**
- `TeleopReplayManager` (`data/replay.py`) exists and is used inside the teleop app's
  replay cycle (`run_replay_cycle`).
- The hardware bridge (`_driver_bridge`) is active during replay mode and
  `_publish_hardware_packet` is called — suggesting real-robot replay may already be
  functional, but this is unconfirmed.
- `we data rerun` is a separate, visualization-only path (Rerun SDK, no robot).

**To sync:**

**Q3.1** Does replay mode (`we teleop` + replay commands) actually publish commands to
the real robot, or is it MuJoCo-only visualization?

**Q3.2** If not yet real-robot: is hardware replay (replaying `action.joint_qpos` to the
robot for success-rate sanity checks) planned, and when? This is our primary workstream A
validation step once data arrives.

**Answers:**

---

## 4 · Action space & DOF coverage

**What we know from code:**
- Current ACT training uses `action.joint_qpos` (full qpos vector = torso + head + arms)
  or the 21D IK EE target. **Hand DOFs are not in the training pipeline.**
- Schema records hand DOFs in `action.hand.left/right` (20D each) and
  `observation.hand.left/right` — data is there if collected with gloves.
- Torso (6), head (2), left arm (7), right arm (7) are all covered in the qpos vector and
  τ_ext / effort signals (except τ_ext on torso/head returns None from the driver).

**To sync:**

**Q4.1** In current data collection sessions, which subsystems are actually moving — whole
body (torso + head + arms), or just arms? Is the torso locked by default?

**Q4.2** **Hand integration:** If hand control is part of the task, who adds hand DOFs to
the ACT/DP training adapter? This is a non-trivial change (action dimension grows, policy
architecture may need adjustment). Timeline?

**Q4.3** Can we start with an arm-only configuration (torso + head locked, only arms move)?
This simplifies the action space significantly for early IL experiments.

**Q4.4** If head is locked: does the head camera still observe the relevant workspace
(tabletop, hands)? Does the head currently track the task during teleop?

**Answers:**

---

## 5 · Development plans

**What we know from code:**
- Policy training (`src/train/`) is inside this repo. A template exists for adding new
  policy backends (`train/policies/_template.py`).
- Live deployment is explicitly disabled (`plan` command raises `NotImplementedError`
  with a note to reintroduce with code + tests + docs).

**To sync:**

**Q5.1** Is policy development (new architectures, force integration, human-data co-training)
expected to stay in this repo, or will a separate training repo be created?

**Q5.2** If staying here: what is the process for contributing to the training adapter
(PR, direct push, etc.)?

**Q5.3** Any planned breaking schema changes in the near term (schema version bumps) —
e.g., adding wrist cameras, changing joint coverage? Good to know before building on the
current format.

**Q5.4** Who is leading deployment / closed-loop eval on the real robot, and what is the
rough timeline? (The `plan` command infrastructure needs to be built before we can run
real-robot policy eval.)

**Answers:**

---

## 6 · Robot API & control modes

**What we know from code** (`robots/tianji/driver.py`):

Four control modes confirmed in `GentoV0404Driver`:

| Mode | SDK call | What it does |
|------|----------|--------------|
| `position` | `switch_to_position_mode` | Pure position control — drives to target, fights external forces |
| `impedance` | `switch_to_imp_joint_mode` | Joint-space spring-damper: τ = Kp×error + Kd×vel_error. Compliant to external forces. Current gains: Kp=(3,3,3,3,0.5,0.5,0.5), Kd=(0.6,0.2,0.2,0.16,0,0,0) |
| `imp_cart` | `switch_to_imp_cart_mode` | Cartesian-space spring-damper. Kp/Kd in 7D (x,y,z,rx,ry,rz,nullspace). Allows per-axis compliance tuning. |
| `force_impedance` | `switch_to_imp_force_mode` | Hybrid position + contact force: sends both joint position target (per cycle) and a force direction + magnitude. Limits: ±10 N, 30 mm, 1 Nm, 10 deg. |

**Key distinction clarified:**
- `impedance` and `imp_cart` both still track a position target, but compliantly — external
  forces deflect the robot proportionally to Kp (lower = more compliant).
- `force_impedance` is hybrid position/force control: position target + a desired contact
  force in a specified direction, set each cycle via `runtime_set_force_ctrl`.

**To sync:**

**Q6.1** Which control mode is used during normal teleoperation — `impedance` or `position`?
Is it configurable per launch profile?

**Q6.2** Is `force_impedance` mode working and tested on the real robot? Is it exposed
during teleoperation or only for scripted/diagnostic motions?

**Q6.3** Is the commanded force in `force_impedance` mode echoed back anywhere in the
snapshot? (We see τ_ext is recorded, but not the force setpoint.)

**Q6.4** **τ_ext on torso/head:** The driver returns `None` for `joint_external_torques`
on torso and head — only arm joints have τ_ext. Hardware limitation or software gap?

**Q6.5** **Wrist F/T sensor:** Is `flange_force` actually available on the robot?
The schema records it; the `*_available` mask will be all-zeros if not fitted. Do both arms
have wrist F/T sensors, or just one, or neither?

**Q6.6** **τ_ext calibration:** Is τ_ext zeroed/biased at startup? Does it drift during
a session? Matters for using it as a reliable contact signal in training.

**Answers:**

---

## 7 · Supplementary / lower priority

**Q7.1** Recording FPS: default is 30 Hz (`config/runtime.yaml`). Is this adjustable?
For force-sensitive contact tasks, higher rate may matter.

**Q7.2** Typical ZMQ round-trip latency between host and robot driver (250 ms timeout is
configured, but what's the normal operating latency)? Matters for understanding how tightly
host IK state and robot sensor state are aligned in recorded data.

**Answers:**

---

## After the meeting

Update:
- This doc with answers
- [data-schema.md](data-schema.md) §force fields with confirmed sensor availability
- [code-structure.md](code-structure.md) open threads section
- [force-feedback.md](../design/force-feedback.md) §4 if flange sensor availability is confirmed
- [decisions-and-caveats.md](../decisions-and-caveats.md) if any design decisions are unlocked
