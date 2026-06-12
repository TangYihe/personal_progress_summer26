# Handoff — where we left off

> Session bookmark. **Current state only** — no history (that lives in the README progress
> log). Update at the end of every working session: what's done, where to pick up next.
>
> Last session: 2026-06-11

---

## Where we are

**GPU procurement (workstream C):** RTX 4090 still pending colleague's case-clearance + PSU-label
report. Large-scale training compute from advisor also still awaited (expected ~2026-06-04/05 —
now overdue; worth following up).

**Reading mode:** I read papers; agent assists, verifies, and organizes. Collaborative, paper-by-paper.

---

## What was done this session (2026-06-11)

**Ego-Pi (`2606.08107`) read in full** — PI dexterous humanoid, egocentric human + robot co-training
on π₀.₅. Documented in [survey/mixed-training.md](notes/survey/mixed-training.md). Key takeaways:
- Focus is semantic compositionality (subtask ordering, instruction following), not low-level
  dexterity — limited direct relevance to our hard problem.
- Portable engineering tricks: deliberate viewpoint alignment at collection time (table-mounted
  Zed Mini for human, matched angle to robot head cam); 40% wrist-cam dropout during robot
  training to bridge the human↔robot observation gap; joint-space retargeting (not fingertip).
- Two-token action design is an architectural constraint of π₀.₅ (32D token cap), not a
  principled independence argument.
- 50-50 co-train ratio; authors report policy is not sensitive to it.

**π₀ / π₀.₅ architecture understood:**
- π₀: pretrained VLM (vision + language) + separate action expert (random init, trained on robot
  data). Joint angles + noised actions → action expert via flow matching. VLM frozen or lightly
  fine-tuned.
- π₀.₅: same VLM + action expert split at inference. Adds a FAST-token pre-training stage
  (joint angles as text tokens, actions as discrete FAST tokens — no action expert yet). Action
  expert added at post-training, initialized from scratch. Subtask prediction is an auxiliary
  objective on the VLM side (text output), not the action expert.
- **Why torque → action expert (TA-VLA):** torque correlates with joint state (HSIC), not
  semantic/visual context → belongs in the same stream as proprioception, not the VLM.

**FACTR 2 (`2606.12406`) read** — two contributions:
- **NEXT:** learns robot dynamics from ~10 min free-space data → neural τ_ext estimation for
  arms without torque sensors. Not needed for us (our arm exposes τ_ext), but clarifies what
  the API computes.
- **FIRST:** contact-phase labeling via τ_ext threshold (hysteresis); pre-contact = 1 s window;
  phase-weighted batch sampling. Self-supervised from τ_ext, no manual labels. Directly portable.
- Cross-paper finding: TA-VLA auxiliary torque prediction outperforms input-only even in ACT
  settings → strategy is backbone-agnostic.
- Updated [design/force-feedback.md](notes/design/force-feedback.md): §6 Option B strengthened
  (backbone-agnostic note); §6 Option D added (FIRST as a complementary training trick).

**FILIC (`2509.17053`) — agent-summarized, not read.** Key finding: converting joint torque to
6D EEF wrench via Jacobian inversion outperforms raw joint torque by 10–17 pp on insertion tasks.
Adds evidence for §4 Option C in force-feedback.md. Deferred to reading queue.

---

## Pick up next

**1. FILIC (`2509.17053`)** — read when ready. Key finding already captured; read for deeper
understanding of dual-loop IL + impedance and Jacobian-inversion force estimation.

**2. Slow-fast policy papers:**
- Slow-fast trio (`2605.27886`)
- ImplicitRDP (`2512.10946`)
Goal: fill in §8 (slow-fast structure) of [design/force-feedback.md](notes/design/force-feedback.md).

**3. Adaptive compliance (`2410.09309`)** — deferred (theory-heavy; revisit after real-robot
experience). Variable impedance (`2603.14068`) likewise.

---

## Watch out / open threads

- **Action item:** check which torque signal(s) our robot exposes (raw motor torque vs.
  τ_ext via API) — gates §4 preprocessing choice in force-feedback.md.
- GPU: awaiting colleague's case-clearance + PSU-label report.
- Large-scale training-resource details from advisor (~2026-06-04/05 expected — overdue, follow up).
- Design decisions indexed at [decisions-and-caveats.md](notes/decisions-and-caveats.md).
- After D: Park et al. `2501.04169`; then plan workstream A once data arrives.
