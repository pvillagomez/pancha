# Pancha 🦆

Project home for **Pancha**, my [Pollen Robotics MicroDuck](https://pollen-robotics.com/microduck/) — a ~25cm, ~800g open-source bipedal RL robot. Pre-ordered Aug 2026, expected delivery before Christmas 2026.

## Repos

- [`microduck`](https://github.com/pollen-robotics/microduck) — the robot's runtime (Rust daemons, on-device control loop).
- [`microduck_rl`](microduck_rl) — RL training environments (submodule, forked to [pvillagomez/microduck_rl](https://github.com/pvillagomez/microduck_rl) for custom skill development).

## Goal

Train Pancha a custom skill: autonomously find toys scattered around the house and put them in a toy basket. See [ROADMAP.md](ROADMAP.md) for the phased plan — this is a multi-stage project (grasp/carry RL skills + vision + navigation), not a single training run.

## Training infra

- Local dev: `microduck_rl` syncs and runs its CPU-only tests fine on macOS (no GPU needed for code/tests).
- Real training: [Hugging Face Jobs](https://huggingface.co/docs/huggingface_hub/guides/jobs) (`--hf-jobs` flag, default flavor `l4x1`, ~$0.80/hr). Azure was considered but the current subscription (Visual Studio Enterprise) has zero GPU quota and a pending quota request that may be denied (Dev/Test subscriptions are frequently restricted).
