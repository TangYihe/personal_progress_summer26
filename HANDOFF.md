# Handoff — where we left off

> Session bookmark. **Current state only** — no history (that lives in the README progress
> log). Update at the end of every working session: what's done, where to pick up next.
>
> Last session: **2026-07-31** — **offline-provisioning Part A DONE.** Built + verified the
> **teleop-only offline seed** (`/home/yihetang/project26/onsite-seed`, **5.8 GB**, also on the USB stick,
> integrity-checked). Rewrote the runbook against the *post-merge* tree and caught a bug that would have
> stranded us onsite: **the nix tarball was never captured** (`nix-installer` downloads a nix package →
> offline install fails at step one; fixed with a shipped, version-matched nix 2.34.7 +
> `--nix-package-url file://`). Also established that **`~/.cache/nix` is load-bearing**, not optional —
> refactor-0722 bumped the robo-nix pin so **two revs** must resolve offline (`3c55fa1` for the robo CLI,
> `fb50985` for the repo lock), which is exactly what broke the Orin on 07-29. **The rehearsal (Part C) is
> NOT done — deliberately deferred.** No robot/teleop work this session.
> Full detail: [notes/weekly-notes/week-2026-07-27.md](notes/weekly-notes/week-2026-07-27.md) (07-31 log).
>
> **07-30 (logged retroactively 07-31):** took fork **(A)** — tried collecting the **multi-step** rubik
> twist and hit a wall: **very hard to teleoperate because hand retargeting isn't smooth.** Reported to
> the retargeting owner; it's their thread now, and it **gates the whole "more data → train" plan**.
> The 07-28 drained robot battery came back **fine** (charges normally, no BMS lockout).

---

## WHEN YOU'RE BACK — next session (start here)

**Session start:** `git pull` in `we-teleop` (check labmate pushes — the shared branch is
`origin/refactor-0722`, which includes our `sim_glove_to_hand` commit `69a309c`). Pull the notes repo.

**🔀 Pick ONE. The provisioning rehearsal is short and self-contained; the data/compare fork is the
bigger thread.**

