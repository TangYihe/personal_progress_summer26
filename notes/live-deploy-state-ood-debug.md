# Live Deploy — Proprioceptive State OOD Bug (investigation)

> Investigation log for the ACT live-deploy failure on the pick-toy task.
> Started 2026-07-02. **Status: RESOLVED 2026-07-02 evening — root cause revised (see
> "Resolution" at the bottom); fixed by the upstream deploy rewrite; verified live on robot.**
> ← back to [weekly notes](weekly-notes/week-2026-06-29.md) · [HANDOFF](../HANDOFF.md)

---

## TL;DR (read this first)

- **Symptom:** `we deploy act_pick_toy_0702 --live` makes **one fast move (enough to shake the body), then freezes.** Sim eval and gt-obs deploy were fine.
- **We ruled out, in order:** camera/visual gap ✗, chunk-velocity (fast motion) ✗, missing driver proprioceptive section ✗.
- **Confirmed:** the **proprioceptive state fed to the policy is grossly out-of-distribution** (`max|z| ≈ 220`, 52/65 state dims outside the training 1st–99th percentile). Feeding the *recorded* state with a live image (`--gt-state`) runs **fine**, proving the image is OK and the **state is the entire problem**.
- **Root cause (identified):** with `--enable-wuji-hands`, deploy uses a **hand-augmented MuJoCo model** whose qpos ordering puts the **20 right-hand finger joints at qpos indices 16–35**, i.e. *between* the right arm (9–15) and left arm (36–42). `observation.state`'s "body" slice is the first 25 qpos entries, so **dims 16–24 are right-hand finger joints**, not body joints. The driver sends **no hand feedback**, so those finger values come from the **simulator's commanded hand pose** (set by `apply_policy_qpos`), which at deploy differs from the near-constant (~±1.57) values training saw there. → dims 16–24 go OOD → bad prediction → lurch + freeze.
- **Fix: NOT yet applied.** This is shared runtime code and connects to the earlier `init_qpos` / hand-reset hand-augmented-model bugs. **Flag to labmate.** See "Next steps / fix direction."
- **⚠️ 2026-07-02 evening update: RESOLVED — and the root cause above is partially superseded.** Training's dims 16–24 turned out to be the **left arm + head** (plain 25-joint layout), not finger rest values; the upstream deploy rewrite (deploy now runs inside TeleopApp on the plain model) fixes the layout mismatch, verified live on the robot. **Read the "RESOLUTION" section at the bottom.**

---

## Background / goal

Today's goal was the full pipeline end-to-end on pick-toy: collect → train → **live eval**.

- Dataset: `artifacts/data/robot/pick-toy-0702__20260702-112947` (~50 demos, right-hand only; left glove hardware dead).
- Checkpoint: `act_pick_toy_0702`, trained 100k steps, **`action_horizon: 100`** (→ `chunk_size=100`, `n_action_steps=100`). `best@60000` (val loss min), `last@100000`. Run dir: `artifacts/checkpoints/act_pick_toy_0702/20260702-042205-be3c15a56f5a/`.
- **Sim eval** (`we eval act_pick_toy_0702 --split val`): looked good.
- **gt-obs deploy** (recorded obs through the real stack): worked earlier (2026-07-01).
- **Live deploy**: `we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 --ckpt-name 060000` → **one quick move, then frozen.** This investigation.

---

## Experiments, outputs, analysis (in order)

### 1. Camera / visual gap — RULED OUT
Wrote `scratchpad/check_obs.py` to grab the live driver-snapshot frame (`DriverSnapshotClient`, exactly what `we deploy --live` feeds the policy) and compare to a training frame. Images saved to workspace `obs_debug/` (`live_obs.png`, `train_obs.png`, `obs_compare.png`, `obs_blend.png`).
- **Output:** 50/50 blend shows bin/table/walls/arms razor-sharp (aligned), only the toy ghosts (different placement).
- **Analysis:** live camera viewpoint matches training. Tape remount is fine. Camera not the issue.
- Gotcha: LeRobot videos are **AV1**; decode with system `/usr/bin/ffmpeg`, NOT the robo-shell nix ffmpeg (GLIBC mismatch) and NOT OpenCV.

