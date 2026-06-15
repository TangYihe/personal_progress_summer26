# we-teleop: codebase structure

> Reference for the model-training workstream. Covers repo layout, data format, and
> policy training pipeline. Not a full audit — focus is on what's relevant to training.
> ← back to [decisions index](../decisions-and-caveats.md)
>
> Last updated: 2026-06-15

---

## Repo purpose

`~/we-teleop/` is the lab's minimum teleop stack for the Tianji humanoid. It supports:
- Real-time teleoperation (keyboard / PICO controller / PICO SMPL-X body tracking / Manus/Wuji gloves)
- Native recording to raw Parquet datasets
- Policy training via **ACT** and **Diffusion Policy** backends
- Open-loop policy evaluation in MuJoCo

Live deployment is intentionally disabled (raises `NotImplementedError`).

---

## Top-level layout

```
we-teleop/
├── src/
│   ├── teleop/          # Runtime, data, control, robots, policy eval
│   └── train/           # Training dispatch + policy wrappers
├── config/              # YAML: profiles, cameras, embodiments, policy templates
├── third_party/         # Git submodules: ACT, Diffusion Policy, Tianji driver, XRobo SDK
├── scripts/             # Calibration, diagnostics, camera servers
├── tests/               # ~26 test modules
└── docs/                # Architecture + data schema docs
```

---

## Entry points

All exposed via a single CLI: `we`

| Command | What it does | Primary module |
|---------|-------------|----------------|
| `we teleop` | Launch teleoperation + recording | `runtime/cli/main.py` → `TeleopApp` |
| `we train act` | Train ACT policy | `train/cli.py` → `train/policies/act.py` |
| `we train dp` | Train Diffusion Policy | `train/cli.py` → `train/policies/dp.py` |
| `we policy eval` | Open-loop eval in MuJoCo | `teleop/policy/cli.py` |
| `we data rerun` | Visualize recordings with Rerun SDK | `teleop/data/cli.py` |
| `we pico` | PICO diagnostics/calibration | `runtime/cli/pico_runtime.py` |
| `we tianji-driver` | Robot-side hardware bridge | `robots/tianji/driver.py` |

---

## Module map

### Teleop runtime (`src/teleop/runtime/`)

- `app/core.py` — `TeleopApp`: main event loop (~600+ lines). Manages MuJoCo sim, input handling, recording, ZMQ hardware comms. Spawns control thread (IK solver), watchdog, and background save thread.
- `app/state.py` — `TeleopState`: runtime state machine.
- `app/commands.py` — keyboard/UI command dispatch (record, pause, target edit).
- `launch/profiles.py` + `launch/resolve.py` — profile resolution from `config/launch_profiles.yaml`.
- `mujoco/runtime.py` — async IK solver backend (pyroki / JAX).
- `robot_commands.py` — ZMQ bridge to real hardware.

**Concurrency:** control thread (IK + frame capture + hardware publish) | main thread (MuJoCo render) | save thread (Parquet finalization). Synchronized via `threading.RLock`.

### Control (`src/teleop/control/`)

- `controller.py` — `TeleopController`: applies qpos/targets, IK jump guard.
- `modes.py` — enums: `InputSource` (keyboard, pico_controller, pico_smplx), `SolverMode` (ee_absolute, ee_relative).
- `inputs/pico/session.py` + `pose.py` — XRobo SDK adapter, pose streaming, controller→wrist transforms.
- `inputs/manus/` + `inputs/wuji/` — glove adapters.
- `kinematics/pyroki_backend.py` — async IK solver.
- `kinematics/preset.py` — `TeleopPreset`: IK chain config (torso, arms, head, hands, base).

### Data (`src/teleop/data/`)

- `recording/manager.py` — `TeleopRecordingManager`: frame enqueue, episode save.
- `recording/dataset.py` — `RawRecordingDataset`: Parquet table write logic.
- `recording/frames.py` — frame payload construction (images, state, actions).
- `schema.py` — **full feature schema** (see §Data format below).
- `snapshot.py` — `DriverSnapshotClient`: ZMQ client for hardware state during recording.
- `replay.py` — `TeleopReplayManager`: load + playback for sanity checks.
- `rerun_view.py` — Rerun SDK integration.

### Policy training (`src/train/`)

