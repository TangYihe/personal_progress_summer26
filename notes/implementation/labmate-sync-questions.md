# Labmate sync — codebase investigation & open questions

> Prep doc for the meeting with 师弟 (teleop/hardware side).
> Covers what we found from code inspection, what's already documented, and what needs syncing.
> ← back to [code-structure](code-structure.md) · [data-schema](data-schema.md)
>
> Investigation date: 2026-06-15 · Partial sync: 2026-06-15 · Remaining questions: open

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
  `imp_cart` (Cartesian spring-damper), `force_impedance` (force direction + max travel distance
  — **not** hybrid position/force as initially assumed; see §6 answers).
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
- **Q1.1 (partial):** Wrist cameras will be set up later. Not on the immediate roadmap but planned.
- **Q1.3:** Currently head camera only — confirmed. Wrist cameras coming later.
- **Camera type note:** The wrist cameras will be **fisheye**. This has design implications for
  training: fisheye images either need to be rectified before use or handled by
  architectures/augmentations that tolerate the distortion. Need to review how previous works
  (e.g. DexWild) handle fisheye wrist cameras. Q1.2 (integration path) still open.

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
- **Q3.1 / Q3.2 (resolved):** Real-robot policy replay is working. Labmates have verified
  end-to-end: collected a simple pick task dataset, trained ACT (overfit), and replayed
  on the real robot — no issues. The basic pipeline (record → train → replay) is validated.

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
- **Q4.1 / Q4.2 (partial):** Hand teleop integration is **not yet complete** — hand is
  not in current data collection or action space. The validated pick task dataset and policy
  do not include hand DOFs. Timeline for hand integration is open.
- Q4.1 (torso/head coverage), Q4.3, Q4.4 still open.

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
| `force_impedance` | `switch_to_imp_force_mode` | Specify a force direction + magnitude + max travel distance. Robot moves in that direction applying the specified force, stopping at the distance limit. No simultaneous pose target. |

**Key distinction clarified:**
- `impedance` and `imp_cart` both still track a position target, but compliantly — external
  forces deflect the robot proportionally to Kp (lower = more compliant).
- `force_impedance` (corrected): **not** hybrid position/force control as initially assumed.
  Input is (force direction, magnitude, max travel distance) — the robot moves in the specified
  direction with the specified force, up to the distance limit. No concurrent pose target.
  This is closer to a "push until contact" primitive than full hybrid control.

**To sync:**

**Q6.1** Which control mode is used during normal teleoperation — `impedance` or `position`?
Is it configurable per launch profile?

**Q6.2** Is `force_impedance` mode working and tested on the real robot? What exactly are
its inputs, and is it suitable for our use case?

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
- **Q6.1 (resolved):** Position control triggers safety protection easily in practice.
  **Joint impedance (`impedance`) is the chosen control mode going forward.**
- **Q6.2 (resolved — and corrected):** `force_impedance` mode has been tested. Its actual
  interface is **(target force direction + magnitude + max travel distance)** — not a
  simultaneous pose + force target as we initially assumed from the code. Semantically it is
  a "push in direction X with force Y, stop after Z mm" primitive. This makes it **less
  suitable for general contact-rich manipulation** (no concurrent pose control); it is more
  useful for scripted insertion/pressing primitives. We will not rely on it for our IL setup.
- Q6.3–Q6.6 still open.

---

## 7 · Supplementary / lower priority

**Q7.1** Recording FPS: default is 30 Hz (`config/runtime.yaml`). Is this adjustable?
For force-sensitive contact tasks, higher rate may matter.

**Q7.2** Typical ZMQ round-trip latency between host and robot driver (250 ms timeout is
configured, but what's the normal operating latency)? Matters for understanding how tightly
host IK state and robot sensor state are aligned in recorded data.

**Answers:**

---

## Status summary (2026-06-15 partial sync)

**Resolved:**
- Control mode: joint impedance confirmed as primary. Position mode ruled out (safety trips).
- `force_impedance` interface clarified: force + distance primitive, not hybrid position/force.
  Not suitable for our general manipulation use case.
- End-to-end pipeline validated: record → ACT train → real-robot replay works on a pick task.
- Hand: not yet integrated into data collection or training. No timeline yet.
- Wrist cameras: planned but not imminent; will be fisheye when added.

**Still open (need follow-up):**
- Q1.2: Wrist camera integration path (schema change ownership, format)
- Q4.1: Which subsystems move in current recordings (torso, head?)
- Q4.3 / Q4.4: Arm-only config feasibility + head camera coverage
- Q5.1–Q5.4: Dev plans, repo ownership, deployment timeline
- Q6.3–Q6.6: force setpoint echo, τ_ext on torso/head, flange F/T sensor fitted?, τ_ext bias
- Q7.1 / Q7.2: Recording FPS, ZMQ latency

**Downstream updates to make once remaining questions are answered:**
- [data-schema.md](data-schema.md) §force fields — confirm flange sensor availability
- [force-feedback.md](../design/force-feedback.md) §4 — update based on confirmed signals
- [decisions-and-caveats.md](../decisions-and-caveats.md) — unlock any blocked design decisions
- [code-structure.md](code-structure.md) — close open threads as confirmed
