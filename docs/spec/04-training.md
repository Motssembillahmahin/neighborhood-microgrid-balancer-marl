# Spec 04 — Training Architecture

## Two-Phase Approach

### Phase 1 — SB3 IPPO (Prototype)

**Goal:** fast iteration on reward shaping and environment debugging before investing in full MARL.

```bash
python train.py --mode sb3 --config config.yaml
```

- 5 independent SB3 PPO agents, one per home
- `Discrete(14)` action space
- Each agent trains on its own `home_agent_env` instance
- Shared blackout penalty still applies via `microgrid_env`
- No shared critic — each agent is fully independent
- Logs to MLflow or W&B (configurable)
- **Success gate:** positive avg reward on `sunny_day` within 500 episodes

### Phase 2 — RLlib MAPPO (Full MARL)

**Goal:** proper cooperative MARL with CTDE.

```bash
python train.py --mode rllib --config config.yaml
```

- All 5 home agents + coordinator trained via RLlib
- `Parameterized` action space (Discrete(6) type + Box price)
- **CTDE:** shared critic sees concatenated state of all N agents during training
- Decentralized execution: each agent's policy sees only its own 7-dim obs at inference
- Coordinator trains separately as a hierarchical agent observing 35-dim state (5 × 7)
- **Success gate:** blackout rate < 5% on `cloudy_peak` within 1000 episodes

## MAPPO Architecture (CTDE)

```
Training time:
┌─────────────────────────────────────────────┐
│  Shared Critic (centralized)                │
│  Input: [obs_h1, obs_h2, ..., obs_hN] = 35-dim  │
│  Output: V(s) — estimates joint value       │
└─────────────────────────────────────────────┘
         ↑ gradient flows during training only

Home Agent Policies (decentralized):
  policy_h1(obs_h1) → action_h1   # 7-dim obs only
  policy_h2(obs_h2) → action_h2
  ...

Inference time:
  Each home agent uses only its own policy + its own obs.
  Coordinator issues price signal from its own policy.
  No shared critic needed.
```

## Coordinator Agent

- Observes: concatenated state of all N homes (35-dim for N=5)
- Acts: continuous price signal → fed into each home agent's `grid_price` observation
- Trains: hierarchical RL (RLlib supports this via multi-agent config)
- **Does not directly control homes** — only nudges via price signals

## config.yaml Structure

```yaml
env:
  num_agents: 5
  episode_length: 48          # 48 × 30-min = 24 hours
  battery_capacity_kwh: 10.0
  ev_min_charge_pct: 0.8

training:
  mode: sb3                   # sb3 | rllib
  total_episodes: 2000
  checkpoint_every: 50
  tracker: mlflow             # mlflow | wandb

ppo:
  learning_rate: 3.0e-4
  n_steps: 2048
  batch_size: 256
  gamma: 0.99
  gae_lambda: 0.95
  clip_range: 0.2
  n_epochs: 10

reward:
  blackout_penalty_lambda: 0.5
  ev_penalty_beta: 0.3
  coordinator_alpha: 0.8

market:
  p2p_price_bins: [0.06, 0.08, 0.10, 0.12, 0.14]  # Phase 1 discrete bins
  feed_in_tariff: 0.05
  grid_buy_price: 0.18

pricing:
  peak_price: 0.18            # 6–9pm
  off_peak_price: 0.06        # midnight–6am
  shoulder_price: 0.10
```

## Experiment Tracking

All runs logged via `MLflow` or `W&B` (set `training.tracker` in config.yaml):

- Per-episode: avg reward, policy loss, value loss, entropy, blackout count, P2P trade volume
- Per-agent: individual reward, bill saved, action distribution histogram
- Checkpoints saved as `models/ppo_ep{N}.zip` every `checkpoint_every` episodes
