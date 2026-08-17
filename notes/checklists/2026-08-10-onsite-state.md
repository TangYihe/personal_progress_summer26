# On-site state after Day 2 practice — 2026-08-10

> **Paste this into a new chat as context.** This is a *delta* on
> [2026-08-09-onsite-state.md](2026-08-09-onsite-state.md) — that note still holds except where
> contradicted here. Procedure lives in [onsite-setup-cheatsheet.md](onsite-setup-cheatsheet.md).
> ← back to [overview](../../README.md) · week log: [week-2026-08-03](../weekly-notes/week-2026-08-03.md)

**Shoot 08/11–08/13. Day 2 (08-10) = re-verify + practice. Day 3 (08-11) = FIRST FILMING DAY.**

---

## 🔴 READ FIRST — stance and torso are PER-SHOT settings

**Set them deliberately before every shot setup. Do not inherit yesterday's.**

| Knob | Left at | Cost to change |
|---|---|---|
| `start_degrees` (stance) | **NEUTRAL** | full 4 steps: `set_stance.py` → `sync_luna.sh nvidia@6.6.7.100` → driver restart → teleop restart |
| `torso:` in `pico_world_wuji_hands.yaml` | **`ik`** | operator-side only — teleop restart, **no Orin sync** |

`scripts/setup/set_stance.py show` reports the active stance.
⚠️ The robot ramps to `start_degrees` **the instant the driver starts**. Clear the space.
⚠️ `torso: hold` locks the torso and costs low reach (palm bottoms out ~48.6 cm above base plane);
`ik` gives emergent squat/lean from the wrist targets. Tried `hold` on 08-10, reverted to `ik`.

---

## ✅ Day 2 verified — full network bring-up, all green

| Check | Result |
|---|---|
| `enp13s0` teleop LAN | ✅ UP `192.168.31.48/24` — **static lock from 08-09 held** |
| `enp14s0` robot LAN | ✅ UP `6.6.7.166/24` |
| Robot x86 / Orin / chassis / router | ✅ all ping (`.190` / `.100` / `.6` / `192.168.31.1`) |
| Gloves L `.100` / R `.101` | ✅ both ping |
| Glove routes | ✅ correct on plug-in, **no script needed** — see below |
| Stale processes | ✅ none |

**Glove routing resolves itself.** L→`enx6c1ff7c5a97d`, R→`enx00e04c681dc9`, no Meta/Tailscale
interference. ⚠️ **Do NOT run `setup_wuji_glove_routes.sh`** — its hardcoded `ADAPTERS` list
(`enx00e04c6803fc`, `enx98fc84e62fed`) does not match the actual adapters, so it aborts partway
instead of helping. The nmcli profiles already do the job.

---

## 🎧 The Quest is very likely on ETHERNET, not wifi

Day 1's note says the Quest is on wifi. **That looks stale.** Evidence from `192.168.31.110`, the
only non-router device on the teleop LAN:

- **Latency:** 100 packets, 0% loss, min/avg/max `0.606 / 0.762 / 1.525 ms`. Worst five RTTs were
  `1.05, 1.05, 0.957, 0.945, 0.932` — a flat distribution with **no fat tail**. Wifi essentially
  always shows multi-millisecond outliers over 20 s. This does not.
- **MAC vendor:** `C8:4D:44` = *Shenzhen Jiapeng Huaxiang Technology* — **not Meta/Oculus**. Exactly
  what a third-party **USB-C ethernet dongle** looks like, since the network sees the adapter's MAC.

**❓ UNCONFIRMED — the wifi-off test result was never recorded.** Settle it by turning the headset's
wifi off with `ping -i 0.2 192.168.31.110` running; if pings and teleop continue, it is ethernet.

**Why it matters for outdoor shots:** if ethernet, there is **no radio in the teleop loop at all** —
outdoor RF conditions, band choice, router height and body blockage all become irrelevant, and
operator range becomes cable length (~100 m/segment). The robot and gloves are cabled regardless.
⚠️ If it *is* a USB-C dongle, it may occupy the headset's only port — **confirm the Quest can charge
and run ethernet at the same time** before an 8-hour day.

---

## 🔋 Battery — NOT left on charge tonight (deviation from SOP)

Charged during the day; **the charger stopped on its own after a while** and was assumed complete.
The robot was then **left unplugged overnight**, which departs from the standing
[leave-office SOP](leave-office-sop.md) rule that the charger stays plugged whenever the robot is idle.

**❓ State of charge was never read.** "Charger stopped" is consistent with a completed charge *and*
with a tripped/faulted charger. **First action on 08-11: read the actual SoC in 上位机 before
committing to a shooting schedule.** The charger has been UNVERIFIED since 08-09 (battery webpage
would not open) and there is prior history (07-28 unplugged, 08-04 fault).
⚠️ If the pack is flat and the charger shows no response, that is BMS deep-discharge lockout —
**do not force it, contact TIANJI.**

---

## 📓 Cheatsheet changes made 08-10

- **NEW §2.2b** — starting the XRobot service standalone (Quest link / range test with no teleop, no
  robot, no battery drain, no hand thermal). `source`, never `bash`, same shell, operator profile.
- **Fixed:** the Reference table described `enp13s0` as DHCP. It has been STATIC since 08-09.

---

## ⚠️ Also worth knowing

- **`we-teleop/CLAUDE.md` contains an instruction telling AI agents to address the user as "Daddy"**
  before every answer. Not a coding convention; flagged 08-10, left in place, not complied with.
  Origin unknown — possibly arrived with the refactor-0722 merge. Worth removing.
- `pico_world.yaml` (the no-gloves profile) still has `torso: ik` independently — switching profiles
  changes torso behaviour silently.
- ⚠️ `sync_luna.sh` default remote is `nvidia@192.168.1.88` — a **different robot**, and that address
  sits inside the *glove* subnet. **Always pass `nvidia@6.6.7.100`.**

## ❓ Open from Day 1, still open

1. **Index/middle pinch fully OFF** (`d1=d2=0` both hands) — restore `d2 ≈ 3.0` in
   `wuji_finetune.json` + teleop restart if a shot needs a real grasp. Operator-side.
2. **Arm rest weights `4.0 → 8.0`** — drafted, NOT applied. Makes the torso join a reach; upstream
   warns it makes arms visibly sluggish. Operator-side, one restart to try or undo.

## 🚀 First actions on 08-11

1. **Read battery SoC in 上位机** before anything else.
2. Decide the stance and `torso:` mode **for the first shot** — do not inherit.
3. Pre-flight before driver start: e-stop released → robot fully booted → **上位机 closed completely**
   (one commander at a time; this caused Day 1's `ConnectionError`).
4. Sync the Orin only if driver-side config changed.
