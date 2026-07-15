# Cheatsheet — teleop → data → train → eval

> Quick command reference. Stable ops steps only — session state lives in [HANDOFF.md](HANDOFF.md).
> All paths `~/project26/we-teleop`. Machines: workstation (operator), Orin `6.6.7.100` (driver).

---

## 0. Session start
```bash
git pull                                      # in ~/project26/we-teleop — labmates push fixes silently
scripts/setup/sync_luna.sh nvidia@6.6.7.100   # then mirror to Orin — ALWAYS pass the address (script default is another robot)
```
- Sync before any robot session: the Orin never pulls on its own, it only gets what we mirror (working tree incl. uncommitted edits).
Networking now (see [HANDOFF.md](HANDOFF.md) for details):
- Robot ↔ workstation: direct ethernet `enp14s0` = `6.6.7.166`. Check up: `ping 6.6.7.100`.
- PICO ↔ workstation: Xiaomi router `enp13s0` = `192.168.31.48`, PICO `.110` (wired via USB-C ethernet adapter since 2026-07-06; was `.231` on WiFi — IP is DHCP, may change with adapter swaps; workstation side needs no update).

---

## 1. Orin — driver
`ssh nvidia@6.6.7.100` → `robo shell --profile tianji-driver` → `~/project26/we-teleop`
```bash
# reset to safe pose
we driver --reset-once --acc 20 --control-mode pd --auto-reset-errors

# start driver (dual-hand)
export WUJI_RIGHT_HAND_SERIAL=306735523434
export WUJI_LEFT_HAND_SERIAL=366239543134
we driver --operator-address 6.6.7.166 --control-mode pd --auto-reset-errors \
  --enable-wuji-hands --wuji-hand-side both --camera-serial 419222302053
```
- Torso lock (team decision 2026-07-06, steadier manipulation) is now set via config, NOT `--lock-torso` (labmates 2026-07-09): `ik_preset.joints.torso: false` in `config/teleop/pico_ee_absolute_wuji_hands.yaml` — torso excluded from the workstation IK solve. Set back to `true` if the task needs torso motion.
- ⚠️ Export BOTH serials — the in-repo defaults are other robots' hands, not ours.
- `--wuji-hand-side both` is the default on `feat/multiprocessing`, but pass it explicitly.
- Hand droops in reset→start gap? Skip `--reset-once`; add `--move-to-init --acc 20` to the driver (homes + holds in PD).
- Camera `419222302053` = D455 (record cam). `--enable-wuji-hands` mandatory when hand mounted.

---

## 2. Workstation — XRobot PC service (for PICO)
`robo shell -p operator` → `~/project26/we-teleop`
```bash
export TELEOP_XROBOT_HOST_IP=192.168.31.48   # MUST set on WiFi, else it grabs the WiFi IP
source scripts/setup/bootstrap_xrobot_pc_service.sh
# verify:
grep -iE 'ServerIP|ConnectedIP' third_party/XRoboToolkit-PC-Service/RoboticsService/bin/setting.ini
```
Then connect the PICO app to `192.168.31.48`. (Do this once per fresh operator shell.)

---

## 3. Glove pinch calibration
Operator shell:
```bash
python scripts/inputs/calibrate_wuji_pinch.py --hand right --backup
python scripts/inputs/calibrate_wuji_pinch.py --hand left --backup
```
- Relaxed hand during the ~2s baseline. Then pinch firmly + press `v` for each: index → middle → ring → pinky.
- Writes thresholds to `config/inputs/retargeting/wuji_{right,left}.yaml`. Confirm `enabled: true`.
- ⚠️ **SANITY-CHECK the printed thresholds before trusting a run** (2026-07-07 incident: degenerate output = hair-trigger pinch). Healthy: `p_on` well above the printed baseline floor, `p_full` well above `p_on` (defaults `0.15/0.35` for scale). Bad numbers → bad glove fit / weak pinch / hot taxel, NOT a config to keep.
- Escape hatch: `git checkout -- config/inputs/retargeting/wuji_{right,left}.yaml` → known-good defaults (current working tree: defaults + `d_near_cm: 3.0`, 2026-07-08).
- Tactile debugging: `python scripts/diagnostics/monitor_wuji_tactile.py --hand right|left` — live per-taxel grid + the pair pressures the gate sees (rest ≈ 0; firm pinch ≳ 0.3).

### 3b. Sim visualization (no robot needed; at a physical display)
```bash
# glove -> retargeter -> MuJoCo hand (one side per window; run two for both)
python scripts/inputs/sim_glove_to_hand.py --side right
python scripts/inputs/sim_glove_to_hand.py --side left
# full teleop sim (arms + hands, PICO, NO robot): just drop --driver-address
we teleop --profile pico_ee_absolute_wuji_hands
```
- `sim_glove_to_hand.py` loads the same `wuji_{side}.yaml` as live teleop → the place to preview pinch-threshold changes. Defaults already right (`--kinematic`, `--hand-model assets`). `--demo` = synthetic curl, no glove.

---

## 4. Workstation — teleop + data collection
Operator shell (XRobot service already up from step 2):
```bash
we teleop --profile pico_ee_absolute_wuji_hands --driver-address 6.6.7.100:5559
```
- Press `Enter` to anchor + start.
- `R` = toggle record · `S` = save episode as success.
- Data lands in `artifacts/data/robot/<task>__<timestamp>/`.

Racked-PICO rules (headset on rack, not worn):
- Park the headset BEFORE anchoring. If it gets moved mid-session → auto-pause (IK jump guard) → `Space`, settle, `Enter` re-anchors jump-free.
- Directions 180° off → recenter (long-press home) with headset facing the operator's direction, re-anchor.
- Keep the headset on USB-C power on the rack (stay-awake-while-charging set via adb 2026-07-08 — under verification; if it still dims: `adb shell dumpsys power | grep -iE "wakefulness|timeout|sleep"` right after).

---

## 5. Training
Workstation, in tmux (~2h):
```bash
tmux new -s training
we train <policy_name>            # e.g. act_pick_toy_0702
# detach: Ctrl-b d   ·   reattach: tmux attach -t training
```
New task → copy `config/policy/act_pick_toy.yaml`, set `dataset:` + `output_dir:`. Checkpoints every 20k (`best@`, `last@`).

---

## 6. Eval / deploy
Workstation at a **physical display** (all open a MuJoCo viewer → fail headless):
```bash
# sim, recorded obs — "did it learn"
we eval <policy_name> --split val

# deploy w/ recorded obs through real stack (no live sensors)
we deploy <policy_name> --gt-obs

# LIVE — real obs, closed loop; needs driver up on Orin
we deploy <policy_name> --live --driver-address 6.6.7.100:5559
```
⚠️ Config defaults `deploy.gt_obs: true` → bare `we deploy` is recorded-obs. Pass `--live` for the real robot.

---

## 7. 上位机 (GentoPlatform) — manual robot reset / recovery
Workstation, at physical display (tkinter GUI; needs `6.6.7.x` ethernet up):
```bash
cd ~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM
python3 UI.py
```
- Connects to x86 board `6.6.7.190` (IP pre-filled). **Stop `we driver` first — one commander at a time.**
- Flow: Connect Robot → speed **5%** + Confirm Speed per component → paste pose into Position Cmd → Add → Position mode → Run.
- Full SOP + reference poses (遥操/打包/原点): [notes/implementation/shangweiji-sop.md](notes/implementation/shangweiji-sop.md)

