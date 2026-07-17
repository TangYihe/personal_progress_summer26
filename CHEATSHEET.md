# Cheatsheet — teleop → data → train → eval

> Quick command reference. Stable ops steps only — session state lives in [HANDOFF.md](HANDOFF.md).
> All paths `~/project26/we-teleop`. Machines: workstation (operator), Orin `6.6.7.100` (driver).
> **Updated 2026-07-17 for the rewrite-0711 tree** (schema v10, four-process operator,
> `we driver run`, task poses, single-side teleop). Old-master commands removed — see git
> history of this file if ever back on an old branch.

---

## 0. Session start
```bash
git pull                                      # in ~/project26/we-teleop — labmates push fixes silently (skip ON PURPOSE on frozen-tree collection days)
scripts/setup/sync_luna.sh nvidia@6.6.7.100   # mirror working tree to Orin — ALWAYS pass the address (script default is another robot)
```
- Sync before any robot session: the Orin never pulls on its own; the script mirrors the working tree incl. uncommitted edits.
- ⚠️ `sync_luna.sh` does NOT refresh the Orin venv. If deps/pins changed (`pyproject.toml`/`uv.lock`), on the Orin run:
  `robo run -p tianji-driver -- uv sync --locked --extra camera --extra wuji-hand`
- ⚠️ If `robo.nix`/`flake.lock` changed: EXIT and re-enter every open robo shell on both machines (in-place re-eval leaves a broken PATH).
- Networking (details in [HANDOFF.md](HANDOFF.md)):
  - Robot ↔ workstation: direct ethernet `enp14s0` = `6.6.7.166`. Check: `ping 6.6.7.100`.
  - PICO ↔ workstation: Xiaomi router `enp13s0` = `192.168.31.48`, PICO wired via USB-C ethernet adapter (IP is DHCP, workstation side needs no update).
- PICO: reset the controllers (ring button — procedure TBC with teammates) before every session; park headset on rack BEFORE anchoring.

---

## 1. Orin — driver
`ssh nvidia@6.6.7.100` → `robo shell --profile tianji-driver` → `~/project26/we-teleop`
```bash
# preflight the record camera (D455 profile is ours, local patch):
we camera check --profile tianji_head_d455 --serial 419222302053 --exposure 100

# start driver (data collection — camera REQUIRED for episodes):
we driver run --auto-reset-errors --camera-name tianji_head_d455 \
  --camera-serial 419222302053 --camera-exposure 100

# teleop-only / no recording:
we driver run --auto-reset-errors --no-camera
```
- ⚠️ **ALWAYS pass `--camera-serial 419222302053` (our D455)** — the profile sets only resolution/fps; without a serial the pipeline grabs the FIRST enumerated RealSense = the robot's built-in head cam (looks plausible in the viewer, records the wrong view — 07-17 incident). `we camera list` shows attached serials.
- ⚠️ **D455 must be on USB3 (5000M) for 640x480@60** — the hub's A→C converter passes SuperSpeed in ONE plug orientation only (marked 07-17; wrong way = silent USB2 = driver fails `Couldn't resolve requests`). Check after any replug:
  `ssh nvidia@6.6.7.100 'for d in /sys/bus/usb/devices/*; do [ -f "$d/product" ] && echo "$(cat $d/product) | $(cat $d/speed) Mbps"; done'` — pass = hub AND D455 at 5000 Mbps.
- Gento robot address `6.6.7.190` is the default now — override with `--robot-address` only if the x86 box changes.
- Hand serials live in `config/actuators/wuji.yaml` (ours are committed locally — the old env-var exports are gone). Rewrite ships the labmates' serials; ours are a keeper patch.
- `--move-to-init` is the default (homes + holds in PD). `--control-mode pd` is the default.
- `--camera-exposure 100` = 10 ms in D455 100 µs units (D405 default 10000 µs — units differ per sensor!).
- ⚠️ **Hands OPEN on driver shutdown** (`b81567c`) — held objects drop when you stop the driver.
- Simple error clears without the 上位机: `we driver reset-errors`.
- Driver debug goes to files (`artifacts/logs/driver-*/`), not the terminal.

---

## 2. Workstation — operator shell + teleop
`robo shell -p operator` → `~/project26/we-teleop`
```bash
unset __NV_PRIME_RENDER_OFFLOAD              # EVERY operator shell, INSIDE the robo shell —
                                             # robo.nix shellHook re-exports it on entry (black MuJoCo window otherwise).
                                             # `robo run -p operator -- …` from outside re-sets it → must use robo shell.
export TELEOP_XROBOT_HOST_IP=192.168.31.48   # before pico.sh, else the service grabs the wrong IP

# bimanual (pose capture, normal teleop):
scripts/runtime/pico.sh pico_world_wuji_hands

