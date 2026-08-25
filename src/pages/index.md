---
layout: ../layouts/BlogPost.astro
title: "RaceLab: a research harness for generated racing environments"
description: "Submission for GI's infinite world generation harness challenge."
author: "Farhan Baig"
date: "Aug 2026 · 10–15 min read"
---

As a candidate for the GI Research Tech challenge we present RaceLab. RaceLab is an agent harness which seeks to answer two questions: how well can an agent generate a game environment from a natural-language specification, and how well can another agent play it? The generator agent writes a typed racing-track plan, local code compiles and certifies it, and player policies run on it in the game engine.

We limited the harness to a racing game largely because its behavior is easy to verify by eye, while the domain still offers wide variation and a natural path to 3D. We could have used a more complex engine or visual stack (i.e. there are many impressive examples of Claude/Codex generated Three.js environments on Twitter) but we wanted to stay within the constraints of a commonly available model, without hours-long sessions and without burning millions of tokens. Also, despite the visual impressiveness, many of those demos lack playability and physics, which is hard to reliabily implement for brand new assets without an intense harness.

We treat **environment generation and gameplay as separate harnesses with separate information boundaries**. The creator sees the world request and fidelity reports while the player sees only what it needs to race. We prevent leakage so that the player agent results are meaningful. We evaluated the harness in both 2D and 3D to check its portability across different input types (granted the underlying engine was more or less the same).

If you are quickly reading through this we would like to draw your attention to the challenges section,  experimental meta harness section, research workflow, and limitations section.

![A generated RaceLab circuit in the 2D renderer, including barriers, sector gates, and two cars.](/trask-writeup/images/racelab-2d.png)

## Generated circuit gallery

These three examples were authored by the live environment agent, then compiled and certified by the same deterministic racing engine used for experiments.

### Compact asphalt technical circuit

> **Prompt used:** “Make a compact 2D asphalt technical circuit with several linked turns, continuous edge barriers, and exactly two aggressive opponents.”

![Top-down render of the generated compact asphalt technical circuit, showing its linked turns, continuous barriers, and two opponents.](/trask-writeup/images/generated-circuit-technical.png)

### Two-lap rolling asphalt circle

> **Prompt used:** “Make a 3D circular asphalt circuit with no corners, two laps, gentle rolling elevation, and visible edge barriers.”

![Top-down render of the generated rolling circular circuit.](/trask-writeup/images/generated-circuit-rolling-circle.png)

*Oblique 3D engine render, included so the gentle rolling elevation is visible rather than inferred from the map.*

![Oblique 3D engine render of the generated rolling circular circuit.](/trask-writeup/images/generated-circuit-rolling-circle-3d.png)

### High-elevation square asphalt circuit

> **Prompt used:** “Make a 3D square high-grip asphalt circuit with four clean 90-degree corners, long sides, high elevation differentials, continuous edge barriers, and one racer opponent.”

![Top-down render of the generated high-elevation square circuit.](/trask-writeup/images/generated-circuit-elevated-square.png)

*Oblique 3D engine render, included so the elevation changes and edge barriers are directly visible.*

![Oblique 3D engine render of the generated high-elevation square circuit.](/trask-writeup/images/generated-circuit-elevated-square-3d.png)

## Introduction

A researcher gives the coordinator a brief such as:

> “A slippery, curvy track with a 90-degree bend in the top right and three aggressive opponents.”

The environment harness turns that request into a constrained `TrackPlan`: corners, surface, grip, width, laps, barriers, and opponent temperaments, but no raw game engine map. A deterministic "compiler" then closes and validates the loop, reports fidelity, and requires a reference driver to finish before accepting the scene.

The player harness then allows an agent to play through this track. Because a normal model (i.e. our execution context) cannot control the car at tick rate, it selects or produces longer-horizon strategies that local code executes at 10 Hz.

In short, the harness provides:

