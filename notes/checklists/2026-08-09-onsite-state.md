# On-site state after Day 1 setup — 2026-08-09

> **Purpose: paste this into a new chat as context.** Current state only, plus what Day 1 learned.
> Procedure lives in [onsite-setup-cheatsheet.md](onsite-setup-cheatsheet.md) — this does not replace it.
> ← back to [overview](../../README.md) · week log: [week-2026-08-03](../weekly-notes/week-2026-08-03.md)

**Shoot 08/11–08/13. Day 1 (08-09) = arrival + full bring-up + shot design. Day 2 = practice.**
Teleop-with-hands only. **No camera, no recording, no autonomous policy.**

---

## 🚫 Two rules that still break everything

1. **NEVER `git pull` and NEVER touch `flake.lock` on-site.** Set wifi (`景一片场`) exists, so there
   *is* internet — the rule stands anyway. Config/YAML edits fine; dependency changes are not.
2. **`unset __NV_PRIME_RENDER_OFFLOAD` in EVERY operator shell**, in the same interactive
   `robo shell` (not `robo run`), or the MuJoCo window is black.

---

## ✅ What Day 1 verified — all transport risks cleared

| Check | Result |
|---|---|
| NIC add-in card (both ports) | ✅ both links UP — **no reseat needed** |
| `enp13s0` teleop LAN | ✅ `192.168.31.48/24` — **now locked STATIC** (was DHCP) |
| `enp14s0` robot LAN | ✅ `6.6.7.166/24` static |
| Robot x86 / Orin / chassis / router | ✅ all ping (`6.6.7.190` / `.100` / `.6` / `192.168.31.1`) |
| Gloves L `.100` / R `.101` | ✅ both ping, host routes present — **not cross-wired** |
| Orin ↔ workstation config | ✅ in sync on arrival (rsync dry-run delta empty) |
| Driver bring-up | ✅ ran, reached init pose |
| Teleop + Quest + hands | ✅ ran, operator confirmed working |
| Sim test of the stand-up ramp | ✅ looked correct |

**🔴 The one pre-pack TODO that was never done, now DONE:** `enp13s0` was still DHCP and held `.48`
only by lease. Locked static on 08-09:
```bash
sudo nmcli con mod "有线连接 1" ipv4.method manual ipv4.addresses 192.168.31.48/24 \
     ipv4.gateway "" ipv4.dns ""
sudo nmcli con up "有线连接 1"
```
⚠️ `有线连接 1` = `enp13s0` (teleop). `有线连接 2` = `enp14s0` (robot) — **never touch that one.**

## 📌 Three corrections to the 08-04 handoff

1. **The Orin repo is at `/home/nvidia/we-teleop`**, NOT `~/project26/we-teleop`. `sync_luna.sh`
   targets `~/we-teleop` — it was right; the handoff prose was wrong.
2. **The Orin WAS synced.** The 08-04 note "the Orin was never synced, disagrees on stance and
   `arm_pd_kp`" is **wrong** — arrival dry-run delta was empty, and it carried the 08-04 stance block
   and `arm_pd_kp: 9.0`.
3. **The stance that travelled was NEUTRAL, not LOW.** The 08-04 "LOW active" note is stale.

---

## 🔧 Current config (exact, as of end of Day 1)

**⚠️ ACTIVE STANCE IS `LOW`** — left there after Day 1 stance testing. **The robot ramps to whatever
is in `start_degrees` the moment the driver starts.** Set it deliberately before bring-up.

| Setting | Value |
|---|---|
| `start_degrees.torso` | **LOW** `[0.0, 81.5, -118.0, 36.5, 0.0, 0.0]` |
| `ik.rest_joint_weights.torso` | `[12, 5, 5, 8, 12, 6]` ← 08-04 KEEPER, unchanged |
| `ik.rest_joint_weights.left/right_arm` | `[4, 4, 4, 4, 1, 1, 1]` ← **unchanged, see open decision** |
| `ik.smoothness_joint_weights.torso` | `[6, 6, 6, 6, 6, 6]` |
| `ik.frozen_joints` | `[]` — nothing frozen, correct |
| `gento.arm_pd_kp` | `[9, 9, 9, 9, 1.5, 1.5, 1.5]` — default |
| `pico_world_wuji_hands` | `arms: pico_world`, `head: hold`, `torso: ik`, `hands: wuji`, `driver_address: 6.6.7.100` |
| `pico_world` (no gloves) | `head: hold`, `torso: ik`, `hands: hold` |
| pinch `index`/`middle` d1,d2 | **`0.0` / `0.0` both hands — pinch fully OFF** |
| `thumb_abd_track.w` | `0.0` both hands — keep off for the demo |

