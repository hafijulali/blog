---
title: "Agenty World: AI Multi-agent RPG style game"
date: 2026-06-20
draft: false
description: "A technical walkthrough of Agenty World, a Three.js and TypeScript RPG-style simulation where AI agents talk, form bonds, reshape the world, and evolve over time."
tags: ["ai", "game-dev", "typescript", "threejs", "multi-agent"]
---

Agenty World is an experiment in turning an AI agent system into a playable RPG-style world.

Instead of treating agents as a chat window, the project puts them inside a 3D cyber-fantasy city. You can walk around, talk to NPCs, switch into a simulation mode, follow agents, watch their relationships change, and let an LLM act as a story engine behind the scenes.

The source code is built around a simple idea: keep the game loop deterministic and fast, then let AI influence the narrative layer asynchronously.

## Stack

The project uses a compact browser-game stack:

- **TypeScript + Vite** for the frontend app.
- **Three.js** for rendering the world, agents, lighting, post-processing, and camera.
- **Node HTTP server** for story and simulation APIs.
- **Codex exec, OpenAI-compatible API, or local fallback runtime** for AI story generation.
- **Local storage snapshots** for simulation save/resume.

The main application starts in `src/main.ts`, which creates a `Game` instance, wires the blocker menu, checks LLM health, and exposes three player paths:

- Explore mode
- Simulation mode
- Resume previous simulation

That split is important. Explore mode behaves like a first-person RPG. Simulation mode behaves more like a living ant-farm of AI agents.

## Runtime shape

`src/game/Game.ts` is the orchestration layer. It owns the renderer, world, player, NPC agents, dialogue UI, HUD, simulation UI, and simulation engine.

At startup, the game can spawn 10 initial personas from `src/agents/personas.ts`. Each persona has an identity, role, traits, color palette, greeting, and a `lifeCredo` generated from the simulation rules.

The loop is a standard `requestAnimationFrame` loop with a capped delta:

```ts
const dt = Math.min(this.clock.getDelta(), 0.05);
```

That cap matters because simulation and rendering share a frame loop. A browser stall should not create a huge physics or AI update jump.

The loop branches by mode:

- **Explore:** update player movement, scan nearby agents, open dialogue on interaction, animate visible agents, update HUD.
- **Simulation:** update follow camera, run `SimulationEngine.update(dt)`, refresh simulation stats, update HUD.

## Game-loop flow

The project includes a full game-loop diagram in `docs/gameloop-flow.md`. The rendered version shows the complete path from boot to frame loop, simulation tick, server calls, callbacks, world updates, UI, and final render.

![Agenty World game-loop flow](/images/agenty-world-gameloop-flow.png)

The most useful design decision here is the separation between high-frequency and low-frequency work:

- Rendering and movement run near 60 FPS.
- Neighbor scans and social bond updates are throttled by frame counters.
- HUD refresh runs on a shorter timer.
- AI simulation ticks run asynchronously about every 10 seconds in the game configuration.

This keeps the experience responsive even when AI calls are slower than a frame.

## Agents as game entities

`src/game/NPCAgent.ts` turns a persona into a Three.js object. An agent is not just a name in a list. It has:

- A voxel-style body mesh.
- An aura ring.
- A floating holo panel.
- Simulation state such as energy, mood, generation, relationships, and life-rule labels.
- Render LOD state: `full`, `simple`, or `far`.

The holo panel is a good UI trick. It makes simulation state visible inside the world instead of hiding everything in a dashboard. The player can see mood, energy, relationships, and life state directly above the character.

The rendering budget is explicit in `src/game/renderBudget.ts`:

- Pixel ratio capped at `1.5`.
- Full holo detail only within 28 units.
- Simplified body rendering until 48 units.
- Distant agents become `far`.
- Built structures and decorative skyline pieces are culled by distance.

This is the kind of practical constraint a simulation game needs early. Multi-agent systems tend to grow object counts quickly.

## The AI story layer

Explore mode uses `src/agents/StoryEngine.ts`.

The local player can choose preset actions like:

- Take action
- Dig deeper
- Broker a deal
- Write a custom move

`StoryEngine` sends those actions through `fetchStoryWeave`, which calls the server endpoint `/api/story/weave`. If the remote runtime fails, it falls back to a local generator. The result is converted back into a `StoryNode` with narrative text and next choices.

This gives the game an RPG dialogue loop without hardcoding every branch.

The important part is that the LLM does not own the whole game. It only returns structured narrative output. The client still owns state, UI, rendering, and world rules.

## Simulation mode

`src/simulation/SimulationEngine.ts` is the heart of the multi-agent system.

Each simulated agent tracks:

- Energy
- Mood
- Age
- Generation
- Neighbor count
- Life verdict
- Relationships
- Personality memory
- Last life event

The simulation has two layers:

1. **Frame-level behavior:** movement, neighbor refresh, bond tracking, holo-panel updates.
2. **Tick-level behavior:** life pressure, deaths, pair selection, LLM dialogue, reproduction, world changes, partner matching.

