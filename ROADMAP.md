# Roadmap: "Pick up toys and put them in the basket"

Goal: Pancha autonomously collects scattered toys around the house and places them in a basket.

This is a multi-phase integration project (RL skills + computer vision + navigation/orchestration),
not a single trainable "skill". Phases are ordered by dependency; each is independently useful.

## Phase 1 — Grasp & lift (RL, sim) — DONE in sim (trained + video-reviewed)

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

**Video reviewed and passed** ([recording](https://github.com/user-attachments/assets/65d89d42-52fc-4d8a-b67f-e86c868edf5d)):
stand -> deep crouch with the beak to the floor -> latch -> smooth rise -> stand holding the
block at beak tip, feet planted throughout. Two observations from watching it:

- The block is held **against** the beak tip, not enclosed in it. With no jaw servo that is
  the correct sim outcome (the weld latches where the beak contacts), but "can the beak grip
  vs merely nudge" stays a real-hardware question — see Phase 6.
- The grasp fires around phase 0.27, i.e. during the DESCENT, not in the dedicated hold
  window (0.375-0.45). The latch is not phase-gated by design, so the policy grabs as soon as
  it legitimately can. The gate works, but the hold segment earns less than the phase profile
  assumed and could be shortened in a revision.

## Phase 2 — Carry while walking (RL, sim) — TRAINED, but the policy does NOT walk

### Result of the first full run (4000 iters, 4096 envs, ~2h50 on l4x1)

**The mechanism works; the reward does not.** The toy is carried perfectly — but the robot
marches in place instead of walking, so Phase 2 is NOT done.

Rollout eval of `model_3999` (`scripts/carry_eval.py`, CPU) under a constant 0.3 m/s forward
command with pushes disabled, 6 s:

| | |
|---|---|
| expected displacement | 1.80 m |
| **actual displacement** | **0.072 m** |
| achieved forward velocity | 0.013 m/s (**4 % of commanded**) |

Command-response curve — it responds to a faster command by lifting its feet more, not by
translating:

| cmd vx (m/s) | achieved (m/s) | ratio | fraction of time feet airborne |
|---|---|---|---|
| 0.00 | 0.006 | — | 0.134 |
| 0.10 | 0.006 | 0.06 | 0.134 |
| 0.20 | 0.007 | 0.03 | 0.134 |
| 0.30 | 0.016 | 0.05 | 0.384 |
| 0.40 | 0.028 | 0.07 | 0.731 |

### Why: the tracking Gaussian cannot tell walking from standing

`track_linear_velocity` uses `std = sqrt(0.1) = 0.316 m/s` against a command range of only
+/-0.4 m/s. The std is nearly as large as the whole command range, so the Gaussian barely
discriminates:

| tracking error | reward (weight 2.0) |
|---|---|
| 0.00 m/s (perfect) | 2.000 |
| 0.25 m/s (**stationary**) | **1.071** |

The policy scored 1.228 at iteration 3999 — so **standing still was worth 87 % of what it
actually earned**. Meanwhile `air_time` (weight 3.0) is the largest positive term and pays for
lifting feet whenever a non-zero command exists, with no requirement to translate. Marching in
place is therefore very close to optimal, and with a payload making real walking harder
(CoM coupling), the degenerate gait wins.

This is the AGENTS.md lesson exactly: *"Tracking Gaussian std: ~ the error you still care
about, not the max error — too loose has no gradient at small errors"*, plus *"RL optimizes the
letter of the reward"*. The escapability test that AGENTS.md demands before tightening a std is
passed here: unlike head oscillation (inherent), translating IS escapable — walking is the task.

**Note this is inherited from the velocity recipe, not introduced by Phase 2.** The same std and
`air_time` weight train a walking policy without a payload; the payload appears to have tipped a
marginal trade-off into the degenerate basin. Whether the base velocity task shows the same
weakness has NOT been tested and is worth checking.

### Suggested fix for run 2 (not yet applied)

1. Tighten `track_linear_velocity` std to ~0.10-0.15 m/s, so a stationary robot scores 0.12
   rather than 1.07 out of 2.0.
2. Gate `air_time` on actual progress, so lifting feet without translating pays nothing —
   AGENTS.md: encode the maneuver in hard state-based gates, not in penalty nudges.
3. Compare reward MASS, not weights: `air_time` at 3.0 currently outweighs the entire
   tracking signal the policy can realistically capture.

### What DID work (mechanism validated, keep it)

- **Carrying is physically real.** The toy stays within **1.5-1.8 mm** of its weld pose through
  350 steps of a real gait with pushes.
- **Robustness is fine at the trained push rate**: 0 falls in 350 steps for every checkpoint.
  An earlier 100 %-fall reading was an artifact of the inherited PLAY config pushing every
  0.5-1.0 s versus 3.0-6.0 s in training — a 6x rate the policy never saw.
- Every penalty term stayed <= 0 for the entire run (the AGENTS.md infallible check).
- **Measured grip force for Phase 6**: 0.35 N mean, **~4-5 N peak** during gait, against a
  0.29 N static hold for the 30 g toy.
- Payload motion stayed low: `carried_toy_accel` ~2.2 m/s^2, so the
  `PAYLOAD_FREE_ACCEL = 30 m/s^2` guard rail was correctly inert (`payload_violence` -0.0006)
  and never taxed the gait.

### Implementation notes

Combine the held object from Phase 1 with the walking gait (`Mjlab-Velocity`). Balance changes
with an off-center held mass swinging near the head during locomotion.

Implemented as the `Mjlab-Carry-Flat-MicroDuck` task in `microduck_rl` (+ a `-Backlash-` twin),
built on `make_microduck_velocity_env_cfg`.

**It is a perfect-grip upper bound, by design.** A MuJoCo `mjEQ_WELD` does not break, so with
the toy welded from step 0 and no latch, it *cannot be dropped* — a 100 % carry rate is
guaranteed by construction rather than earned, and the task is really "walk with 10–80 g of
extra beak mass". That isolates the difficulty the handoff flagged (CoM coupling) and avoids
inventing a beak grip-strength threshold, which is unknowable until hardware (Phase 6).
Instead `carried_toy_grip_force` logs the force the weld actually supplies, so a measured grip
strength can later be compared against a real distribution. A breakable-grip variant is then a
small additive change.

- **Carry pose, measured** (`scripts/carry_sweep.py`, CPU): in the head frame +X is world up
  and −Z is world forward; `mouth_tip` sits at (−8.09, 0, −77.74) mm. Sweeping the toy down
  from there, interpenetration with the head collision mesh reaches zero at **25 mm** below the
  beak tip (24 mm still overlaps 0.35 mm; centring it on `mouth_tip` overlaps 12.5 mm). That is
  the carry analogue of where Phase 1's latch fired, since Phase 1 welded on *contact*.
- **CoM coupling, measured at that pose**: 0.9 mm at 10 g, 2.7 mm at 30 g, **6.7 mm at 80 g
  (14.5 % of the 46 mm sole)** — *gentler* than Phase 1's 8.2 mm, because the toy hangs closer
  to the body than wherever the latch happened to catch it. The static shift is not the hard
  part; walking makes it dynamic.
- **The head-tracking lesson needed no new work.** The velocity recipe already implements
  AGENTS.md's fix (`head_pose_bias_penalty`: L1 on a 1 s EMA, curriculum-ramped), which prices
  the escapable DC droop and lets the unavoidable oscillation cancel. Building on velocity
  inherits it. Do *not* fix a droop symptom by tightening `head_pose_tracking`.
- **A new trap, found and guarded.** mjlab fires `reset` events *before* the `forward()` at the
  end of `step`, and the head is not the root body — so its `xpos` during a reset event is
  still the **previous episode's**. Placing the toy off an unrefreshed head pose spawns it
  metres from the beak while `eq_active`, the held flag, NaN guards and rewards all still look
  healthy. Measured: **0.00006 mm** placement error with the kinematics refresh, **1836 mm**
  without. This is the reset-side mirror of the Phase 1 sensor-staleness trap.
- Validated on CPU: toy welded at the beak from step 0, held to <0.1 mm through 40 steps of
  random actions, 61 D actor obs preserved, `eq_data` expanded per-world, all rewards finite,
  NaN-free. A 5-iteration / 64-env HF Jobs smoke test completes with every penalty <= 0.
- **First measured payload numbers** (smoke test, untrained policy, so a baseline rather than a
  result): `carried_toy_accel` ~10.3 m/s^2 mean, `carried_toy_grip_force` 0.42-0.52 N against a
  0.38 N static weight. The `PAYLOAD_FREE_ACCEL` = 30 m/s^2 guard rail therefore sits ~3x above
  current payload motion and is nearly inert (`payload_violence` = -0.0001), which is what a
  backstop should be — re-set it from the trained distribution.
- 17 new tests, **17/17 mutation-checked**. Mutation testing found two real holes: the toy-mass
  DR test derived its expectation from the same constant it was checking (tautological), and an
  `isinstance` check on the twist command was near-vacuous because `GroundPickPhaseCommandCfg`
  *subclasses* `UniformVelocityCommandCfg`.
- **The first smoke test caught a third bug the tests had missed**: both payload metrics were
  dead, always reading zero. mjlab computes rewards *before* metrics, and all three payload
  terms differenced one shared previous-velocity buffer, so the penalty consumed it and the
  metrics measured `vel - vel`. `carried_toy_grip_force` read exactly (mean toy mass) x g —
  static weight, dynamic term missing, and it looked like a healthy number. Fixed with a
  per-step shared cache; the suite is now 18 tests, 18/18 mutations caught.

Still open: run 2 with the reward fix above. A play-video review is NOT the gate yet — the
headless eval already shows the policy does not translate, so there is nothing to review.

### Handoff notes (written while building Phase 1 — kept for context)

**The twist command slot collides.** Phase 1 encodes the pick phase in the twist slot as
`[cos(2*pi*phi), sin(2*pi*phi), 0]` (`GroundPickPhaseCommand`), but Phase 2 needs that same
slot for real velocity commands. The 61 D obs contract is shared across the whole policy
family and slots cannot be added, so **Phase 2 cannot reuse Phase 1's phase machinery**.

The design that falls out of that:

- **Spawn the toy already held.** Activate the weld at reset (`eq_active` on from step 0) and
  skip the latch entirely — Phase 2 is about carrying, not acquiring. `mdp.reset_grasp`
  already releases and clears state; the mirror-image "attach at reset" is a few lines using
  the same `eq_data` maths as `update_grasp_latch`.
- **Base on `make_microduck_velocity_env_cfg`, not ground_pick.** Phase 2 is the walking
  recipe plus a payload, so it should inherit the gait rewards, not the phased pick stack.
- **The real difficulty is the CoM coupling.** The payload sweep measured a static 8.2 mm CoM
  shift at 80 g (18 % of the sole); walking makes that a *dynamic* disturbance on a head that
  is already 38 % of body mass and must oscillate. `AGENTS.md` has a directly relevant lesson:
  a tight instantaneous head-tracking std once taxed walking so hard the policy stood still —
  price only the escapable part (e.g. L1 on a 1 s EMA).
- Keep `mdp.update_grasp_latch` registered but consider raising its gates, or omit it, so the
  policy cannot re-grab a dropped toy mid-episode and mask a failure to carry.
  *(Resolved: omitted. And moot in the other direction — a weld cannot break, so there is no
  drop to mask.)*

## Phase 3 — Release on command (RL, sim) — depends on Phase 1

Small episodic skill: open beak, dip head, drop the object. Similar scale/difficulty to
`standup`/`ground_pick`.

### Handoff notes

`mdp.release_grasp(env, env_ids)` already exists and is the counterpart to the Phase 1 latch —
it deactivates the weld and clears the held flag. Phase 3 is therefore mostly reward design
plus a command to trigger it, not new mechanism work. Note the beak cannot open (no jaw
servo), so "release" is deactivating the weld at a chosen moment; the reward has to make the
policy dip and aim first, since dropping from a standing pose scatters the toy.

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

- [x] Phase 1 — Grasp & lift (#1) — trained & video-reviewed, 100% grasp+lift in sim
- [ ] Phase 2 — Carry while walking (#2) — trained; carry mechanism works, but the policy marches in place instead of walking (reward fix needed)
- [ ] Phase 3 — Release on command (#3)
- [ ] Phase 4 — Perception (#4)
- [ ] Phase 5 — Navigation / orchestration (#5)
- [ ] Phase 6 — Real hardware validation (#6)