- **An authoring boundary.** Models express intent through a stable game grammar and local code owns geometry and physics.
- **Fidelity accounting.** Supported requests are honored while unsupported requests are noted and told to user.
- **Deterministic certification.** Fixed seeds, fixed probes, and the same authoritative runtime make comparisons replayable.
- **Separated agent roles.** Environment-generation context never leaks into the player’s observation stream.
- **Scalability.** Runs are isolated artifacts that can later move to cloud storage, containers, or scheduled clusters without changing the experiment contract.

The rest of this post covers the harness design and observed results.

---

## 1. Challenges

### Improvements must be cheap but noticeable

Our intended evaluation asks three questions:

- Does the harness produce environments that satisfy more of the original specification?
- Does the harness produce player behavior that is more successful or controllable?
- Is the harness token efficient?

Harnesses should not overaxaggerate the necessary token spend for an underlying task. It would be easy to construct a very complicated harness with many layers of token calls and cross verification. However, due to both the time and budget limitations of this challenge, as well as a desire to adhere to traditional token efficient harnesses, we kept the architecture small but powerful.

### Generated worlds cannot exceed the engine

“Infinite world generation” is still bounded by the engine. Unless the environment is a learned world model, it is very difficult for the agent to add mechanics or assets the engine does not support, especially without blowing up the token budget.

We make that boundary explicit. The harness attempts supported edge cases and explicitly states if certain portions of the request are unsupported such as jumps or damage. It can thus grade the compiled scene against the original request. Otherwise, producing any valid circuit could be mistaken for prompt faithful generation.

### Language models do not operate at tick speed

A racing policy must act quickly. Model decode for Codex, Claude, and other general purpose LLMs is slower than the control loop and latency varies so repeated tick-level prompts waste context and are slow. Lower call frequency then requires intermediate control which can be bridged by local microcontrollers.

We considered several control patterns:

1. a model call on every tick
2. model calls only at detected events, with a microcontroller between them
3. evenly spaced model calls, with a microcontroller between them
4. overlapping the current microcontroller window with the next model call
5. overlapping and selecting from existing control skills, with the model choosing a skill from its interpretation of the scene.

We started by moving the model up the control hierarchy: a slow planner chooses a maneuver while fast local code handles intermediate controls and safety. This follows the aforementioend planner/controller split which notably is also used in autonomous driving, robotics, and game agents that emit reusable skills.

Note that for these overlapped cases we still face a large problem which is when we overlap the microcontroller with the next model call, we still only have the current state to feed into the model call. The current primary path which solves this is **predictive skill selection**. While one reusable closed-loop skill drives, the model predicts the coarse visual state expected when its response arrives and selects the next skill for that activation state. We compared it with predictive script generation, event-triggered script generation, fixed-interval script generation, and a model call on every simulation tick.

Note here that the predictive skill selection differs from predictive script generation in that skill selection chooses an existing skill from our harness based on its prediction of the next state, while script generation writes its own new microcontroller given its prediction of the next state.

We gave the 2D agent a forward cone view and the 3D agent a first-person (FPV) view. This preserves the current-direction visual evidence a general game-playing agent would plausibly receive, without granting a map or top-down route.

![The exact player-facing observation contracts: a 2D forward cone and a 3D first-person camera.](/trask-writeup/images/player-camera-contracts.png)

*The player sees only the left-hand forward cone in 2D or the right-hand in-car camera in 3D. Top-down circuit and trajectory views in this post are evaluator visualizations, not player inputs.*

Our first matched pilot used one certified 2D circuit, `gpt-5.6-luna`, an 800-tick limit, and the same contract in every arm: forward-cone RGB plus speed, with no map, pose, heading, progress, checkpoint, centerline, or privileged rollout.

**A quick note on timing:** a tick is one discrete update of the game simulation, not one second of real time. Simulated ticks describe how much in-game control occurred, while wall time measures how long the run actually took on the clock, including model inference and harness overhead. A method can therefore use fewer ticks but still take longer to run.

| Player method | Completed | Sim. ticks | Model calls | Total tokens | Wall time |
| --- | :---: | ---: | ---: | ---: | ---: |
| Predictive + skill library | Yes | 438 | 6 | 5,254 | 23.7 s |
| Predictive + custom scripts | No | 800 | 8 | 15,589 | 61.7 s |
| Event-triggered custom scripts | No | 800 | 28 | 132,148 | 74.6 s |
| Interval custom scripts | No | 800 | 8 | 15,585 | 56.7 s |
| Model call every tick | Yes | 245 | 245 | 210,890 | 426.5 s |

