# Force & impedance control

> Workstream **D** (background knowledge), folder index. ← back to [overview](../../../README.md).
>
> Last updated: 2026-06-03

---

## Why this matters

Our robot data will record **all joint torques / forces** and the robot supports **impedance
control**. We want to test whether we can **incorporate force into the policy** to leverage this
(force/tactile is a core research interest). I'm relatively **less familiar** with this area, so
this stream is partly upskilling — budget time for building intuition.

## Goal

- Build working intuition for **force vs. position control** and **impedance control**.
- Survey how recent policy-learning work **handles / incorporates force**.
- Land on **concrete options** for adding force to our policy.

---

## Sub-pages

- [concepts.md](concepts.md) — running conceptual understanding (force vs. position, impedance,
  admittance, etc.). The place to build and refine intuition.

## Papers — to read (inbox)

> Dump works here (title / link / one-line why). Move to a per-paper note once read.

- _(add works to read here)_

1. https://arxiv.org/html/2605.27886v1 A recent work on tactile / force manipulation in sim. They have some novel part on how they customized the impedence controller & seperate high- and low-level (force-aware, fast) policy. Read in detail. 
2. https://arxiv.org/abs/2603.14068 Labmate recommended. On learnable / variable impedance control.
3. https://arxiv.org/pdf/2512.10946 ImplicitRDP. On slow-fast policy with force input. Need to read. 
4. https://arxiv.org/abs/2410.09309 Adaptive compliance control. Pretty famous one. Maybe also want to checkout its follow-up works. 
5. https://arxiv.org/abs/2509.07962 Torque-aware VLA. Proposed future prediction as an auxiliary training target to avoid mode collapse. Also a while ago maybe worth checking its follow-ups. Also good to see how they incoporated force/torque in VLAs, instead of smaller DPs. 


## Notes (per paper)

> ### <Title> (<venue/year>) — [link]()
> - **Idea / problem:**
> - **Force vs. position handling:**
> - **How force enters the policy / controller:**
> - **Key takeaway:**
> - **What to try / relevance to us:**

- _(no papers processed yet)_

---

## Options to try (force in our policy)

- _(add candidates as they surface)_