### 2. Inference / execution mechanics — understood
Traced `_deploy_live` in `src/train/policies/lerobot/__init__.py`:
- Each loop step: `predict_action_chunk` returns the **full chunk** (`chunk_size` rows; 100 here) — `_predict_policy_action_rows`.
- **Temporal aggregation ON** (`_TemporalActionAggregator`, `k=0.01` ≈ uniform weights): every chunk contributes a prediction to each future frame it overlaps; each frame executes the weighted average. ACT-style temporal ensembling.
- **Executes ONE aggregated action per loop, re-queries every step.** Does **NOT** use LeRobot's `n_action_steps` action-queue → so `action_horizon`'s `n_action_steps` half has no effect on deploy; only `chunk_size` matters (aggregation horizon).
- **Rate = 100 Hz** (`eval_rate_hz`: `config.fps` unset → `rate_hz=100` from metadata = recording `fps=100`). Matches demos → **frequency is not the cause.**
- Targets are **absolute qpos**, published to the driver which PD-tracks them (no interpolation beyond `safety.sanitize`). A far target → fast slew.

### 3. Step-size comparison (`--debug-actions`) — chunk velocity RULED OUT
Compared demo per-step motion vs the predicted chunk's internal per-step motion (L2 of consecutive action vectors):
```
demo ep per-step L2:  mean=0.0174  median=0.0074  max=0.3021
chunk per-step L2:    mean≈0.006–0.010  max≈0.017–0.033
```
- **Analysis:** the chunk is internally *smooth* — per-step motion ≤ demo, max 10× smaller. The policy is NOT predicting fast motion. So the lurch is not chunk velocity; it's an **offset** (chunk[0] far from the current pose).

### 4. Offset + state OOD (`--debug-actions`) — STATE is grossly OOD
```
offset ||chunk[0]-state|| ≈ 2.9–3.5   (over 25 "arm/body" dims)
state max|z| ≈ 220–222                (z = (live-mean)/std vs training stats)
out_of_[q01,q99] = 52/65 dims
```
- Training stats loaded from `<lerobot_root>/meta/stats.json` (`observation.state`, 65-dim). `lerobot_root = .cache/we/training/lerobot/e27ff90a910aafff`.
- **Analysis:** the live state vector is in a completely different space than training. Because these stats come from the recording itself, the live *construction* must differ from the recording's.

### 5. Live image + recorded state (`--gt-state`) — IMAGE fine, STATE is everything
Ran `--gt-state --warmup-episode 48 --warmup-steps 10`: feeds the **recorded** state from episode 48 while keeping the **live camera image**.
- **Output:** robot runs **nicely**, follows sensible motion, no lurch/freeze.
- **Analysis:** the live image (through the full policy pipeline) is fine. The **state is the entire problem.** Also implies normalization + the policy itself are fine (recorded state normalizes and drives correctly).

### 6. Per-dim mismatch (`--debug-actions`, frame-0 top-15)
```
dim: live | train_mean | train_std | z
dim17: +0.2258 | -1.5776 | 0.0081 | +221.7
dim18: -0.2058 | -1.5582 | 0.0140 |  +96.9
dim19: -0.0898 | -1.5833 | 0.0172 |  +87.0
dim21: -0.4044 | +0.0048 | 0.0063 |  -65.3
dim16: +1.0710 | +1.5725 | 0.0143 |  -35.0
dim20: +0.3240 | +0.0124 | 0.0109 |  +28.6
dim24: +0.2081 | +0.0252 | 0.0206 |   +8.9
dim23: +0.0660 | +0.0097 | 0.0111 |   +5.1
...
dim9:  -1.5710 | -1.0769 | 0.3060 |   -1.6   (matches, larger std)
dim11: +1.5712 | +1.4010 | 0.1404 |   +1.2   (matches)
dim12: -1.5676 | -1.0785 | 0.3832 |   -1.3   (matches)
dim13: +0.0017 | -0.3340 | 0.1912 |   +1.8   (matches)
dim15: +0.0022 | -0.2343 | 0.1792 |   +1.3   (matches)
dim3:  -0.0018 | -0.0122 | 0.0070 |   +1.5   (matches)
```
- **Analysis:** NOT a units bug (no constant multiplier). Clean break: **dims 0–15 match (z≈1–2); dims 16–24 grossly wrong (z up to 222).** The wrong dims had **tiny training std (0.006–0.02)** → held nearly fixed in all demos; live has them at scattered but plausible joint angles.

### 7. Driver proprioceptive sections (`--debug-actions`, frame 0) — missing-section RULED OUT
```
live proprioceptive joint_positions (section->len, None=driver did not report):
{'left_arm': 7, 'right_arm': 7, 'head': 2, 'torso': 6, 'chassis': None}
```
- **Analysis:** all body sections ARE reported with correct lengths (chassis None is expected). So the mismatch is NOT a fall-back-to-MuJoCo from a missing/mismatched section. The real joint values are being written.

