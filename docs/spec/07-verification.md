# Spec 07 — Verification & Success Criteria

## Test Commands

```bash
# Phase 1: environment unit tests
pytest tests/test_envs/ -v                          # Gymnasium compliance, shapes, reset/step
pytest tests/test_market/ -v                        # Double-auction clearing, price band invariant

# Phase 2: agent + full suite
pytest tests/test_agents/ -v                        # Reward computation, action selection
pytest tests/ -v --cov=env --cov=agents --cov=utils --cov=dashboard

# Smoke: SB3 training (10 episodes, must not crash, rewards must be non-NaN)
python train.py --mode sb3 --config config.yaml --episodes 10

# Evaluation across all scenarios
python evaluate.py --checkpoint models/ppo_ep010.zip --all-scenarios

# Phase 3: dashboard
docker compose up --build
# → open http://localhost:3000, Live Grid tab must show WebSocket data within 5s

# Lint + type check
ruff check . && mypy env/ agents/ utils/ dashboard/
cd frontend && npm run lint && npm run type-check && npm run build
```

## Phase-by-Phase Success Gates

| Phase | Gate | Metric |
|---|---|---|
| 1 — Env Core | `pytest tests/test_envs/ tests/test_market/ -v` all pass | 0 failures, 0 errors |
| 2 — SB3 Proto | Positive avg reward on `sunny_day` | < 500 episodes to converge |
| 2 — SB3 Proto | Blackout rate on `cloudy_peak` | < 5% after ep 1000 |
| 3 — Dashboard | Live Grid WebSocket latency | < 500ms observed in browser |
| 4 — RLlib MARL | MAPPO vs IPPO on `storm_outage` | MAPPO ≥ 15% higher avg reward |

## Behavioral Sanity Checks

After Phase 2 training, verify emergent behaviors manually using `evaluate.py`:

1. **Battery arbitrage:** agents should charge during off-peak (low price) and discharge during peak
2. **P2P preference:** P2P trade volume should increase over training (agents learn it's cheaper)
3. **EV compliance:** agents with imminent EV departure should prioritize charging over selling
4. **Cooperation under threat:** when grid load approaches 85%, agents should reduce grid buys

## MLflow / W&B Run Health Checks

A healthy training run should show:
- Episode reward: monotonically increasing trend (with noise) in first 200 episodes
- Policy loss: decreasing
- Entropy: should not collapse to 0 (agents must not become deterministic too fast)
- Blackout count per episode: should decrease as cooperation emerges
- P2P trade volume: should increase as agents learn market participation

## Anti-Patterns to Watch

| Symptom | Likely cause | Fix |
|---|---|---|
| Reward stays negative forever | `λ` (blackout penalty) too high | Reduce `reward.blackout_penalty_lambda` |
| Agents never use P2P | Price bins too close to grid price (no incentive) | Widen spread in `market.p2p_price_bins` |
| EV always depleted | `β` too low — agent ignores EV | Increase `reward.ev_penalty_beta` |
| Reward collapses mid-training | Learning rate too high | Reduce `ppo.learning_rate` |
| All agents buy from grid, no P2P | P2P not profitable — check `feed_in_tariff` vs `grid_buy_price` gap | Adjust pricing config |
