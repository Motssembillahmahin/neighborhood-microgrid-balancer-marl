# Spec 03 — Reward Design & P2P Market

## Reward Design

### Home Agent Reward (per timestep)

```python
R_home = (
    - cost_bought_from_grid               # $/kWh × kWh bought — minimize electricity bill
    + revenue_sold_to_grid                # feed-in tariff × kWh sold — earn from excess solar
    + p2p_trade_profit                    # (p2p_price - cost_basis) × kWh traded — P2P is better than grid
    - β * ev_departure_penalty            # penalty if EV below min_charge at departure_time
    - λ * max(0, total_demand - grid_cap) # shared blackout penalty — ALL agents receive this
)
```

### Shared Blackout Penalty

```python
# In reward_model.py — applied to EVERY agent in the same timestep
blackout_shortfall = max(0, sum(agent.grid_demand) - grid_capacity)
shared_penalty = λ * blackout_shortfall   # λ from config.yaml → reward.blackout_penalty_lambda

# Economic rationale: if any agent over-consumes relative to grid capacity,
# everyone pays. Selfish agents discover cooperation reduces collective risk.
```

### EV Departure Penalty

```python
# Applied only at the timestep matching ev_departure_time
if timestep == ev_departure_timestep and ev_charge < ev_min_charge_pct:
    penalty = β * (ev_min_charge_pct - ev_charge)   # β from config → reward.ev_penalty_beta
# Rationale: agents must balance profit-seeking with mobility obligations.
```

### Grid Coordinator Reward

```python
R_coord = (
    - total_neighborhood_cost             # sum of all home agent costs this timestep
    - α * blackout_risk_score             # α from config → reward.coordinator_alpha
    + grid_stability_bonus                # bonus when load stays below safe threshold (e.g. 70%)
)
# Coordinator optimizes neighborhood, not individual homes.
# Issues price signals (not direct control) to preserve decentralized execution.
```

### Tunable Coefficients (config.yaml)

```yaml
reward:
  blackout_penalty_lambda: 0.5   # shared penalty weight — increase to force more cooperation
  ev_penalty_beta: 0.3           # EV constraint weight
  coordinator_alpha: 0.8         # coordinator blackout risk weight
```

**Always comment reward term changes with economic rationale.**

---

## P2P Energy Market — Double Auction (energy_market.py)

### Mechanism

Runs once per timestep after all agents have acted. Steps:

1. **Collect orders**: agents on action 4/4a-e → ask (seller); action 5/5a-e → bid (buyer)
2. **Sort**: asks ascending by price; bids descending by price
3. **Find clearing price P***: walk both lists; clearing price = price where `cumulative_supply ≥ cumulative_demand`
4. **Execute trades**: all matched pairs trade at P*
5. **Grid fallback**: unmatched demand falls back to utility grid at `grid_buy_price`
6. **Compute P2P profits**: sellers earn `P* × kWh`; buyers save `(grid_buy_price - P*) × kWh`

### Price Band Invariant

```python
# Enforced in energy_market.py — tested in tests/test_market/
assert feed_in_tariff < clearing_price < grid_buy_price
# Sellers always earn more than feed-in.
# Buyers always pay less than grid price.
# Violation → raise MarketInvariantError.
```

### Example Clearing

```
Asks (sellers, ascending):  H3@$0.08/kWh 2.0kWh, H1@$0.10/kWh 1.5kWh
Bids (buyers, descending):  H2@$0.14/kWh 1.5kWh, H5@$0.12/kWh 2.0kWh

Clearing: 3.5kWh supply vs 3.5kWh demand → P* = $0.11/kWh
H4 residual 1.0kWh → falls back to grid @ $0.18/kWh

Outcome:
  H3: +$0.22  (vs feed-in $0.10 → +$0.12 better)
  H1: +$0.165 (vs feed-in $0.075 → +$0.09 better)
  H2: -$0.165 (vs grid $0.27 → saves $0.105)
  H5: -$0.22  (vs grid $0.36 → saves $0.14)
```

### Key Interface

```python
class EnergyMarket:
    def clear(self, orders: list[Order]) -> MarketResult:
        # Returns: trades executed, clearing_price, grid_fallback_volume
        ...

@dataclass
class Order:
    agent_id: str
    side: Literal["ask", "bid"]
    quantity_kwh: float
    price: float           # explicit bid price from agent

@dataclass
class MarketResult:
    clearing_price: float
    trades: list[Trade]
    grid_fallback: dict[str, float]  # agent_id → kWh bought from grid
```
