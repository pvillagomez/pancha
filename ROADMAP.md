# Roadmap: "Pick up toys and put them in the basket"

Goal: Pancha autonomously collects scattered toys around the house and places them in a basket.

This is a multi-phase integration project (RL skills + computer vision + navigation/orchestration),
not a single trainable "skill". Phases are ordered by dependency; each is independently useful.

## Phase 1 — Grasp & lift (RL, sim) — TRAINED, sim-complete

Crouch, close the beak on a small light object, lift head back to standing while holding it.

Implemented as the `Mjlab-GraspLift-Flat-MicroDuck` task in `microduck_rl`, trained to
**100 % grasp-and-lift** in sim (64/64 eval episodes, mean lift 21.4 cm to beak height).

- **Graspable object**: `robot/microduck/toy.xml` — a 30x30x24 mm, 30 g block. A box, not
  a ball: a rolling prop escapes the beak on first touch, which turns "grasp" into
  "chase". Spawns at x = 80 +/- 20 mm in the robot's yaw frame.
- **The grasp is a real weld.** The Microduck has NO jaw servo (its 14 joints are 5+5
  leg and 4 neck/head, and the beak geoms are rigidly fixed to the head), so "closing
  the beak" is not an action. The grasp is a latched `mjEQ_WELD` equality between the
  head and the toy, installed inactive at compile time and switched on per-env when the
  beak actually closes on the toy: real contact + mouth within 35 mm + mouth pointing
  down + low relative speed. The speed gate is what stops a fast head slam from
  counting as a grasp.
- **Payload limits, measured** (`scripts/payload_sweep.py`, CPU, no GPU needed):
  servo torque is *not* the binding constraint — even a 200 g payload needs only 26 %
  of the XL330's stall torque at the neck. What binds is BALANCE: an 80 g toy drags the
  whole-body CoM 8.2 mm forward, 18 % of the 46 mm sole the robot must keep its weight
  over. Training therefore randomizes the toy over 10–80 g rather than jumping to
  dog-toy masses.
- Validated on CPU end-to-end: the weld holds the toy to within ~1 mm, releases cleanly,
  `eq_data` expands per-world, the 61 D obs contract is preserved, every penalty term is
  <= 0, and a 5-iteration / 64-env HF Jobs smoke test completes.

### Training result (3000 iters, 4096 envs, ~1h50 on l4x1, first attempt)

The grasp emerged around iteration 200 and went 0 % -> 84 % in ~150 iterations, then held
94-98 % for the rest of the run. No reward-hacking iterations were needed.

| | iter 0 | iter 354 | iter 2132 |
|---|---|---|---|
| grasp rate | 0 % | 84 % | 95 % |
| falls (`fell_over`) | 12.12 | 0.29 | 0.08 |
| `dof_pos_limits` | -0.000 | -0.146 | -0.018 |
| `head_impact_penalty` | -0.0005 | -0.043 | -0.008 |

Two failure modes appeared mid-run and resolved themselves as the policy refined: joints
parking near their limits (peaked at iter ~350, then fell 8x) and the beak scuffing the
floor on descent (peaked at iter ~354, then fell 5x). Neither needed the `AGENTS.md`
limit-proximity fix.

Rollout eval of `model_2999` (64 envs, play cfg with pushes enabled): **64/64 episodes
grasped and lifted the toy >5 cm**, mean lift 21.4 cm (median 21.4, max 24.0) — i.e. the
toy goes from the floor to standing beak height. Artifacts:
[checkpoints + `exported/policy.onnx`](https://huggingface.co/Pablooooooooo/mjlab-grasplift-flat-microduck-20260828-225507).

Still open: nobody has *watched* a rollout. Sim metrics can pass while the video fails the
human eye, so a play-video review is the remaining gate before calling the skill good.

## Phase 2 — Carry while walking (RL, sim) — depends on Phase 1

Combine the held object from Phase 1 with the walking gait (`Mjlab-Velocity`). Balance changes
with an off-center held mass swinging near the head during locomotion.

## Phase 3 — Release on command (RL, sim) — depends on Phase 1

Small episodic skill: open beak, dip head, drop the object. Similar scale/difficulty to
`standup`/`ground_pick`.

## Phase 4 — Perception (NOT RL — computer vision) — independent, can start anytime

Detect toys and the basket from Pancha's onboard camera. None of the existing RL policies see
images at all (observations are pure proprioception + commands) — this needs a separate object-
detection model, likely running on the RK3566's AI accelerator, or offloaded. Toy variety
(colors/shapes) makes this the hardest non-RL piece.

## Phase 5 — Navigation / orchestration (classical robotics, not RL) — depends on Phases 2 & 4

A controller that takes "toy at bearing X, distance Y" from Phase 4 and issues velocity commands
to the Phase 2 walk-and-carry policy: go to toy → grasp → go to basket → release → repeat.
State machine handling failures (dropped toy, lost toy, no path found).

## Phase 6 — Real hardware validation — depends on all above, needs Pancha (ships ~Dec 2026)

Sim2real validation for everything trained above: payload weight limits, confirming the beak can
actually grip (vs. just nudge) real toys, balance under real domain randomization.

## Suggested near-term milestone

Before attempting full autonomy, target a **gamepad-assisted** version first: aim Pancha at a toy
manually and press a button for grasp/carry/release (Phases 1-3 only), skipping perception/
navigation (Phases 4-5) entirely. Achievable well before December; full autonomy is a follow-on
project.

## Status

- [x] Phase 1 — Grasp & lift (#1) — trained, 100% grasp+lift in sim; pending video review
- [ ] Phase 2 — Carry while walking (#2)
- [ ] Phase 3 — Release on command (#3)
- [ ] Phase 4 — Perception (#4)
- [ ] Phase 5 — Navigation / orchestration (#5)
- [ ] Phase 6 — Real hardware validation (#6)
