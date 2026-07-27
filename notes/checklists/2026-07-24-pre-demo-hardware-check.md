# Pre-demo hardware check — 2026-07-24 (with operator)

> On-hardware checklist before the 07-27 demo week. Goal: two vetted hand configs,
> confirmed thumb/body behavior, and a smoothing baseline — all folded into the
> **stable demo branch**. ⚠️ Any source/config edit here must be (1) captured on the
> stable branch and (2) synced to the Orin (`sync_luna.sh nvidia@6.6.7.100`, + venv
> refresh if deps changed). Smoothing constants are Python, not yaml → source edit → re-sync.

Session-start: `git pull` we-teleop · `unset __NV_PRIME_RENDER_OFFLOAD` in every operator
shell · Orin sync if tree changed · know the pause/Esc path before teleop.

---

## a. HANDS

### a1. Two retargeting config sets — enhanced-pinch vs pinch-off, easy switch

**What controls it** (both `config/inputs/retargeting/wuji_left.yaml` + `wuji_right.yaml`):
- Distance pinch snap: `retarget.pinch_thresholds.<finger>.{d1,d2}` — currently index/middle
  `1.5/2.0`. **`d1=d2=0` = snap fully OFF** (in-file comment: twisting much better off).
- Press-pinch (tactile) gate: `retarget.pinch_pressure.enabled` (true) + `press_grip.enabled` (true).

- [ ] **PRESET A — "enhanced pinch"** (grasp thin items): keep current distance thresholds
      (~1.5/2.0 on index/middle) + `pinch_pressure.enabled: true`. Visually check: can pinch a
      thin object; fingers not glued.
- [ ] **PRESET B — "pinch off / natural"** (twist-heavy, no sticky fingers): set index/middle
      (and ring/pinky as needed) `d1=d2=0`; consider `pinch_pressure.enabled: false`. Visually
      check: fingers move independently, no snap-stick during twist.
- [ ] **Decide the switch mechanism** — ⚠️ NOT built yet (loader reads fixed `wuji_{side}.yaml`).
      Options, lightest first:
      1. **Swap script** (recommended for demo week, zero code-path risk): keep presets as
         `wuji_{side}.pinch.yaml` / `wuji_{side}.twist.yaml`, a one-liner script copies the
         chosen one over `wuji_{side}.yaml`. Claude can build this tonight/tomorrow AM.
      2. Small feature: profile- or env-selected retargeting variant (cleaner, but a code change
         → coordinate with retargeting owner; not during demo week).
- [ ] Record which preset each demo TASK wants (twist → B; pinch/grasp → A).

### a2. Thumb movement — natural?

- Current state: `retarget.thumb_abd_track.w: 0.0` in BOTH yamls = thumb amplification OFF
  (pure geometry). Earlier A/B: labmates' `w:30` gave more flex but **overshot on our glove**.
- [ ] Watch thumb during teleop: is spread/opposition natural, or under-reaching / reverse-bending?
- [ ] If under-reaching → run `uv run python scripts/calibration/calibrate_thumb_spread.py --side <side>`
      (holds 2 poses ~4s; writes a per-glove `thumb_abd_track` into `wuji_finetune.json`). Verify
      with `uv run we hands experiment --side <side> --thumb-norm-delta 0.04`.
- [ ] If still unnatural after our own calibration → **coordinate with the retargeting owner**
      (do NOT hand-tune his experimented gains; report the behavior + our calibration frames
      `artifacts/diagnostics/thumb-calib-frames-<side>.jsonl`).

⚠️ Grip-force / thermal (left hand faulted 07-22: over-temp → transmission-slip critical):
- [ ] Keep commanded grip force modest in the reset/task pose; collect in batches w/ cooldowns.
- [ ] Confirm with ops: is a Wuji left-hand replacement available on short notice if the slip fails?

---

## b. BODY / TORSO — lower pose, stability, joint locking