![Token usage, model calls, simulated time, wall time, and completion for five player-control methods on one matched 2D circuit.](/trask-writeup/images/player-method-comparison.png)

*Figure: Predictive skill selection completed with about 40 times fewer calls and tokens.*

![Top-down trajectories for the same five matched player-control runs.](/trask-writeup/images/player-method-trajectories.png)

*Figure: Evaluator-only top-down paths from the same pilot.*

The charts show one exploratory run per method. However success rates of the predictive skill approach was the highest across different tracks at 100% across 24 trials with event triggered custom scripts, interval custom scripts, and model call on each tick being at 0%, 8.33%, and 12.5%. To be fair, model call every tick is quite effective but far too slow.

#### Experimental meta-harness for multi lap races

We also tested adaptation within one five-lap race. During a 500-tick exploration window, the harness recorded permitted image features, speed, and the visible outcome of each chosen skill. A later visual match could replay that skill locally without another API call. Memory never received lap, pose, heading, checkpoint, centerline, or evaluator measurements. Through this we can show that if we allow longer playthroughts agents can selfreinforce the harness and play better with less model reliance. Racing is an especially difficult domain to include a metaharness since slight track variations like speed, car weight, tire grip, track curve, etc can necessitate vastly different actions. However this experimental meta harness as well as the success of the skill library in predictive skill based stategy is quite promising.

In one exploratory `gpt-5.6-luna` run, the car finished five laps in 5,532 ticks using 24 calls and 45,834 tokens. The first two laps were model-led but laps three through five required no new calls.

| Lap | Sim. ticks | Model calls | Local skill replays | Evaluator on-track |
| --- | ---: | ---: | ---: | ---: |
| 1 | 1,993 | 19 | 5 | 58.2% |
| 2 | 1,817 | 5 | 5 | 71.3% |
| 3 | 619 | 0 | 4 | 91.8% |
| 4 | 541 | 0 | 4 | 88.0% |
| 5 | 562 | 0 | 5 | 94.3% |

![Evaluator-only trajectory, lap pace, and the transfer from API calls to locally replayed skills in the five-lap adaptation run.](/trask-writeup/images/player-multilap-meta.png)

*Figure: The policy finished all five laps. The late-lap speedup coincided with zero new model calls.*

Note though that this is not yet general circuit memory. Single screenshots can confuse similar track sections since again slight variations are huge in racing so the next version should match short camera-and-action histories or let the model choose among a small retrieved set.

#### 3D player comparison

We repeated the comparison on one compact elevated circuit using `gpt-5.6-luna`, an 800-tick limit, and first-person RGB plus speed. The player received no map, pose, heading, progress, checkpoint, centerline, elevation telemetry, or privileged rollout.

For 3D we collapsed the event based and tick based custom script methods into just tick based and called it blocking generated controllers. Overlapped stale controllers are controllers which do not predict the state of the world after the current microcontroller actions complete so we expect worse performance.

| 3D player method | Completed | Sim. ticks | Model calls | Total tokens | Wall time |
| --- | :---: | ---: | ---: | ---: | ---: |
| Blocking generated controllers | No | 800 | 8 | 15,329 | 60.2 s |
| Overlapped stale controllers | No | 800 | 8 | 16,354 | 64.5 s |
| Predictive + custom scripts | Yes | 541 | 6 | 12,086 | 44.8 s |
| Predictive + skill library | Yes | 484 | 6 | 5,706 | 25.4 s |

![Token usage, model calls, simulated time, wall time, and completion for four player-control methods on one matched 3D circuit.](/trask-writeup/images/player-method-comparison-3d.png)

*Figure: Prediction with skills is still the most efficient in 3D.*

![Top-down evaluator trajectories for the same four matched 3D player-control runs.](/trask-writeup/images/player-method-trajectories-3d.png)

*Figure: The player acted from first-person frames, while the top-down paths were reconstructed only for evaluation.*

