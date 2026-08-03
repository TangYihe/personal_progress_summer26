# Demo shoot prep — 08/11–08/13

> Living doc for the filming shoot. Three jobs: (1) a **risk pass over the shoot script**,
> (2) **props + site requirements** to send to the partner side, (3) the **packing list** — which
> became the critical prep item once we decided (2026-08-03) to bring the whole workstation.
>
> ← [week-2026-08-03](weekly-notes/week-2026-08-03.md) ·
> [pre-pack verification](checklists/2026-08-04-pre-pack-verification.md)
>
> Status: **§1 risk pass DONE and §2 prop guide SENT to the filming team, 2026-08-03.**
>
> **🔑 Headline finding from the pass: the scripted actions are not very dexterous, but there is a
> lot of whole-body movement.** That inverts the hardware priorities — **stand up / squat down is
> the top risk**, and the hand/retargeting work drops to a sanity check. It also means the
> multi-step twist blocker no longer sits on the critical path for the shoot.

---

## 1. Shoot-script risk register

**How to use this.** Go through the script substep by substep. For each substep, ask *what
capability does this need*, find it in the capability table below, and copy its risk into the
per-step table. A substep that maps to a 🔴 needs either a mitigation or a fallback shot **decided
before we travel** — on-set is the wrong place to discover that a shot is impossible.

Severity: 🔴 can stop the shot · 🟡 costs time / needs retakes · 🟢 minor, worth knowing.

### 1a. Known capability risks (pre-seeded from our own history)