**Init poses** (`config/init_poses/`): `stance_high`, `stance_low`, **`stance_neutral` (NEW 08-09)**,
`test`. All three stance poses carry **identical arm values**, so `Tab` between them moves the torso
only — the arms do not move at all. That property is what the stand-up shot depends on.

**Dirty working tree** (unchanged policy — local, do not push): `embodiments.yaml`,
`wuji_finetune.json`, both teleop profiles, `driver_supervisor.py`, plus untracked
`init_poses/stance_*.yaml` and `scripts/setup/set_stance.py`.

---

## 🎬 The stand-up shot — designed and sim-verified on Day 1

**Goal:** robot stands up with arms completely still, then reaches high with arms + some torso.

**The key insight: make `start_degrees` the END state of the shot, not the start.** `start_degrees`
is simultaneously the driver's move-to-init target *and* the IK rest reference, so setting it to
**HIGH** means the rest term pulls toward exactly where the shot finishes. Nothing fights.

**Setup:** `set_stance.py high` → `sync_luna.sh nvidia@6.6.7.100` → restart driver → restart teleop.

**Then in the teleop window:**
| Step | Keys | Effect |
|---|---|---|
| 1 | `Space` | **pause — arms lock**, track nothing. Critical: live teleop makes arms follow the operator |
| 2 | `/` then `init_pose stance_neutral`, Enter | select the crouched pose |
| 3 | `Tab` | robot crouches, ~6 s, **arms rigid** |
| 4 | — roll camera — | |
| 5 | `/` then `init_pose stance_high`, Enter | select the tall pose |
| 6 | `Tab` | **stands up**, ~6 s, arms still rigid |
| 7 | `Enter` | re-anchors at HIGH; arms take over; torso **stays up** (rest ref = HIGH) |

- `/` opens the command entry (`:` also works). `init_pose list` shows stored names. Poses are
  re-read from disk on every command — **no restart needed to edit or add one.**
- Ramp rate is `0.2` rad/s / `0.4` rad/s² (`runtime.yaml:58-59`) — visibly smooth, and a real
  cinematic knob. ⚠️ Driver-side → changing it costs an Orin sync + driver restart.
- Rise sizes: NEUTRAL→HIGH = **+12.3 cm** (~6 s) · LOW→HIGH = **+28 cm** (~10 s).

**🔑 Re-anchor semantics on `Enter` (verified in code):** the robot's **current** wrist poses become
the anchor and the operator's **current** hand pose becomes the origin; motion is relative from
there. **The robot does NOT jump to the operator's hands** on resume, and re-engagement eases in
over `action_engagement_ms: 1000`.

**⚠️ The stance gotcha this design defeats:** re-anchoring changes the *targets*, NOT the IK *rest
reference*. With `start_degrees` at LOW/NEUTRAL, pressing `Enter` after ramping to HIGH makes the
torso visibly sink back down while the arms stretch. Only a matching `start_degrees` holds it.

---

## Standing procedure

**Stance switch — all four steps, none optional:**
```bash
python3 scripts/setup/set_stance.py high     # or: low | neutral | show
scripts/setup/sync_luna.sh nvidia@6.6.7.100  # ⚠️ ALWAYS pass the address (default = another robot)
# restart the DRIVER on the Orin   -> move-to-init + convergence gate
# restart TELEOP on the workstation -> IK rest reference
```
Do the first two with driver + teleop stopped. Skipping either restart makes the two hosts disagree
— the torso gets dragged one way while the arms compensate, which reads as a tracking bug.

**Driver (Orin, `robo shell -p tianji-driver` in `~/we-teleop`):**
```bash
we driver --auto-reset-errors --no-camera
```
**Teleop (workstation, separate terminal):**
```bash
cd ~/project26/we-teleop && robo shell -p operator
unset __NV_PRIME_RENDER_OFFLOAD
export TELEOP_XROBOT_HOST_IP=192.168.31.48
scripts/runtime/pico.sh pico_world_wuji_hands      # pico_world = no gloves, no hand thermal
```
`pico.sh` starts the XRobot service itself; it does **not** survive a reboot, and a bare `we teleop`
will not bring it up.

**Safe read-only probe before commanding motion:** `we driver status` — links, prints each section's
state + error code, disconnects. No motion, nothing armed.

---

## 🐛 Day 1 incidents and their causes