## 2. Why a racing game?

Racing has a clear goal but very ambiguous intermediate actions which makes control an interesting problem. Several lines through a corner can work, and a locally fast move can hurt the next turn. This makes racing harder to reduce to a short script, even compared to open world games like Minecraft.

We also considered a fighting game, but its animation, hit logic, move system, and opponents would shift too much work toward the engine. Racing let us build 2D and 3D versions with shared rules to test our harness with different inputs.

![The same racing domain rendered from a 3D driver-facing camera.](/trask-writeup/images/racelab-3d-driver-view.png)

The 3D engine is not a second game. It subclasses the 2D world and adds road grade and bank while checkpoints, laps, collision, traffic, nitro, and replay remain shared. On flat ground, tests require tick-for-tick agreement with 2D.

![An overhead 3D view of a banked circuit, showing the shared track geometry from another camera.](/trask-writeup/images/racelab-3d-overhead.png)

### Other domains we considered

Card games are another interesting option because a model can act at native turn cadence. But that tests discrete strategy rather than the gap between slow planning and continuous control which makes the player agent side of the harness less challenging. Also this is less faithful to the infinite world generation challenge.

## 3. Harness design

### Two harnesses

RaceLab has two agents with deliberately separate jobs:

| Environment generation | Player control |
| --- | --- |
| Reads the user’s world specification | Reads game state |
| Authors a typed track plan | Produces or selects driving behavior |
| Can inspect compiler and fidelity feedback | Cannot inspect creator reasoning or evaluator state |
| Searches over scene parameters | Acts inside one fixed, certified scene |
| Optimizes specification satisfaction | Optimizes race behavior |

### Context

We keep reusable context in small Markdown cards grouped by role. The harness checks the selected cards before each run, prevents environment-only information from entering the player prompt, and records the final context card pack. Most of the player prompt stays fixed. Only the current image, speed, active skill, and recent controls change between calls.

The two agents also use memory differently. The player may receive up to four lessons from repeated, visually confirmed failures. The environment agent queries SQLite for a small number of prior examples that match the current requirement type. This keeps context focused and prevents an unrelated precedent from influencing a run.

### Environment generation

The creator translates the researcher’s request into `track-grammar-v1`, a typed plan covering track geometry, surface, grip, barriers, laps, and opponents.

![Flow of the environment-generation harness from a researcher brief to a certified scene, including measured repair feedback.](/trask-writeup/images/environment-harness-flow.svg)

*Figure: The model translates intent into a typed plan. Deterministic code turns this into a renderable scene.*

Local code handles coordinates and validates the circuit and finally runs a deterministic reference driver before accepting it. A `TrackReport` records how closely the result matched the request and any compromises made along the way.

We tested nine prompts/briefs across three seeds. Every approach used `claude-haiku-4-5-20251001` and the same compiler and grader. One-shot produced one plan, self-judge ranked four of its own plans, and the harness used simulator feedback to revise and search its candidates.

| Generation method | All requirements met | Mean requirements met | Model calls / env. | Tokens / env. | Wall time / env. | Search ticks / env. |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Non-harnessed one-shot | 29.6% | 76.0% | 1.04 | 2,664 | 7.37 s | 0 |
| Non-harnessed self-judge | 33.3% | 76.9% | 4.00 | 10,723 | 29.77 s | 0 |
| Harnessed generation | 74.1% | 93.2% | 2.85 | 7,855 | 28.87 s | 4,763 |

![Environment-generation fidelity by brief type for harnessed generation, one-shot generation, and a self-judging compute control.](/trask-writeup/images/environment-generation-quality.png)

*Figure: 81 trials: nine briefs × three seeds × three arms.*

![Model calls, token usage, wall time, and deterministic simulation cost for the three environment-generation methods.](/trask-writeup/images/environment-generation-cost.png)

*Figure: The harness exchanges cheap deterministic simulation for model calls.*

### Player control

The player receives camera frames and speed, but no map, centerline, target heading, creator reasoning, or evaluator state. Speed is the only non-visual signal exposed directly from the game engine; we included it because performance was very poor when the player had to infer speed from images alone. The model selects longer-horizon skills while a deterministic local loop handles tick-level control.