| Capability the script might ask for | Status | Sev | Mitigation / fallback |
|---|---|---|---|
| **Multi-step / long-horizon cube twist** | **Blocked** — teleop-limited by hand-retargeting smoothness (07-30); the retargeting owner's thread | 🔴 | **Do not script it** unless the owner has shipped a fix and we verified it 08-04. Fallback: the *single* horizontal twist, which is validated |
| **Single horizontal cube twist, autonomous** | **Validated 16/16** (07-24) | 🟢 | Our strongest shot. Now available on-site because the full deploy stack travels |
| **Fine pinch / picking up thin objects** | Unreliable — index/middle pinch is currently **OFF** (`d1=d2=0`); thumb IP-vs-MCP bending unresolved | 🟡 *(downgraded 08-03 — the script isn't dexterous)* | Props chosen for a power grasp, not a pinch. One open decision on 08-04: if *any* action needs a real index/middle grasp, restore `d2`≈3.0 |
| **🔴 Stand up / squat down** | **The dominant ask of the script.** Untested since the merge; `torso: hold` today (our own edit) | 🔴 | **Top priority 08-04** ([Part B](checklists/2026-08-04-pre-pack-verification.md#part-b--🔴-whole-body-stand-up--squat-down-the-days-main-event)). If not natural: lever ladder → per-joint leg freeze (`J1/J5/J6` are free to lock) → fallback to a captured squatted init pose with `torso: hold` |
| **Chassis (底盘) driving** | **Cannot be teleoped** — fails closed in `gento.py`, needs a separate controller | 🔴 | Choreograph the base separately from upper-body teleop. ⚠️ **Confirm 08-04 that base + upper body can even move simultaneously** — open since 07-24 |
| **Live height change during a take** | `profile.torso` is all-or-nothing (`hold`/`ik`) and there is no height parameter — but **per-joint leg freezing IS available** via `ik.frozen_joints` (verified 08-03) | 🟡 | If live IK can't be made natural, the take runs at a **fixed captured height** — the script needs to know whether height can change *mid-take* or only *between* takes |
| **Head motion** | `head: hold` (LOCAL) | 🟡 | Re-enable + operator-test, or accept a fixed head |
| **Long continuous takes** | Wuji hand **overheats** under sustained teleop | 🟡 | Batch takes with cooldowns; tell the crew the robot needs rest between setups, and build it into the schedule |
| **Robot holds an object at the end of a take** | Hands ramp **open** on driver exit (our fix) — a held object drops | 🟡 | Don't stop the driver mid-hold; catch/place the prop first |
| **Any take right after a reset** | Reset ramps to the captured pose; **hands do not open on reset** | 🟡 | Place/remove props before the reset, not after |
| **Recording video from the robot camera** | Frame drops unfixed (~8–11% of episodes faulty) | 🟡 | Only matters if we record for training. Cinema camera is the crew's, unaffected |
| **Bright / film-set lighting** | ⚠️ **Our camera runs with auto-exposure OFF and a fixed exposure tuned to office light** | 🔴 | See the warning below — this is the most under-appreciated risk in the whole shoot |
| **First bring-up after transport** | Init-gate trip likely (happened after the 07-27 走线); NIC card may unseat | 🟡 | Budget setup time; recover via 上位机; reseat the NIC before debugging |
| **Anything needing a code change on-site** | **No internet, no VPN.** A `git pull` or `flake.lock` bump breaks `robo shell` | 🔴 | **Freeze the tree at 08-04.** Config/yaml edits are fine; dependency changes are not |

> **🔴 Lighting vs. the autonomous policy.** `_lock_camera_exposure` pins the D455 exposure with AE
> **off** (`--camera-exposure 100` = 10 ms), and the gain freezes wherever AE left it. Every frame
> the policy was trained on came from *our office lighting*. A film set is lit very differently and
> often changes between setups. Consequences: (a) the image may blow out or crush, and (b) even if
> it looks fine to a human, it is **out of distribution for the policy** — which is precisely the
> failure mode that made the v0 policies go OOD. **Mitigations, in order of preference:** ask the
> crew to light the robot's working area close to normal room lighting for the autonomous takes ·
> re-tune `--camera-exposure` on-site under the actual lighting (fast, but does not fix the OOD
> problem) · shoot autonomous takes under our own lighting and save the dramatic lighting for
> teleoperated shots. **Raise this with the crew before the shoot day, not on it.**

### 1b. Per-step pass — to fill

| # | Script step / shot | Capability needed | Sev | Risk | Mitigation | Fallback shot |
|---|---|---|---|---|---|---|
| | | | | | | |

*(fill from the script; every 🔴 needs a decided fallback before we travel)*

---

## 2. Props — general requirements

Requirements that come from what the robot and hands can actually do. Send the *principles* to
whoever is sourcing props, not just a list — it lets them substitute sensibly.

**Graspability**
- **Bigger than feels necessary.** Index/middle pinch is currently off and thumb flexion is limited
  → the reliable grip is a **power grasp**, not a fingertip pinch. Props sized for a whole hand.
- **Rigid.** Deformable props change shape under grip force and make the grasp unrepeatable.
- **Light.** The arms droop under load (no brake), and hand weight already shows up in the
  init-gate error trend. Light props, no heavy tools.
- **Not slippery.** Matte, high-friction surfaces. Glossy plastic slips out of the Wuji fingers.
- **Buttons / switches: large-format, stiff, and rigidly mounted.** A big button is only useful if
  it does not slide away when pressed — it needs to be fixed to the table or heavy enough not to
  move. Travel should be forgiving (a few mm of positional error must still register a press).

**Placement and repeatability**
- Every prop that the robot interacts with needs a **repeatable position** — a taped mark, a jig, or
  a shallow tray. Autonomous takes especially: the policy starts from a captured pose and does not
  correct for a badly placed object (07-24 generalization probe: it retries but does not recover).
- Props must sit within the robot's reachable workspace at the **table height we specify** — with
  the torso possibly frozen at a captured pose, table height is a hard constraint, not a preference.

**On camera**
- **Visually distinct from the background and from each other**, and **matte** — the fixed-exposure
  camera has no headroom for specular highlights.
- Bring **spares of anything the robot touches**, especially the Rubik's cube. A dropped or damaged
  hero prop with no backup stops the shoot.

**To fill:** the actual prop list from the script, marked buy / already-have / need-to-source.
⚠️ Anything to buy is on a **shipping clock** for 08/11 — this is the standing lead-time item.

---

## 3. Additional items we need on-site (asks for the partner / TIANJI)

We are bringing the workstation, the robot, the Orin, the router and our cabling. We are **not**
bringing furniture, power distribution, or lighting. Everything below is an ask.

**Furniture**
- **A movable desk (on casters) for the workstation + display + keyboard + mouse.** It must be
  movable because the operator's position relative to the robot matters for teleop — the
  robot-right / operator-left arrangement is what made cube teleop workable, and the crew will want
  to reposition between setups.
- **A display, keyboard and mouse** — ❓ confirm whether the site provides these or we bring our
  own. **The MuJoCo viewer needs a real display; there is no headless path for teleop.**
- **A stable table for the robot's task surface**, at a specified height (❓ fix the number on
  08-04 once the torso question is settled), that does not slide when the robot applies force.
- **A chair for the operator** at a height suited to sustained teleop.

**Power** — the single most common way a setup like this fails on arrival.
- Workstation (RTX 4090 desktop — high draw), robot charger, Orin, router, USB hub, monitor.
- ❓ How many outlets, on how many circuits, and what is the circuit capacity? We will bring power
  strips, but strips do not add amperage.
- ❓ A place to **charge the robot overnight**, and somewhere secure to store it between shoot days.

**Space and safety**
- Robot footprint + operator working area + clear floor for the cable runs (robot ethernet, teleop
  LAN, glove dongles) — cables must be routable without being a trip hazard for the crew.
- Clear e-stop access at all times; the crew should be briefed on where it is and that the robot
  can move unexpectedly.

**Network** — we bring our own router and both networks are self-contained.
- ❓ Confirm we may run our own router/switch on-site.
- **We need no site internet**, and will not use it. (Worth stating explicitly so nobody plans
  around giving us wifi.)

**Lighting** — see the 🔴 warning in §1a. Needs a real conversation with the crew, early.

---

## 4. Packing list

> Now the critical prep item: the workstation is a single point of failure with no spare on-site,
> and a missing cable is unrecoverable. **Tick items physically, not from memory.**

**Compute**
- [ ] Workstation (⚠️ NIC is an **add-in card** — may unseat in transit; reseat first if both
      ethernet links misbehave on arrival)
- [ ] Workstation power cable
- [ ] Display + display cable, keyboard, mouse *(unless the site provides them — confirm)*
- [ ] USB seed stick — **disaster recovery** if the workstation doesn't survive transit; already
      built and verified, costs nothing to carry
      ([runbook](checklists/2026-07-29-onsite-provisioning-runbook.md))

**Robot**
- [ ] TIANJI robot + Orin
- [ ] **Robot charger** ⚠️ (the 07-28 drain happened because a charger went unplugged during cable
      work — [leave-office-sop](checklists/leave-office-sop.md))
- [ ] Both Wuji hands (mounted) + spares/tools for the mounts

**Teleop**
- [ ] Both gloves + both USB-ethernet dongles (**labelled L / R**)
- [ ] Headset (Quest / PICO) + its USB-C ethernet adapter + mount + charger
- [ ] Xiaomi router + its power supply + switch
- [ ] D455 camera + mount *(⚠️ still taped — bring tape / a better mount)*

**Cabling — pack generous, a missing cable is unrecoverable**
- [ ] Ethernet cables: robot↔workstation, workstation↔router, headset↔router, + **spares**
- [ ] 2× USB-ethernet adapters (spares beyond the glove dongles)
- [ ] USB hub ⚠️ **the A→C converter orientation is MARKED — do not replug blind** (SuperSpeed
      passes in one orientation only)
- [ ] Powered USB3 hub *(long-standing purchase, still unbought — decide now or accept the
      bus-power risk)*
- [ ] Power strips (see the circuit question above)
- [ ] Spare USB-C / USB-A cables, zip ties, tape, cable labels

**Paper / reference** (assume no internet on-site)
- [ ] CHEATSHEET printed or on a phone — §1 USB check, §3 glove troubleshooting, §7 上位机 recovery
- [ ] **Photos of the pre-teardown cabling** (Part E of the pre-pack checklist)
- [ ] The hand-yaml swap command, written down