- `cli.py` — `dispatch_train()`: routes to ACT or DP; manages isolated uv venvs per backend.
- `policies/act.py` — ACT wrapper: submodule init, venv setup, calls ACT training script via subprocess.
- `policies/dp.py` — Diffusion Policy wrapper: same pattern.
- `contract.py` — `PolicyEnv` dataclass; action spaces (joint, ee); action modes (absolute, delta).

Each policy backend runs in an **isolated uv venv** (fingerprinted; rebuilt on dep changes).

### Policy eval (`src/teleop/policy/`)

- `cli.py` — `eval-act` and `eval` commands.
- `act_open_loop.py` — loads checkpoint, builds MuJoCo env, replays episodes with model predictions. Supports temporal aggregation.

### Robots (`src/teleop/robots/`)

- `registry.py` — `EmbodimentAdapter` base; `TianjiEmbodiment`.
- `tianji/model.py` — URDF path, body/joint name constants (7-DOF arms; torso; head; legs).
- `tianji/driver.py` — hardware driver (ZMQ cmd/snapshot, motor commands).

---

## Data format

**On-disk layout:**
```
data/robot/<task-slug>__<timestamp>/
├── episodes/
│   ├── episode_0.parquet
│   ├── episode_1.parquet
│   └── ...
└── meta/
    ├── info.json           # schema version, fps, robot_type, episode count
    ├── teleop_session.json # profile, input source, git hash, PICO calibration
    └── description.md      # task description
```

**Parquet schema** (defined in `data/schema.py`; current: schema version 7 for ACT):

| Column group | Contents |
|---|---|
| Observation: state | joint qpos, qvel, effort |
| Observation: force | external torque (τ_ext), forces, gyros |
| Observation: images | per-camera RGB frames |
| Observation: hand poses | Manus/Wuji glove finger poses |
| Action | joint_qpos targets; IK targets (wrist L/R, headset); chassis command |
| Source (PICO) | raw controller poses, headset pose, SMPL-X joint poses |
| Metadata | sample index, timestamps, camera frame IDs |

**Key note:** τ_ext (external torque) is recorded as a native column — the robot exposes it directly via API. This directly answers the open action item in [force-feedback.md](../design/force-feedback.md) §4.

---

## Policy training detail

### ACT

- Backend: `third_party/policy/act` (submodule).
- Schema version: 7.
- Action spaces: `joint` or `ee`; modes: `absolute` or `delta`.
- Configurable: `batch_size`, `num_epochs`, `lr`, `kl_weight`, `chunk_size`, `hidden_dim`.
- Config template: `config/policy/act.yaml`.

### Diffusion Policy

- Backend: `third_party/policy/diffusion_policy` (submodule).
- Schema version: 5.
- Configurable: `batch_size`, `num_epochs`, `obs_horizon`, `pred_horizon`, `action_horizon`, `diffusion_steps`.
- Config template: `config/policy/dp.yaml`.

Both train pipelines write checkpoints with metadata that `we policy eval` reads back for eval.

---

## Launch profiles (config/launch_profiles.yaml)

| Profile | Input | Solver mode |
|---------|-------|-------------|
| `tianji_local_keyboard` | Keyboard/mouse | ee_absolute |
| `tianji_pico_ee_absolute` | PICO controller | ee_absolute |
| `tianji_pico_ee_relative` | PICO controller | ee_relative (to headset) |
| `tianji_pico_smplx_relative_upper_body` | PICO SMPL-X | ee_relative |
| `..._manus_glove` variants | above + Manus | — |
| `..._wuji_glove` variants | above + Wuji | — |

---

## Related notes

- [data-schema.md](data-schema.md) — detailed field semantics, shapes, action representation, sync model

---

## Open threads surfaced by this investigation

- **τ_ext confirmed recorded** — the schema includes external torque natively. Action item from force-feedback.md §4 ("check which torque signal our robot exposes") is answered: τ_ext is available.
- **Replay pipeline exists** (`data/replay.py`) — `TeleopReplayManager` can load + playback recordings. Relevant for workstream A (replay success-rate sanity check).
- **Live deployment disabled** — `policy/cli.py plan` raises `NotImplementedError`. Any deployment path will need to be built.
- **Schema versioning** — ACT at v7, DP at v5. Watch for version mismatches if mixing checkpoints.
