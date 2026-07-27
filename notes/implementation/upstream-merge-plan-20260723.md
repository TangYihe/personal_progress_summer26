# Upstream merge plan (drafted 2026-07-22 eve, execute 07-23)

> Big merge to absorb the rewrite-0711 refactor, now landed on `master`. Planned
> tonight; **execute tomorrow as a dedicated block, not before the eval work.**
> Claude cannot fetch/push — Yihe runs all `git fetch`/`push`.

## Scope (measured 2026-07-22 18:45 after fetch)

- **Target is now `origin/master`**, not `rewrite-0711`. They merged
  `rewrite-0711 → master` on 07-21 (`664fdd3`); `rewrite-0711 ⊆ master`;
  `origin/HEAD → master`. Confirm with the owner tomorrow that master is the
  canonical branch going forward before basing/pushing.
- **History was force-pushed/rebased again.** Our old base `09b7273` is gone from
  `rewrite-0711` but **still in `master`'s history** (via the merge), so our
  merge-base with master is still `09b7273`.
- **Divergence vs our base:** 47 commits, **379 files, +34.3k / −18.5k**. The bulk
  is ONE squashed commit `3e489cf "Rewrite teleoperation, recording, and policy
  stack"` (657 files) — reconcile against one giant diff, not 47 small ones.
- Our branch `wip/collection-integration-20260720` = `09b7273` + **12 local
  commits** (10 older + `cbd02af` replay --pose + `4c851b5` gt-obs deploy).
  Restore points intact: `-20260717`, `-20260720`.

## Mechanical approach

Since our base is still in master's history, replay our 12 commits onto the new
tip with git assistance (heavy conflicts expected on reset/control_process/cli):

```
git rebase --onto origin/master 09b7273 wip/collection-integration-20260720
# or: fresh branch off origin/master + cherry-pick our 12 in dependency order
```
Work on a fresh `wip/collection-integration-20260723`; keep -20260720 as restore.

## Collision zones (all touch control_process.py / cli.py)

1. **Unified reset** (`47aeb3d unified reset`, `f27ea68 unify hand reset`,
   `5e7630b optimized reset further`) — collides with our task-reset-pose
   (`a340e2f`, `1c0d501`, `144cc45`, `399cf81`) + driver-reset-honors-commanded-
   body (`3513b8a`). **Hardest part.** Re-implement our staged body→fingers task
   pose ON TOP of their unified reset. **Owner decision needed: whose reset wins**
   before pushing our features to a shared branch.
2. **Deploy flags replaced** (`0a8b749 replace rollout flags with one strategy
   option`, `006b29c allow ACT action horizon at deployment`) — re-port our gt-obs
   feature (`4c851b5`) and our `--temporal-aggregation`/`--live` usage onto the new
   single strategy option + deploy-time action-horizon knob. ⚠️ this also changes
   the deploy commands we use for eval — our eval work stays on the 20260720 tree.
3. **CLI split + control_process rewrite** — re-port both new features onto the new
   module layout (`we/cli.py` split into driver/eval/policy/train modules).

## Design-conflict resolution — owner coordination (traced in origin/master 07-22 eve)

Four places where our features collide on DESIGN with upstream's parallel work.
Yihe's steer noted inline; **#1 direction decided, #2 to discuss tomorrow.**

### 1. Reset semantics — per-task pose vs. one global reset pose ✅ direction set
- **Ours:** per-task **body+hand** poses (`artifacts/task_poses/<name>.yaml`) via
  `setpose`/`usepose`/`home`; staged reset (body ramps → converge → fingers close);
  drives teleop + collection + deploy start-state. `3513b8a` makes reset honor the
  **commanded body target** (else a task pose silently resets to default init).
- **Theirs** (`47aeb3d`/`f27ea68`/`5e7630b`): `_fold_wuji_reset_pose()` in
  `control_process.py` folds ONE global **hand-only** pose
  (`config/inputs/wuji_reset_pose.yaml`, `capture_wuji_reset_pose.py`) into every
  reset (startup/Tab/post-episode); **body still ramps to the hardcoded operational
  pose** — exactly what `3513b8a` overrides.
- **→ DECISION (Yihe 07-22): ours is the superset. Keep THEIR global reset pose as
  the default, and keep OUR per-task customizable reset-pose feature layered on top
  (task pose overrides when active).** Same shape as the 07-20 integration. Re-apply
  our task-reset commits on top of their unified reset; preserve `3513b8a`. Stretch
  ask to owner: generalize `wuji_reset_pose.yaml` into a named body+hand pose store
  so our task poses ARE their reset poses (stop diverging).

