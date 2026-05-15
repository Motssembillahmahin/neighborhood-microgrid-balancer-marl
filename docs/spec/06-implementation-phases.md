# Spec 06 — Implementation Phases

Implement in strict phase order. Each phase has a gate — do not advance until the gate passes.

---

## Phase 1 — Environment Core

**Gate:** `pytest tests/test_envs/ -v` passes + `check_env()` passes

| # | File | What to build |
|---|---|---|
| 1 | `utils/solar_simulator.py` | Sinusoidal solar curve + Gaussian noise + cloud multiplier from scenario |
| 2 | `utils/demand_simulator.py` | Morning/evening peak demand profile, configurable per home via config |
| 3 | `utils/pricing_engine.py` | Time-of-use price lookup (peak/shoulder/off-peak) from config.yaml |
| 4 | `agents/reward_model.py` | `compute_home_reward()` and `compute_coordinator_reward()` — all coefficients from config |
| 5 | `env/energy_market.py` | Double-auction clearing, price band invariant, `Order` / `MarketResult` dataclasses |
| 6 | `env/home_agent_env.py` | Gymnasium env: 7-dim obs Box, Discrete(14), reset/step, calls reward_model + energy_market |
| 7 | `tests/test_envs/` | Gymnasium compliance, obs/action shapes, reward non-NaN, step/reset determinism |
| 8 | `tests/test_market/` | All clearing cases, price band invariant on 100 random order books |

---

## Phase 2 — SB3 Prototype

**Gate:** positive avg reward on `sunny_day` within 500 episodes; no blackouts after ep 300

| # | File | What to build |
|---|---|---|
| 9  | `scenarios/sunny_day.yaml` | solar_mult=1.5, demand_mult=0.8, grid_cap=1.0 |
| 10 | `scenarios/cloudy_peak.yaml` | solar_mult=0.4, demand_mult=1.4, grid_cap=0.85 |
| 11 | `scenarios/storm_outage.yaml` | solar_mult=0.1, demand_mult=1.2, grid_cap=0.5 |
| 12 | `config.yaml` | Full hyperparameter config (see spec 04 for structure) |
| 13 | `env/microgrid_env.py` | PettingZoo ParallelEnv wrapping N home_agent_env instances |
| 14 | `agents/home_agent.py` | SB3 PPO wrapper: `train()`, `predict()`, `save()`, `load()` |
| 15 | `train.py` | `--mode sb3`: loop over episodes, log to MLflow/W&B, save checkpoints |
| 16 | `evaluate.py` | Load checkpoint, run N episodes on given scenario, return metrics dict |
| 17 | `tests/test_agents/` | Reward computation unit tests, action selection smoke test |

---

## Phase 3 — Dashboard

**Gate:** `docker compose up --build` → Live Grid tab shows WebSocket data within 5s

| # | File | What to build |
|---|---|---|
| 18 | `dashboard/schemas.py` | All Pydantic models: HomeState, GridState, TradeEvent, TrainingEvent |
| 19 | `dashboard/grid_visualizer.py` | FastAPI app: all REST + WebSocket endpoints from spec 05 |
| 20 | `frontend/src/types/api.ts` | TypeScript types mirroring Pydantic schemas |
| 21 | `frontend/src/hooks/` | useWebSocket, useGridState, useTrainingMetrics, useScenarios |
| 22 | `frontend/src/components/EnergyFlowD3.tsx` | D3 SVG energy flow with animated arrows |
| 23 | `frontend/src/components/` | StatBar, StabilityGauge, P2PFeed, CoordinatorPanel, HomeCard |
| 24 | `frontend/src/pages/live-grid.tsx` | Live Grid tab layout |
| 25 | `frontend/src/pages/training.tsx` | Training tab: controls + RewardChart + LossChart + log |
| 26 | `frontend/src/pages/scenarios.tsx` | Scenarios tab: cards + eval controls + comparison table |
| 27 | `frontend/src/pages/agents.tsx` | Agents tab: AgentTable + CoordinatorSummary + CheckpointManager |
| 28 | `docker-compose.yml` | backend (port 8000) + frontend (port 3000) |
| 29 | `Dockerfile.backend` | Python 3.10 slim, installs requirements.txt |
| 30 | `Dockerfile.frontend` | Node 18 alpine, builds Next.js |

---

## Phase 4 — RLlib MARL

**Gate:** MAPPO outperforms IPPO baseline by ≥15% avg reward on `storm_outage`

| # | File | What to build |
|---|---|---|
| 31 | `env/home_agent_env.py` | Add parameterized action space (Dict) behind `--mode rllib` flag in config |
| 32 | `env/grid_coordinator.py` | Coordinator Gymnasium env: 35-dim obs, continuous price signal action |
| 33 | `agents/home_agent.py` | Add RLlib MAPPO policy class alongside SB3 class |
| 34 | `agents/coordinator_agent.py` | RLlib hierarchical agent: trains on coordinator env |
| 35 | `train.py` | `--mode rllib`: RLlib multi-agent config, shared critic, coordinator integration |
| 36 | `.github/workflows/ci.yml` | Lint (ruff) + type-check (mypy) + pytest + Next.js build on every PR |

---

## Critical Files

These files are load-bearing — get them right before anything else:

- `env/home_agent_env.py` — everything depends on this being Gymnasium-compliant
- `env/energy_market.py` — price band invariant must never be violated
- `agents/reward_model.py` — most tuning iterations will happen here
- `config.yaml` — single source of truth; never hardcode hyperparameters anywhere
- `train.py` — phase-switching entrypoint; keep `--mode sb3` and `--mode rllib` cleanly separated
