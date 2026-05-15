# CLAUDE.md — Neighborhood Microgrid Balancer (MARL)

## Project Overview

Multi-agent RL system where each home is a PPO agent managing solar, battery, and EV resources,
coordinated by a grid-level hierarchical agent. Core tension: individual reward (minimize
electricity bill) vs shared penalty (blackout penalizes everyone → emergent cooperation).

**Full design specs:** `docs/spec/01` through `docs/spec/07`

## Stack

| Layer | Technology |
|---|---|
| RL env (single-agent) | Gymnasium |
| RL env (multi-agent) | PettingZoo parallel API |
| RL training Phase 1 | Stable-Baselines3 PPO |
| RL training Phase 2 | RLlib MAPPO (shared critic, CTDE) |
| Experiment tracking | MLflow or W&B (set in config.yaml) |
| Backend API | FastAPI + WebSocket |
| Frontend | Next.js (TypeScript), D3.js, Recharts, Tailwind |
| Infrastructure | Docker Compose, GitHub Actions CI |
| Testing | pytest + pytest-cov, ruff, mypy |

## Commands

```bash
# Training
python train.py --mode sb3 --config config.yaml       # Phase 1: SB3 prototype
python train.py --mode rllib --config config.yaml     # Phase 2: RLlib MAPPO

# Evaluation
python evaluate.py --checkpoint models/ppo_ep140.zip --scenario scenarios/cloudy_peak.yaml
python evaluate.py --checkpoint models/ppo_ep140.zip --all-scenarios

# Backend (FastAPI)
uvicorn dashboard.grid_visualizer:app --reload --port 8000

# Frontend (Next.js)
cd frontend && npm run dev

# Docker
docker compose up --build

# Tests
pytest tests/ -v --cov=env --cov=agents --cov=utils --cov=dashboard

# Lint + type check
ruff check . && mypy env/ agents/ utils/ dashboard/
cd frontend && npm run lint && npm run type-check
```

## RL Design (Summary — see docs/spec/ for full detail)

### State Space (7-dim per home agent)
0. `battery_level` — 0–1 normalized
1. `solar_generation` — kW from solar_simulator
2. `ev_charge_status` — 0–1
3. `ev_departure_time` — hours remaining, normalized
4. `grid_price` — $/kWh from pricing_engine
5. `neighbor_agg_demand` — kW aggregate (partial observability)
6. `time_of_day` — 0–1 normalized

### Action Space
- **Phase 1 (SB3):** `Discrete(14)` — 4 base actions + 5 P2P offer bins + 5 P2P buy bins
- **Phase 2 (RLlib):** `Parameterized` — `Discrete(6)` type + `Box` price head

### Reward (comment every term with economic rationale)
```python
R_home = (
    - cost_bought_from_grid
    + revenue_sold_to_grid
    + p2p_trade_profit
    - β * ev_departure_penalty
    - λ * max(0, total_demand - grid_cap)   # shared — forces cooperation
)
```
Coefficients `α`, `β`, `λ` always from `config.yaml`, never hardcoded.

### Training
- **Phase 1:** SB3 IPPO (5 independent PPO agents) — fast iteration, reward shaping
- **Phase 2:** RLlib MAPPO + shared critic — full MARL, CTDE paradigm
- Coordinator: hierarchical RL, 35-dim obs (5 homes × 7), continuous price signal output

## P2P Market
- Double-auction clearing every timestep
- Agents submit explicit price bids (action 4/5 + price)
- **Invariant:** `feed_in_tariff < P* < grid_buy_price` — always enforced, tested in `tests/test_market/`

## Coding Standards

### Python
- Type hints and docstrings on all public functions/classes
- Never hardcode hyperparameters — always read from `config.yaml`
- Comment reward function terms with economic rationale explaining *why* each term exists
- Use `pytest` for all tests. Fixtures in `conftest.py` for shared config and mock envs
- Format with `ruff`. Type-check with `mypy`

### TypeScript (Frontend)
- Strict TypeScript — no `any` types
- Functional React components with hooks
- D3 for custom SVG energy flow diagrams; Recharts for standard time-series charts
- Keep `src/types/api.ts` in sync with FastAPI Pydantic schemas

### General
- Conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- PRs require passing CI (lint + test + build) before merge
- `config.yaml` is the single source of truth — all hyperparameters live there

## Implementation Order

Follow `docs/spec/06-implementation-phases.md` strictly:
1. Environment core (env + simulators + market + reward model)
2. SB3 prototype (training loop + scenarios + evaluation)
3. Dashboard (FastAPI + Next.js)
4. RLlib MARL (MAPPO + coordinator + CI)

Do not skip phases. Each phase has a gate test that must pass before advancing.

## Key Files

| File | Role |
|---|---|
| `env/home_agent_env.py` | Core single-agent Gymnasium env — everything depends on this |
| `env/energy_market.py` | P2P double-auction — price band invariant must never be violated |
| `agents/reward_model.py` | Reward computation — most iteration happens here |
| `config.yaml` | Single source of truth for all hyperparameters |
| `train.py` | Phase-switching entrypoint (`--mode sb3 \| rllib`) |
| `docs/spec/` | Full design specs 01–07 — read before modifying any component |
