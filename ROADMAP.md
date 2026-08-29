# Roadmap: "Pick up toys and put them in the basket"

Goal: Pancha autonomously collects scattered toys around the house and places them in a basket.

This is a multi-phase integration project (RL skills + computer vision + navigation/orchestration),
not a single trainable "skill". Phases are ordered by dependency; each is independently useful.

## Phase 1 — Grasp & lift (RL, sim) — NOT YET STARTED

Crouch, close the beak on a small light object, lift head back to standing while holding it.

- Needs a new graspable object entity in the sim scene (like `ball_kick`'s ball), spawned near
  `mouth_tip`.
- Needs a *real* grasp mechanism: today's `ground_pick`/`apply_mouth_payload_force` only applies
  a simulated payload force (10-40g) during the return phase — there is no actual attach/weld
  constraint or graspable body anywhere in `microduck_rl`. This has to be built from scratch.
- 10-40g is very light for an 800g robot — many dog toys (50-200g+) may exceed what the neck/body
  can compensate for while balancing. Real payload limits need validating in sim first.
- Estimated effort: ~1-2 weeks of reward-design iteration (per `AGENTS.md`'s "2-5 iterations of
  reward-hacking whack-a-mole" pattern).

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

- [ ] Phase 1 — Grasp & lift (#1)
- [ ] Phase 2 — Carry while walking (#2)
- [ ] Phase 3 — Release on command (#3)
- [ ] Phase 4 — Perception (#4)
- [ ] Phase 5 — Navigation / orchestration (#5)
- [ ] Phase 6 — Real hardware validation (#6)
