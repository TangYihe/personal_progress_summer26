# Pre-pack verification — 2026-08-04 AM (robot packs in the afternoon)

> **Last hardware window before the 08/11–08/13 shoot** (unless a setup day at the site gives us
> another — ❓ confirm). Everything that needs the real robot happens this morning. The afternoon is
> teardown and packing.
>
> Goal: (1) the pipeline still works end-to-end post-merge, (2) hand configs are the ones we want to
> travel with, (3) **whole-body movement is characterized** — and if it's shaky, tuned *today*.
>
> Detailed knob reference lives in [2026-07-24-pre-demo-hardware-check.md](2026-07-24-pre-demo-hardware-check.md)
> — this doc is the ordered run for 08-04, not a replacement. File pointers below were
> **re-verified against the post-refactor-0722 tree on 2026-08-03**.
>
> ← [week-2026-08-03](../weekly-notes/week-2026-08-03.md) · [packing list](../demo-shoot-prep.md#4-packing-list)

---

## Order matters

Parts A–C need the robot. Part D is the config freeze and must happen **after** any tuning in C,
because tuning changes files that have to be synced and captured. Part E is teardown.

**⚠️ Anything you change in C must reach the Orin (Part D2) or it doesn't exist on the robot.**

---

## Part 0 — Session start (~10 min)

- [ ] `git pull` in `we-teleop` (shared branch `origin/refactor-0722`; check labmate pushes).
      ⚠️ **Last chance** — do not pull on-site (no uplink → a bumped `flake.lock` breaks
      `robo shell`, exactly as it broke the Orin on 07-29).
- [ ] Confirm the LOCAL uncommitted edits are still present and are the ones we want to travel:
      `wuji_finetune.json`, `config/teleop/pico_world_wuji_hands.yaml`, `driver_supervisor.py`.
- [ ] Every operator shell: `unset __NV_PRIME_RENDER_OFFLOAD` (same interactive `robo shell`,
      not via `robo run`) — else the MuJoCo window is black.
- [ ] Both Yihe and the operator know the stop path (`Space` pause / `Esc`) before anything moves.

---

## Part A — Bring-up + pipeline sanity (~30 min)

- [ ] **Orin driver:** `robo shell -p tianji-driver` →
      `we driver --auto-reset-errors --camera-serial 419222302053 --camera-exposure 100`
      (defaults: robot `6.6.7.190`, `pd`, `--move-to-init`, bimanual hands + camera).
      - ⚠️ **Init-gate may trip** → recover via 上位机 (CHEATSHEET §7). **Record the joint + error +
        velocity** before touching the gate — the right-wrist error is on a growing trend
        (07-15 ~0.7° → 07-20 Arm_R6 2.01°, gate now 2.5°). Do not bump it as a reflex.
      - ⚠️ Watch for Wuji **`Bus undervoltage` F4J3/F4J4** (07-28). If it recurs: stop, check the
        hand power rail and connectors, and log which hand.
- [ ] **Gloves reachable** before starting teleop — a wuji profile fails without them.
      Connect-timeout with carrier up ⇒ suspect cross-wiring (CHEATSHEET §3).
- [ ] **D455 on USB3 5000M** — check before trusting any recording:
      `ssh nvidia@6.6.7.100 'for d in /sys/bus/usb/devices/*; do [ -f "$d/product" ] && echo "$(cat $d/product) | $(cat $d/speed) Mbps"; done'`
      Pass = hub **and** D455 at 5000. Fallback if drops persist: `config/cameras.yaml` d455 60→30 fps.
- [ ] **Workstation teleop:** `scripts/runtime/pico.sh pico_world_wuji_hands`, `Enter` to anchor.
      Both arms + both hands live; head/torso currently `hold` (LOCAL).
- [ ] **End-to-end proof** — one short recorded episode, or a `we data replay` on the
      `_repaired`/`_merged` set. ⚠️ Raw datasets cannot be replayed (one faulty episode rejects the
      whole set). Confirms arms + hands + camera + recorder all still work post-merge.
- [ ] **Optional but valuable: one autonomous run of the validated policy.** Now that the full
      deploy stack travels with us, an autonomous shot is on the table for the script — but only if
      we know it still runs. Confirming it here is much cheaper than discovering it on set.

---

## Part B — Hand configs (~30 min)

**Decide what travels.** Current LOCAL `wuji_finetune.json` state: thumb `w=30` both hands;
**index/middle pinch fully OFF (`d1=d2=0`, both hands)**; thumb `segment_scaling` = base default.

- [ ] **Sanity-check the current config in sim first** (no robot time burned):
      `scripts/inputs/sim_glove_to_hand.py` — watch the per-second `dist_cm` / `pinch_alpha` /
      `press_gate` print. `--record <mp4>` captures an A/B candidate.
- [ ] **⚠️ Decide the pinch question before packing.** `d1=d2=0` was tuned for the *single*
      horizontal twist. If the script needs a real **index/middle grasp** (picking up a prop,
      holding an object), restore `d2`≈3.0 and restart teleop. Getting this wrong is not fixable
      on-site without an operator A/B.
- [ ] **Prepare TWO stable hand yamls to travel** (long-deferred item, now or never):
      pinch-enhanced vs plain. No switch mechanism exists → the swap is `cp` the preset over
      `wuji_{side}.yaml` + restart teleop. Write the exact swap command into the packing notes so it
      can be done under time pressure on set.
- [ ] Record **which preset each scripted shot wants** → feeds the
      [script risk register](../demo-shoot-prep.md#1-shoot-script-risk-register).
- [ ] **Thermal:** the hand overheats under sustained teleop. Confirm the plan is batched takes with
      cooldowns, and that the shot list is compatible with that.
- [ ] ⚠️ **Do not expect the multi-step twist to be drivable** — it is still blocked on
      hand-retargeting smoothness (07-30, the owner's thread). Verify today whether it has improved;
      if not, the script must not depend on it.

---

## Part C — 🔴 WHOLE-BODY MOVEMENT (the main event, ~60–90 min)

This is the item with real tuning risk, so give it the biggest block and start it early enough that
tuning still fits before packing.

### C1. What is and isn't possible (know before testing)

- **Chassis (底盘) CANNOT be teleoped** — fails closed in `gento.py`; it needs a separate
  controller. If the script has the robot **driving while the upper body moves**, that is
  choreographed separately, not teleoped. ⚠️ **Confirm base + upper-body simultaneity works at all**
  — this has been an open question since 07-24 and it is a script-level constraint.
- **Torso = 6 leg joints** (`Luna_Leg_J1..J6`), all-or-nothing: `torso: hold` (frozen) or
  `torso: ik` (fully IK-driven). **No per-joint locking, no body-height parameter.**
  Default standing: `Leg_J2=60°, Leg_J3=-70°`.
- Currently **`head: hold`, `torso: hold`** in `config/teleop/pico_world_wuji_hands.yaml:12-13`
  (our own LOCAL edit — this is what turned torso teleop off).

### C2. Test

- [ ] **Sim first**, then robot. Set `torso: ik` in the profile and re-run teleop.
- [ ] Teleop the body down/up. Judge: **stable, or oscillating/lurching under hand weight?**
- [ ] Note the range operators can safely reach — write it down; it becomes a script constraint.
- [ ] Re-check the init-gate error afterwards (leg motion loads the arms differently).

### C3. If it's shaky — tune, in this order

All smoothing is a **1-euro filter with values hardcoded in Python**, not yaml → every edit is a
source change → **must be re-synced to the Orin** (Part D2). *Lower `min_cutoff_hz` and/or lower
`beta` = smoother, at the cost of lag.* Verified 2026-08-03 against the current tree:

| Knob | Location | Current |
|---|---|---|
| Arm/joint smoothing (**active path**) | `src/teleop/control/inputs/pico/smoothing.py:60` `DEFAULT_JOINT_SMOOTHING` | `min_cutoff_hz=2.0, beta=5.0` |
| PICO pose-space smoothing | `src/teleop/control/inputs/pico/smoothing.py:50-51` | **inactive** — `PicoReceiver(alignment, smoothing=None)` at `control_process.py:502` |
| Hand/finger smoothing | `src/teleop/control/inputs/hands.py:70` `DEFAULT_HAND_SMOOTHING` | `min_cutoff_hz=0.5, beta=1.0` |
| Hand first-order stage | `config/inputs/retargeting/wuji_{left,right}.yaml` `lp_alpha` | `0.5` (lower = smoother) |
| Wuji finger actuator LPF | `config/actuators/wuji.yaml:10` `low_pass_cutoff_hz` | `10.0` (leave unless fingers buzz) |
| Driver extrapolation | `src/teleop/runtime/driver_supervisor.py:532` | `extrapolation=False` — **keep off** (on = less lag, but amplifies steps into chatter). ⚠️ moved into code; it is no longer a `runtime.yaml` toggle |

- [ ] Change **one knob at a time** and re-feel. Log every value you try and the verdict — otherwise
      you cannot get back to a known-good state under time pressure.
- [ ] **Fallback if live torso teleop stays unstable: a static captured lower pose.** Hold the robot
      at the lowered posture → `init_pose capture <name>` (upstream's name for our old `setpose`;
      captures all joints incl. the 6 leg joints) → `init_pose <name>` to activate, with
      `torso: hold`. The torso is then frozen at the low pose for the whole take. **This is the safe
      demo path** and it is worth capturing the pose *even if live teleop works*, as insurance.
- [ ] Whatever the outcome, **write the verdict into the script risk register** — "torso live" vs
      "torso frozen at captured pose" changes what the script can ask for.

---

## Part D — Config freeze (~30 min, AFTER all tuning)

- [ ] **D1. Lock `enp13s0` static** — it is DHCP today, and `192.168.31.48` is hardcoded in
      `TELEOP_XROBOT_HOST_IP` and in the headset app's connect target. Same router on-site ⇒ same
      subnet, but the *lease* is not guaranteed.
      ```bash
      sudo nmcli con mod "有线连接 1" ipv4.method manual ipv4.addresses 192.168.31.48/24 \
           ipv4.gateway "" ipv4.dns ""
      ```
      Also clears the stale `192.168.123.123`. ⚠️ Make sure `.48` is outside the router's DHCP pool
      (or reserve it) so nothing else claims it first. Verify: `ip addr show enp13s0`, then a full
      teleop bring-up to prove the headset still connects.
- [ ] **D2. Final Orin sync** — `scripts/setup/sync_luna.sh nvidia@6.6.7.100` (⚠️ always pass the
      address; the script's default is another robot). This is also what finally puts the
      **hands-open-on-shutdown fix** (`driver_supervisor.py`, LOCAL + uncommitted) on the Orin.
      Restart the driver and confirm hands ramp **open** on driver exit.
      - Deps/flake unchanged ⇒ rsync only, no venv refresh. If C3 changed nothing, this is quick —
        but do it anyway; the smoothing edits are Python and only reach the robot this way.
- [ ] **D3. Commit the travelling state.** The tree currently carries LOCAL do-not-push work on
      `wip/collection-integration-20260728` plus uncommitted edits. Get it onto a branch so a
      mistake on-site is recoverable with one `git switch` — this is the long-deferred
      **stable demo branch**, and it matters more now that the tree itself is what travels.
      ⚠️ Do **not** push the LOCAL commits (`1c7a1c3` init-gate loosen, `06757f6` machine config).
- [ ] **D4. Smoke test from a cold start** — reboot the workstation, then bring the whole stack up
      from scratch without fixing anything by hand. That is the on-site experience; if something
      only works because of state in your current shell, find out now.

---

## Part E — Teardown + pack (PM)

- [ ] **📸 Photograph everything before unplugging anything** — both NIC ports, both glove dongles,
      the USB hub, the D455, the robot-side connections. The 07-27 走线 rework cross-wired the two
      glove ethernet cables and cost a full debugging session; photos make that a 30-second check.
- [ ] **Label the cables**: glove L / glove R, `enp13s0` (teleop LAN / router), `enp14s0` (robot).
- [ ] ⚠️ **The A→C converter orientation is MARKED — do not replug it blind.** SuperSpeed passes in
      one orientation only.
- [ ] **Robot charged and the charger packed.** Per [leave-office-sop.md](leave-office-sop.md) — the
      07-28 drain happened exactly because a charger went unplugged during cable work and nobody
      said anything.
- [ ] **Bring the USB seed stick** — the workstation is now a single point of failure with no
      spare, and the seed is already built and verified. It costs nothing to carry.
- [ ] Pack to the [packing list](../demo-shoot-prep.md#4-packing-list); tick items physically, not
      from memory.
- [ ] ⚠️ **The workstation NIC is an add-in card.** Transport shock can unseat it — if both ethernet
      links misbehave on arrival, **reseat the card first**, before debugging cables or config.
