# Spec 02 — Environment Design

## State Space — per home agent (7 dimensions)

| Dim | Feature | Range | Source |
|---|---|---|---|
| 0 | `battery_level` | 0–1 normalized | battery model |
| 1 | `solar_generation` | kW | `solar_simulator.py` |
| 2 | `ev_charge_status` | 0–1 normalized | EV model |
| 3 | `ev_departure_time` | hours remaining, normalized 0–1 | per-home config |
| 4 | `grid_price` | $/kWh | `pricing_engine.py` |
| 5 | `neighbor_agg_demand` | kW aggregate (partial obs — not per-agent) | `microgrid_env.py` |
| 6 | `time_of_day` | 0–1 normalized (step / 48) | env step counter |

Observation space: `Box(low=0, high=1, shape=(7,), dtype=np.float32)` with appropriate
scaling per dimension. `grid_price` and `solar_generation` normalized against config max values.

## Action Space

### Phase 1 — SB3 Prototype: `Discrete(14)`

```
0:    Buy from utility grid
1:    Sell to utility grid
2:    Charge battery from solar
3:    Draw from battery to cover demand
4–8:  P2P Offer energy at price bins [p1, p2, p3, p4, p5]
9–13: P2P Buy energy at price bins [p1, p2, p3, p4, p5]
```

Price bins are defined in `config.yaml` under `market.p2p_price_bins`.
All bins must satisfy: `feed_in_tariff < bin_price < grid_buy_price`.

### Phase 2 — RLlib Full MARL: Parameterized

```python
action_space = Dict({
    "action_type": Discrete(6),       # 0=buy_grid, 1=sell_grid, 2=charge, 3=draw, 4=p2p_offer, 5=p2p_buy
    "p2p_price":   Box(low=feed_in, high=grid_buy, shape=(1,))
})
# p2p_price is only used when action_type ∈ {4, 5}; ignored otherwise
```

## Simulators

### solar_simulator.py

```python
def generate_solar_curve(date, cloud_factor=1.0) -> np.ndarray:
    # Returns array of shape (48,) — one kW value per 30-min timestep
    # Base: sinusoidal curve peaking at solar_noon (configurable)
    # Noise: Gaussian σ from config
    # Cloud multiplier: 0–1 from scenario YAML (cloud_factor)
```

### demand_simulator.py

```python
def generate_demand_curve(home_id, season="summer") -> np.ndarray:
    # Returns array of shape (48,) — household demand kW per timestep
    # Profile: morning peak (7–9am) + evening peak (6–9pm)
    # Per-home variation: each home has a base_demand_kw from config
```

### pricing_engine.py

```python
def get_price(timestep: int) -> float:
    # Time-of-use pricing lookup from config.yaml
    # Peak hours (6–9pm): high price
    # Off-peak (midnight–6am): low price
    # Shoulder: mid price
    # Returns $/kWh
```

## home_agent_env.py (Gymnasium)

- Inherits `gymnasium.Env`
- `observation_space`: `Box(7,)` as above
- `action_space`: `Discrete(14)` (Phase 1) or `Dict` (Phase 2)
- `reset()`: randomize battery (40–80%), solar/demand from simulators, EV departure in 4–16h
- `step(action)`: apply action, call `energy_market`, compute reward, advance state
- Must pass `gymnasium.utils.env_checker.check_env()` before integration into PettingZoo

## microgrid_env.py (PettingZoo Parallel)

- Wraps N instances of `home_agent_env`
- Implements `ParallelEnv` API: `step(actions: dict)` → `obs, rewards, terms, truncs, infos`
- Each step: collect all actions → run `energy_market.clear()` → distribute rewards
- `possible_agents`: `["home_0", "home_1", ..., "home_{N-1}"]`
- N is configurable via `config.yaml → env.num_agents`
