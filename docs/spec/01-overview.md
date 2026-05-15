# Spec 01 — Project Overview & Key Decisions

## What We're Building

A multi-agent reinforcement learning system where N homes (each with solar panels, a battery,
and an EV) act as independent RL agents that negotiate energy via a P2P double-auction market.
A grid-level coordinator agent maintains neighborhood stability by issuing dynamic price signals.

**The core MARL challenge:** agents are individually incentivized to minimize their own
electricity bill, but a shared blackout penalty forces emergent cooperation. No agent is ever
told to cooperate — they discover it's cheaper than triggering the collective penalty.

## Stack

| Layer | Technology |
|---|---|
| RL environments | Gymnasium (single-agent), PettingZoo parallel API (multi-agent) |
| RL training (Phase 1) | Stable-Baselines3 PPO |
| RL training (Phase 2) | RLlib MAPPO with shared critic |
| Experiment tracking | MLflow or W&B (configurable in config.yaml) |
| Backend API | FastAPI + WebSocket |
| Frontend | Next.js (TypeScript), D3.js, Recharts, Tailwind CSS |
| Infrastructure | Docker Compose (backend + frontend), GitHub Actions CI |
| Testing | pytest + pytest-cov, ruff, mypy |

## Key Design Decisions (Locked)

| Decision | Choice | Rationale |
|---|---|---|
| P2P price bids | Explicit — agents submit action type + price | More realistic; agents learn pricing strategy |
| Action space Phase 1 | Discrete(14): 4 base + 5 offer bins + 5 buy bins | SB3-compatible, fast iteration |
| Action space Phase 2 | Parameterized: Discrete(6) type + Box price head | RLlib native, continuous price |
| Training paradigm | MAPPO with shared critic (CTDE) | Standard cooperative MARL; shared critic attributes shared penalty correctly |
| Training path | SB3 IPPO prototype → RLlib MAPPO full MARL | Two-phase: debug reward first, scale second |
| Episode resolution | 48 timesteps × 30 min = 24-hour episodes | Captures full daily solar/demand cycle |
| Dashboard tabs | Live Grid, Training, Scenarios, Agents | Designed in visual companion session |

## High-Level Data Flow

```
Simulators (solar, demand, pricing)
    ↓
microgrid_env.py  ←→  home agents (7-dim obs, 14/parameterized actions)
    ↓                        ↑
energy_market.py  ←→  P2P bids/asks cleared → rewards computed
    ↓
grid_coordinator.py  →  price signal → fed back into next timestep obs
    ↓
FastAPI WebSocket  →  Next.js dashboard (live during training and inference)
```

## Project Structure

```
env/              # RL environments
agents/           # RL agent wrappers
utils/            # Simulators and pricing engine
scenarios/        # YAML scenario definitions
dashboard/        # FastAPI backend
frontend/         # Next.js dashboard
tests/            # pytest test suite
docs/spec/        # This spec directory
train.py          # Training entrypoint (--mode sb3 | rllib)
evaluate.py       # Scenario-based evaluation
config.yaml       # Single source of truth for all hyperparameters
```
