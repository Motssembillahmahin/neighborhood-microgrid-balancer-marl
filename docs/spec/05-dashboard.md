# Spec 05 — Dashboard Design

## Backend — FastAPI (dashboard/grid_visualizer.py)

### REST Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/state` | Current `GridState` snapshot (latest timestep) |
| GET | `/metrics` | Training metrics: reward curve, loss, episode count, P2P rate |
| GET | `/scenarios` | List available scenario YAML files |
| POST | `/evaluate` | Trigger evaluation run (body: `{checkpoint, scenario, n_episodes}`) |
| GET | `/agents` | Per-agent stats for current episode |
| GET | `/checkpoints` | List saved model checkpoints with reward scores |

### WebSocket Endpoints

| Path | Push frequency | Payload |
|---|---|---|
| `/ws/grid` | Every sim timestep (30-min sim = ~100ms real) | `GridState` |
| `/ws/training` | Every episode completion | `TrainingEvent` |

### Pydantic Schemas (dashboard/schemas.py)

```python
class HomeState(BaseModel):
    agent_id: str
    battery_level: float       # 0–1
    solar_generation: float    # kW
    ev_charge_status: float    # 0–1
    ev_departure_time: float   # hours remaining
    last_action: int           # 0–13 (Phase 1)
    last_reward: float
    last_p2p_price: float | None

class GridState(BaseModel):
    timestep: int
    episode: int
    homes: list[HomeState]
    grid_load_pct: float       # total demand / grid capacity
    clearing_price: float | None
    trades: list[TradeEvent]
    coordinator_price_signal: float

class TradeEvent(BaseModel):
    seller_id: str
    buyer_id: str
    quantity_kwh: float
    price: float
    timestep: int

class TrainingEvent(BaseModel):
    episode: int
    avg_reward: float
    policy_loss: float
    value_loss: float
    blackout_count: int
    p2p_trade_volume: float
```

---

## Frontend — Next.js (TypeScript)

### Pages

| Page | Route | Tab |
|---|---|---|
| `pages/index.tsx` | `/` | → redirects to `/live-grid` |
| `pages/live-grid.tsx` | `/live-grid` | Live Grid |
| `pages/training.tsx` | `/training` | Training |
| `pages/scenarios.tsx` | `/scenarios` | Scenarios |
| `pages/agents.tsx` | `/agents` | Agents |

### Key Components

**Live Grid tab:**
- `<EnergyFlowD3 />` — D3.js SVG showing real-time energy arrows between homes, P2P market, utility grid
- `<StatBar />` — 5 top-level KPIs (solar, grid load, P2P volume, cost, avg reward)
- `<StabilityGauge />` — horizontal bar with blackout risk zone
- `<P2PFeed />` — live scrolling list of recent trades
- `<CoordinatorPanel />` — current price signals from coordinator
- `<HomeCard />` × N — per-home solar/battery/EV bars, current action badge, last reward

**Training tab:**
- `<TrainingControls />` — Start / Stop / Pause / Save checkpoint buttons
- `<RewardChart />` — Recharts line chart of episode reward history
- `<LossChart />` — PPO policy + value loss curves
- `<P2PRateChart />` — P2P demand share over episodes
- `<HyperparamSliders />` — sliders that write to config.yaml (applied on next run)
- `<TrainingLog />` — live scrolling log (WebSocket `/ws/training`)

**Scenarios tab:**
- `<ScenarioCard />` × 4 — sunny_day, cloudy_peak, storm_outage, custom
- `<EvalControls />` — select checkpoint + scenario, run 10-episode eval
- `<ScenarioComparisonTable />` — reward, blackout rate, P2P%, cost saving across scenarios

**Agents tab:**
- `<AgentTable />` — per-agent reward, bill saved, P2P trades, action distribution bar, EV violations
- `<CoordinatorSummary />` — coordinator obs dim, action type, CTDE note
- `<CheckpointManager />` — list checkpoints with reward bars, load/delete buttons

### Hooks

```typescript
useWebSocket(url: string)           // generic WS connection with reconnect
useGridState()                      // subscribes to /ws/grid, returns GridState
useTrainingMetrics()                // subscribes to /ws/training, accumulates history
useScenarios()                      // fetches /scenarios list
```

### TypeScript Types

Keep `src/types/api.ts` in sync with FastAPI Pydantic schemas. No `any` types.
