# BOOMSCAN

**Autonomous Earth-Observation Imaging Planner**
*Team 404 — Lost in Space Track*
Gara Shanmukh Sai · Rishika Reddy · Taniskha Reddy · Shaswath Rekham

---

## 1. Problem

Commercial Earth-observation operators get paid for **valid ground coverage**, not for commanded shutter clicks. During a 12-minute LEO pass, the satellite, AOI footprint, and feasible pointing geometry change continuously, and three hard gates silently invalidate frames: smear during the 120 ms exposure (≤ 0.05 °/s), reaction-wheel momentum (≤ 30 mNms), and off-nadir angle (≤ 60°). The customer is a constellation tasking or mission-operations engineer who today either replans manually or accepts a naive nadir-greedy schedule that hits the geometric sweet spot and then loses every frame to the smear gate. They would pay for BOOMSCAN because it converts the *same* satellite passes into more sellable area — more revenue per revisit with no additional spacecraft, downlink, or human in the loop.

## 2. What we built

BOOMSCAN is a single-file deterministic planner ([`plan_imaging.py`](plan_imaging.py)) exporting the required `plan_imaging(...)` entry point. Architecture: SGP4 propagates the orbit at 50 Hz over the pass, an AOI grid is screened for visibility and off-nadir reachability, a greedy mosaic selects frames against a margin-aware coverage objective, and each accepted frame is committed as a **stop-and-stare** waypoint — the same body-to-inertial quaternion is held before, during, and after the 120 ms shutter. SLERP between identical waypoints yields ω ≈ 0 during exposure, pinning `Q_smear = 1.0` by construction. Slews between frames are wheel-momentum validated end-to-end. There is **no dataset, no fine-tuning, no model** — this is orbital geometry plus constraint-aware scheduling. Dependencies: `numpy`, `sgp4`, optional `shapely` (pure-numpy fallback). Margin policy: plan to 58° off-nadir (2° margin) and 25 mNms effective wheel (5 mNms margin) to absorb real controller dynamics.

## 3. How we measured it

Evaluated with the organizer harness in `--mock` mode against a structural-stub baseline that returns a valid no-imaging schedule (scores zero). Command: `python run_evaluation.py --submission plan_imaging.py --all --mock`.

| Submission       | Case 1 | Case 2 | Case 3 (hard, 60° offset) | Weighted `S_total` |
| ---------------- | -----: | -----: | ------------------------: | -----------------: |
| Structural stub  | 0.0000 | 0.0000 |                    0.0000 |             0.0000 |
| **BOOMSCAN**     | **1.0194** | **1.1158** |             **1.2824** |     **1.1583** |

Component breakdown — Case 1: `C=0.845`, `η_E=0.467`, `η_T=0.895`, 33/33 frames kept. Case 2: `C=0.911`, `η_E=0.534`, `η_T=0.920`, 32/32 kept. Case 3: `C=0.965`, `η_E=0.930`, `η_T=0.966`, 7/7 kept. **Q_smear = 1.0000 on all cases. Zero rejections across 72 commanded frames.**

## 4. Orbital-compute story

BOOMSCAN is built to fit on a smallsat flight computer, not a workstation. The whole planner is **one Python file, no model weights, no network calls**, with a memory footprint dominated by the 50 Hz pass arrays and the AOI grid (single-digit MB). Runtime is deterministic and CPU-only — no GPU, no stochastic search, no ML inference — so behavior is reproducible and auditable for flight qualification. Per-case planning runs comfortably below the 120 s/case contest budget on a laptop CPU; the dominant cost is SGP4 propagation and quaternion construction, both of which are well within the duty cycle of an OBC like the Xiphos Q7 / GR740 class. **Latency:** plan once per uplinked AOI, then stream 50 Hz attitude samples and 120 ms shutter windows. **Power:** no accelerator required; the planner is dwarfed by the ADCS and payload draw it commands. This is the orbital-compute argument: a planner small enough to live next to the bus software, not in a ground datacenter.

## 5. What doesn't work yet

**We do not have a verified Basilisk leaderboard score.** Basilisk would not run on our machine, so every number above is from the local mock simulator, which (a) tracks commanded attitude perfectly — no controller lag or overshoot — and (b) credits coverage with a centroid-in-FOV test rather than exact polygon intersection. Both gaps land hardest on Case 3: 7 frames covering 96.5 % of cells is plausible under centroid coverage but unlikely under strict intersection, and edge-of-envelope frames near the 58° margin are exactly where Basilisk's tracking error would matter. Our 2° / 5 mNms margins are the buffer, but they are not a substitute for a real run. Other known gaps: greedy frame selection is not globally optimal (a beam search or small ILP would likely lift Cases 1–2); slew effort is heuristic, not optimal-control; and the pure-numpy area fallback is less precise than exact clipping. **Next questions we'd want a day for:** wire up Basilisk and recheck Case 3, replace greedy with beam search, and add a closed-loop ACS overshoot model so the off-nadir margin is derived rather than chosen.

---

**Files:** [`plan_imaging.py`](plan_imaging.py) (submission) · [`test_plan.py`](test_plan.py) (local tests) · [`PRESENTATION_2MIN.md`](PRESENTATION_2MIN.md) (demo script)
