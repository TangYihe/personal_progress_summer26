# Pipeline shakedown issues — 2026-07-17 (post 50 Hz merge)

> Evidence file for reporting to labmates. All found while trying to collect cube-twist
> data on `wip/collection-integration-20260717` = origin/rewrite-0711 @ `dfddafb`
> (50 Hz control / 100 Hz driver) + our task-pose/single-side commits + local patches.
> Workstation = operator host; driver on Orin `6.6.7.100`; robot `6.6.7.190`.
> Settings were kept at upstream values for all repros (input_timeout_ms 100 etc.).

---

## Issue 1 — UNINTENDED ROBOT MOVEMENT + driver playout faults ⚠️ SAFETY-RELEVANT

**Symptom:** with the PICO controller held still, the arm keeps moving on its own.
Operator-observed repeatedly when the arms are near the intended init/grasp pose
(first seen ~16:29 local, reproduced ~17:44 local during a bimanual session).

**Driver log at the moment of a 17:44 repro (verbatim, times are UTC = local−8):**

```text
07-17 09:44:36.799 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.799 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.815 ERROR    [hardware] canonical action playout fault  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_hardware:430
07-17 09:44:36.818 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.835 ERROR    [hardware] canonical action playout fault  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_hardware:430
07-17 09:44:36.837 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.855 ERROR    [hardware] canonical action playout fault  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_hardware:430
07-17 09:44:36.857 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.875 ERROR    [hardware] canonical action playout fault  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_hardware:430
07-17 09:44:36.879 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
07-17 09:44:36.895 ERROR    [hardware] canonical action playout fault  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_hardware:430
07-17 09:44:36.897 WARNING  [observation] playout quality degraded; recording will retain markers  error=interpolation lookahead rejected  teleop.runtime.driver_supervisor:_run_observation:830
```

Fault repeats every ~20 ms (= every 100 Hz driver tick) — sustained rejection, not a
one-off. New code path: the 50→100 Hz playout/interpolation is from today's upstream
work (`29b0a04` 100Hz control, `5cc9d04`, `78248c1`).

**Working hypothesis (unverified):** operator command stream develops gaps (see Issue 2's
host-side stalls) → driver playout queue starves → "interpolation lookahead rejected" →
driver-side extrapolation/shaping continues producing motion the operator didn't command.
Would unify Issues 1+2 under one root cause (operator-host stalls) with two failure
surfaces. `runtime.yaml` notes `driver_extrapolation` defaults on and "amplifies
frame-to-frame steps"; an A/B with it off would discriminate.

**Discriminator still to run:** watch the wrist target spheres during a repro —
spheres move while operator still = input side (PICO); spheres still but robot moves =
driver/playout side (consistent with these logs).

**UPDATE 17:56 — third stalled stream + likely root cause found.**

New failure with the operator still connected (verbatim):

```text
07-17 17:56:31.520 WARNING  [control] control safety alert  alert=robot feedback unavailable or stale; forced pause
07-17 17:56:31.536 ERROR    [recording] recording status changed  phase=failed task=recording_bug_0717 episodes=1 frames=1 error=action 4103 fell back to hold
07-17 17:57:06.428 ERROR    [recording] ... error=recording command failed: recording operation requires recording, currently failed
```

So the DRIVER→operator feedback stream also goes stale (>100 ms), forcing PAUSE/hold.
Commands starving (Issue 1) + feedback going stale point at the ROBOT HOST side.

**Root-cause candidate: the `78248c1` desched fix needs `SCHED_FIFO/20` and our Orin
denies it.** Driver startup warns every launch:

```text
07-17 04:07:43.766 WARNING [hardware] hardware scheduling configured  cpu=11 policy=SCHED_OTHER detail=SCHED_FIFO unavailable; grant CAP_SYS_NICE or realtime rtprio cyclic_gc=disabled after startup
```

i.e. on our robot the stutter fix silently degrades to SCHED_OTHER → the kernel
desched stutter the fix targets persists → playout faults + stale feedback.
Works-on-their-machine instance #4 (after exposure units, camera profile, robo.nix
graphics). Suggested fix: grant rtprio on the robot host (limits.d) + make the
driver FAIL LOUD (or at least banner-level) when SCHED_FIFO is denied; document the
grant in bootstrap_luna_orin.sh.

Supporting datapoint: a ~30 s episode with the operator perfectly STILL saved
successfully (episodes=1 above); episodes fail when the operator moves — consistent
with load-sensitive scheduling stalls.

---

## Issue 2 — episodes fail within ~10 s: "missing required left hand data"

**Symptom:** strict recorder fails episodes quickly and repeatedly:

```text
07-17 16:30:00.061 ERROR [recording] recording status changed  phase=failed task=unintended_movement_0717 episodes=0 frames=542 error=action 13438 is missing required left hand data
07-17 17:37:29.946 ERROR [recording] recording status changed  phase=failed task=recording_test    episodes=0 frames=487 error=action 6455 is missing required left hand data
```