# single-side collection (right arm+glove teleoped; left frozen at the task pose):
scripts/runtime/pico.sh pico_world_wuji_right
```
- ⚠️ **Profiles now default `driver_address: 6.6.7.100` (07-17 upstream)** — a bare `we teleop --profile …` targets the REAL robot. **Pure sim needs the selector now**: `we teleop --select-profile`, pick a profile, type `local` in the driver field.
- Control clock is now **50 Hz** (driver 100 Hz, 1:1 with Gento's internal servo loop — 07-17 upstream anti-jitter change).
- `pico.sh` sources the XRobot PC service bootstrap itself — no separate step. Connect the PICO app to `192.168.31.48`.
- Glove preflight: `we hands check --side right` (or `--side both`); network only: `we hands network`.
- `pico_world_wuji_right` needs only the RIGHT glove adapter + RIGHT PICO controller (left stays racked).

### Teleop keys & command language
Hotkeys: `Enter` resume/anchor · `Space` pause · `Tab` reset (ramps to active pose + holds) · `R` record · `S` save · `Esc` quit.
`/` or `:` opens the command line:

| command | effect |
|---|---|
| `rc <task>` | select recording dataset name (required before `R`) |
| `setpose <name>` | capture the LAST LIVE pose (body + hands) → `artifacts/task_poses/<name>.yaml` + activate |
| `usepose <name>` | activate a saved task pose (any session; name independent of `rc` task) |
| `home` | clear task pose + reset to default init pose |
| `help` | list everything |

- Active task pose = target of `Tab`/`R` resets. Pose WITH hand targets = **staged reset**: body ramps first → converge → hands commanded → 1 s settle.
- ⚠️ **Strict episode rules (schema v10): pause/reset/hold fallback mid-episode FAILS the episode.** Glove dropout mid-episode fails it too. `R` mid-recording = discard.
- Recording REQUIRES the image modality on wuji profiles → no camera, no episodes.
- Data lands in `artifacts/data/robot/<task>__<timestamp>/` (workstation).

---

## 3. Gloves — thresholds & sim preview
- Pinch thresholds live in `config/inputs/retargeting/wuji_{left,right}.yaml`. Current tree: index/middle `d1/d2 = 1.5/2.0` both hands, **tactile gates OFF** (`p_on: 1.0/p_full: 2.0`; keep `p_full > p_on` or the gate locks ON).
- **Pinch snap is TASK-DEPENDENT** (07-16): twist-heavy tasks → right hand `d1=d2=0` (snap off); pinch/grasp tasks → normal (~1.5/2.0).
- Sim preview (no robot, physical display): `python scripts/inputs/sim_glove_to_hand.py --side right` — loads the same yaml as live teleop; prints `dist_cm`/`pinch_alpha`/`press_gate`.
- Old `calibrate_wuji_pinch.py` is not in the rewrite tree; calibration/finetune tooling is the labmate's workstream (his branch is merged into our integration branch — SDK pin 2026.6.18).

---

## 4. Cube-twist collection — two-session workflow (single-side teleop)

**Session A — capture the task pose (bimanual, once per pose):**
1. Driver up (§1). Teleop: `scripts/runtime/pico.sh pico_world_wuji_hands`.
2. `Enter` to anchor + start. Teleop the cube into a solid LEFT-hand grasp; put the right hand at its twist-start position.
3. While holding the grasp steady: `/` → `setpose rubik_twist` → "task pose saved and active".
4. Verify the staged reset ONCE: move both arms away → `Tab` → body ramps back first, THEN fingers close (watch for log "reset body converged; commanding captured hand targets", 1 s settle). Re-seat the cube into the closing grasp.
5. `Esc` to quit teleop. **Leave the driver running** — it PD-holds; the pose file persists on the workstation.

**Session B — single-side collection:**
1. Teleop: `scripts/runtime/pico.sh pico_world_wuji_right`. Left glove/controller not needed.
2. `/` → `usepose rubik_twist` → "task pose active".
3. `/` → `rc <dataset>` (e.g. `rc rubik_twist_v0`) — dataset name is independent of the pose name.
4. `Tab` once → robot ramps to the task pose (arms → fingers). Seat the cube into the left grasp.
5. Episode loop: `R` (ramps to pose, auto-records at pose) → `Enter` if paused → do the twist with the right hand → `S` (save) → re-seat cube → `R` → …
6. Left arm stays frozen at the pose all episode; left hand action records the captured grasp constants (NOT zeros) — verify once in the first parquet.
7. `home` at session end (clears pose, default init). Stop teleop before the driver; **hands open on driver stop — catch the cube.**

---

## 5. Training
Workstation, in tmux (~2h):
```bash
tmux new -s training
we train <policy_name>            # e.g. act_pick_toy_0702
# detach: Ctrl-b d   ·   reattach: tmux attach -t training
```
New task → copy `config/policy/act_pick_toy.yaml`, set `dataset:` + `output_dir:`. Checkpoints every 20k (`best@`, `last@`).
⚠️ New strict policy families on rewrite-0711 — our 4 old `act_pick_toy*.yaml` configs are NOT restored (on snapshot branches).

---

## 6. Eval / deploy
Workstation at a **physical display** (all open a MuJoCo viewer → fail headless):
```bash
we eval <policy_name> --split val                       # sim, recorded obs — "did it learn"
we deploy <policy_name> --gt-obs                        # recorded obs through the real stack
we deploy <policy_name> --live --driver-address 6.6.7.100:5559   # LIVE closed loop
```
- ⚠️ Config defaults `deploy.gt_obs: true` → bare `we deploy` is recorded-obs. Pass `--live` for the real robot.
- Deploy starts **paused**: `Enter` run · `Space` pause · `Tab` reset+clear policy state · `--max-frames N` auto-pauses (~2000 ≈ 20 s).
- Deploy reproduces a task-pose start by construction (same RESET path as collection).

---

## 7. 上位机 (GentoPlatform) — manual robot reset / recovery
Workstation, at physical display (tkinter GUI; needs `6.6.7.x` ethernet up):
```bash
cd ~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM
python3 UI.py
```
- Connects to x86 board `6.6.7.190` (IP pre-filled). **Stop `we driver` first — one commander at a time.**
- Flow: Connect Robot → speed **5%** + Confirm Speed per component → paste pose into Position Cmd → Add → Position mode → Run.
- Simple error clears don't need this anymore: `we driver reset-errors` (§1).
- Full SOP + reference poses (遥操/打包/原点): [notes/implementation/shangweiji-sop.md](notes/implementation/shangweiji-sop.md)
