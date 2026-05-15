# Spec 08 — Parallel Development with Git Worktrees

## Overview

4 terminals run simultaneously, each in its own worktree directory on its own branch.
All worktrees share one `.git` repo. No stashing, no branch switching.
Conflicts are near-zero by design through file ownership rules.

## Pre-flight (once, on staging — before any worktrees)

Create skeleton files so every branch has a common ancestor:

```bash
# Run from main project directory on staging branch
touch config.yaml
touch requirements.txt
mkdir -p tests && touch tests/conftest.py
touch env/__init__.py agents/__init__.py utils/__init__.py tests/__init__.py
mkdir -p tests/test_utils tests/test_market tests/test_agents tests/test_envs

git add .
git commit -m "chore: project skeleton for parallel worktrees"
```

## Spin Up 4 Worktrees

```bash
# Run from main project directory (staging branch)
git worktree add ../microgrid-wt1 -b feat/utils-simulators
git worktree add ../microgrid-wt2 -b feat/market-engine
git worktree add ../microgrid-wt3 -b feat/reward-model
git worktree add ../microgrid-wt4 -b feat/config-infra
```

Each terminal opens its own directory:
```
Terminal 1 → cd ../microgrid-wt1    # feat/utils-simulators
Terminal 2 → cd ../microgrid-wt2    # feat/market-engine
Terminal 3 → cd ../microgrid-wt3    # feat/reward-model
Terminal 4 → cd ../microgrid-wt4    # feat/config-infra
```

## File Ownership (strict — no two terminals touch the same file)

| Terminal | Branch | Owns exclusively |
|---|---|---|
| T1 | `feat/utils-simulators` | `utils/solar_simulator.py`, `utils/demand_simulator.py`, `tests/test_utils/` |
| T2 | `feat/market-engine` | `utils/pricing_engine.py`, `env/energy_market.py`, `tests/test_market/` |
| T3 | `feat/reward-model` | `agents/reward_model.py`, `tests/test_agents/test_reward_model.py` |
| T4 | `feat/config-infra` | `config.yaml`, `scenarios/*.yaml`, `tests/conftest.py`, `requirements.txt` |

**Rule:** if you need a value from `config.yaml` during development, hardcode a test default
locally — T4 will provide the real config. Do not modify `config.yaml` from T1/T2/T3.

## Merge Order (run from main worktree, on staging)

```bash
# 1. Merge T4 first — config and fixtures needed by others
git merge feat/config-infra --no-ff -m "feat: config, scenarios, test fixtures"

# 2. Merge T1, T2, T3 in any order — no deps between them
git merge feat/utils-simulators --no-ff -m "feat: solar and demand simulators"
git merge feat/market-engine --no-ff -m "feat: pricing engine and P2P market"
git merge feat/reward-model --no-ff -m "feat: reward model"

# 3. Run Phase 1 gate
pytest tests/ -v -q

# 4. Clean up worktrees
git worktree remove ../microgrid-wt1
git worktree remove ../microgrid-wt2
git worktree remove ../microgrid-wt3
git worktree remove ../microgrid-wt4
git branch -d feat/utils-simulators feat/market-engine feat/reward-model feat/config-infra
```

## Phase 2 Worktrees (after Phase 1 gate passes)

New wave of 4 worktrees branched from the now-merged staging:

```bash
git worktree add ../microgrid-wt1 -b feat/microgrid-env        # microgrid_env.py (PettingZoo)
git worktree add ../microgrid-wt2 -b feat/sb3-agent            # home_agent.py SB3 wrapper
git worktree add ../microgrid-wt3 -b feat/training-loop        # train.py + evaluate.py
git worktree add ../microgrid-wt4 -b feat/scenarios-eval       # scenario YAML + evaluate tests
```

## Why Conflicts Won't Arise

- **Pre-flight skeleton** → every branch has a common ancestor for shared files
- **File ownership** → no two branches ever modify the same file
- **T4 sole ownership** → `config.yaml` and `conftest.py` only ever modified by one branch
- **`__init__.py` additions** → git auto-merges non-overlapping line additions cleanly

## List Active Worktrees

```bash
git worktree list
```