**`ConnectionError: Failed to init.` on first driver start.** The Gento SDK's `robot.link()` to
`6.6.7.190` was refused — ping worked (x86 OS up) but the controller wouldn't accept a link.
**Cause: the e-stop was pressed during transport.** Fixed by releasing it + a restart with the TIANJI
engineer. **Check order for a repeat:** e-stop released? → robot fully booted (ping answers long
before the controller is ready)? → is 上位机 open and holding the link? **One commander at a time —
close 上位机 completely before starting the driver.**

**`robo shell` on the Orin takes ~2 min** with `Could not resolve host` warnings. **Not broken —**
the Orin has `default via 6.6.7.1 dev eth10`, a default route to a network with no internet, so
nix's substituter requests **hang until timeout** instead of failing fast. It completes and works.
**Do not "fix" this by pulling.** Optional real fix: a `substituters =` line in
`/etc/nix/nix.custom.conf` on the Orin (the system config reserves that file for user overrides).
**NOT applied.**

---

## ❓ Open decisions for Day 2

1. **🖐️ Index/middle pinch is fully OFF** (`d1=d2=0` both hands). If any scripted action needs a real
   index/middle grasp, restore `d2 ≈ 3.0` in `config/inputs/retargeting/wuji_finetune.json` +
   teleop restart (**operator-side, no Orin sync**). **Decide before a shoot day, not during one.**
2. **💪 Arm rest weights `4.0 → 8.0`?** Drafted but **NOT applied.** At HIGH the torso is already at
   its rest pose, so during a reach the solver strongly prefers the arms and the torso barely joins
   in. Raising arm proximal weights is the lever that makes the torso participate. ⚠️ Upstream's own
   comment warns stronger proximal weights make the arms **visibly sluggish**. Operator-side, one
   teleop restart to try or undo. Note the comment above those lines claims "J1–J3 are held at 2.0" —
   **stale, the real values are 4.0.**
3. **🔌 Charger works but is UNVERIFIED** — the battery webpage wouldn't open. Treat battery as
   unknown: watch powered-on time, don't idle the robot live between setups. ⚠️ If it goes flat
   unexpectedly, that is the 08-04 charger fault recurring, and it is probably not the charger.

## ⚠️ Constraints to keep in mind

- **No collision checking anywhere in the code.** Nothing stops the hand hitting the chassis. Joint
  limits and the residual gate are *not* collision protection.
- **Arm stops dead then re-ramps ~1 s** = the IK residual gate (>50 mm rejects; >100 mm with a joint
  on its limit hard-fails to HOLD). It means the arm can't follow you — **not** a fault.
- **Torso and arms move SIMULTANEOUSLY on driver start**, each rate-limited by its own envelope
  (torso 0.2 rad/s, arms/head 0.35). There is no body-first staging on that path.
- **At LOW there is only 8° of J2 headroom** (81.5° vs the 89.5° limit) — that is all the further
  squat the IK has from that stance, and a pinned joint feeds the saturation/HOLD path.
- **`head: pico`** rotates the robot's head with the operator's (headset worn **on the head**), but
  feeds tracking noise into the torso → shaky. Head-motion shots only. Currently `hold`.
- **Wuji hand thermal** — overheats under sustained teleop. Batch takes with cooldowns.
- **Left glove tactile sensor is faulty** (hardware, vendor silent since 07-08).
- **Quest drops wifi** and it looks like a server fault: app reconnect fails while the service looks
  healthy. **Restart wifi on the Quest, not the service.** Tell: headset IP stops answering ping.
- Glove latency is **left ~2 ms / right ~0.15 ms**, consistently. Normal for this rig, not a fault.

## Reference

| Thing | Value |
|---|---|
| Workstation teleop LAN | `192.168.31.48` (`enp13s0`, **STATIC**, `有线连接 1`) |
| Workstation ↔ robot | `6.6.7.166` (`enp14s0`, static, `有线连接 2`) |
| Robot x86 / Orin / chassis | `6.6.7.190` / `6.6.7.100` / `6.6.7.6` |
| Gloves L / R | `192.168.1.100:50001` / `192.168.1.101:50001` |
| Router | `192.168.31.1` |
| Repo — workstation / Orin | `~/project26/we-teleop` / **`/home/nvidia/we-teleop`** |
| SSH | `nvidia@6.6.7.100`, `root@6.6.7.190` (password), `sunrise@6.6.7.6` |
| 上位机 | `~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM` → `python3 UI.py` |
| Stance geometry | LOW 0.551 m · NEUTRAL 0.710 m · HIGH 0.833 m (shoulder height) |