- Profile keys (`config/teleop/pico_world*.yaml`): `torso ∈ {hold, ik}`, `head ∈ {hold, pico}`.
  Currently `torso: hold`, `head: hold`. **`torso` = 6 leg joints** (Gento Luna_Leg_J1..J6) —
  "lower body / squat" = changing these leg angles. Default standing:
  `Leg_J2=60°, Leg_J3=-70°` (the main knee/hip pair).
- ⚠️ There is **no per-joint lock config** and **no body-height parameter** — torso is all-or-nothing
  (`hold` = frozen, `ik` = fully IK-driven). No `ik_preset`.

- [ ] **Test live torso/leg teleop**: set `torso: ik` in the profile, teleop the body down/up.
      Is the motion **stable** (no oscillation/lurch under hand weight)?
- [ ] **If stable** → good; note the range operators can safely reach.
- [ ] **If unstable** → fall back to a **static captured lower pose**: hold robot at the lowered
      posture → `setpose <name>` (captures 22 joint positions incl. 6 leg joints) → `usepose <name>`
      makes it the reset/start pose, with `torso: hold` (torso frozen at the low pose during the task).
      This is the safe demo path if live squat teleop is jittery.
- [ ] Note: "lock some joints" isn't supported granularly — if partial leg locking is truly needed,
      that's a code change → coordinate with owner (not demo week).
- [ ] Watch the init-gate trend (now 2.5°, Arm_R6 was 2.01° under hand load) — record joint+error+vel
      before bumping; if it hits ~2.5° again, investigate don't bump.

---

## c. OVERALL — smoothing parameters

⚠️ All smoothing is a **1-euro filter**, values **hardcoded in Python** (not yaml). Lower
`min_cutoff_hz` and/or lower `beta` = smoother (more lag). Editing = source change → re-sync Orin.

- [ ] **Arm (PICO) motion** — active path is joint-space: `DEFAULT_JOINT_SMOOTHING` in
      `src/teleop/control/inputs/pico/smoothing.py:60` (currently `min_cutoff_hz=2.0, beta=5.0`).
      Pose-space smoothing is currently **disabled** (`smoothing=None`, `control_process.py:329`).
      If arm feels jittery → lower those two numbers first.
- [ ] **Hand/finger motion** — CONFIRM the hand smoothing IS merged (it is on this tree):
      `DEFAULT_HAND_SMOOTHING` in `src/teleop/control/inputs/hands.py:119`
      (`min_cutoff_hz=0.5, beta=1.0`), plus `retarget.lp_alpha: 0.5` in both wuji yamls
      (lower = smoother). ✅ present — just verify it's still there after any pull.
- [ ] **Driver chatter**: keep `driver_extrapolation` OFF (commented in `config/runtime.yaml:25`;
      on = less lag but amplifies steps into chatter). Leave off for a smooth demo.
- [ ] Actuator low-pass (Wuji fingers): `config/actuators/wuji.yaml` `low_pass_cutoff_hz: 10.0`
      — leave unless fingers buzz.

---

## d. (to be added)

- [ ] Verify D455 on **USB3 5000M** (camera frame-drops still unfixed — batch1 ~8% / batch2 ~11%
      faulty; matters if the demo records). Fallback: `config/cameras.yaml` d455 `fps: 60→30`.
- [ ] Quest tracking placement — controller visibility limits; the PICO/right-controller snap
      (arm lurch) wasn't reproduced last session but confirm before trusting; task-space step
      clamp suggestion still stands.
- [ ] Emergency/safety: operator + Yihe both know pause (`Space`) / stop (`Esc`) before any teleop.
- [ ] **⚠️ CHASSIS (底盘) motion + upper body simultaneously** — the film plan has shots where the
      robot DRIVES while the upper body also moves. Currently the 底盘 is **controller-only, NOT in
      our teleop pipeline** (our profiles cover arms/torso-legs/head/hands via PICO/Quest+gloves, no
      base velocity command). CHECK: can base be driven (controller) *at the same time* as upper-body
      teleop without conflict? If simultaneous base+upper-body is required and not possible → raise
      early (owner + advisor): is base teleop in-pipeline planned, or do we choreograph base separately?
- [ ] _(add more as they come up)_