![Flow of the predictive player harness, where a model call overlaps the real-time skill loop.](/trask-writeup/images/player-harness-flow.svg)

*Figure: While the active skill drives from fresh allowed observations, the model predicts the state at activation time and prepares the next skill.*

![Two player trajectories under matched conditions with different aggression settings. The conservative policy stays on track more often but times out; the aggressive policy completes with a riskier line.](/trask-writeup/images/player-aggression-ab.png)

Here the game, seed, and observations stay fixed while only player aggression changes.

### Drawing for environment creation

The Draw view lets a researcher sketch a centerline when language is too ambiguous. Drawing is an especially promising approach for more complex scenes in more complex engines. Even for the case of adding assets to a scene, a draw tool augmented with edge tracking/intermediate shape tracking could help decompose the desired asset into a more easibly interpretable form than via natural language.

## 4. Research workflow

### Graphical workflow

The browser interface follows one artifact flow across five views:

1. **Coordinator.** A researcher describes a 2D or 3D circuit or references a sketch. The coordinator generates and certifies the track.
2. **Draw.** A normalized canvas captures a reusable closed centerline. The normal compiler still closes, smooths, and certifies it.
3. **Environments.** The library shows each scene as a 2D plan or orbitable 3D view, alongside scene information.
4. **Experiments.** Each circuit has a persistent chat for requesting drivers, seeds, perturbations, or behavior settings. Large matrices stream progress and ETA.
5. **Runs.** Results are grouped as experiment → run → fork. Entries report status, ticks, calls, tokens, behavior, and seed. Researchers can scrub replays and fork from a selected tick.

### Why graphical

**A terminal is enough to launch a matrix, but the main research objects are spatial and temporal.** Researchers need to compare tracks with requirements, orbit 3D surfaces, scrub and fork replays, draw routes, and see which controller owned each tick interval.

The browser adds persistent conversations, progress, and most importantly visual information.

## 5. Limitations + Future Work

RaceLab currently is quite a focused prototype, not a general game-generation benchmark. It covers just a racing domain, so we cannot guarantee a similar approach will transfer. However, in designing the harness we wanted to ensure that it could theoretically expand. This is part of the reason we decided to use both a 2D and 3D racing environment.  

Regardless of the engine we believe by translating natural language into a strong typed engine specific spec, we can generalize these results. There is also room in other engines for more rigorous verification. For example, our engine is completely built from the ground up, and so physics and interactions are less explicitly defined as they would be in a more fleshed out counterpart. This would theoretically make the "compiler" approach even stronger and averse to failure.

The main next step would be to test the harness with a more flexible engine, such as Unity, and across a wider range of environments.

We also theorize that a smaller model trained through self-play (i.e. Qwen 6B) could improve both environment generation as well as play. There is precedent for using such RL techniques to make agents particularly good at using tools in specialized contexts.

## Acknowledgements and references

Part of the inspiration for RaceLab’s use of model-authored skills was influenced by prior work on code-generating embodied agents. Additionally, there were just some papers and references we found interesting and slightly relevant to this project which are attached below.

- Wang et al., [“Voyager: An Open-Ended Embodied Agent with Large Language Models”](https://arxiv.org/abs/2305.16291), 2023.
- Liang et al., [“Code as Policies: Language Model Programs for Embodied Control”](https://arxiv.org/abs/2209.07753), 2022.
- Naveed, Qiao, and Dolan, [“Trajectory Planning for Autonomous Vehicles Using Hierarchical Reinforcement Learning”](https://arxiv.org/abs/2011.04752), 2020.
- Waddle Team, [“Introducing Waddle: Agents that Control Robots”](https://www.waddlelabs.ai/research/introducing-waddle), 2026.
- [Pi](https://github.com/badlogic/pi-mono), an open-source agent toolkit and coding-agent harness.
- Prime Intellect, [Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent), an open-source harness for coding.
- [OpenCode](https://github.com/anomalyco/opencode), an open-source coding agent.

---
