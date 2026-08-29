# AGENTS.md

Project home for **Pancha**, a Pollen Robotics MicroDuck. This repo tracks the roadmap and (via
the `microduck_rl` submodule, forked to `pvillagomez/microduck_rl`) custom RL skill development.
Pancha hasn't shipped yet (ordered Aug 2026, expected before Dec 2026) — everything here is
sim-only until then.

## Repo map

- `README.md` — project overview.
- `ROADMAP.md` — the 6-phase plan for the "collect toys, put them in the basket" goal. Each phase
  is also a GitHub Issue (#1-#6) linked to the "Toy pickup & basket delivery" milestone — keep both
  in sync when phase status changes.
- `microduck_rl/` — git submodule, forked from `pollen-robotics/microduck_rl` (`origin` = fork,
  `upstream` = original). **Read `microduck_rl/AGENTS.md` before writing any RL task/reward code**
  — it has extensive, hard-won conventions (obs layout, reward sign conventions, DR rules, etc.)
  that this file does not repeat.

## Training commands (microduck_rl)

Local dev (macOS, CPU-only, no GPU in this environment):
```bash
cd microduck_rl
uv sync
uv run list-envs
uv run --with pytest pytest tests/
```

Real training runs on Hugging Face Jobs (no local/Azure GPU available — Azure quota request is
pending/likely denied, VS Enterprise subscriptions are frequently restricted for GPU quota):
```bash
uv run python -m mjlab_microduck.train_cli <TASK_ID> \
  --env.scene.num-envs <N> --agent.max_iterations <N> \
  --agent.logger tensorboard --no-wandb --hf-jobs --namespace Pablooooooooo
```

Two gotchas discovered while setting this up:
- **Use `python -m mjlab_microduck.train_cli`, not the bare `train` command** — the `mjlab`
  dependency's own console script shadows this project's `train` entry point in this environment.
- **Both `--no-wandb` AND `--agent.logger tensorboard` are required** to avoid wandb entirely.
  `--no-wandb` only stops the *submission* script from requiring local wandb credentials;
  `--agent.logger tensorboard` stops the *trainer itself* from calling `wandb.init()` remotely.
  Passing only one fails (either locally before submission, or remotely after billing).

Always run a 5-iteration/64-env smoke test before a full run (~a few cents on `l4x1`, the default
HF Jobs flavor, ~$0.80/hr). Full runs cost roughly $2-5 depending on task complexity; the 12h job
timeout caps worst-case cost at ~$9.60.

## Current status

See the "Toy pickup & basket delivery" milestone: https://github.com/pvillagomez/pancha/milestone/1
