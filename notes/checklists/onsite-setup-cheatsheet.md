# On-site setup cheatsheet — demo shoot 08/11–08/13

> Follow top to bottom on arrival. Part 1 is once per site. Part 2 is every session.
> ← back to [overview](../../README.md) · detail: [week-2026-08-03](../weekly-notes/week-2026-08-03.md)

## 🚫 Two rules that break everything

1. **NEVER `git pull` and NEVER touch `flake.lock` on-site.** No uplink → `robo shell` tries to fetch
   and fails. This is exactly what broke the Orin on 07-29. **Config/YAML edits are fine; dependency
   changes are not.**
2. **`unset __NV_PRIME_RENDER_OFFLOAD` in EVERY operator shell**, before teleop. Must be the same
   interactive `robo shell`, not via `robo run`. Otherwise the MuJoCo window is black.

---

## PART 1 — On arrival (once)

### 1.1 Physical
- [ ] Unpack; **cable up from the labels/photos** taken at teardown.
- [ ] ⚠️ **If BOTH ethernet links misbehave → reseat the NIC add-in card FIRST.** Both ports live on
      one card and transport shock unseats it. Don't debug cables or config before reseating.
- [ ] Robot charger plugged in and charging.
- [ ] Display + keyboard + mouse connected (MuJoCo needs a real display).
- [ ] Router powered on.

### 1.2 Network — check, don't assume
```bash
ip -4 addr show enp13s0     # expect 192.168.31.48/24   (teleop LAN -> Quest)
ip -4 addr show enp14s0     # expect 6.6.7.166/24       (robot, static)
ping -c2 192.168.31.1       # router
ping -c2 6.6.7.190          # robot x86
ping -c2 6.6.7.100          # Orin
```
**If `enp13s0` is NOT `192.168.31.48`** — it is DHCP, and `.48` is hardcoded in
`TELEOP_XROBOT_HOST_IP` *and* in the headset app. Symptom is "teleop doesn't track". Fix:
```bash
sudo nmcli con mod "有线连接 1" ipv4.method manual ipv4.addresses 192.168.31.48/24 \
     ipv4.gateway "" ipv4.dns ""
sudo nmcli con up "有线连接 1"
```
⚠️ `有线连接 1` = `enp13s0` (teleop). `有线连接 2` = `enp14s0` (robot) — **do not touch that one.**
⚠️ Reserve `.48` in the router or keep it out of the DHCP pool.

### 1.3 USB — verify, silent failures here
```bash
cat /sys/bus/usb/devices/*/speed        # D455 must be 5000, not 480
```
⚠️ USB3 drops to 480 on an under-seated A plug, and the **A→C converter passes SuperSpeed in ONE
orientation only** (it is marked). Prefer Type-C.

### 1.4 Gloves
```bash
we hands network      # glove dongle routes are SCRIPT-managed, not NetworkManager -> must run here
we hands check        # live skeleton -> 20-DoF retarget, reports rate
```
Gloves are `192.168.1.100:50001` (L) / `192.168.1.101:50001` (R).
⚠️ **Connect timeout while the carrier is UP = cross-wired glove cables** (cost a session on 07-27).
tcpdump the dongle; swap if the source IP is the other glove's.

### 1.5 Quest
- [ ] Headset on the **teleop LAN** (`192.168.31.x`) — not office wifi.
- [ ] Headset app connect target = `192.168.31.48`.

### 1.6 First bring-up
- [ ] ⚠️ **Expect the init gate to trip after transport.** Recover via 上位机. **Record joint + error +
      velocity BEFORE bumping the gate** — the right-wrist error trend is being tracked (was 2.5°).
- [ ] Decide the stance for the day → Part 3.

---

## PART 2 — Every teleop session

### 2.0 Sync the Orin FIRST — if any config changed since the last driver start
```bash
scripts/setup/sync_luna.sh nvidia@6.6.7.100
```
⚠️ **ALWAYS pass the address — the script's default is a DIFFERENT robot** (`nvidia@192.168.1.88`).
- Needed after changing anything the **driver** reads: `start_degrees` (stance), `gento.*` gains,
  `runtime.yaml` rate caps, `cameras.yaml`, driver source.
- **Not** needed for operator-only changes: IK weights, `head:`/`torso:` profile selectors,
  `wuji_finetune.json`, init-pose files.
- rsync + `--delete`, so the Orin mirrors the working tree. **No venv refresh** — restart the driver.
- It rebuilds the Tianji SDK on the Orin only if the SDK sources changed (prints which).