**Analysis:**
- Check = `recording/service.py` per-frame required-modality guard (rewrite-0711
  strict recorder; NOT new today — morning episodes passed it).
- Hand command goes empty when EITHER glove's sample age exceeds `input_timeout_ms`
  (100 ms) — `LatestHands.sample()` returns None for the whole tick; the error names
  "left" only because left is checked first.
- **`we hands check --side both` soak: both gloves pin 120 Hz normally, but show
  correlated dips hitting BOTH sides simultaneously** (worst observed: 37 Hz with
  ~80 ms ages; several 60–100 Hz dips). Cable-independent (fresh cables, dips not
  correlated with wiggling; single-glove cabling can't slow the other side).
- Both gloves' receive+retarget workers run as THREADS inside the one operator
  supervisor process (~125% CPU observed) — unlike the four main processes. GIL/CPU
  contention there would stall both streams at once. Suggested fix: promote hand
  input to its own process.
- Left-glove hardware WAS also flaky earlier (adapter carrier=0 pre-session) — fixed
  with new cables + connector reseat; soak now clean at baseline. The remaining
  correlated-stall issue is host-side, not cabling.

**UPDATE ~18:10 — TRIGGER ISOLATED: FINGER MOVEMENT (operator-controlled repro).**
Hands moving in space with fingers still → episodes record fine. The moment fingers
move, even slightly → immediate failure. Reproduced multiple times; operator has a
recording of the repro.

Mechanism this points at: the per-frame MuJoCo retarget optimization warm-starts to
near-zero cost while fingers are still, but spikes when finger pose actually changes →
retarget worker latency > input_timeout (100 ms) → hand action empties for the tick.
Both retargeters share the supervisor process's GIL, so one side's expensive solve
stalls BOTH streams. For the retargeting owner: `we hands check --deep --output …`
captures per-frame `optimizer_evaluations` + timings — a 30 s trace while wiggling
fingers should show the cost spike directly.

**QUANTIFIED (60 s `--deep` capture, both hands, still + finger-movement phases;
`we-teleop/artifacts/logs/hand_check_0717.jsonl`, 87 MB):**

| | fingers STILL | fingers MOVING |
|---|---|---|
| retarget ms (median / p99 / max) | 3.6 / 22 / 154 | 24 / 60 / 155 |
| optimizer_evaluations (median) | 9 | **50 = the cap, every moving frame** |

- Publish gaps > 100 ms: **3–4 per minute per side** (worst 167 ms) → with the 100 ms
  input timeout + strict recorder, expected episode failure every ~15–20 s of recording.
- Gaps land on BOTH sides at identical timestamps (t=32.6 s / 43.4 s / ~57.5 s on both)
  — two independent devices stalling in lockstep = shared-process GIL coupling, proven.
- Mean delivered retarget rate 88.8 Hz vs the 120 Hz glove stream.
- Suggested directions for the owner: per-side (or per-glove) process for hand input;
  and/or a per-frame optimizer time budget (the 50-eval cap costs ~24 ms median when
  active — it saturates on any finger motion).

**SUB-BUG — misleading error message:** with the LEFT hand deliberately still and only
RIGHT fingers moving, the failure still reads "missing required left hand data"
(`recording/service.py` checks sides in ("left","right") order and blames the first,
while the actual stale-side information is discarded in `LatestHands.sample()` which
returns None wholesale when EITHER side is stale). Costs real debugging time — we
chased the left glove's cabling for an hour. Suggest: name the stale side, or report
"hand input stale (worst side: X, age: Y ms)".

**History note:** pre-rewrite datasets (e.g. 07-10) were collected on old master whose
recorder did NOT enforce per-frame hand modality — glove gaps went silently into data.

---

## Issue 3 — upstream tree hygiene (rewrite-0711 @ dfddafb)

1. **Merge-backup test files committed**: `tests/control/inputs/test_pico_resample_{BACKUP,BASE,LOCAL,REMOTE}_4044680.py` contain raw conflict markers → ruff + pytest collection fail on a clean checkout (185 syntax errors). We removed them on our branch.
2. **JPEG-quality contract test stale**: `5cc9d04` changed `jpeg_quality` to 60; `tests/robots/test_camera_clock.py` still expects 90 → fails on clean tip.
3. (Still open from 07-16) `tests/contracts/test_launch_profiles.py` expects `pico_world` `head: hold`; config ships `head: pico` → fails on clean tip.
4. (Standing) init-pose gate not configurable (0.5°/1°/s too strict for hand-loaded wrists — we carry a local patch).

---

## Context for reproduction

- PICO: third-person, headset racked. Glove pair: the newer pair (post 07-15 swap).
- Camera: D455 `419222302053` @ 640x480@60, exposure 100 (100 µs units), USB3 via
  hub (A→C converter orientation-sensitive! marked).
- Both hands mounted; serials in local `config/actuators/wuji.yaml` patch.
- To capture full evidence: operator `pico.sh <profile> --debug` + driver `--debug`,
  then `artifacts/logs/operator-*/` + `artifacts/logs/driver-*/`.
