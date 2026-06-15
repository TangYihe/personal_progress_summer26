# Handoff — where we left off

> Session bookmark. **Current state only** — no history (that lives in the README progress
> log). Update at the end of every working session: what's done, where to pick up next.
>
> Last session: 2026-06-15

---

## Where we are

**GPU procurement (workstream C):** RTX 4090 still pending colleague's case-clearance + PSU-label
report. Large-scale training compute from advisor also still awaited (expected ~2026-06-04/05 —
now overdue; worth following up).

**Reading mode:** I read papers; agent assists, verifies, and organizes. Collaborative, paper-by-paper.

**Codebase investigation (workstream E):** `~/we-teleop/` fully mapped this session. Notes written.
Pending: labmate sync meeting to answer open hardware/dev questions.

---

## What was done this session (2026-06-15)

**Full `~/we-teleop/` codebase investigation.** Three notes written:

- [code-structure.md](notes/implementation/code-structure.md) — overall repo layout, module map,
  entry points, launch profiles, data layout, training pipeline, open threads.
- [data-schema.md](notes/implementation/data-schema.md) — detailed Parquet schema: state origin
  and shape, all force fields with semantics, action representation (two levels: joint-space +
  EE Cartesian), cross-stream sync model and timestamp columns.
- [labmate-sync-questions.md](notes/implementation/labmate-sync-questions.md) — 7-topic prep doc
  for the 师弟 sync meeting (cameras, demo data, replay, action space, dev plans, control modes,
  supplementary). Includes what we already know from code vs. what needs confirming.

**Key findings from code investigation:**

- τ_ext (`joint_external_torque`) confirmed natively recorded for both arms → answers
  force-feedback.md §4 action item; directly usable for FACTR/TA-VLA approaches.
- Force fields: three layers — per-joint τ_ext/effort, arm-base 6D wrench + IMU, wrist
  flange 6D wrench. `flange_force` column exists; hardware availability unconfirmed.
- Action space has two levels in every recording: joint-space (`action.joint_qpos`) and
  Cartesian (`action.ik_target.*`). Hand DOFs recorded but not in training pipeline.
- ACT/DP are original unmodified submodules; connected via Parquet→HDF5 adapter with z-score
  normalization. Open-loop eval runs on recorded data in MuJoCo only (no real-robot eval yet).
- Single head camera only (RealSense D405, 640×480). Wrist cameras not integrated.
- Four robot control modes: position, impedance (joint), imp_cart (Cartesian), force_impedance
  (hybrid position + contact force). force_impedance availability on real robot unconfirmed.

---

## Pick up next

**1. Labmate sync meeting** — highest priority gate.
Prep doc ready at [labmate-sync-questions.md](notes/implementation/labmate-sync-questions.md).
After meeting: fill in answers, update data-schema.md §force fields and force-feedback.md §4
with confirmed sensor availability, unlock any blocked design decisions.

**2. FILIC (`2509.17053`)** — read when ready. Key finding already captured (EEF wrench via
Jacobian inversion +10–17 pp); read for deeper understanding of dual-loop IL + impedance.

**3. Slow-fast policy papers:**
- Slow-fast trio (`2605.27886`)
- ImplicitRDP (`2512.10946`)
Goal: fill in §8 (slow-fast structure) of [design/force-feedback.md](notes/design/force-feedback.md).

**4. Adaptive compliance (`2410.09309`)** — deferred (theory-heavy; revisit after real-robot
experience). Variable impedance (`2603.14068`) likewise.

---

## Watch out / open threads

- **τ_ext confirmed available** for arm joints (resolved action item from §4 force-feedback.md).
  Still open: τ_ext calibration/bias behavior (Q6.6 in sync doc).
- **flange_force** (wrist F/T): column exists but hardware availability unknown → Q6.5.
- **Replay on real robot:** unclear if currently supported → Q3.1 in sync doc.
- **Hand DOFs:** not in training pipeline → needs ownership decision → Q4.2 in sync doc.
- GPU: awaiting colleague's case-clearance + PSU-label report.
- Large-scale training-resource details from advisor (~2026-06-04/05 expected — overdue, follow up).
- Design decisions indexed at [decisions-and-caveats.md](notes/decisions-and-caveats.md).
- After D: Park et al. `2501.04169`; then plan workstream A once data arrives.