### 2.1 Orin — driver (no camera)
```bash
robo shell -p tianji-driver
we driver --auto-reset-errors --no-camera
```
- Defaults: robot `6.6.7.190`, `pd`, `--move-to-init`, bimanual hands.
- ⚠️ **The robot moves to the stance on driver start.** Clear the space, e-stop in hand.
- ⚠️ **No camera = you can teleop but CANNOT record.** `pico_world_wuji_hands` requires the `image`
  modality, so saving an episode fails without it. If you need to record:
  ```bash
  we driver --auto-reset-errors --camera-serial 419222302053 --camera-exposure 100
  ```
  The serial matters — without it RealSense grabs the robot's built-in head cam.

### 2.2 Workstation — teleop
```bash
robo shell -p operator                        # interactive shell
unset __NV_PRIME_RENDER_OFFLOAD
export TELEOP_XROBOT_HOST_IP=192.168.31.48
scripts/runtime/pico.sh pico_world_wuji_hands --driver-address 6.6.7.100
```
`Enter` to anchor.

| Profile | Hands | Use |
|---|---|---|
| `pico_world_wuji_hands` | wuji | normal — arms + hands + torso |
| `pico_world` | hold | torso/whole-body only; no gloves, no hand thermal |

`pico.sh` **starts the XRobot service itself** (it does **not** survive a reboot, so the first run
after power-on brings it up; a bare `we teleop` will not).

### 2.3 In-session commands
| Command | Effect |
|---|---|
| `Enter` / `Space` | start-resume / pause |
| `Tab` | ramp to the active init pose |
| `init_pose list` | list stored poses |
| `init_pose <name>` | activate a pose (read from disk each time — no restart) |
| `init_pose capture <name>` | capture the current pose |
| `init_pose default` | back to `start_degrees` |

---

## PART 3 — Stance switching (low / neutral / high)

**`start_degrees.torso` is BOTH the driver's move-to-init target AND the IK rest reference.** An
`init_pose` alone does *not* change the stance teleop holds.

```bash
scripts/setup/set_stance.py show              # what is active now
scripts/setup/set_stance.py low               # or: neutral | high
scripts/setup/sync_luna.sh nvidia@6.6.7.100   # REQUIRED — the driver reads it too
# then restart the driver (Orin) AND teleop (workstation)
```

| Stance | Shoulder | vs neutral | Fwd | Use |
|---|---|---|---|---|
| `low` | 0.551 m | −16.0 cm | +8.9 cm | ground / low reach. Only 8° J2 headroom left. |
| `neutral` | 0.710 m | — | — | **ORIGINAL — required for autonomous policy shots.** |
| `high` | 0.833 m | +12.3 cm | +1.5 cm | high reach. Stays over the base. |

⚠️ **Per-scene, not per-take** (edit + sync + two restarts). ⚠️ **Autonomous shots must use `neutral`** —
the validated policy was trained there, and deploy/replay key off the same value.

### Other tuning levers (all config, all need a teleop restart)
| Want | Change | Sync Orin? |
|---|---|---|
| Robot head rotates with operator's head | `head: hold` → `pico` in the teleop profile; operator wears the headset **on the head** | no |
| Softer arms | `gento.arm_pd_kp` → `[6,6,6,6,1.5,1.5,1.5]` (leave `arm_pd_kd`) | **yes** + driver restart |
| Torso squats more / less | `ik.rest_joint_weights.torso` — keeper is `[12,5,5,8,12,6]`; `3/3/4` was too shaky | no |

⚠️ `head: pico` feeds headset tracking noise into the torso → shaky. Head-motion shots only.
⚠️ Softer `kp` under-sizes `operating_limit_margin_degrees` (measured at `kp: 9`) — don't drive
softened joints to their operating limits.

---

## PART 4 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Quest can't find the workstation | XRobot service not running | `pico.sh` starts it. Check: `pgrep -af RoboticsServic` — **the name truncates at 15 chars**, so `pgrep -x RoboticsServiceProcess` gives a false negative |
| Was connected, dropped, app reconnect fails, service looks fine | **Quest dropped wifi**; service still holds a dead ESTAB session | **Restart wifi on the Quest**, not the service. Tell: headset IP no longer answers `ping` |
| "Teleop doesn't track" | `enp13s0` IP changed | §1.2 |
| Glove connect timeout, carrier UP | Cross-wired glove cables | tcpdump the dongle, swap |
| Black MuJoCo window | `__NV_PRIME_RENDER_OFFLOAD` set | `unset` it in the same interactive `robo shell` |
| Both ethernet links flaky | NIC add-in card unseated | **Reseat the card first** |
| Camera at 480 Mb/s / no 60 fps | Under-seated USB-A, or A→C converter flipped | §1.3 |
| Init gate trips on bring-up | Transport | Try `we driver reset-errors` first; else 上位机 → **Part 5**. **Record joint + error + velocity BEFORE bumping the gate** |
| Arm stops dead then re-ramps ~1 s | **IK residual gate** — target out of reach, not a fault | Move back into reach. >50 mm residual rejects; >100 mm with a joint on its limit hard-fails to HOLD |
| Torso/arm shaky | Too-low torso rest weights, or `head: pico` | Revert to `[12,5,5,8,12,6]` and `head: hold` |
| Wuji `Bus undervoltage` F4J3/F4J4 | Hand power rail | Stop, check the rail/connectors |
| Hand sluggish / hot | **Thermal** — overheats under sustained teleop | Batch takes with cooldowns |
| `robo shell` tries to fetch and fails | Someone pulled or touched `flake.lock` | Don't. Restore the lock file |

