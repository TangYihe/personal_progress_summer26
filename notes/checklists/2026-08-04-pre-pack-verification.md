# Pre-pack verification — 2026-08-04 (pack after 17:00)

> **Last hardware window before the 08/11–08/13 shoot** (unless a setup day on-site gives another —
> ❓ confirm). Operator available all day; **teardown starts after 17:00** when they go off-shift.
>
> **🔴 TOP PRIORITY (set 2026-08-03): whole-body movement — can the robot stand up / squat down
> naturally?** The script leans heavily on whole-body motion and *not* on dexterity, so the hand
> work is demoted to a sanity check. If the squat isn't natural, we have the whole day to fix it,
> and two fallbacks already scoped (captured squat pose · per-joint freezing).
>
> Knob reference: [2026-07-24-pre-demo-hardware-check.md](2026-07-24-pre-demo-hardware-check.md)
> ⚠️ **that doc says "no per-joint lock config" — that is WRONG on this tree; see B3 lever 4.**
> All file pointers below re-verified against the post-refactor-0722 tree on 2026-08-03.
>
> ← [week-2026-08-03](../weekly-notes/week-2026-08-03.md) · [packing list](../demo-shoot-prep.md#4-packing-list)

---

## Order and timing

| When | What | Needs |
|---|---|---|
| Early | **Part 0** session start | — |
| Early | **Part A** bring-up + pipeline sanity | robot |
| **Most of the day** | **🔴 Part B — WHOLE-BODY** | robot **+ operator** |
| Short | **Part C** hands — sanity check only | robot + operator |
| After tuning | **Part D** config freeze | robot |
| **After 17:00** | **Part E** teardown + pack | — |

**The operator is the scarce resource** — use them for the feel judgments in B and C. Config edits,
sim jogging (B1) and the freeze work in D don't need them.

**⚠️ Anything changed in B or C must reach the Orin (D2) or it does not exist on the robot.**

---

## Part 0 — Session start (~10 min)

- [ ] `git pull` in `we-teleop`. ⚠️ **Last chance** — do not pull on-site (no uplink → a bumped
      `flake.lock` breaks `robo shell`, exactly as it broke the Orin on 07-29).
- [ ] Confirm the LOCAL uncommitted edits are still present: `wuji_finetune.json`,
      `config/teleop/pico_world_wuji_hands.yaml`, `driver_supervisor.py`.
- [ ] `unset __NV_PRIME_RENDER_OFFLOAD` in every operator shell (same interactive `robo shell`, not
      via `robo run`) — else the MuJoCo window is black.
- [ ] Both Yihe and the operator know the stop path (`Space` pause / `Esc`) before anything moves.

---

## Part A — Bring-up + pipeline sanity (~30 min)

- [ ] **Orin driver:** `robo shell -p tianji-driver` →
      `we driver --auto-reset-errors --camera-serial 419222302053 --camera-exposure 100`
      - ⚠️ **Init-gate may trip** → recover via 上位机 (CHEATSHEET §7). **Record joint + error +
        velocity** before touching the gate — the right-wrist error is on a growing trend
        (07-15 ~0.7° → 07-20 Arm_R6 2.01°, gate now 2.5°). Do not bump as a reflex.
      - ⚠️ Watch for Wuji **`Bus undervoltage` F4J3/F4J4** (07-28) — stop and check the hand power
        rail if it recurs; log which hand.
- [ ] Gloves reachable (a wuji profile fails without them). Connect-timeout with carrier up ⇒
      suspect cross-wiring (CHEATSHEET §3).
- [ ] D455 on **USB3 5000M**:
      `ssh nvidia@6.6.7.100 'for d in /sys/bus/usb/devices/*; do [ -f "$d/product" ] && echo "$(cat $d/product) | $(cat $d/speed) Mbps"; done'`
- [ ] Teleop up: `scripts/runtime/pico.sh pico_world_wuji_hands`, `Enter` to anchor.
- [ ] **End-to-end proof** — one short recorded episode, or a `we data replay` on the
      `_repaired`/`_merged` set (⚠️ raw datasets can't be replayed).
- [ ] **Worth doing while the robot is here: one autonomous run of the validated policy.**
      Autonomous shots are newly on the table; confirming the deploy stack runs is much cheaper now
      than discovering it on set.

---

## Part B — 🔴 WHOLE-BODY: stand up / squat down (the day's main event)

### B0. What the profile switch actually does (confirmed in code 2026-08-03)

- `profile.torso ∈ {hold, ik}` — validated at [launcher.py:129](../../../we-teleop/src/teleop/runtime/launcher.py#L129).
  **At the profile level it is genuinely all-or-nothing.**
- With `torso: ik`, [ik_process.py:59-63](../../../we-teleop/src/teleop/runtime/ik_process.py#L59-L63)
  adds **all 6 leg joints** to the IK's active set.
- **But `ik.frozen_joints` then subtracts from that set** —
  [solver.py:249](../../../we-teleop/src/teleop/control/kinematics/solver.py#L249):
  `optimized_joints = active_joints - frozen`. **So per-joint leg locking IS available**, just at
  the embodiment level rather than the profile level. This is exactly idea #2.
- Currently `head: hold`, `torso: hold` in `config/teleop/pico_world_wuji_hands.yaml:12-13`
  (our own LOCAL edit — this is what turned torso teleop off in the first place).

**The standing pose tells us how the squat is built** — `config/embodiments.yaml:46-47`:

```yaml
start_degrees:
  torso: [0.0, 60.0, -70.0, 10.0, 0.0, 0.0]   # J1  J2  J3  J4  J5  J6
```

**J2 (+60°), J3 (−70°), J4 (+10°) sum to 0** — that is the sagittal chain (hip / knee / ankle
pitch) arranged to keep the upper body vertical. **J1, J5, J6 sit at 0** → they are the
out-of-plane DOFs (yaw / roll), and they contribute nothing to a straight-ahead squat.

> ⚠️ This mapping is **inferred** from the start values and the sum-to-zero pattern, not from the
> URDF axes. Confirm it in sim (B1) before relying on it — it's 10 minutes and no robot needed.

### B1. Sim first — identify the joints (~15 min, no robot, no operator)

- [ ] Jog each leg joint **individually** in sim and write down what it does. Confirm or correct:
      J2/J3/J4 = sagittal squat chain, J1/J5/J6 = out-of-plane.
- [ ] Sanity-check the squat range in sim before asking the robot to do it.

### B2. Test on the robot (with the operator)

- [ ] Set `torso: ik` in `config/teleop/pico_world_wuji_hands.yaml`. **Sim → then robot.**
- [ ] Operator squats down and stands up. Judge honestly: **natural, or shaky / oscillating /
      lurching?** Hand weight loads the arms and changes leg dynamics, so judge it with the hands on.
- [ ] Record **the usable range** (how low, how fast) — it becomes a script constraint either way.
- [ ] Re-check the init-gate error afterwards; leg motion loads the arms differently.
- [ ] 🎥 **Video the squat**, good or bad. It is the artifact to send the retargeting/teleop owner,
      and the before/after reference once you start tuning.

### B3. If it's shaky — the lever ladder, softest first

Work **one lever at a time**, re-feel after each, and **log every value tried with its verdict**.
Under time pressure you must be able to get back to a known-good state.

| # | Lever | Where | Current | Effect |
|---|---|---|---|---|
| 1 | **Driver velocity / accel cap on the torso** | `config/runtime.yaml:45-46` | `max_velocity_rad_s: 1.0`, `max_acceleration_rad_s2: 10.0` | Hard cap on how fast legs may move. **Try this first** — pure config, no IK behavior change. ℹ️ The *reset* ramp already uses `0.2 / 0.4` (lines 58-59) and is visibly smooth, so there is a proven-calm setting to interpolate toward |
| 2 | **Per-joint smoothness weight** | `config/embodiments.yaml:210-211` `smoothness_joint_weights.torso` | `[3.0]×6` (arms are `1.0`) | Pulls each joint toward the *preceding solution* → graded damping. Raise the specific joint that shakes. Global multiplier `smoothness_weight: 0.0035` |
| 3 | **Per-joint rest weight** | `config/embodiments.yaml:202` `rest_joint_weights.torso` | `[12.0]×6` (arms `4.0/1.0`) | Pulls toward the start pose → resists drifting away from standing. Global multiplier `rest_weight: 0.01` |
| 4 | **🔒 Hard per-joint freeze — idea #2** | `config/embodiments.yaml:141` `ik.frozen_joints: []` | `[]` | Frozen joints are **excluded from IK and held at the solve seed**. **Recommended first freeze: the out-of-plane DOFs `J1, J5, J6`** — they sit at 0° for a sagittal squat, so freezing them is kinematically free and removes 3 redundant DOF the IK can wiggle. Only freeze J4 (ankle) if shake persists — that one costs real squat range |
| 5 | **1-euro joint smoothing** | `src/teleop/control/inputs/pico/smoothing.py:60` `DEFAULT_JOINT_SMOOTHING` | `min_cutoff_hz=2.0, beta=5.0` | Lower either = smoother, more lag. ⚠️ **Python, not yaml** → source edit → must re-sync the Orin. Affects the whole PICO joint path, arms included |
| 6 | **🅰️ Captured squat pose — idea #1** | `init_pose capture <name>` → `config/init_poses/<name>.yaml`, with `torso: hold` | — | Legs frozen at a captured squat for the entire take. **The guaranteed-stable path.** Costs live height change |

**Exact freeze syntax** (names must match exactly, including the space after `Gento`):
```yaml
ik:
  frozen_joints: [Gento Luna_Leg_J1_Joint, Gento Luna_Leg_J5_Joint, Gento Luna_Leg_J6_Joint]
```
Unknown names raise at load (`tianji.ik.frozen_joints contains unknown joints`), so a typo fails
fast rather than silently. Guard: **you cannot freeze every active joint** — for "all legs off" use
`torso: hold`, not a 6-joint freeze.

> ⚠️ **`ik.frozen_joints` and the weight arrays live in `config/embodiments.yaml` under `tianji.ik`
> — they are GLOBAL to the robot model, not per-profile.** Whatever you freeze applies to *every*
> profile on this embodiment, **including deploy/replay**. If you freeze leg joints for teleop, re-run
> the Part A autonomous check afterwards before assuming the policy path is unaffected.

### B4. Decision gate — before packing

- [ ] **Decide and write down what travels:** live torso IK (with which levers set), or a captured
      squat pose with `torso: hold`.
- [ ] **Capture the fallback squat pose regardless** (`init_pose capture <name>`) — insurance costs
      one minute now and is unobtainable on-site.
- [ ] Push the verdict into the [script risk register](../demo-shoot-prep.md#1-shoot-script-risk-register):
      "torso live, range X" vs "torso frozen at captured pose" changes what the script can ask for.
- [ ] ⚠️ **Still unanswered since 07-24: can the chassis be driven at the same time as upper-body
      teleop?** The 底盘 cannot be teleoped at all (fails closed in `gento.py`). If the script has the
      robot driving *while* the upper body moves, confirm the two can be commanded together — and if
      not, raise it today, not on set.

---

## Part C — Hands: sanity check only (~20 min, demoted 2026-08-03)

The script's actions are **not very dexterous**, so this is a "still works" check, not a tuning
session. Current LOCAL state: thumb `w=30` both hands; **index/middle pinch OFF (`d1=d2=0`)**;
thumb `segment_scaling` = base default.

- [ ] Hands respond and grasp in teleop. That's the bar.
- [ ] **One question still worth deciding:** do any scripted actions need a real **index/middle
      grasp**? If yes, restore `d2`≈3.0 and A/B it today — it is not fixable on-site. If no, keep
      `d1=d2=0` as-is and move on.
- [ ] **Thermal:** the hand overheats under sustained teleop → the shot schedule needs batched takes
      with cooldowns. Tell the crew.
- [ ] ⏭️ **Skipped deliberately:** the two-stable-hand-yamls A/B and the multi-step twist work. The
      multi-step twist is still blocked on retargeting smoothness (the owner's thread) and the script
      no longer depends on it.

---

## Part D — Config freeze (after all tuning, before teardown)

- [ ] **D1. Lock `enp13s0` static** — it is DHCP today, and `192.168.31.48` is hardcoded in
      `TELEOP_XROBOT_HOST_IP` and in the headset app's connect target. Same router on-site ⇒ same
      subnet, but the *lease* is not guaranteed.
      ```bash
      sudo nmcli con mod "有线连接 1" ipv4.method manual ipv4.addresses 192.168.31.48/24 \
           ipv4.gateway "" ipv4.dns ""
      ```
      Also clears the stale `192.168.123.123`. ⚠️ Ensure `.48` is outside the router's DHCP pool (or
      reserved). Verify with `ip addr show enp13s0` **and** a full teleop bring-up.
- [ ] **D2. Final Orin sync** — `scripts/setup/sync_luna.sh nvidia@6.6.7.100` (⚠️ always pass the
      address). This is what finally puts the **hands-open-on-shutdown fix** (`driver_supervisor.py`,
      LOCAL + uncommitted) on the Orin, **and** any Part B lever-5 smoothing edit. Restart the driver
      and confirm hands ramp **open** on driver exit.
- [ ] **D3. Commit the travelling state** — get the LOCAL + uncommitted work onto a branch so a
      mistake on-site is one `git switch` from recovery. This is the long-deferred **stable demo
      branch**, and it matters more now that the tree itself is what travels. ⚠️ Do not push the
      LOCAL commits (`1c7a1c3` init-gate loosen, `06757f6` machine config).
- [ ] **D4. Cold-boot smoke test** — reboot the workstation and bring the whole stack up from
      scratch, fixing nothing by hand. That is the on-site experience; if something only works
      because of state in your current shell, find out now.

---

## Part E — Teardown + pack (after 17:00)

- [ ] **📸 Photograph everything before unplugging anything** — both NIC ports, both glove dongles,
      the USB hub, the D455, the robot-side connections. The 07-27 走线 rework cross-wired the two
      glove ethernet cables and cost a full debugging session; photos make that a 30-second check.
- [ ] **Label the cables**: glove L / glove R, `enp13s0` (teleop LAN / router), `enp14s0` (robot).
- [ ] ⚠️ **The A→C converter orientation is MARKED — do not replug blind.**
- [ ] **Robot charged, charger packed** ([leave-office-sop](leave-office-sop.md)).
- [ ] **Bring the USB seed stick** — the workstation is now a single point of failure with no spare.
- [ ] Pack to the [packing list](../demo-shoot-prep.md#4-packing-list); tick items physically.
- [ ] ⚠️ **The NIC is an add-in card** — transport can unseat it. If both ethernet links misbehave on
      arrival, **reseat the card first**.
