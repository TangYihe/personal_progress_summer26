# Wuji LEFT hand — over-temperature + transmission slip (possible hardware damage)

> Date: **2026-07-22**. Reporter: Yihe. Status: **OPEN** — hand connected to the workstation,
> checking with Wuji HMI; transmission-slip hint says contact customer service.
> Context: single-side cube-twist collection (batch 1, 97 eps) — the LEFT hand holds/grips the
> Rubik's cube at the task pose while the right arm teleoperates the twist. Sustained hard grip
> on the cube is the suspected load cause.

---

## Verbatim device log

```
[2026-07-22 08:02:03.351] [wuji] [error] Joint Motor F1J2 Reports an exception: Overtemperature.
[2026-07-22 08:02:03.351] [wuji] [error] Hint: Try improve cooling and reduce load.
[2026-07-22 08:06:20.142] [wuji] [critical] Joint Motor F1J3 Reports an exception: Transmission slip detected.
[2026-07-22 08:06:20.142] [wuji] [critical] Hint: Possible hardware damage, please contact customer service.
```

(Timestamps are the device's own clock — appear to be UTC; the collection ran ~15:10 local /
UTC+8, so these correspond to ~16:02–16:06 local.)

## Timeline

- Earlier in the session: **F1J3 reported an over-temperature warning.** Action taken: let the
  hand **cool for ~20 min**, then resumed collection.
- **08:02:03** — `F1J2` **Overtemperature** (error) — "improve cooling and reduce load."
- **08:06:20** — `F1J3` **Transmission slip detected** (critical) — "possible hardware damage,
  please contact customer service."

## Suspected cause

- **Sustained, high-force grip on the Rubik's cube** by the left hand during single-side
  collection (left hand welded to the grasp task pose, pressing the cube for the duration of each
  episode across ~97 episodes). Over-temperature (F1J2, then F1J3) preceded the transmission-slip
  event on F1J3 — thermal stress plausibly contributed to the slip.
- Affected joints: **finger 1, joints J2 and J3** (`F1J2`, `F1J3`) on the **left** hand.

## Actions taken / next

- Cooled the hand ~20 min after the first F1J3 over-temp warning (cleared, resumed).
- After the transmission-slip critical: **collection STOPPED**; hand **connected to the
  workstation to inspect via the Wuji HMI**.
- **Follow-up:** verify joint health in HMI; transmission-slip hint → **contact Wuji customer
  service** with this log. If F1J3 is damaged, batch-2 collection and bimanual use are blocked
  until repaired/replaced.

## Mitigations to discuss

- Reduce grip force on the cube (the grasp pose / retargeting may be commanding more closing
  force than needed to hold it).
- Batch collection with cooldowns (see [[wuji-hand-thermal]] — hand overheats under sustained
  teleop).
- Consider a lighter/softer contact or a fixture that reduces continuous holding load.