### 2. Single-side — profile arm+hand welding vs. availability-driven hand_sides ⏳ discuss tomorrow
- **Ours:** `sides` PROFILE key (`pico_world_wuji_right.yaml`, `sides:[right]`) —
  task design: **freezes the non-selected ARM** at the task pose (IK welding),
  restricts glove input, sets modalities.
- **Theirs:** `hand_sides` on `OperatorConfig`, derived from **hardware
  availability** (`prepare_wuji_network(allow_partial=True)` → connected gloves,
  `cli.py:263`); **hands-only, no arm-freezing**, no profile key.
- **Tension:** two meanings of "active sides" (authored+arm-aware vs discovered+
  hands-only). Clash: profile `sides:[right]` but both gloves plugged → ours ignores
  left (task design) vs theirs enables both (availability).
- **→ Yihe 07-22:** upstream's single-hand is probably just handling a **broken/absent
  hand** (availability). **Resolve details tomorrow** — likely: keep our profile
  `sides` for arm-freezing + task design, feed it INTO their `hand_sides`, decide
  precedence (profile overrides availability). Owner decision: accept a profile key
  that gates arms too?

### 3. Deploy rollout strategy — our gt-obs axis vs. their `--rollout` enum (re-port)
- **Theirs** (`0a8b749`): one `--rollout async|sequential|temporal-aggregation|rtc`
  enum (`PolicyRollout`) replaces the old booleans. **Our `--temporal-aggregation`
  → `--rollout temporal-aggregation`.**
- **Ours** (`4c851b5`): gt-obs HARDWARE deploy = observation-source axis (orthogonal
  to rollout), but touches the same rewritten files. Re-port as a separate option;
  confirm with owner it fits their deploy model (they may treat recorded-obs as
  eval-only). Related: `006b29c` "ACT action horizon at deployment" may let us DROP
  the dual chunk100/chunk200 configs (set horizon per-deploy on one model) — align.

### 4. Init gate — migration, not a real conflict
- Our `4244cbe` hardcodes the startup-gate tolerance (2.5°) in `interfaces.py`;
  theirs is parameterized. Adopt theirs; ask for per-profile configurability with our
  evidence (Arm_R6 2.01° under hand load).

### Meta-ask to the owner
Upstream has rebuilt our reset + single-hand work THREE times. Durable fix = upstream
our IL-collection workflow (per-task body+hand start pose, single-arm collection,
recorded-obs deploy) INTO their reset/deploy modules, not re-merge weekly.

## Our 12 commits — re-apply grouping

- **Low conflict:** `c763c14` (--exposure), `6738358` (driver_address default),
  `6304413` (merge-backup removal — likely obsolete now), `0be7efa` (config
  snapshot: serials, d455 profile, retargeting, holds → mostly re-apply as config).
- **Collides — reset:** the 4 task-reset commits + `3513b8a` → zone 1 above.
- **Collides — single-side:** `325c392` → merge with their single-hand preflight
  (as done in the 07-20 integration: profile sides restrict + their reachability
  discovery filters).
- **Superseded?** `4244cbe` init gate 2.5° → check for their configurable IK/init
  gate and adopt it instead of our hardcoded bump.
- **Re-port onto rewritten files:** `cbd02af` (replay --pose), `4c851b5` (gt-obs)
  → zones 2 + 3.

## Free wins that come with the merge

- IK joint-boundary-normalization (was a candidate surgical cherry-pick).
- wuji runtime reconnect (glove-dropout watchdog).
- `babc8fd` GR00T real-time chunking, `6c9e909` tolerate absolute policy inference
  lag, `115e992` remote policy version recovery, `757cd68` quest calibration.

## Verify + rollout (after re-apply)

1. `ruff check` · `pytest -q` (expect known fails — re-baseline them) ·
   supervisor smoke · headless sim.
2. Orin `sync_luna.sh nvidia@6.6.7.100` + venv refresh (uv.lock WILL change) +
   exit/re-enter robo shells if flake.lock changed.
3. Robot re-validation: finger-movement repro, single-side, `usepose`, staged
   reset, short teleop feel.
4. Coordinate reset/single-hand ownership with the owner → **Yihe pushes.**

## Local uncommitted state to preserve through the merge

- `config/inputs/retargeting/wuji_finetune.json` (both hands recal, thumb w=0)
- `config/teleop/pico_world.yaml` (head/torso → hold)
- Today's untracked: `scripts/data/{repair_missing_images.py,merge_recordings.py}`,
  the 4 `act_rubik_twist*` policy configs, the merged dataset (artifacts, not git),
  `docs/agent/20260722T085421Z_*` incident note.
