<div align="center">

# 🛰️ BOOMSCAN

### *The Satellite Imaging Planner That Wastes Zero Sky*

**Autonomous Earth-Observation Imaging Planner — Lost in Space Track**

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=flat-square&logo=python)](https://python.org)
[![SGP4](https://img.shields.io/badge/Orbit-SGP4%20Propagation-orange?style=flat-square)](https://pypi.org/project/sgp4/)
[![Score](https://img.shields.io/badge/S__total-1.1583-brightgreen?style=flat-square)](/)
[![Smear](https://img.shields.io/badge/Q__smear-1.0000%20%E2%9C%94-success?style=flat-square)](/)
[![Frames](https://img.shields.io/badge/Frame%20Rejections-ZERO-red?style=flat-square)](/)

**Team 404 — Lost in Space**
Gara Shanmukh Sai · Rishika Reddy · Taniskha Reddy · Shaswath Rekham

</div>

---

## The Problem Nobody Talks About

Commercial Earth-observation satellites get paid for **valid ground coverage** — not for pressing the shutter button.

During a 12-minute LEO pass, three silent gates can discard every single frame a satellite captures:

| Gate | Limit | What Kills It |
|------|-------|---------------|
| Angular velocity smear | ≤ 0.05 °/s during 120 ms exposure | Any rotation during shutter open |
| Reaction wheel momentum | ≤ 30 mNms | Aggressive slewing between targets |
| Off-nadir angle | ≤ 60° | Imaging at the edge of coverage |

The naive approach — point straight down, fire every frame, hope for the best — **hits the geometric sweet spot and loses half the frames to the smear gate.** Every lost frame is lost revenue. No additional spacecraft needed. No more ground time. Just smarter scheduling.

**BOOMSCAN turns the same satellite passes into more sellable area.** More revenue per revisit. Zero extra hardware.

---

## What We Built

> One Python file. No ML. No model weights. No network calls. Just physics.

**BOOMSCAN** is a single-file deterministic imaging planner ([`plan_imaging.py`](plan_imaging.py)) built on four elegant ideas:

### The Architecture

```
SGP4 Propagation (50 Hz)
        ↓
   Visibility Screen
   (horizon + nadir geometry)
        ↓
   Off-Nadir Reachability Filter
   (≤ 58° with 2° margin)
        ↓
   Greedy Mosaic Planner
   (max new coverage per frame)
        ↓
   Stop-and-Stare Waypoints ← THE KEY INSIGHT
   (same quaternion: before + during + after shutter)
        ↓
   Wheel Momentum Validation
   (end-to-end slew budget check)
```

### The Key Insight: Stop-and-Stare

Most planners rotate the satellite and fire the shutter mid-slew. That's why they get smear.

BOOMSCAN does something different: it **locks the attitude** before the shutter opens and holds it until after it closes. The same quaternion is emitted at `t_settle`, `t_shutter_open`, and `t_shutter_close`. When the grader SLERPs between identical waypoints, angular velocity during exposure is **mathematically zero** — not approximately zero, not "well within tolerance" — exactly, provably **zero**.

```
                 ┌── SAME QUATERNION ──┐
                 │                    │
t_settle ──► t_open ──► [120ms] ──► t_close
               ω = 0 by construction
               Q_smear = 1.0000 guaranteed
```

This is how BOOMSCAN achieves **Q_smear = 1.0000 across every single case, every single frame.**

---

## Results

Evaluated against the organizer harness in `--mock` mode. Baseline: a structural stub returning a valid no-imaging schedule (scores zero).

```bash
python run_evaluation.py --submission plan_imaging.py --all --mock
```

### Score Table

| Submission | Case 1 | Case 2 | Case 3 (hard, 60° offset) | Weighted `S_total` |
|------------|-------:|-------:|--------------------------:|-------------------:|
| Structural stub | 0.0000 | 0.0000 | 0.0000 | 0.0000 |
| **BOOMSCAN** | **1.0194** | **1.1158** | **1.2824** | **1.1583** |

### Component Breakdown

| Metric | Case 1 | Case 2 | Case 3 |
|--------|-------:|-------:|-------:|
| Coverage `C` | 0.845 | 0.911 | 0.965 |
| Energy efficiency `η_E` | 0.467 | 0.534 | 0.930 |
| Time efficiency `η_T` | 0.895 | 0.920 | 0.966 |
| Frames commanded | 33 | 32 | 7 |
| Frames rejected | **0** | **0** | **0** |
| `Q_smear` | **1.0000** | **1.0000** | **1.0000** |

> **72 commanded frames. 72 valid frames. Zero rejections.**

---

## Why This Runs in Space

BOOMSCAN isn't just designed for the contest — it's designed to actually fly.

| Constraint | BOOMSCAN |
|-----------|----------|
| Dependencies | `numpy`, `sgp4`, optional `shapely` |
| Model weights | None |
| Network calls | None |
| GPU required | No |
| Stochastic behavior | None — fully deterministic |
| Memory footprint | Single-digit MB (50 Hz pass arrays + AOI grid) |
| CPU target | Xiphos Q7 / GR740 class OBC |
| Planning time | Well under 120 s/case on a laptop CPU |

**This planner is small enough to live next to the bus software, not in a ground datacenter.**

One file. One function call. One uplink per AOI. Then the satellite knows what to do.

---

## Margin Policy

We don't fly at the edge of the envelope — we build in margin:

- **Off-nadir:** plan to **58°** (2° margin below 60° spec)
- **Wheel momentum:** plan to **25 mNms** (5 mNms margin below 30 mNms spec)

These margins absorb real controller dynamics, thermal drift, and attitude determination error — the things mock simulators don't model but flight hardware absolutely will throw at you.

---

## How to Run

### Dependencies

```bash
pip install numpy sgp4
pip install shapely  # optional — pure-numpy fallback is built-in
```

### Run Evaluation

```bash
python run_evaluation.py --submission plan_imaging.py --all --mock
```

### Use the Planner

```python
from plan_imaging import plan_imaging

schedule = plan_imaging(
    tle_line1="1 ...",
    tle_line2="2 ...",
    aoi={"type": "Polygon", "coordinates": [...]},
    t_start=1700000000.0,
    t_end=1700000720.0,
)
# Returns: list of imaging waypoints with attitude quaternions
```

---

## Honest Limitations

**We don't have a Basilisk leaderboard score.** Basilisk wouldn't run on our hardware. Every number above is from the local mock simulator, which:

1. Tracks commanded attitude perfectly — no controller lag, no overshoot
2. Credits coverage with a centroid-in-FOV test, not exact polygon intersection

Both gaps bite hardest on **Case 3**: 7 frames covering 96.5% of cells is plausible under centroid coverage, but strict polygon intersection would be tighter. Our margins exist for exactly this reason, but they are not a substitute for a real Basilisk run.

**Known algorithmic gaps:**
- Greedy frame selection is not globally optimal (beam search or a small ILP would lift Cases 1–2)
- Slew effort is heuristic, not optimal-control
- Pure-numpy area fallback is less precise than exact Shapely polygon clipping

**What we'd do with one more day:**
- Wire up Basilisk and recheck Case 3
- Replace greedy with beam search
- Derive off-nadir margin from a closed-loop ACS overshoot model instead of choosing it

---

## File Map

| File | Purpose |
|------|---------|
| [`plan_imaging.py`](plan_imaging.py) | The entire planner — one file, one function |
| [`test_plan.py`](test_plan.py) | Local test suite |

---

<div align="center">

**Built in one hackathon. Designed for orbit.**

*BOOMSCAN — Because every photon counts.*

</div>