### 8. qpos layout dump (`--debug-actions`, frame 0) — THE reveal
```
deploy model qpos layout (addr:joint):
0:base_x, 1:base_y, 2:base_yaw,
3:Gento Luna_Leg_J1_Joint ... 8:Gento Luna_Leg_J6_Joint,     # TORSO (named "Leg")
9:Arm_R1_Joint ... 15:Arm_R7_Joint,                          # RIGHT ARM
16:right_finger1_joint1 ... 35:right_finger5_joint4,         # RIGHT HAND (20 finger joints)
36:Arm_L1_Joint ... 42:Arm_L7_Joint,                         # LEFT ARM
...
```
- `Gento Luna_Leg_J*` = the **torso** joints (that's just their name; not an actual leg — see `TIANJI_TORSO_JOINTS`).
- The log truncated at addr ~39 (`layout[:40]`), which is why only Arm_L1–L4 showed — L5–L7 are at 40–42.
- **KEY:** the model orders qpos as `base(3) + torso(6) + right_arm(7) + RIGHT_HAND fingers(20) + left_arm(7) + ...`. The 20 finger joints are at **16–35**.

---

## State vector composition (why dims 16–24 are the culprit)

- `observation.state` is 65-dim. It is built (in `_live_policy_frame`) as:
  1. `state = policy_vector_from_keys(item, observation_keys)` → the model's full `qpos`.
  2. `_trim_live_base_vector(state, base_state_dim)` → **first `base_state_dim` (=25) qpos entries** = the "body" slice.
  3. append the hand_fields → +40 dims → 65 total.
- In the **deploy** (hand-augmented) model, the first 25 qpos entries are `base(3) + torso(6) + right_arm(7) + first 9 right-hand finger joints (dims 16–24)`.
- So **dims 16–24 of the body slice are right-hand FINGER joints**, not body joints.
- Cross-check with the per-dim data (consistent): dims **9–15 = right arm** (moved in demos → larger std 0.14–0.38, and they *match* live because arm feedback is real); dims **16–24 = right-hand fingers** (near-constant ±1.57 in demos → tiny std, and they're *wrong* live). This also resolves the earlier "which arm moved" confusion: the **right arm is at dims 9–15 and did move** (correct — right-hand teleop).

### Where the finger values come from live (the "MuJoCo commanded hand pose")
- `compose_state_vector` (`frame_payload.py:358`) overwrites only `base/torso/head/left_arm/right_arm` from the driver proprioceptive. **There is no hand section** → the finger qpos entries are NOT overwritten and keep the value already in `snapshot.qpos`.
- `snapshot.qpos = runtime.data.qpos` = the **MuJoCo simulation's** joint positions. Each loop `apply_policy_qpos(runtime, ..., left_hand, right_hand)` sets the sim's finger joints to the **hand portion of the last applied action** (policy prediction in closed loop; recorded hand action during warmup).
- So the finger values in the state are the **sim's commanded hand pose, not a hardware measurement.** At deploy they differ from the near-constant (~±1.57) values training saw at dims 16–24 → OOD.

---

## Root cause (current understanding)

With `--enable-wuji-hands`, `create_open_loop_runtime` (`src/train/policies/workflow.py`) builds a **hand-augmented render model** whose qpos interleaves the 20 right-hand finger joints at indices 16–35. `observation.state` takes the first 25 qpos entries as the "body," so **9 finger joints (16–24) land inside the body slice**. Those finger values are the **sim's commanded hand pose** (no driver hand feedback exists), which at deploy sits at a different configuration than the near-constant values present during recording. Result: 9 state dimensions are severely OOD → the policy predicts a trajectory anchored far from reality → the driver slews there fast (the lurch that shakes the body) → deeper OOD → temporal-aggregated near-constant garbage → freeze.

Same family as the earlier **uncommitted `init_qpos` fix** ("hand-augmented render model has larger nq") and the hand-reset bugs. All are `--enable-wuji-hands` deploy-path issues.

### Remaining uncertainty to verify (before fixing)
Why were dims 16–24 **near-constant ±1.57 in training**? Leading hypothesis: during **recording**, the sim/render finger joints were **not driven by the real hand** — they sat at the model's default rest pose (±1.57), constant — while the actual hand actuation was captured in the *appended* hand_fields (dims 25–64). At **deploy**, `apply_policy_qpos` *does* drive the sim fingers → dims 16–24 now vary → mismatch. **This must be confirmed** by checking how the recording populates the finger qpos vs how deploy does. (If true, the training state has effectively "dead"/constant values at 16–24 that deploy fills with live commanded values.)

---

## Code investigated (read-only, for reference)

- `src/train/policies/lerobot/__init__.py`
  - `_deploy_live` (~1064): live loop; `host_snapshot.qpos = runtime.data.qpos`; `_live_policy_frame`; aggregator; `apply_policy_qpos`; `build_robot_command_packet`; `bridge.publish`.
  - `_live_policy_frame` (~1337): builds state via `build_frame_payload(host_snapshot, remote=remote)` → `policy_vector_from_keys` → `_trim_live_base_vector(state, base_state_dim)` → append hand_fields.
  - `_predict_policy_action_rows`, `_TemporalActionAggregator`, `_request_latest_driver_sample`.
  - `base_state_dim`/`base_action_dim` computed ~1092–1093; `_joint_address_maps` (~1309) builds `qposadr`/`dofadr` from `runtime.model`.
  - `create_open_loop_runtime` (in `workflow.py`) — swaps in the hand-augmented render model (root of the layout divergence).
- `src/teleop/data/recording/frame_payload.py`
  - `build_frame_payload` (126): `state_vector = snapshot.qpos`; if `remote`, `compose_state_vector(...)`. Live image via `_image_from_remote` (uses `remote["image_bgr"]`).
  - `compose_state_vector` (358): writes base/torso/head/left_arm/right_arm from `proprioceptive.joint_positions` — **no hand section**.
  - `write_section` (445): if values `None` or length-mismatch → returns (keeps existing qpos).
- `src/teleop/robots/tianji/driver.py`
  - `snapshot_robot_state` (1254): builds `proprioceptive.joint_positions` = `{left_arm, right_arm, head, torso, chassis}` (each `None` if its section is disabled). **No hand.** Values are `np.deg2rad(fb_pos)`.
- `src/teleop/robots/tianji/model.py`: `TIANJI_TORSO_JOINTS` = `Gento Luna_Leg_J1..J6`; `TIANJI_HEAD_JOINTS` (2); `TIANJI_LEFT/RIGHT_ARM_JOINTS` (7 each).
- `src/teleop/robots/embodiment.py`: `hardware_joint_sections` order = torso, head, left_arm, right_arm.
- `src/teleop/data/snapshot.py`: `DriverSnapshotClient` (400) — `request_status()` / `request_sample(idx)` → `image_bgr`, `proprioceptive`, etc.

---

## Code CHANGED (all UNCOMMITTED, debug-only, deploy path) — flag to labmate

> All in the deploy/live path; none affect training. Added as diagnostics — decide whether to keep/clean before committing anything.

- `src/train/cli.py` — `deploy_command`: added flags `--warmup-steps`, `--warmup-episode`, `--debug-actions/--no-debug-actions`, `--gt-state/--no-gt-state`; threaded into `compact_values`.
- `src/train/policies/lerobot/__init__.py` — `_deploy_live`:
  - **Warmup block:** if `--warmup-steps N`, replay the first N ground-truth frames of `--warmup-episode` (publish recorded actions) to seed proprio before closed loop.
  - **Debug baseline (once):** demo per-step L2; load training state stats from `<lerobot_root>/meta/stats.json`; dump the model qpos layout (`addr:joint`).
  - **gt_states block:** if `--gt-state`, load recorded states from `--warmup-episode`.
  - **In-loop:** if `--gt-state`, override `frame.state` with the recorded state (keep live image). If `--debug-actions` and `frame_index<20`: log chunk per-step L2, offset `||chunk[0]-state||`, state `max|z|` + `out_of_[q01,q99]`; at frame 0 also log the proprioceptive section summary and the top-15 per-dim mismatches.
- Configs (new): `config/policy/act_pick_toy_0702.yaml` (action_horizon 100), `config/policy/act_pick_toy_0702_chunk200.yaml` (chunk_size=200, 40k steps — a separate experiment, not yet needed given the root cause).
- Scratch: `scratchpad/check_obs.py`; workspace `obs_debug/*.png` (throwaway).

---

## How to reproduce

Driver on Orin (`robo shell --profile tianji-driver`, hand enabled):
```bash
export WUJI_RIGHT_HAND_SERIAL=306735523434
we driver --operator-address 6.6.7.166 --control-mode pd --auto-reset-errors \
  --enable-wuji-hands --wuji-hand-side right --camera-serial 419222302053 --move-to-init --acc 20
```
Workstation (physical display, `robo shell -p operator`) — the diagnostic run:
```bash
we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 \
  --ckpt-name 060000 --debug-actions --warmup-steps 10 --warmup-episode 48 --max-frames 2
```
(⚠️ frame 0 lurches — keep a hand near e-stop.) The control experiment that runs fine:
```bash
we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 \
  --ckpt-name 060000 --gt-state --warmup-episode 48 --warmup-steps 10
```

---

## Next steps / fix direction

1. **Confirm** the finger hypothesis: check how the *recording* populates the finger qpos (dims 16–24) vs how `_deploy_live` (`apply_policy_qpos`) drives them. Determine whether training's dims 16–24 were a constant default.
2. **Fix options** (for labmate / next session — shared runtime code):
   - Make the deploy body-state slice use the **intended body joints by name/order** (base+torso+head+left_arm+right_arm) instead of the raw `qpos[0:base_state_dim]` prefix, so interleaved hand joints can't leak in; OR
   - Ensure the sim finger joints at deploy sit at the **same configuration** training recorded (don't drive them into the body slice); OR
   - Fix `create_open_loop_runtime` so the render-model qpos layout matches the training/recording layout when hands are enabled.
3. This gates the **live-eval pipeline milestone**. Until fixed, the pipeline is validated through **gt-obs / gt-state** but not true live closed-loop.
```

---

## RESOLUTION (2026-07-02 evening) — root cause revised; fixed upstream; verified live

### Revised root cause: dims 16–24 were LEFT ARM + HEAD, not fingers

The "remaining uncertainty" above is settled, and it **revises** the root cause. The recording/teleop
runtime uses the **plain (non-hand) model** with exactly 25 qpos:

```
0–2 base | 3–8 torso | 9–15 Arm_R1–R7 | 16–22 Arm_L1–L7 | 23–24 Head
```

So training's dims 16–24 were never finger joints — they were the **left arm + head**, near-constant
because pick-toy is right-hand-only (left arm parked at home ±1.57). Verified against
`meta/stats.json`: dims 16–19 mean ≈ ±1.57 with std 0.008–0.017 — a parked arm with **real encoder
jitter** (a sim constant would give exactly-zero std).

The old deploy's corruption was therefore two-sided:
1. dims 16–24 got **finger commands** (sim's commanded hand pose) — as diagnosed above; AND
2. the **real left-arm/head feedback was lost entirely** — `compose_state_vector` wrote it by joint
   name to the hand-model addresses 36–44, past the first-25 trim.

Consequence for the fix options above: option (b) ("park sim fingers at training values") was wrong —
those values are a left-arm pose. Options (a)/(c) (compose by name / match layouts) were right.

Extra findings while verifying:
- **Hand obs dims 25–64 are all-zero in training AND live** — the driver reports no hand feedback, so
  `hand_vector(None)` → zeros on both sides. Consistent → not part of the bug (but note: the policy
  never sees real hand state; hand behavior is learned from action supervision only).
- The earlier "52/65 dims out of [q01,q99]" was inflated by a quantile artifact: zero-variance dims
  have q01 ≈ 4e-16, so a live value of exactly 0.0 counts as "out". The true OOD signal was exactly
  dims 16–24 (z up to 222), as the per-dim table showed.

### Fix: upstream deploy rewrite (pulled 2026-07-02)

Upstream rewrote deploy entirely (`we deploy` now logs `route: TeleopApp`): `_deploy_live`'s
standalone loop and its `create_open_loop_runtime` hand-model swap are gone; deploy runs inside
**TeleopApp** — the same app/model/code path as teleop recording. State is composed on the **plain
25-joint model** with real driver feedback, identical to how the training data was built. The layout
mismatch is structurally eliminated (deploy and recording can no longer disagree).

### Verified live (2026-07-02 evening)

`we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 --ckpt-name 060000` on the new
code, **old (collection-era) driver kept on the Orin** so kp/kd/alpha match training dynamics →
**works, no lurch, task behavior close to demos, full episodes run clean.** Full
collect → train → live-eval pipeline validated end-to-end.

New deploy app keys: `Enter` = start/resume · `Space` = pause · `Tab` = reset robot to init (ramps at
reset velocity + holds) · auto-pauses at `--max-frames` (no longer exits).

The old diagnostic tooling (`--debug-actions`, `--gt-state`, `--warmup-*`) died with the rewritten
code but is preserved on branch **`wip/live-deploy-state-ood`** (commit `a96f840`) if ever needed.