⚠️ **There is NO collision checking anywhere in the code.** Nothing stops the hand hitting the
chassis — matters now that the torso moves. Joint limits and the residual gate are *not* collision
protection.

⚠️ **Index/middle pinch is fully OFF** (`d1=d2=0`, both hands). A real index/middle grasp needs
`d2`≈3.0 in `wuji_finetune.json` + a teleop restart. **Thumb abduction weight is off** (`w: 0.0`) —
keep it off for the demo. **Left glove tactile sensor is faulty** (hardware).

---

## PART 5 — 上位机 (GentoPlatform): manual reset / init-gate recovery

**This is the init-gate recovery tool.** Run on the **workstation** at a physical display (tkinter
GUI; needs the `6.6.7.x` link up). Full SOP + all reference poses:
[shangweiji-sop.md](../implementation/shangweiji-sop.md) · CHEATSHEET §7.

```bash
cd ~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM
python3 UI.py
```

⚠️ **Stop `we driver` first — one commander at a time.**
ℹ️ **Try the cheap fix first:** simple error clears no longer need this — `we driver reset-errors`.

**Flow:** *Connect Robot* (turns green) → per component set speed to **5%** + *Confirm Speed* →
paste angles into **Position Cmd** → *Add* → *Position* (Status switching) → *Run*. Repeat per
component.

| Component | Is |
|---|---|
| ARM0 / ARM1 | left arm / right arm |
| BODY | torso (the 6 leg joints) |
| HEAD | head |

**遥操姿态 — upright, use this to recover after a gate trip:**

| Component | Angles (paste comma-separated) |
|---|---|
| ARM0 (left) | `90, -90, -90, -90, 0, 0, 0` |
| ARM1 (right) | `-90, -90, 90, -90, 0, 0, 0` |
| BODY | `0, 0, 0, 0, 0, 0` |
| HEAD | `0, 0, 0` |

**打包姿态 — packing pose (✅ verified 2026-08-04; speed 10%). Needed to RE-PACK after the shoot:**

| Component | Angles |
|---|---|
| 左臂 ARM0 (left) | `120, -90, -90, -90, 0, 0, 0` |
| 右臂 ARM1 (right) | `-120, -90, 90, -90, 0, 0, 0` |
| 头 HEAD | `0, 0` |
| 腰 BODY | `0, 87, -140, 52.3, 0, 0` |

**⚠️ Order matters — and step 3 needs a hand on the robot:**
1. Arms + head to the pose (10% speed, position mode, paste → Add → Run).
2. **Arms 下使能 → click `idle`**, then fold the dexterous hands up toward the body by hand.
3. **Support the hands** while the body (腰) goes to the packing pose.

ℹ️ Body pitch `87 − 140 + 52.3 = −0.7` → torso stays vertical, same family as the teleop stances.
⚠️ `J3 = −140` is just outside our `−139.5°` operating limit → **上位机 only**, not commandable from
teleop or an `init_pose`. Full procedure: [shangweiji-sop.md](../implementation/shangweiji-sop.md).

---

## Reference

| Thing | Value |
|---|---|
| Workstation teleop LAN | `192.168.31.48` (`enp13s0`, DHCP, `有线连接 1`) |
| Workstation ↔ robot | `6.6.7.166` (`enp14s0`, static, `有线连接 2`) |
| Robot x86 / Orin / chassis | `6.6.7.190` / `6.6.7.100` / `6.6.7.6` |
| Gloves L / R | `192.168.1.100:50001` / `192.168.1.101:50001` |
| Router | `192.168.31.1` |
| D455 serial / exposure | `419222302053` / `100` |
| SSH | `nvidia@6.6.7.100`, `root@6.6.7.190`, `sunrise@6.6.7.6` |
| Sync to Orin | `scripts/setup/sync_luna.sh nvidia@6.6.7.100` — ⚠️ **always pass the address**, default is another robot |
| 上位机 | `~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM` → `python3 UI.py` (Part 5) |
| Stance switch | `scripts/setup/set_stance.py {show\|low\|neutral\|high}` (Part 3) |
| Disaster recovery | `/home/yihetang/project26/onsite-seed` + USB stick (5.8 GB) — [runbook](2026-07-29-onsite-provisioning-runbook.md) |
