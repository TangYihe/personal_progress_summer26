# Leave-office SOP (end of every session)

> Standing checklist — run before walking out. Goal: nothing left powered/uncharged.
> Born from the 2026-07-28 incident: TIANJI's 走线 rework left the **robot charger unplugged**,
> nobody said anything, and the battery drained mid-session. ⚠️ A full drain may lock the BMS —
> see §3.4. Order matters: **software down → arms safe → gloves → PICO → robot off + CHARGING**.

---

## 0. Software down (workstation + Orin)
- [ ] Teleop: `Esc` to quit the operator/teleop window first (before the driver).
- [ ] Driver (Orin): stop `we driver`. ⚠️ **Hands OPEN on driver stop** — catch anything held.
- [ ] ⚠️ **Arms have NO brake** — on driver stop the arms go IDLE and **droop**. Move to a safe
      resting/packing pose BEFORE stopping the driver (§1), or support them by hand as it stops.

## 1. Arms → safe/packing pose (do this while the driver is still up)
- [ ] Bring the robot to the **打包 / 原点** pose via 上位机 (GentoPlatform) so nothing droops onto
      the table or the hands. SOP + reference poses: [notes/implementation/shangweiji-sop.md](../implementation/shangweiji-sop.md).
      ⚠️ Stop `we driver` first — one commander at a time.
- [ ] Confirm arms are settled and not resting weight on the Wuji hands.

## 2. PICO / Quest headset
- [ ] Exit the teleop app on the headset.
- [ ] **Power OFF** the headset (hold power button until it shuts down — don't leave it sleeping;
      sleep ≠ off, and the PICO sleep bug drains it).
- [ ] Park it on the rack.
- [ ] Plug the headset in to charge (it's the daily-use device).

## 3. Wuji gloves — POWER DOWN
- [ ] **Power off / unplug both gloves** (each glove's power lead). "手套是否断电" = confirm the
      glove LEDs are off, not just the software readers stopped.
- [ ] ⚠️ **A→C converter orientation is MARKED** — do NOT replug blind next time (wrong orientation =
      silent USB2, D455/glove drops). Note which cable came off which dongle (cross-wire risk, §3 CHEATSHEET).
- [ ] Left glove tactile sensor is a known-faulty unit — no special handling, just don't lose track of it.

## 4. Robot — power off + ON CHARGE  ← the one that bit us
- [ ] Shut down the onboard computers cleanly first (so nothing corrupts on hard power cut):
  - Orin `nvidia@6.6.7.100`, x86 `root@6.6.7.190`, chassis `sunrise@6.6.7.6`.
  - `ssh <host> 'sudo shutdown -h now'` for each, or per TIANJI's documented sequence.
- [ ] Power off the robot main switch/e-stop per TIANJI's shutdown procedure.
      ⚠️ TODO: fill in the exact button/switch sequence with TIANJI — don't guess it.
- [ ] 🔴 **PLUG IN THE CHARGER and CONFIRM it's actually charging** — look for the charging
      indicator (LED / 上位机 battery status). **This is the whole point of this SOP.**
- [ ] 🔴 **Deep-discharge caveat:** if the battery fully drained, the charger may show NO response
      (BMS deep-discharge lockout). **Do NOT keep forcing it** — leave it plugged and **contact TIANJI**
      for the recovery/wake procedure. We do not have a confirmed self-recovery step for a dead-flat pack.

## 5. Final glance before the door
- [ ] Robot charger: **plugged, indicator lit.** (Say it out loud.)
- [ ] Headset: off + charging.
- [ ] Gloves: powered off.
- [ ] Workstation: safe to leave (long training in tmux is fine; note it on the whiteboard/HANDOFF).

---

## Standing reminders
- ⚠️ **Charger stays plugged whenever the robot is idle** — not just overnight. After ANY hardware/走线
  work by TIANJI, re-verify the charger is connected and charging BEFORE starting a session (they've
  pulled it once without telling us — 07-28).
- ⚠️ If TIANJI touches cabling, also re-check: glove cables not cross-wired (§3 CHEATSHEET),
  D455 on USB3 5000M (§1 CHEATSHEET), init-gate on driver startup.