The tick loop is intentionally slower. In `Game.startSimulation()`, the engine is created with a 10-second tick interval:

```ts
this.simulation = new SimulationEngine(callbacks, 10, this.world.collision);
```

That gives the game enough time to animate between decisions while still letting the world evolve.

## Conway-inspired life pressure

The simulation borrows from cellular-automata thinking. Agents evaluate their local neighborhood and get a life verdict such as underpopulated, stable, crowded, or risky.

On each tick, the engine:

- Refreshes neighbors.
- Applies life pressure to energy and mood.
- Ages agents.
- Applies death risk from life verdict, energy collapse, or old age.
- Decays distant bonds.
- Selects pairs for conversation.
- Applies AI or local tick effects.
- Queues births and processes reproduction.
- Emits callbacks to the game.

This makes the simulation feel less like random chat and more like a living system with pressure, scarcity, proximity, and consequence.

## AI as a judgment layer

The code does not ask the LLM to simulate every frame. It asks the LLM to judge important moments.

`src/simulation/SimulationLLM.ts` calls `/api/sim/tick` for a selected pair. The server returns structured effects:

- Dialogue
- Outcome
- Mood delta
- Energy delta
- World mood
- Life event
- Optional world modifications
- Optional death or reproduction signal

Then `SimulationEngine.applyTickEffects()` applies those results to the real game state.

This is the right boundary for an AI game system. The model contributes interpretation and narrative; deterministic code applies constraints.

## Server-side runtime adapters

The server in `server/index.ts` exposes these main routes:

- `GET /api/story/health`
- `GET /api/sim/health`
- `POST /api/story/weave`
- `POST /api/sim/tick`
- `POST /api/sim/partner`
- `POST /api/sim/director`
- `POST /api/dialogue/log`

The runtime is selected through config and environment variables in `server/config.ts`.

There are three runtime styles:

- **Codex runtime:** `server/codex/CodexExecClient.ts` shells out to `codex exec` with `--ephemeral`, `--json`, and `--output-last-message`.
- **OpenAI-compatible runtime:** uses chat completions with JSON-object responses.
- **Local runtime:** template fallback for story weaving and local simulation decisions.

The Codex path is interesting because it treats an agent CLI as a structured backend runtime. The server wraps it behind the same interface as the OpenAI-compatible runtime, so the game code does not care which implementation produced the story.

## Director prompts

Simulation mode also has a player-director interface. The UI offers example commands such as:

- `Alex Rivera kills Priya Sharma`
- `create building at 1,1`
- `charity drive in the plaza`
- `wedding for Sofia Reyes and Marcus Chen`

`src/simulation/playerDirective.ts` first tries to parse commands locally. Simple directives like killing an agent or building a structure can be applied without a model call. Ambiguous commands are escalated to `/api/sim/director`.

This hybrid approach is useful:

- Fast, predictable commands stay local.
- Creative or ambiguous commands go to the LLM.
- The engine still validates the result before changing state.

## The world can change

The world is not only scenery. `src/game/World.ts` owns static environment pieces and dynamic modifications. Agents and director commands can create:

- Buildings
- Roads
- Power nodes
- Power lines
- Plazas
- Lights

`src/game/worldModGovernance.ts` keeps that under control with caps and probabilities:

- Max agent-built structures: 12
- Max mods per tick: 1
- Low chance of automatic world modification per simulation tick

That prevents the LLM from flooding the map with geometry.

## Save and resume

Simulation state is persisted with `src/game/gameSave.ts`.

The save snapshot includes:

- Agent personas and positions
- Sim state
- World modifications
- Conversations
- Pair bonds
- Assigned partners
- Relationship metadata
- Motion state
- Pending reproduction
- Current follow target

This is more than a simple checkpoint. It captures enough social state to resume the simulation without flattening agents back to generic NPCs.

## Why this architecture works

Agenty World works because it does not confuse AI with the entire game engine.

The stable parts are deterministic:

- Rendering
- Movement
- Collision
- Culling
- Save/load
- State mutation
- Population constraints

The AI parts are isolated:

- Dialogue weaving
- Pair judgment
- Director interpretation
- Narrative events
- Optional world modifications

That separation keeps the project debuggable. When something goes wrong, you can ask whether the issue is in the render loop, simulation rules, prompt output, parser, or state application.

## What I would improve next

The next technical step would be stronger observability for simulation decisions:

- A timeline inspector for every tick.
- A debug panel showing why a pair was selected.
- Visual overlays for neighbor counts and life verdicts.
- Replayable simulation seeds.
- Tests around parsers for LLM responses and player directives.

The game already has the right foundation: a deterministic loop, structured server endpoints, runtime adapters, and an explicit simulation state model. The next challenge is making emergent AI behavior inspectable enough to tune.

## Takeaway

Agenty World is a good example of how to build AI into a game without handing the whole game to the model.

The model creates narrative pressure. The engine owns reality.

That is the pattern I would reuse for any AI RPG: deterministic world, structured AI decisions, bounded side effects, and a game loop that never waits on the model to draw the next frame.