**(0) 🔴 PROVISIONING REHEARSAL — ~20–30 min, no robot needed, everything is already prepared.**
The seed is built and on the stick. On the **spare workstation, network cable UNPLUGGED**:
```bash
sudo -v
/mnt/seed/provision.sh        # or wherever the stick auto-mounts there
```
It self-checks and **fails the run if `cache.nixos.org`/`github.com` answer**, so the spare's own
internet can't fake a pass. Then the one manual step (needs a display):
```bash
cd /home/yihetang/project26/we-teleop
robo shell -p operator
unset __NV_PRIME_RENDER_OFFLOAD          # shellHook re-exports it on EVERY entry
we teleop --profile keyboard --driver-address local
# a MuJoCo window must actually RENDER, not be black
```
Bring the drive back → `provision-<host>-<date>.log` is on it. ⚠️ Two things change how to read the
result: **does the spare already have nix?** (then the offline-install unknown stays untested) and
**is it NVIDIA?** (if not, a viewer failure is a REAL preview of the TIANJI risk —
`robo.nix` hardcodes `hostGraphics = "nixgl-nvidia"` for `operator`). Then decide the **uv-cache
question** (13 GB not shipped → no `uv sync --offline` fallback; there's room on the drive).
Seed is also at `/home/yihetang/project26/onsite-seed` on the workstation, and reproducible from the
runbook's Part A block.

**(A) ⛔ Multi-step data collection is BLOCKED — do not just retry it.** 07-30 showed the task is
teleop-limited by **hand-retargeting smoothness**, not by data volume. Reported to the retargeting
owner; **follow up with them for status first.** Demos an operator can barely produce would encode the
struggle, so fixing the teleop feel precedes collecting. What we can do meanwhile:
- Feed the owner the specifics in one go: **thumb IP-vs-MCP** unresolved (`thumb_skip_pip: true` caps it
  upstream of `segment_scaling`), and our LOCAL `wuji_finetune.json` has **index/middle pinch fully OFF
  (`d1=d2=0` both hands)** — tuned for the *single* twist, likely wrong for a multi-step sequence.
- Capture A/B candidates with `scripts/inputs/sim_glove_to_hand.py --record` (pushed 07-29) — sim only,
  no robot needed.
- Re-decide pinch `d2` (~3.0 restores a real index/middle grasp) + prep the **two stable hand yamls**
  (pinch-enhanced vs plain; manual swap = `cp` preset + restart teleop).
- ❓ Check `artifacts/data/robot/` for anything saved during the 07-30 attempt before assuming it's empty.

**(B) Side-by-side compare** our teleop system against another one — still available, and arguably the
better use of a session while retargeting is someone else's blocker.

Both use the teleop bring-up below.

**Teleop bring-up (either track):**
1. **Orin (driver):** `robo shell -p tianji-driver` →
   `we driver --auto-reset-errors --camera-serial 419222302053 --camera-exposure 100`
   (defaults: robot `6.6.7.190`, `pd`, `--move-to-init`, bimanual hands + camera). ⚠️ init-gate may trip
   on startup → recover via 上位机. ⚠️ Watch the Wuji **`Bus undervoltage` F4J3/F4J4** (07-28) — stop +
   check the hand power rail if it recurs.
2. **Workstation (operator):** fresh `robo shell -p operator` → `unset __NV_PRIME_RENDER_OFFLOAD` →
   `scripts/runtime/pico.sh pico_world_wuji_hands` (Quest tracks via the XRobot service, headset-agnostic;
   head/torso are held LOCAL; both arms + hands live). `Enter` to anchor.
3. **If track (A):** write a fresh v3 policy config (copy `config/policy/act_turn_cube.yaml` → set
   `dataset`/`output_dir`, `num_workers: 4`, `include_hands`) once data lands. Capture the grasp start with
   upstream `init_pose capture <name>` (was our `setpose`).

**Open threads to carry:**
- **🔴 Hand-retargeting smoothness — REPORTED to the owner 07-30, awaiting them.** This is the blocking
  thread for the multi-step task. Sub-thread: **thumb IP-vs-MCP bending** unresolved —
  `segment_scaling.thumb` reset to base default (both hands) didn't move it, and `thumb_skip_pip: true`
  (SDK reconstructs thumb MCP inaccurately) limits it upstream of scaling. Follow up for status; don't
  re-attempt multi-step collection before it improves.
- **Still-unpushed SHAREABLE:** `9c83c95` (retargeting pinch 1.5/2.0 + tactile gates off) — held pending
  owner sign-off, because "gates off" bakes in our phantom-pressure + faulty-left-sensor assumption.
- **Report to william (still owed):** `config/policy/act_turn_cube.yaml` ships `num_workers: 32` → the
  video-DataLoader **segfault** ([[train-dataloader-segfault]]); we run 4. (Leave his `Call me Daddy` line
  in `CLAUDE.md` alone — intentional per Yihe.)
- **Open-shutdown fix is LOCAL + uncommitted** in `driver_supervisor.py` and NOT on the Orin (post-dates the
  last sync). To get hands-open-on-shutdown on the robot: `scripts/setup/sync_luna.sh nvidia@6.6.7.100`
  (rsync only this time — no deps/flake change → no venv refresh, just restart the driver).

**Branch / local state (`wip/collection-integration-20260728`):**
- `origin/refactor-0722` tip = **`69a309c`** (our `sim_glove_to_hand`, pushed 07-29) on `49199da`.
- Above the OLD base on wip: `77d169b` (dup of the pushed commit — drops on the next rebase onto
  `origin/refactor-0722`), `1c7a1c3` + `06757f6` (**LOCAL, do-not-push**), `9c83c95` (SHAREABLE, unpushed).
- **Uncommitted LOCAL** (do-not-push): `wuji_finetune.json` (thumb `w=30`; **index+middle pinch snap OFF
  `0/0` both hands**; thumb `segment_scaling` = base default), `pico_world_wuji_hands.yaml` (head/torso
  `hold`), `driver_supervisor.py` (open-on-shutdown). ⚠️ index/middle pinch is fully off — if a task needs a
  real index/middle grasp, raise `d2` (~3.0) + restart teleop.

**Housekeeping (optional):** `third_party/GMR` (1.2 GB) + `third_party/policy` are untracked leftovers
NOT referenced by refactor-0722 — removable. Submodules `Isaac-GR00T`/`Psi0`/`Unity-Client` are
uninitialized — `git submodule update --init --recursive` only if those workflows are needed.

**Rollback anytime:** `git switch wip/pre-refactor-0722-merge-20260728` (`d5f7e6a`).

⚠️ Glove gotcha (standing): connect-timeout with carrier-up = check for **cross-wired glove cables**
(tcpdump the dongle, swap if the source IP is the other glove's). CHEATSHEET §3.

---

## WHEN YOU'RE BACK — 2026-07-24 (archived — pipeline validated ✓)

**State:** Full pipeline works (train → deploy). v0 policies go OOD on the precision twist
(actions fine — open-loop MSE ~5e-5, gt-obs ~75%; killers = live-obs gap + thin data). Pivoted to
more+cleaner data (v1 merged, 195 clean eps). **07-23: the v1 policies were NOT trained** — both
horizontal runs had crashed overnight (DataLoader worker segfault; `num_workers:32` too high for a
video policy — see [[train-dataloader-segfault]]). Root-caused + fixed; retraining relaunched fresh.
Replay validated on hardware ✓.

**🌙 TRAINING RUNNING OVERNIGHT (started 07-23 18:43, tmux `train`):**
- Fresh runs, `num_workers:4`, **50k steps** each, BOTH sequential (chunk-100 `&&` chunk-200).
  Configs `config/policy/act_rubik_twist_horizontal.yaml` + `_chunk200.yaml` (edited: num_workers
  32→4, steps 100k→50k — UNCOMMITTED). Crashed 32-worker dirs parked at `*_crashed20k`/`*_crashed10k`.
- **FIRST THING: check they finished** → `we_policy.json` present at
  `artifacts/checkpoints/act_rubik_twist_horizontal/` (+ `_chunk200`). Missing / only partial
  checkpoints = crashed again → `tmux attach -t train`, read the error. **The tell the fix held is
  clearing ~20k steps** (both prior runs died at 13k/20k).
- Relaunch a run (fresh, NOT `--resume` — resume restores the old 32-worker config):
  `robo run -p training -- we train act_rubik_twist_horizontal`. Loss plateaus ~0.05 early.

**🎯 PLAN 07-24 (busy: hardware onsite; operator here):**
1. **EVAL the v1 policies** (once trained). Commands CORRECTED for this tree:
   - **gt-obs** (policy actions from recorded obs, open-loop on robot) — from a training superset
     shell (deploy recipe [[deploy-venv-gap]]) with xrobot paths exported, run **`we deploy` DIRECTLY
     — NOT via pico.sh (pico.sh re-runs the cmake build → GLIBC crash):**
     ```
     we deploy act_rubik_twist_horizontal \
       --gt-obs artifacts/data/robot/2026-07-22__rubik_twist_horizontal_merged \
       --episode N --pose rubik_twist_horizontal_v1 \
       --profile pico_world_wuji_right --driver-address 6.6.7.100 --checkpoint-name last
     ```
   - **live** closed-loop: same but DROP `--gt-obs …`, ADD `--temporal-aggregation` (uses live camera).
   - ⚠️ `--live`/`--ckpt-name` are GONE on this tree; live = default when `--driver-address` set and
     no `--gt-obs`; checkpoint = `--checkpoint-name last`/`--checkpoint-step`.
   - ⚠️ Profile **`pico_world_wuji_right`** (single-side — dataset meta confirms `sides:[right]`).
     A wuji profile REQUIRES the gloves connected (IPs 192.168.1.100/.101) or replay AND deploy fail.
     In-session hand-open (`home`) needs a wuji profile; under `keyboard` the hand stays gripped
     (bounce the driver to open — hands open on driver shutdown).
2. **Teleop-pipeline checks w/ operator** — full checklist:
   [notes/checklists/2026-07-24-pre-demo-hardware-check.md](notes/checklists/2026-07-24-pre-demo-hardware-check.md).
   Covers: 2 hand-retarget presets (enhanced-pinch vs pinch-off — swap mechanism NOT built yet),
   thumb naturalness (coord w/ retargeting owner), body/torso lower pose (torso is all-or-nothing
   `hold`/`ik`; static captured pose = safe fallback), smoothing (hand smoothing IS merged; values
   are hardcoded Python not yaml), ⚠️ **CHASSIS not in pipeline** (fails-closed per CLAUDE.md — film
   plan needs base+upper-body together → raise w/ owner+advisor), D455 USB3, Quest tracking, e-stop.
3. **THEN soften the reset grip** — only after eval baselines as-trained behavior (transmission-slip
   mitigation); keep the pose name so deploy-reset matches the data.
4. **Advisor reply** — scope / on-site decision-maker / hard deadline (asked 07-23). 🛒 asset buy-list
   STILL open. Big upstream merge = DEFERRED post-demo (plan:
   [notes/implementation/upstream-merge-plan-20260723.md](notes/implementation/upstream-merge-plan-20260723.md)).
5. **Stable branch (07-23 item #3, NOT done)** — cut a self-maintained demo branch off the validated
   07-20 tree + today's edits; isolate from labmate churn. Uncommitted edits to fold in: num_workers/
   steps (2 configs), `pico_world.yaml` head/torso hold, `wuji_finetune.json` recal,
   `repair_missing_images.py`. Also fix the stale CHEATSHEET deploy commands. Claude to draft.

⚠️ camera frame-drops still unfixed (batch1 ~8% / batch2 ~11% faulty) — fix at source (USB3 5000M /
`config/cameras.yaml` d455 60→30 fps / powered hub) when there's a window.

**Two new features committed (local, NOT pushed — Yihe pushes):** `cbd02af` `we data replay --pose`;
`4c851b5` gt-obs HARDWARE deploy (`we deploy --gt-obs <ds> --driver-address <host>` runs the policy
on RECORDED obs while publishing its actions to the robot). ⚠️ both touch `control_process.py` →
re-port after the big merge; CLAUDE.md owning-doc update owed. Deploy env recipe (non-durable) + the
`we_policy.json`/vision-policy gotcha are in the 07-22 week log.

**🔴 Camera frame-drops STILL UNFIXED** — at 200 eps this faulty-outs a fraction / leans on the
forward-fill repair (bad for a vision policy). Fix at source when there's a window: confirm D455 on
**USB3 5000M** (CHEATSHEET §1 sysfs check), or drop `config/cameras.yaml` d455 `fps: 60`→`30`, +
powered USB3 hub (still unbought).

**Then:**
1. **Thumb:** run `calibrate_thumb_spread.py` on the RIGHT glove — A/B proved `thumb_abd_track`
   is the right lever for thumb flexibility; fit it to OUR glove (don't reuse labmates' `w:30`).
2. **Watch right pinch `d2`** (large: index 5.56 / middle 6.51) — lower if fingers over-stick.
3. **Train a first ACT — READY, dry-run verified.** `config/policy/act_rubik_twist.yaml` points
   at the repaired 50-ep dataset (`include_hands: true`). Just run:
   `robo run -p training -- we train act_rubik_twist` (dry-run resolved clean: ACT/cuda/50k/batch32).
   ⚠️ Training venv had a stale launcher (`typer_main`→`main`) — already fixed via
   `robo run -p training -- uv pip install -e . --no-deps`; if it recurs, same fix.
4. 🛒 **Asset purchase** (Yihe) — buy-list + arrival date; awaiting TIANJI reply on setup/wifi.
5. Upstream-report: camera frame-drop; curate can't repair `missing_required_image`;
   `--driver-address local` flag not normalized to sim.

**🚩 #1 BIG UPSTREAM MERGE = TOMORROW's milestone (07-22).** User wants it done (fast-iterating
project). Treat as a milestone: Claude examines the 239-file rewrite-0711 refactor and brings a
STAGED plan to review BEFORE executing. Conflict scope mapped (07-21 log): ~14 files incl.
`control_process.py` (rewritten → re-implement our task-pose logic), `src/we/cli.py` (split →
re-port), `driver.py` (collides w/ their reset-unification). ⚠️ Claude can't fetch/push (HTTPS
creds) → Yihe runs those; ⚠️ upstream built PARALLEL reset/single-hand → coordinate w/ owner on
whose version wins before pushing our feature to the shared branch.

**#2 DONE (07-21):** `feat/repair-missing-frames` — `scripts/data/repair_missing_images.py`
forward-fills dropped frames; verified 50/50 clean. Still fix the camera drop at the source
(top priority above) so future collection doesn't lean on the repair.

**07-22 MORNING — first steps (planned 07-21 eve):**
0. Check overnight training: checkpoints in `artifacts/checkpoints/act_rubik_twist` (10k/20k/…),
   loss curve sane (wandb). Driver up (hands + camera); Orin sync if the tree changed.
1. **Replay validation — feature BUILT 07-21 eve** on `feat/replay-initial-pose` (sim-verified,
   HARDWARE-UNTESTED). `we data replay <ds> --episode N --pose rubik_twist_test --profile
   <hands> --driver-address 6.6.7.100 --yes` now starts PAUSED with the pose pre-activated →
   **`Tab`** ramps to the pose (staged: body then hand closes — place the cube in the window) →
   **`Enter`** plays the recording (RESUME already triggers replay `begin_approach`+`play`).
   ⚠️ First hardware run: workspace clear, hands out, ready on `Space`/`Esc`. ⚠️ Also fixed a
   PRE-EXISTING replay crash (`replay.task`→`replay.dataset_label`; replay didn't run at all
   before). ⚠️ OPEN Q: which profile for hands replay WITHOUT live gloves (a `wuji` profile may
   demand glove processes) — sort at first run. ⚠️ Edits `control_process.py` → re-port after
   the big merge. Files: `control_process.py`, `supervisor.py`, `runtime/cli/data.py`.
2. **Inference w/ custom init = READY, no new code.** `we deploy act_rubik_twist --profile
   pico_world_wuji_right --driver-address 6.6.7.100` starts PAUSED (shares run_operator) →
   `:usepose rubik_twist_test` → `Tab` (staged reset, place cube in the window) → `Enter`
   (policy drives from the grasp pose). Needs a checkpoint from #0. Deploy default is gt_obs →
   pass `--live`? (verify: rewrite deploy route; `--driver-address` = live hardware.)
   ⚠️ Deploy chunk knobs: n_action_steps 100 (executes full 2 s chunk) — sweep smaller for
   tighter closed-loop if the twist drifts.

(Big upstream merge = the other 07-22 milestone; separate from this morning block.)

**Local working-tree changes this session (uncommitted, on `wip/collection-integration-20260720`):**
- `config/teleop/pico_world.yaml`: head/torso → `hold`.
- `config/inputs/retargeting/wuji_finetune.json`: left + right fresh recal; `thumb_abd_track`
  removed from both (both thumbs `w=0`).
- NEW `scripts/data/repair_missing_images.py` (on `feat/repair-missing-frames`).

---

## PICK UP NEXT (as of 2026-07-20)

1. ~~**Confirm HR/operator scheduling**~~ ✅ **DONE 07-21** — operators locked, on-site starts
   Mon 07-27.
2. **PICO tracking-snap (⚠️ safety, root-caused 07-20)**: right-controller VIO snap caused
   two arm lurches; full evidence + suggested fix (task-space step clamp) in
   [notes/issues/2026-07-20-pico-tracking-snap.md](notes/issues/2026-07-20-pico-tracking-snap.md)
   — send to teleop owner if not sent yet. Teammates ruled out battery; suggest trying a
   different PICO angle/placement next session (untested).
3. ~~**Right-hand Wuji calibration didn't save**~~ ✅ **DONE 07-21** — recal saved; both hands
   at parity; labmates' `thumb_abd_track w:30` removed (both thumbs default). Proper thumb-spread
   calib on our glove (`calibrate_thumb_spread.py`) still TODO for more thumb flexibility.
4. **Quest teleop verified in sim (zero code changes needed)** — `pico_world` profile
   tracks correctly via the existing PICO receiver once the headset is on the teleop LAN
   (192.168.31.x, not office wifi) and the XRobo PC service is running. Mount is
   3D-printing (blocks robot trial). Once printed: robot-validate Quest teleop; Quest also
   has controller-visibility limits (design mount placement with camera visibility in
   mind, not just ergonomics).
5. **07-27 (Mon) demo shoot needs prep** — multiple shots planned, prep not yet scoped.
   **First step: ask teammates what props/equipment they already have** before buying
   anything. Then shot list + task selection + run-through, done early in the week.
6. **Then: real cube-twist collection** (pipeline READY once #3 is done): driver w/
   camera serial + exposure → `pico_world_wuji_hands` → real grasp → `setpose rubik_twist`
   → `pico_world_wuji_right` → `usepose` → `rc <dataset>` → `R`/`S` loop (CHEATSHEET §4).
   Verify first parquet (50 Hz spacing, left-hand grasp constants, D455 frames).
7. Standing: notes repo committed — **push it yourself**; jitter-parquet verdict
   follow-up; 50 Hz feel verdict for the labmate.

**Tree state:** workstation on `wip/collection-integration-20260720` (= origin
rewrite-0711 `09b7273` + our 9 commits + 1 driver-reset restore fix), NOT pushed. Orin
needs `sync_luna.sh nvidia@6.6.7.100` + venv refresh (uv.lock changed) before next driver
start. Suite: 536 passed / 8 known fails (6 pre-existing on clean upstream tip, 2
init-gate locals). wuji-sdk 2026.7.14 + matching glove/hand firmware. Init gate now 2.5°
(bumped from 2.0° — 3rd bump, trend tracked in week file, Arm_R6 measured 2.01°). Saved
task pose `rubik_test` (throwaway); capture a real `rubik_twist` before collecting.

---

## Where we are

**Workstation AND Orin both on `master` (Orin synced 07-09 via `sync_luna.sh nvidia@6.6.7.100` — now in CHEATSHEET §0 as a session-start step; ⚠️ always pass the address, script default is another robot).** Master tip includes `dc100cc` — driver emits a fixed **60 Hz snapshot stream**, recorder stores 1:1 (state read internally at 120 Hz). Serials in CHEATSHEET (`R=306735523434`, `L=366239543134`).

**Colleague's retargeting: sim-tested 07-09 am, GOOD** with one tweak — right middle finger over-stuck → its `d2` 4.0→3.0 in `wuji_right.yaml` (uncommitted). On-robot feel still unverified. **Finetune store still DISABLED** (`~/.we-teleop/wuji_finetune.json.off`) — Test B (store overlay on, watch right-pinky `d1 7.95` suspect) never run; do it before relying on our 07-08 calibration. Debug rules: finger over-sticks → lower its `d2`; tactile gate = min(thumb, finger taxel) — pads must press together.

**Camera FIXED (07-09): the all-white cam view was a units bug in master's exposure lock.** `_lock_camera_exposure` (Xiawei `296574d`) pins `D405_EXPOSURE_US=10000` with AE off — that's 10 ms on their D405 (µs units) but **1 SECOND on our D455 RGB sensor (100µs units)** → solid white. Local fix `D405_EXPOSURE_US = 100.0` (=10 ms) **VERIFIED on robot — view normal**. Committed on **`wip/d455-exposure-fix-20260709`** (`96cc236`; recovery after any force-push reset: `git cherry-pick 96cc236`). TODO: upstream a `--camera-exposure` flag + tell labmates (not portable across cameras; also gain freezes wherever AE left it). Note: with AE off, one `wait_for_frames timeout` warning at driver start is benign; repeated = USB/cable (D455 mount still taped — check `rs-enumerate-devices` USB type 3.x vs 2.1).

**Arm-jitter data: RE-RECORDED WITH CAMERA 07-10 + SENT to labmate** — `rubric-cube-0710__20260710-105655` (6 eps, 94,756 frames @ 60 Hz, camera confirmed) + `lagging_movement_2.zip`; parquets with the labmate for analysis — **follow up on their verdict after the break**. ⚠️ Make sure they know state is 60 Hz sampled — jitter >30 Hz aliases. (07-09 camera-less `teleop-bug1..3` recordings now redundant.) **Labmate 07-09: the 一卡一卡/stationary jitter is a "PICO reading issue"** → see PICO section below. **Operator view verdict 07-10: robot-right/operator-left WORKS — cube teleop a lot better** ✓.

**Recording keys (verified in code):** `r` start = ramp to init THEN record (skip-ramp if already at reset pose); `s` = save+reset-ramp; `r` mid-recording = discard+reset-ramp. Task must be selected first (`rc`). ⚠️ **Hands do NOT open on reset** — upstream regression still present (reset packet sends last hand pose): drop held objects before `s`. Use `s` even for jitter episodes (failures may get filtered out by the team's tooling).

**Torso lock is now config, NOT `--lock-torso`** (labmates 07-09): `ik_preset.joints.torso: false` in `config/teleop/pico_ee_absolute_wuji_hands.yaml` (uncommitted). CHEATSHEET §1 updated. First session: watch torso doesn't sag (driver no longer actively holds it).

**`we data rerun` — viewer status (07-10 retest):** native viewer FAILS in robo env (`winit: Failed to load one of xlib's shared libraries` — egl-wayland did NOT fix it). **Browser mode works**: `we data rerun <name> -e N --no-robot-model --browser` then open `http://127.0.0.1:9090/?url=rerun%2Bhttp%3A%2F%2F127.0.0.1%3A9876%2Fproxy` (plain `:9090` = disconnected welcome screen; the serving process must stay running). ⚠️ **Freezes ~10 s into a full episode** (~15k frames + images) — next: `--max-frames` windows, or `--save ep.rrd` + standalone viewer outside robo env, or fix xlib in robo.nix. Venv note: `uv sync --extra operator --extra data-rerun --group dev --group wuji-glove` dropped torch/lerobot — add `--group train-lerobot` back before any `we eval`/`we deploy`. After robo.nix changes, EXIT and re-enter open robo shells (in-place re-eval leaves broken PATH — `No module named mujoco`).

**PICO — status update 07-15:**
1. *Tracking in third-person view still a bit shaky*; hanging the headset over the operator's neck works better. **Teammates suggest switching to Meta Quest** (large code change on their side) — ask progress at the meeting. Interim tip: **reset the PICO controllers (ring button? — verify) before every teleop session.**
2. *Sleep bug* (unchanged, moot if Quest happens): adb settings + USB-C power don't prevent rack sleep; interim = nudge every few minutes (`Space`, settle, `Enter` after).
3. *Stationarity A/B* (rack tilted vs straightened) — deprioritized 07-15 pending the Quest answer; details in [week-2026-07-06](notes/weekly-notes/week-2026-07-06.md) TODO #3.

**Left glove tactile sensor still FAULTY (hardware)** — vendor contacted 07-08, no reply yet; follow up. Until replaced, left press-pinch/pressure can't work (distance-gate retargeting unaffected).

**Glove cabling (07-10): running WITHOUT the extension cable** — passive USB3 extension caused `-EPROTO`/-71 storms on the right adapter (details + diagnostic recipe in weekly notes 07-10). Proper fix later: active repeater / powered hub / ethernet-side extension. If a glove ever shows carrier UP but ping dead + frozen RX counters → glove processor hung → power-cycle the glove.

**Operator view: robot moved to the RIGHT (07-09 evening)** — replaces the seat-move idea; judge next session. Bimanual ergonomic conflict (seeing right hand's fingers = tiring left-arm pose) still open. Getting more teleop tasks/objects from Eden (in progress 07-09).

Note: local snapshot branches: **`wip/d455-exposure-fix-20260709`** (`96cc236`, the exposure-units fix — cherry-pick after any reset), `wip/pre-master-switch-20260708` (07-08 pinch-tuning state), `wip/pre-multiproc-20260706` (pre-branch-switch state incl. driver.py gain overrides + old pinch yamls) and `wip/live-deploy-state-ood` (pre-refactor, never delete). Retargeting commits (`83b422e` etc.) are still NOT on master — they also live on active `feat/wuji-hand-v2`; 11 multiprocessing-branch commits not merged. Ask labmates about the merge plan.

**Uncommitted working-tree changes (07-09):** `driver.py` (exposure 100), `config/teleop/pico_ee_absolute_wuji_hands.yaml` (torso false), `config/inputs/retargeting/wuji_right.yaml` (middle d2 3.0), + 4 `act_pick_toy*.yaml` staged. All flow to the Orin via `sync_luna.sh` (mirrors working tree).

**Weekly notes:** [notes/weekly-notes/week-2026-07-20.md](notes/weekly-notes/week-2026-07-20.md) (prev: [week-2026-07-13.md](notes/weekly-notes/week-2026-07-13.md))

---

## What was done (2026-07-02, evening session — OOD bug resolved, upstream integrated)

- **State-OOD root cause revised + confirmed**: training state dims 16–24 = **left arm (16–22) + head (23–24)** on the plain 25-joint model (near-constant: right-hand-only task, left arm parked at ±1.57 home). Old deploy built the state on the hand-augmented model → fingers leaked into 16–24 AND real left-arm/head feedback landed at addresses 36–44, outside the first-25 trim. Full detail in the bug note's RESOLUTION section.
- **Upstream integrated (force-pushed/rebased history!)**: local master `reset --hard` to `origin/master` (`6536c22`). All pre-refactor local changes preserved on branch **`wip/live-deploy-state-ood`** (`a96f840`). Restored from it (upstream didn't touch these): pinch-calibration yamls, `act_pick_toy*.yaml` policy configs, diagnostics script.
- **Upstream rewrote deploy** → runs inside TeleopApp on the plain model (same code path as recording) → **fixes the OOD bug structurally**.
- **LIVE DEPLOY VERIFIED** ✓ — `we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 --ckpt-name 060000` with the old driver on the Orin: no lurch, behavior close to demos, full episodes clean. Wire compat old-driver ↔ new-workstation checked (new `state` snapshot command fails gracefully; command path unchanged).

---

## What was done (2026-07-01)

- **`we train` fixed + ACT training running** — two bugs in `2ea54a4` (the QoL training commit):
  1. LeRobot's output dir collided with `we train`'s manifest files → fixed upstream with `LEROBOT_OUTPUT_SUBDIR = "train"`
  2. `_validate_loss` called `policy.eval()` → ACT's VAE encoder returned `None` → KL loss crashed. Labmate had local fix; pushed after we reported. Fix: check `policy.config.use_vae` before switching mode.
  - Both fixes pulled via `git pull` (75da37e → 560e76f). Training now running cleanly.

- **Upstream pull — notable driver.py changes:**
  - `"impedance"` control mode renamed to `"pd"` — `"impedance"` kept as deprecated alias, existing commands still work
  - `alpha` default: `0.95` (was `0.9`)
  - `DEFAULT_OPERATOR_ADDRESS`: `6.6.7.200` (was `192.168.1.196`)

- **Ethernet topology planned** — current shared office WiFi causes teleop jitter/"jumping". Plan confirmed: wall → router → switch → workstation + robot (ethernet); PICO → switch via USB-C ethernet adapter. Not yet executed.

---

## What was done (2026-06-30)

- **Torso startup error fixed** — labmate updated x86 onboard software; no longer recurring.
- **Full pipeline verified end-to-end**: D455 stream (serial `419222302053`, used for data collection), D435i (serial `349622073307`) also verified, right Wuji glove + hand, recording flow (R/S keys).
- **First data collection** ✓ — ~50 demos pick toy + put in bin. Arm motion unstable — to address.

---

## Canonical teleop commands

> ⚠️ **Superseded 2026-07-06 — dual-hand + `--lock-torso` is now canonical; see [CHEATSHEET.md](CHEATSHEET.md) §1–§4** (single source of truth for commands). Kept below for reference only.

**Orin** (`~/project26/we-teleop`, `robo shell --profile tianji-driver`):
```bash
# Reset robot to safe pose first:
we driver --reset-once --acc 20 --control-mode pd --auto-reset-errors

# Then start driver (new terminal) — OLD single-hand version, see CHEATSHEET for current dual-hand:
export WUJI_RIGHT_HAND_SERIAL=306735523434
we driver --operator-address 6.6.7.166 --control-mode pd --auto-reset-errors --enable-wuji-hands --wuji-hand-side right --camera-serial 419222302053
```

**Homing note (from 2026-07-01 deploy testing):** `--reset-once` moves to the operational pose then shuts down → IDLE → **hands droop** (no brake). To home AND hold in PD (no droop), start the persistent driver with **`--move-to-init`** instead of a separate `--reset-once`. Add `--no-enable-camera` when the camera isn't needed (e.g. gt-obs deploy uses recorded obs):
```bash
we driver --operator-address 6.6.7.166 --control-mode pd --auto-reset-errors \
  --enable-wuji-hands --wuji-hand-side right --move-to-init --acc 20 --no-enable-camera
```

**Workstation** (`~/project26/we-teleop`, `robo shell -p operator`):
```bash
source scripts/setup/bootstrap_xrobot_pc_service.sh
we teleop --profile pico_ee_absolute_wuji_hands --driver-address 6.6.7.100:5559
```
Press `Enter` in the teleop window to anchor and start.

**Recording:** `R` = toggle record/start, `S` = save episode as success.

**Note:** `--control-mode impedance` still works as a deprecated alias for `pd`.

---

## Inference / eval commands (as of 2026-07-02 — ALL validated ✓, including live closed-loop)

Run on the **workstation** at a physical display (`robo shell -p operator`). All modes open a MuJoCo viewer → will fail headless (same `xlib` class as the rerun bug).

```bash
# 1. Open-loop eval — RECORDED obs, sim only, no robot (the "did it learn" check)
we eval act_pick_toy_0702 --split val       # val/unseen episodes — honest check

# 2. Deploy w/ RECORDED obs through the real deploy stack (no live sensors)
we deploy act_pick_toy_0702 --gt-obs --dry-run   # print resolved settings only
we deploy act_pick_toy_0702 --gt-obs             # publishes commands on :5558

# 3. Deploy LIVE — real obs, closed loop, needs driver up on Orin (VALIDATED 2026-07-02)
we deploy act_pick_toy_0702 --live --driver-address 6.6.7.100:5559 --ckpt-name 060000 --max-frames 2000
```

**⚠️ Config defaults `deploy.gt_obs: true`** → bare `we deploy <policy>` is RECORDED-obs, not live. Must pass `--live` for the real robot.

**New deploy UX (post-rewrite, `route: TeleopApp`):** starts **paused**. `Enter` = start/resume · `Space` = pause · `Tab` = reset robot to init pose (ramps + holds) + clears policy state. `--max-frames N` now **pauses** (not exits) after N frames — good backstop (~2000 ≈ 20 s at 100 Hz). Episode loop: `Enter` → run → `Space`/auto-pause → `Tab` reset → reposition toy → `Enter`.

**Bonus:** `we deploy --serve-policy` + `--policy-endpoint tcp://...` runs inference as a remote server (decouples GPU from robot machine — relevant to cloud-GPU/intern split).

---

## 07-16 MORNING UPDATE (mid-session)

- **Left hand REMOUNTED** (mount reprinted) → the 3 right-only patches REVERTED (driver_supervisor sides / hands.py ignore / gento.py zero-tool). Both hands default again.
- **Pulled upstream** `f1ea07e → 7983e6e` (8 commits, clean ff, no rebase; stash→merge→pop, zero conflicts). All keeper patches intact. Incoming: 1-euro PICO smoothing (labmate on our jitter!), extrapolation toggle (untested), impedance back, wandb eval, `we hands` CLI expansion.
- **Profile RENAMED**: `pico_ee_absolute_wuji_hands` deleted upstream → `pico_world_wuji_hands`; torso lock now `torso: hold` in it (new schema). Teleop: `scripts/runtime/pico.sh pico_world_wuji_hands --driver-address 6.6.7.100`. Driver: `we driver run --auto-reset-errors --no-camera` (robot address 6.6.7.190 is default now).
- **⚠️ MuJoCo window BLACK after pull — fix: `unset __NV_PRIME_RENDER_OFFLOAD` in every operator shell** (verified renders fine). Cause: upstream robo.nix sets `nixgl-nvidia` + PRIME offload for a hybrid laptop; our desktop has RTX 4090 as primary display GPU. Local robo.nix patch pending (needs full shell re-entry); add to upstream-report list (3rd works-on-their-machine bug).
- ⚠️ Orin: `sync_luna.sh nvidia@6.6.7.100` needed before next driver start (deps unchanged → no venv refresh, but exit/re-enter robo shells: flake.lock changed).

**07-17 AFTERNOON — REBASED ONTO TEAMMATES' NEW LOW-LEVEL WORK. Working tree = `wip/collection-integration-20260717`.**

- **New base**: `origin/rewrite-0711` @ `dfddafb` — control 60→50 Hz / driver 100 Hz (1:1 Gento servo pairing, their fix for a measured 10–35 Hz command-rate beat = likely our 一卡一卡), arm accel 30→60, PICO resample/smoothing rework + hand-EE smoothing, Orin desched stutter fix, wuji-sdk==2026.6.18 pinned UPSTREAM (labmate branch merged — our SDK-pin layer dropped), `driver_address: 6.6.7.100` default in ALL profiles.
- **Our commits cherry-picked on top (7)**: task-reset-pose ×3 + single-side teleop + camera-check `--exposure` + finger ramp + local-patch snapshot. Only 2 trivial conflicts. Plus: removed upstream's accidentally committed merge-backup test files (broke ruff/pytest on their clean tip) + gave our profile the `driver_address` default.
- **Suite: 466 passed; 4 known fails** — 2 upstream (`pico_world head: pico` contract test; JPEG-quality test not updated for `5cc9d04`'s 60) + our 2 init-gate locals. Ruff clean. Supervisor smoke (local sim) green.
- ⚠️ **UX change: bare `we teleop --profile …` now targets the REAL robot** (profile default driver). Pure sim = `--select-profile` + type `local`. CHEATSHEET §2 updated.
- **Old branch `wip/collection-integration-20260716` preserved** (incl. final snapshot commit `7201ad6`) — full restore point for the pre-merge 60 Hz state.
- **Upstream-report list grew**: merge-backup test files committed on rewrite-0711; JPEG-quality test stale; (still) head:pico profile test; init gate still not configurable.
- **NEXT: Orin sync + venv refresh (uv.lock changed!) → robot sanity (reset pose + short teleop feel at 50 Hz — jitter better?) → REAL COLLECTION.** ⚠️ Existing `rubik_test` pose yaml is joint-space, rate-independent — still valid. ⚠️ 07-17 morning validation was at 60 Hz; today's data will be 50 Hz — do NOT mix with any earlier episodes.

**07-17 MIDDAY — EVERYTHING ROBOT-VALIDATED ✓ (reset pose + staged hands + single-side teleop + recording loop). Remaining before real collection: camera serial fix.**

- On-robot ✓: `setpose rubik_test` → staged reset (body→fingers; fingers now RAMP over 2 s, `a53ee67`, after "too fast" feedback) → `usepose` cross-session → `pico_world_wuji_right` teleop (left welded to pose, left controller asleep = no effect) → full `R`/`S` loop.
- **Parquet proof** (`rubik_test_v0__20260717T105348`): `action.hand.left` bit-identical non-zero constants on all 701 rows; left-arm action deviation 0.0. IL data path closed.
- ⚠️ **Camera: driver needs `--camera-serial 419222302053` (our D455)**. Without it the RealSense pipeline grabs the robot's BUILT-IN head cam (that's what today's episodes recorded — throwaway). Canonical driver command now in CHEATSHEET §1. `we camera check` gained `--exposure` (`86780d9`) — re-sync Orin before using it there.
- **Camera 60 fps UNBLOCKED (07-17 pm)**: D455 was stuck on USB2 (no 60 fps mode) because the hub's A→C converter passes SuperSpeed in ONE orientation only — flipped + marked; hub now USB3.2/5000M, D455 5000M. Topology-check command + full story in week file / CHEATSHEET §1. **Buy a POWERED USB3 hub** (bus-power budget is the 07-15 blackout class risk; shopping checklist in week notes).
- ⚠️ In every NEW teleop session: `usepose rubik_test` BEFORE the first `Tab` (bare Tab = default init while left hand keeps gripping the cube).

**07-16 EVENING — TASK RESET POSE (plan 1.a) BUILT + SIM-VERIFIED ✓. Tomorrow = cube collection.**

**Branch layout (workstation):**
- **Working tree = `wip/collection-integration-20260716`** = rewrite-0711 tip + labmate's branch (wuji-sdk **2026.6.18** pin + calibration tools) + `feat/task-reset-pose` + our 6 uncommitted local patches. **This is the tree for tomorrow** — sync THIS to the Orin.
- `rewrite-0711` local = clean at origin (nothing untested on it — merge integration in only after robot validation / team discussion). Feature commits live on `feat/task-reset-pose` (`5e59053`+`37f7706`+`e51e399`) for the eventual upstream PR.

**The feature (all sim-verified 07-16 except staged hands):**
- `setpose <pose>` capture last live pose (body+hands) → `artifacts/task_poses/<pose>.yaml` + activate · `usepose <pose>` re-activate any session (pose name INDEPENDENT of `rc` dataset name — capture once, collect many datasets) · `home` clear + default reset. `Tab`/`R` ramp to active pose; convergence gate follows it; driver honors the RESET command's action (wire-identical for old flows).
- **Staged reset** (`e51e399`): pose WITH hand targets = body ramps first → converge → hands commanded → 1.0 s settle (`_RESET_HAND_SETTLE_NS`) → complete. ⚠️ **UNTESTED — needs gloves plugged** (sim test had no hands). Watch for log "reset body converged; commanding captured hand targets"; tune settle if grasp needs >1 s.
- Sim-verified: setpose/Tab-return/home/usepose + full recording loop (`rc` → `usepose` → `R` auto-records at pose → `S`) in keyboard profile.

**TOMORROW (07-17) cube-collection runbook:**
0. `git fetch` + check teammates' pushes; Orin: `sync_luna.sh nvidia@6.6.7.100` FROM the integration branch + venv refresh if deps complain (SDK pin changed!); exit/re-enter robo shells; `unset __NV_PRIME_RENDER_OFFLOAD` (operator).
0.5. ~~BUILD SINGLE-SIDE TELEOP~~ — **✅ BUILT 07-17 morning (`ac81a41` on the integration branch), suite green, headless smoke OK.** Profile **`pico_world_wuji_right`** (`sides: [right]`): right arm+glove teleoped, left frozen at task pose, left glove/controller not needed, resets whole-body. ⚠️ Spec correction: idle LEFT hand is actively re-commanded each tick from the task pose's captured grasp (a `None` hand action records ZEROS → deploy would open the hand). Recorded action = applied command = grasp constants; both hand modalities still required. Preflight: `we hands check --side right`. Sim check pending (untracked `keyboard_right_tmp.yaml` in tree — delete after; it fails the profile-list contract test while present). Status header w/ details in [design/single-side-teleop.md](notes/design/single-side-teleop.md).
1. **CAMERA IS BLOCKING**: robot recording requires image modality (wuji profile). Unresolved from 07-15: Orin driver didn't see synced `cameras.yaml`. Now easier: `we camera check`, then `we driver run --auto-reset-errors --camera-name tianji_head_d455 --camera-exposure 100` (new flag; 100 = 10 ms in D455 units).
2. Gloves: sanity in sim first (`sim_glove_to_hand.py --side right`), pinch thresholds currently index/middle 1.5/2.0 both hands, tactile gates OFF (phantom pressure likely = SDK 2026.7.2 artifact — RE-TEST gates on 2026.6.18, may re-enable).
3. **Test staged hand reset with gloves** (sim ok) BEFORE robot.
4. Collection: teleop to left-grasp-cube pose → `setpose rubik_twist` → `rc <dataset>` → episodes: `R` (ramps arms→fingers, auto-records) → task → `S` → re-seat cube → `R` … → `home` at end.
5. Report to labmates: `test_launch_profiles` broken on clean rewrite-0711 (test expects pico_world `head: hold`, config ships `pico`); share SDK-artifact finding + phantom-pressure story with retargeting owner.

**07-16 afternoon (pinch debugging + fixes → 2 commits READY TO PUSH):**
- **Pinch over-sensitivity ROOT CAUSE: phantom tactile pressure** (~0.28 at rest, right glove of new pair) firing the `pinch_pressure` press-pinch gate — NOT the distance thresholds. Tactile gates disabled in both hand yamls (`p_on: 1.0 / p_full: 2.0`; keep p_full > p_on or gate locks ON). d1/d2 were a red herring; right hand currently d1=d2=0 (A/B: twisting improved!), restore values in yaml comment.
- **Ported `sim_glove_to_hand.py`** to rewrite APIs (`scripts/inputs/`, untracked) — MuJoCo hand + per-second `dist_cm`/`pinch_alpha`/`press_gate` printout; single-side network preflight.
- **Commits PUSHED to rewrite-0711 ✓**: `b81567c` hands open on driver shutdown (+ incident note; verified on robot via log line; ⚠️ held objects drop on driver stop) · `16cd156` `--camera-exposure` flag (labmates' default kept; our D455 = `we driver --camera-exposure 100`; camera.py local patch RETIRED — local diff now 6 files).
- Teammates pushed: recording overhaul w/ quality markers (likely THE data-loss fix — verify at meeting) + probing experiments; empty branch `fix/hand-retargeting-from-rewrite-0711` — share phantom-pressure findings with them BEFORE they start!
- Torso shakes: `head: hold` + `torso: hold` now both set in `pico_world_wuji_hands.yaml` (rewrite supports it natively — no develop migration needed).

## Pick up next (as of 2026-07-15 EVENING — urgent-video day done, 2 videos captured)

**TOMORROW MORNING (07-16), in order:**
1. **Test the swapped glove pair** — old right glove had degraded signal (weird reconstructed hand). Verify new pair's IPs match `config/inputs/wuji.yaml` (.100/.101 — per-glove static config may differ!), then `we hands check --source wuji` + sim feel (`we teleop --profile pico_ee_absolute_wuji_hands`, no driver = pure sim). ⚠️ **Re-test pinch with stock-ish d2 first (4.0 / middle 3.0)** — today's 2.5/2.2 tuning was against the BAD glove and is probably an artifact.
2. **Continue harder task videos** (single right arm only — bimanual incl. cube-twist blocked by the melted left mount) → send to supervisor.
3. Robot state: **rewrite-0711 + 9-file local patch set** ([diff](notes/implementation/rewrite-0711-local-patches-20260715.diff), also in working tree, synced to Orin). Left hand DISMOUNTED (mount melted — photo it, tell labmates/3D-owner, plan reprint in heat-resistant material). Driver: `we driver run --robot-address 6.6.7.190 --auto-reset-errors --no-camera` on Orin; teleop: `we teleop --profile pico_ee_absolute_wuji_hands --driver-address 6.6.7.100`. SSH now passwordless.
4. **Camera before any data collection**: Orin driver didn't see the synced `cameras.yaml` d455 profile (unresolved); exposure units bug patched locally (round 2 — `daa65c0` re-introduced it).
5. **When left hand remounts**: revert 3 right-only patches (driver_supervisor sides, hands.py ignore, gento.py zero-tool) — all marked with WARN comments.
6. **Upstream/report to labmates**: exposure not camera-portable (again) + `--camera-exposure` flag; init-pose gate too strict for hand-loaded wrists (0.5°); hand-command rejection holds whole robot silently (ignore-vs-reject + `--hand-side` flag); ARM_TOOL timeout on tool-less arm is back; sync_luna doesn't refresh the Orin venv; **robo.nix graphics not host-portable** (nixgl-nvidia + `__NV_PRIME_RENDER_OFFLOAD=1` hardcoded for a hybrid laptop → black MuJoCo window on our NVIDIA-primary desktop; suggest host-configurable `hostGraphics`).

**Big picture: teleop of the two-step cube-twist task WORKS (vertical two-finger twist → horizontal twist). Next big goal = IL / autonomous execution of it.** Full plan + meeting-prep questions in [week-2026-07-13](notes/weekly-notes/week-2026-07-13.md). Full 07-15 session war story in the week file's daily log.

0. **`git pull` first** (labmate pushes/rebases over the break; snapshot + `reset --hard` if diverged; exposure fix recoverable via `git cherry-pick 96cc236`). Then `sync_luna.sh nvidia@6.6.7.100`.
1. **Team meeting** — ask: Quest integration status; is data replay really fixed on the newer branch (which?); who owns the recording data-loss bug (a few seconds lost intermittently); big refactor coming? → then pick our first focus.
2. **Design the reset-to-grasping-pose change (plan 1.a)** — episodes should start with the left hand already grasping the cube, not the default arms-at-side pose; touches teleop + collection + deploy (train/deploy start-pose match). Sketch options before discussing with team. Related: hands do NOT open on `Tab`/`s` reset (upstream regression) — the redesign must decide hand state at reset anyway.
3. **Labmate's verdict on the jitter parquets** (sent 07-10, `rubric-cube-0710__*`) — follow up; confirm they know state is 60 Hz sampled (>30 Hz aliases).
4. **Test B of finetune store** (`mv ~/.we-teleop/wuji_finetune.json.off` → on, rerun sim; watch right-pinky `d1 7.95` suspect) + on-robot feel of colleague retargeting incl. our middle `d2: 3.0`.
5. **Exposure fix follow-ups**: upstream a `--camera-exposure` flag (D405 µs vs D455 100µs-units — tell labmates); optionally pin `rs.option.gain` too.
6. **Left glove tactile sensor** — vendor still silent; follow up on replacement.
7. **Lower the default torso pose** — may fold into the 1.a reset-pose redesign.
8. **Data replay** — pull the newer branch teammates say fixes it, retest before trying our `--max-frames`/`.rrd` workarounds. Bimanual-ergonomics thinking still open. Glove extension proper fix when reach needed. RL-for-dexterous-manipulation reading runs in parallel on the other laptop (notes repo now pushed each session).

**2026-07-07 session summary:** operator trained (pm); cube single-steps good, bimanual hard (ergonomic conflict, open); pinch over-sensitivity root-caused to `calibrate_wuji_pinch.py` degenerate thresholds → reverted to defaults (verified better); tactile monitor diag script added; PICO sleep NOT fixed by white-paper tape (psensor confirmed fooled — deeper cause TBD) and NOT network.

## Pick up next (older list)

0. **Report to labmates / Friday meeting**: full pipeline works end-to-end; state-OOD bug root cause + resolution (bug note has the write-up). Mention the upstream deploy rewrite fixed it and our diagnosis independently confirmed why.
1. ~~Port the ARM_TOOL_DYNA conditional fix before Orin update~~ — **OBSOLETE (2026-07-03): TIANJI firmware/SDK update fixed the underlying controller bug** (it was TIANJI-side all along; non-zero dyna with no hand no longer corrupts mode switching). And since our hand is always mounted now, non-zero kine/dyna offsets are always correct — upstream's unconditional `ARM_TOOL_DYNA` is fine as-is. No port needed before syncing the Orin.
2. **New control params: adopted 2026-07-03, shaking LARGELY MITIGATED** ✓ — Orin synced to new master (`sync_luna.sh`); driver hold test confirms new gains (KP 10 flat, KD (0.6,0.2,0.2,0.16,0,0,0)) mostly fix the arm shaking. Use `--control-mode pd` (not `impedance`) everywhere now. Future data collection happens on the new gains → no train/deploy mismatch going forward.
3. ~~More eval episodes + success rate on pick-toy; record videos~~ — DONE 2026-07-03 (multiple episodes with object-position variation + videos).
4. **Right-hand pinch over-sensitive** — reproduced in sim 2026-07-03 (`sim_glove_to_hand.py --side right`, kinematic mode added + pushed — no robot needed). Colleague's new retargeting algorithm **lands Monday morning** — pull + re-test in sim then.
5. **Left glove** — waiting on replacement (external). #1 blocker for dual-hand.
6. **`we data rerun` replay bug** — display lockup; investigate (check if the upstream rerun changes fixed it).
7. **Background board** — ask W or S.

**Still on radar:**
- PICO wrist calibration (redo once mounts stable)
- Quest headset setup
- Wuji hand TIANJI official mount (power/comm routing under discussion)
- Arm droop (no brake) — awaiting TIANJI response
- Camera mount proper fix (D455 still taped; live obs verified fine, but fragile)

---

## Local changes vs upstream (2026-07-02 — after master reset to origin)

**Branch `wip/live-deploy-state-ood` (`a96f840`)** = full pre-refactor snapshot (old debug flags, old fixes, configs). Never delete it; recover single files with `git checkout wip/live-deploy-state-ood -- <path>`.

**Restored into working tree (untouched by upstream):** `config/inputs/retargeting/wuji_{left,right}.yaml` (pinch calib), `config/policy/act_pick_toy*.yaml`, `scripts/diagnostics/debug_tianji_arm_mode_switch.py`.

**New local changes (2026-07-03):** `driver.py` — `PD_ARM_KP`/`PD_ARM_KD` env-var overrides for gain testing (uncommitted, synced to Orin; watch for conflicts when pulling — labmates touch this file). `sim_glove_to_hand.py` kinematic mode was committed + pushed (not local).

**New local changes (2026-07-08, all uncommitted):** (1) `third_party/wuji_retargeting_v2_pkg/.../opt/enhanced_analytical.py` — per-finger `d_near_cm`/`d_far_cm` via `_parse_per_finger` (**UNTESTED**; vendor pkg is editable-installed so it's live); (2) `config/inputs/retargeting/wuji_right.yaml` — `d_near_cm: {index: 3.5, middle: 3.0, ring: 3.0, pinky: 3.0}`; (3) `wuji_left.yaml` — `d_near_cm: 4.0`; (4) new `scripts/diagnostics/monitor_wuji_tactile.py` (2026-07-07). Stale `wuji_{left,right}.yaml.bak` from the bad 07-07 calibration still on disk — delete when convenient, do NOT restore from them.

**Old uncommitted fixes — status after the upstream rewrite:**
1. **init_qpos crash fix** — obsolete for deploy (path deleted); `create_open_loop_runtime` still exists for `we eval`; re-port only if eval hits the IndexError with hands enabled.
2. **deploy hand-reset fix** — code deleted upstream; NOT re-fixed: new `_send_hardware_reset` still sends the *last* hand pose on reset (`reset()` doesn't clear `hand_targets`). Verify on robot whether the hand opens on `Tab`; re-flag to labmate if it stays pinched.
3. **teleop hand-reset fix** — same situation as (2) for the `S`-save reset.
4. **driver.py ARM_TOOL_DYNA fix** — obsolete (2026-07-03): TIANJI firmware/SDK update fixed the controller-side bug; hand always mounted → non-zero dyna always correct.

## Watch out / open threads

- ~~Workstation and Orin on different we-teleop versions~~ — **SUPERSEDED 2026-07-03: decided to sync Orin to new master** (`sync_luna.sh`). The pick-toy policy was a sanity check only; no need to preserve collection-era deploy dynamics. Old tree still recoverable from `wip/live-deploy-state-ood` if ever needed.
- **⚠️ Upstream changed driver control params**: `PD_ARM_KP` (24,24,20,16,12,8,8)→(10×7), `PD_ARM_KD` (0.3×7)→(0.6,0.2,0.2,0.16,0,0,0), `alpha` 0.95→0.5. Adopting them changes tracking behavior vs. the training data → plan a re-collect/retrain when switching.
- **⚠️ `--control-mode impedance` is NO LONGER a pd alias on NEW driver code** — it's now real joint impedance (`switch_to_imp_joint_mode`, no velocity feedforward). Orin's old driver still treats it as alias. Once the Orin updates, use `pd` explicitly everywhere.
- **⚠️ Upstream history was force-pushed/rebased** — if `git pull` complains about divergence, don't merge; snapshot to a branch and `reset --hard origin/master` (as done 2026-07-02).
- **Two separate workstation networks:** (1) robot ↔ workstation DIRECT ethernet on `enp14s0` = `6.6.7.166` (`--operator-address 6.6.7.166`, unchanged); (2) PICO ↔ workstation via Xiaomi router on `enp13s0` = `192.168.31.48` (DHCP; lock static). PICO = `192.168.31.110` (wired via USB-C ethernet adapter since 2026-07-06; was `.231` on router WiFi — PICO IP is DHCP and doesn't matter to any config, connection is PICO→WS). Static cmd: `sudo nmcli con mod "有线连接 1" ipv4.method manual ipv4.addresses 192.168.31.48/24 ipv4.gateway "" ipv4.dns ""` (also clears stale `192.168.123.123`). PICO bootstrap: `export TELEOP_XROBOT_HOST_IP=192.168.31.48` before sourcing.
- **Right Wuji hand serial**: `306735523434` — export before driver launch.
- **Always pass `--enable-wuji-hands` when hand is physically mounted** — controls dynamics compensation. If omitted with hand mounted, dynamics will be slightly wrong (won't crash).
- **Right hand only on robot**: `--wuji-hand-side right` — left Wuji hand not yet mounted.
- **XRobot PC Service**: re-source bootstrap each new shell: `source scripts/setup/bootstrap_xrobot_pc_service.sh`.
- **Robot SSH credentials**: Orin `nvidia@6.6.7.100` / x86 `root@6.6.7.190` / chassis `sunrise@6.6.7.6`.
- **`we train` test config**: `config/policy/act_pick_toy_val_test.yaml` — triggers validation at step 1000, useful for quickly testing training pipeline changes.
- **Start each session with `git pull`** in `~/project26/we-teleop` — labmates push fixes without announcing (and sometimes rebase!).
