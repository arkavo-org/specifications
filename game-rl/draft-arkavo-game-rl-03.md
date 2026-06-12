# Game-RL Protocol Specification

**Version:** 3.0.0-draft03
**Status:** Draft
**Created:** 2026-06-12
**Supersedes:** 2.0.0-draft02 (draft-02)
**Authors:** Arkavo AI
**License:** Apache 2.0

---

## Abstract

The Game-RL Protocol defines a standard interface for multi-agent AI systems to observe and interact with game environments, functioning either as **embodied characters** (APCs/NPCs) or **systemic controllers** (Game Masters/Directors). Built on the Model Context Protocol (MCP), it enables researchers and developers to:

- Train reinforcement learning agents in existing games
- Drive games with LLM agents — from frontier models to small on-device models
- Orchestrate multiple AI agents with heterogeneous capabilities
- Build AI-native games with LLM-powered NPCs, game masters, and world simulators
- Ensure deterministic reproducibility for scientific research

This specification is implementation-agnostic and supports any game engine or runtime that can expose the required interface.

draft-02 codifies the protocol dialect proven in released implementations (game-rl for RimWorld and Project Zomboid; consumed by Arkavo Edge) and incorporates lessons from extended agent playtesting — most significantly a redesigned **Spatial Intent** model (Section 7) that preserves learning-signal integrity, and a **loud-error requirement** (Section 3.4) that eliminates silent action failures.

draft-03 promotes four conventions now field-validated across two adapters — RimWorld (systemic) and Project Zomboid (embodied) — establishing them as proven rather than aspirational: a textual `RenderMap` viewport (§5.9), structured construction and one-shot actions (§7.8), partial-success feedback (§3.4, REQ-ERR-03), and an embodied-vs-systemic Reference Point clarification (§7.4, REQ-SPA-03).

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture](#2-architecture)
3. [Protocol Foundation](#3-protocol-foundation)
4. [Agent Model](#4-agent-model)
5. [Core Tools](#5-core-tools)
6. [Observation Contract](#6-observation-contract)
7. [Spatial Intents](#7-spatial-intents)
8. [Rewards & Episodes](#8-rewards--episodes)
9. [Determinism & Reproducibility](#9-determinism--reproducibility)
10. [Multi-Agent Coordination](#10-multi-agent-coordination)
11. [Vision Streams](#11-vision-streams)
12. [Input Robustness](#12-input-robustness)
13. [Platform Adaptation](#13-platform-adaptation)
14. [Conformance Requirements](#14-conformance-requirements)
15. [Security Considerations](#15-security-considerations)
16. [References](#16-references)

Appendices: [A. Schemas](#appendix-a-schema-definitions) · [B. Migration from draft-00](#appendix-b-migration-from-draft-00) · [C. Change Log](#appendix-c-change-log) · [D. IPC Bridge Protocol (Informative)](#appendix-d-ipc-bridge-protocol-informative)

---

## 1. Introduction

### 1.1 Motivation

Modern AI research increasingly requires complex, interactive environments that go beyond traditional simulations. Commercial games offer rich, well-tested worlds with emergent behaviors, but lack standardized interfaces for AI integration. Meanwhile, game developers seek to incorporate AI agents but face fragmented tooling and no clear architectural patterns.

The Game-RL Protocol addresses these challenges by defining:

- A **transport-agnostic protocol** for game-AI communication
- A **multi-agent model** supporting heterogeneous agent types
- **Observation and action contracts** that preserve learning signal integrity
- **Determinism guarantees** required for reproducible research
- **Spatial intent semantics** that let agents act spatially without coordinate micromanagement — and without the environment silently overriding agent decisions
- **Vision stream specifications** for high-performance pixel observations

### 1.2 What Changed Since draft-00 (Design Lessons)

draft-02 is informed by two released game adapters (RimWorld, Project Zomboid), a shipping Swift implementation (GameOfMods), and hundreds of hours of LLM-agent playtesting through Arkavo Edge. Three lessons drove the largest changes:

1. **Small models need stable identifiers.** Agents backed by small local models (≤ 8B parameters) frequently corrupted snake_case identifiers (`agent_id` → `agentId` → `agent-id` drift), breaking tool calls. PascalCase payload fields and camelCase tool names proved materially more reliable, and match the .NET conventions of the game runtimes most adapters patch. draft-02 makes this dialect canonical (Section 1.5).

2. **Silent failures destroy both learning and trust.** Playtesting found over 160 action paths that logged a warning and returned success-shaped responses. Agents (RL and LLM alike) cannot recover from failures they cannot observe. draft-02 makes loud errors a Level 1 conformance requirement (REQ-ERR-01).

3. **Over-simplified spatial actions create degenerate policies.** An earlier simplification reduced spatial expression to a single `Near` anchor, with resolution defaults that referenced the map center. The observable result: agents placed *everything in the center of the map*, because (a) the only universally-known anchor was `MapCenter`, (b) ambiguous anchors resolved to the instance nearest the map center, and (c) infeasible placements silently relocated to the colony center, corrupting credit assignment. draft-02 replaces this with Spatial Intents v2 (Section 7): a richer anchor grammar, mandatory landmark discovery, and strict resolution-integrity rules.

### 1.3 Scope

This specification covers:

- Protocol messages and their semantics
- Agent registration and lifecycle
- Observation, action, reward, and spatial-intent schemas
- Synchronization requirements
- Platform adaptation guidelines
- A runnable conformance suite

This specification does not cover:

- Specific game implementations
- AI/ML algorithms or architectures
- Network topology for distributed training
- Authentication and authorization (deferred to future versions)

### 1.4 Terminology

| Term | Definition |
|------|------------|
| **Agent** | An autonomous system that perceives game state and influences it via the Game-RL protocol |
| **Embodied Agent** | An agent controlling a specific entity (Avatar/APC) subject to spatial and simulation constraints (physics, line-of-sight) |
| **Systemic Agent** | An agent controlling environmental or meta-game systems (Game Master), not bound by spatial constraints but by administrative permissions |
| **Avatar** | The in-game entity (mesh/physics body) controlled by an Embodied Agent |
| **Environment** | The game runtime exposing the Game-RL interface |
| **Step** | A single observe-act-reward cycle |
| **Episode** | A sequence of steps from reset to terminal state |
| **Tick** | The smallest discrete time unit in the game simulation |
| **Observation** | State information provided to an agent |
| **Action** | A command issued by an agent to the environment |
| **Reward** | A scalar signal indicating action quality |
| **Spatial Intent** | A coordinate-free action expressing *what* to place/do and *near what*, resolved by the environment |
| **Anchor** | The spatial reference an intent is resolved against |
| **Landmark** | A named, stable spatial reference exposed in observations for use as an anchor |
| **Diegetic** | Within the game world; subject to simulation rules |
| **Non-Diegetic** | Outside the game world; operating on meta-game systems |

### 1.5 Naming Conventions (Normative)

This section is normative and is the single largest dialect change from draft-00.

**REQ-NAME-01:** Tool names MUST be **camelCase**: `registerAgent`, `step`, `observe`, `reset`, `stateHash`, `episodeSummary`, `configureStreams`, `manifest`, `resolveSpatial`, `deregisterAgent`, `saveTrajectory`, `loadTrajectory`, `batchStep`, `sendMessage`.

**REQ-NAME-02:** All payload field names — tool arguments and tool results — MUST be **PascalCase**: `AgentId`, `Action`, `Ticks`, `Observation`, `Reward`, `Done`, `StateHash`, `TotalReward`.

**REQ-NAME-03:** Action type names and their parameters MUST be PascalCase: `{"Type": "SetWorkPriority", "ColonistId": "Lizzie", "WorkType": "Construction", "Priority": 1}`.

**REQ-NAME-04:** MCP protocol-level fields keep MCP's own conventions (`protocolVersion`, `inputSchema`, `serverInfo`) — this specification does not restyle the MCP layer. JSON Schema keywords inside `inputSchema` use standard lowercase JSON Schema spelling (`type`, `properties`, `description`).

**Rationale (informative):**

- *Small-model reliability.* Tokenizers split snake_case identifiers at underscores; in playtesting, small local models drifted between `agent_id`, `agentId`, and `agent-id` within a single session. PascalCase identifiers survived generation far more reliably.
- *Runtime interop.* The majority of adapted game runtimes are .NET (Harmony-patched). PascalCase on the wire eliminates a per-field translation layer in every adapter.
- *Consistency with MCP.* MCP itself is camelCase at the protocol layer; a snake_case payload dialect inside a camelCase protocol was the inconsistency, not the cure.

Implementations migrating from draft-00 names: see Appendix B. Servers MAY additionally accept draft-00 snake_case aliases during a deprecation window (Section 12).

### 1.6 Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## 2. Architecture

### 2.1 System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Processes                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │RL Agent │ │LLM Agent│ │LLM Agent│ │  Human  │               │
│  │(policy) │ │ (NPC)   │ │  (GM)   │ │(player) │               │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘               │
│       │           │           │           │                     │
└───────┼───────────┼───────────┼───────────┼─────────────────────┘
        │    MCP    │    MCP    │    MCP    │
        │  (stdio)  │  (stdio)  │  (stdio)  │
┌───────┼───────────┼───────────┼───────────┼─────────────────────┐
│       ▼           ▼           ▼           ▼                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Game-RL Protocol Server                    │    │
│  │   Agent Registry · Synchronization · Reward Routing     │    │
│  │   Input Normalization · Spatial Resolution Contract     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │ (in-process or IPC)               │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Game Runtime                          │    │
│  │  (Unity, Godot, Custom Engine, .NET/Harmony, etc.)     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                        Environment                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Communication Model

The Game-RL Protocol uses MCP (Model Context Protocol) as its foundation:

- **Transport**: stdio (stdin/stdout) for process-based agents
- **Encoding**: JSON-RPC 2.0 over newline-delimited JSON
- **Session**: One MCP connection per agent process (or per orchestrator multiplexing several agents)

Environments MAY support alternative transports (HTTP, SSE, WebSocket) but MUST support stdio for conformance.

### 2.3 Deployment Patterns

Unchanged from draft-00 in substance:

- **Pattern A — Single-Agent Training:** Agent ↔ Game over stdio.
- **Pattern B — Multi-Agent with Orchestrator:** An orchestrator (e.g., Arkavo Edge) multiplexes N agents over one connection, distinguishing them by `AgentId`.
- **Pattern C — In-Process Agents:** The game embeds a policy (ONNX etc.) directly.
- **Pattern D — Shared Session with Connector:** Desktop AI clients connect to a *running* game via a connector binary bridging stdio ↔ Named Pipe / Unix socket.

### 2.4 Clock Modes

Two clock modes determine simulation timing ownership. Carried from draft-00 with renamed payload fields.

**Training Mode (agent-owned clock):** the simulation pauses between `step` calls; `step(Ticks=N)` advances physics by exactly N ticks; determinism guaranteed; required for RL.

**Live Mode (engine-owned clock):** the game runs at its native tick rate; `step` queues the action for immediate execution; determinism not guaranteed; suited to game mastering, modding, and live NPCs.

```json
{
  "name": "registerAgent",
  "arguments": {
    "AgentId": "rl_policy:walker_1",
    "AgentType": "Entity",
    "Config": { "ClockMode": "training" }
  }
}
```

**Clock Mode Resolution:** if ANY connected agent requests `training`, the environment MUST use lockstep simulation (training wins). Only if ALL agents request `live` does the environment run real-time.

### 2.5 Connector Architecture

Carried from draft-00 unchanged in substance: a lightweight bridge binary lets MCP desktop clients attach to a running game (or spawn one headless), proxying JSON-RPC between stdio and the game's IPC endpoint. On client EOF the connector deregisters its agent and exits; the game continues for other agents.

---

## 3. Protocol Foundation

### 3.1 MCP Handshake

Agents MUST initiate the connection with the MCP initialize handshake:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "tools": {} },
    "clientInfo": { "name": "agent-identifier", "version": "1.0.0" }
  }
}
```

The environment MUST respond with its capabilities:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "tools": { "listChanged": false } },
    "serverInfo": {
      "name": "game-name",
      "version": "1.0.0",
      "gameRlVersion": "2.0.0-draft02"
    }
  }
}
```

`serverInfo.gameRlVersion` declares Game-RL Protocol compliance and dialect. Servers SHOULD accept the MCP protocol versions `2025-11-25`, `2025-06-18`, and `2025-03-26`.

### 3.2 Message Flow

All game interactions use MCP tool calls. Tool names are camelCase; payload fields are PascalCase (Section 1.5):

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "tools/call",
  "params": {
    "name": "step",
    "arguments": {
      "AgentId": "guard_1",
      "Action": { "Type": "Patrol", "Waypoint": 3 },
      "Ticks": 60
    }
  }
}
```

### 3.3 Error Codes

Errors follow JSON-RPC 2.0 conventions with Game-RL specific codes:

| Code | Name | Description |
|------|------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid request | Malformed request |
| -32601 | Method not found | Unknown tool |
| -32602 | Invalid params | Schema validation failed |
| -32603 | Internal error | Environment error |
| -32000 | Agent not registered | Unknown `AgentId` |
| -32001 | Invalid action | Action not in action space / not permitted |
| -32002 | Episode terminated | Step after `Done=true` |
| -32003 | Sync timeout | Agent missed step deadline |
| -32004 | Resource exhausted | Too many agents/streams |
| -32005 | Spatial resolution failed | Anchor unresolvable or placement infeasible (Section 7.5) |

### 3.4 Loud Errors (Normative)

**REQ-ERR-01:** An action that cannot be performed MUST surface as either (a) a JSON-RPC error, or (b) a step result whose `Error` field is non-empty. Environments MUST NOT log-and-continue: a success-shaped response for a failed action is non-conformant.

**REQ-ERR-02:** Error messages MUST identify the failing action and the reason in terms the agent can act on (e.g., `"Equip failed: WoodLog is not a weapon"` rather than `"operation failed"`). Where alternatives exist (valid targets, valid anchors), the error SHOULD enumerate at least one.

**REQ-ERR-03 — Partial-success feedback.** When an action partially succeeds (e.g. N of M requested items placed), the result MUST state the shortfall — both the achieved count N and the requested count M — and SHOULD carry the engine's own rejection reasons for the unmet portion. A partial success reported as a plain success is non-conformant: it corrupts credit assignment exactly as a silent failure does (REQ-ERR-01), because the agent attributes the full requested effect to its action. Example: `"Placed 2 of 3 Beds inside Room_1 — 'Space already occupied' x4."`

*Rationale (informative):* playtesting found 160+ adapter code paths that warned to a log file invisible to the agent and returned normally. RL agents attributed unearned rewards; LLM agents repeated the failed action indefinitely. Loud errors are as important to learning as rewards.

---

## 4. Agent Model

### 4.1 Agent Types

draft-02 canonicalizes the six role names from the released implementations. The draft-00 archetype names are retired (mapping in Appendix B).

| AgentType | Scope default | Description | Typical permissions |
|-----------|---------------|-------------|---------------------|
| `Observer` | systemic | Watch-only; receives observations, cannot act | none (read-only) |
| `Player` | embodied | Controls the player avatar | avatar movement, interaction, combat |
| `Entity` | embodied | Controls a single non-player entity (NPC, unit) | entity-local movement, interaction |
| `Controller` | systemic | Strategic control of many entities (colony/factory manager) | task assignment, priorities, zoning, production |
| `System` | systemic | Environmental systems (weather, economy, spawning) | world parameters, resource spawning |
| `Director` | systemic | Narrative/combat direction (game master) | spawn/kill entities, events, difficulty, narrative |

Implementations MAY extend these types but MUST enforce permission boundaries: an action type outside the agent's permitted set MUST produce error `-32001`.

### 4.2 Agency Scopes

Carried from draft-00: every agent operates in one of two scopes, orthogonal to its type.

- **Embodied (diegetic):** has an `AvatarId` referencing a physics body; observation limited to sensor range; actions bound by physics; mortal; cannot create entities.
- **Systemic (non-diegetic):** no physical presence; global observation; administrative actions; invisible to embodied agents.

Environments MUST validate actions against scope (e.g., `Move` from a systemic agent without a body → `-32001`; `SpawnEntity` from an embodied agent → `-32001`).

The scope-interaction model (a systemic Director spawns a potion; an embodied Entity simply perceives "a health potion nearby" with no attribution) is unchanged from draft-00 §4.1.3.

### 4.3 Registration

**Tool: `registerAgent`** — MUST be called before `step`.

```json
{
  "name": "registerAgent",
  "arguments": {
    "AgentId": "player1",
    "AgentType": "Controller",
    "Config": {
      "Scope": "systemic",
      "ClockMode": "training",
      "SessionType": "exclusive",
      "AvatarId": "npc_ranger_01",
      "SpawnPoint": "tavern_entrance",
      "ObservationProfile": "compact",
      "ActionMask": ["SetWorkPriority", "Draft", "EstablishFarm"],
      "RewardShaping": {
        "Components": ["survival", "progress"],
        "Weights": { "survival": 1.0, "progress": 0.5 }
      }
    }
  }
}
```

`AgentId` pattern: `^[a-zA-Z0-9_:-]+$`. All `Config` fields are optional; `AvatarId` is REQUIRED when `Scope` is `embodied`.

**Response (AgentManifest):**

```json
{
  "AgentId": "player1",
  "Registered": true,
  "Scope": "systemic",
  "ObservationSpace": {
    "Type": "structured",
    "Sections": ["colonists", "resources", "alerts", "landmarks", "actions"]
  },
  "ActionSpace": {
    "Type": "DiscreteParameterized",
    "Actions": [
      { "Name": "SetWorkPriority", "Params": { "ColonistId": "string", "WorkType": "string", "Priority": "integer(0-4)" } },
      { "Name": "EstablishFarm", "Params": { "Near": "anchor", "Crop": "string?", "Size": "integer?" } },
      { "Name": "Wait", "Params": {} }
    ]
  }
}
```

Registration with the same `AgentId` MUST be idempotent.

### 4.4 Lifecycle

```
INIT → registerAgent → REGISTERED → reset (first call) → ACTIVE ⇄ step
ACTIVE → Done=true or deregisterAgent → TERMINAL
TERMINAL → reset → ACTIVE (new episode)
```

Environments that attach to an already-running game session MAY report `ACTIVE` immediately after registration (live mode); training environments MUST require a `reset` before the first `step` of an episode.

**Tool: `deregisterAgent`**

```json
{ "name": "deregisterAgent", "arguments": { "AgentId": "guard_1" } }
```

---

## 5. Core Tools

The complete tool surface. Levels refer to conformance levels (Section 14).

| Tool | Level | Mutates | Purpose |
|------|-------|---------|---------|
| `manifest` | L1 | no | Environment capabilities (Section 5.1) |
| `registerAgent` | L1 | yes | Register an agent (Section 4.3) |
| `deregisterAgent` | L1 | yes | Remove an agent |
| `step` | L1 | yes | Execute action, advance time, return observation + reward |
| `observe` | L1 | no | Read state without advancing time |
| `reset` | L1 | yes | Start/restart an episode |
| `stateHash` | L2 | no | Determinism verification |
| `episodeSummary` | L2 | no | Cumulative episode metrics |
| `configureStreams` | L3 | yes | Vision stream setup (Section 11) |
| `resolveSpatial` | L3 | no | Dry-run spatial intent resolution (Section 7.6) |
| `saveTrajectory` / `loadTrajectory` | L3 | varies | Record/replay |
| `batchStep` | L3 | yes | Synchronized multi-agent stepping (Section 10.1) |
| `sendMessage` | L3 | yes | Agent-to-agent messages via environment (Section 10.2) |

### 5.1 `manifest` — Environment Capabilities

**New in draft-02.** The manifest MUST be reachable as a *tool* (not only as an MCP resource): many lightweight MCP clients and small-model agents implement only `tools/list` + `tools/call`. The `game://manifest` resource (draft-00 §6.1) becomes OPTIONAL; when present it MUST return the same content.

**Input:** `{}` (no arguments)

**Output:**

```json
{
  "Name": "RimWorld",
  "Version": "1.5",
  "GameRlVersion": "2.0.0-draft02",
  "Capabilities": {
    "MultiAgent": true,
    "MaxAgents": 16,
    "AgentTypes": ["Observer", "Player", "Entity", "Controller", "System", "Director"],
    "ClockModes": ["training", "live"],
    "SessionTypes": ["exclusive", "shared"],
    "Deterministic": true,
    "SaveReplay": false,
    "DomainRandomization": false,
    "Headless": false,
    "VariableTimestep": true,
    "SpatialIntent": true
  },
  "DefaultActionSpace": { "Type": "DiscreteParameterized", "Actions": ["..."] },
  "RewardComponents": [
    { "Name": "survival", "Description": "Per-tick colony survival", "Range": [0, 1] },
    { "Name": "colony_growth", "Description": "Population and wealth growth", "Range": [-10, 10] }
  ],
  "StreamProfiles": { "policy_fast": { "Streams": [{ "Name": "rgb", "Width": 224, "Height": 224 }] } },
  "Scenarios": [
    { "Name": "training_base", "Description": "Established colony checkpoint" },
    { "Name": "new", "Description": "Fresh colony, parameterizable" }
  ],
  "TickRate": 60,
  "MaxEpisodeTicks": null,
  "Compliance": { "Level": 2, "Version": "2.0.0-draft02", "TestResultsUrl": null }
}
```

`Capabilities.SpatialIntent: true` declares Spatial Intents support and triggers the Section 7 requirements, including the Landmarks observation contract.

### 5.2 `step` — Execute Action and Advance Simulation

The primary interaction tool.

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "AgentId": { "type": "string", "description": "Your registered agent ID" },
    "Action": {
      "oneOf": [
        { "type": "integer", "description": "Discrete action index" },
        { "type": "array", "items": { "type": "number" }, "description": "Continuous action vector" },
        { "type": "string", "const": "Wait", "description": "No-op" },
        {
          "type": "object",
          "description": "Parameterized action: PascalCase Type plus flattened parameters",
          "properties": { "Type": { "type": "string" } },
          "required": ["Type"]
        }
      ]
    },
    "Ticks": {
      "type": "integer",
      "minimum": 0,
      "default": 1,
      "description": "Simulation ticks to advance. 0 = apply action without advancing time."
    }
  },
  "required": ["Action"]
}
```

`AgentId` MAY be omitted in single-agent sessions; environments then attribute the step to the sole registered agent.

Parameterized actions flatten their parameters beside `Type` (no nested `params` object — a draft-00 change):

```json
{ "Action": { "Type": "PlaceBuildingNear", "Building": "Bed", "Near": "Stockpile", "Count": 3, "Stuff": "WoodLog" } }
```

**Output (StepResult):**

```json
{
  "AgentId": "player1",
  "StepId": 412,
  "Tick": 36000,
  "Observation": { "...": "see Section 6" },
  "Reward": 1.5,
  "RewardComponents": { "survival": 1.0, "colony_growth": 0.5 },
  "Done": false,
  "Truncated": false,
  "TerminationReason": null,
  "Events": [
    { "Type": "RaidArrived", "Tick": 35990, "Severity": 3, "Details": { "Faction": "Pirates" } }
  ],
  "Error": null,
  "Feedback": "Placed 3 Beds near Stockpile_4821",
  "FrameIds": {},
  "StateHash": "sha256:a1b2c3..."
}
```

Required fields: `Observation`, `Reward`, `Done`, `Truncated`. `Error` MUST be non-null when the action failed (REQ-ERR-01). `Feedback` is a human/LLM-readable account of what the action did; environments SHOULD populate it — it is the primary in-context learning signal for LLM agents.

**Default elision:** to conserve agent context, servers MAY omit fields whose value equals the protocol default — `Reward: 0`, `Done: false`, `Truncated: false`, empty `RewardComponents`/`Events`/`FrameIds` — from `step` and `observe` responses. Absence of an omissible field MUST be interpreted as its default value. A non-default value MUST always be present (a terminal step always carries `Done: true`; a failed action always carries `Error`).

**Step responses SHOULD be compact** (alerts, feedback, deltas). Full state belongs to `observe` (Section 6.1). This "observation diet" keeps LLM context windows viable across long episodes.

**REQ-RWD-01 (carried from draft-00):** rewards MUST be returned synchronously in the `step` response containing the causal action. Asynchronous reward delivery causes learning tearing.

### 5.3 `observe` — Read State Without Advancing Time

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "AgentId": { "type": "string" },
    "Include": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Sections to include; dot notation for sub-sections (e.g. \"entities.buildings\")"
    },
    "Limit": { "type": "integer", "minimum": 1, "description": "Max items per list section" }
  }
}
```

With no `Include`, returns a compact overview. Section semantics in Section 6. `observe` MUST NOT advance simulation time or mutate state.

### 5.4 `reset` — Start or Restart an Episode

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "Seed": { "type": "integer", "description": "RNG seed for deterministic episodes" },
    "Scenario": { "type": "string", "description": "Named scenario, checkpoint, or parameterized form" },
    "Config": {
      "type": "object",
      "properties": {
        "DomainRandomization": { "type": "object" },
        "InitialState": { "type": "object" }
      }
    }
  }
}
```

Scenario strings are environment-defined; environments SHOULD support a parameterized form for procedural setup (e.g. `"new:Crashlanded:BorealForest:Randy:Strive:250"`) and MUST list available scenarios in the manifest.

**Output:** an initial `StepResult` (with `Reward: 0`). Long-running resets (full game restarts, checkpoint loads) MAY return immediately with `{"Status": "Restarting"}`; agents then poll `observe` until ready.

### 5.5 `stateHash` — Reproducibility Check

**Input:** `{}` · **Output:**

```json
{ "Hash": "sha256:a1b2c3d4e5f6...", "Tick": 12345, "Components": { "Entities": "sha256:...", "Rng": "sha256:..." } }
```

`Hash` is REQUIRED; `Tick` and `Components` are OPTIONAL. The hash MUST be stable for identical states (REQ-DET-02).

### 5.6 `episodeSummary` — Cumulative Episode Metrics

**New in draft-02** (existed in implementations; absent from draft-00). The episode-boundary signal for trajectory consolidation and lesson synthesis: orchestrators treat `episodeSummary` calls as episode delimiters.

**Input:** `{}` · **Output:**

```json
{
  "TotalReward": 152.5,
  "StepCount": 412,
  "TicksElapsed": 360000,
  "RewardBreakdown": { "survival": 120.0, "colony_growth": 25.5, "farm_fertility_quality": 7.0 }
}
```

### 5.7 `configureStreams` — Vision Setup

```json
{ "name": "configureStreams", "arguments": { "AgentId": "player1", "Profile": "policy_fast" } }
```

Returns an array of Stream Descriptors (Section 11). Custom stream configuration via a `Custom` object MAY be supported.

### 5.8 `saveTrajectory` / `loadTrajectory`

Carried from draft-00 with renamed fields: `Path` (required), `AgentIds`, `IncludeObservations`, `IncludeFrames`, `Format` (`json` | `msgpack` | `hdf5`); load adds `PlaybackMode` (`instant` | `realtime` | `step`) and `VerifyDeterminism` (default true).

### 5.9 `RenderMap` — Textual Viewport (REQ-VIS-01, SHOULD at Level 2)

**New in draft-03.** `RenderMap` is read-only and advances no simulation time (like `observe`). It returns a plain-text ASCII viewport centered on the agent's **Reference Point** (§7.4), annotated with **absolute coordinate rulers** (axis labels along the edges) so that any coordinate read off the map is a valid action anchor or target.

```json
{
  "name": "RenderMap",
  "arguments": { "AgentId": "player1", "Radius": 12 }
}
```

```
       110   115   120   125
   145  . . . T T T . . . . .
   140  . # # B B # # . ~ ~ .
   135  . # @ . . S # . ~ ~ .   @ = agent   # = wall   B = bed
   130  . # # D # # # . . . .   S = stockpile   D = door   T = tree
   125  . . . . . . . . . . .   ~ = water   . = floor
        Legend included in result.
```

**Requirements:**

- **Reference-Point centering (MUST):** the viewport is centered on the agent's locus (§7.4) — colony centroid for a systemic Controller, avatar position for an embodied agent — not the geometric map center.
- **Coordinate rulers (MUST):** the viewport MUST carry absolute-coordinate axis labels. Coordinates and landmark Ids read from `RenderMap` MUST be accepted by the spatial anchor grammar (§7.2) when within map bounds — the textual perception channel and the action channel share one coordinate space.
- **Axis convention (MUST declare):** the viewport is north-up by default; the game MUST declare its axis convention in the legend or manifest (e.g. Project Zomboid uses screen-north = `-y`).
- **Glyphs and legend (MUST):** one glyph per cell, and the result MUST include a legend mapping each glyph. Glyph priority is game-defined but SHOULD layer mobile entities (agents, hostiles) above static structures above terrain, so a cell's most decision-relevant occupant is the one shown.

`RenderMap` is the standard **textual perception** channel for LLM agents and the cheap-perception complement to the observation diet (§5.2): it advances no time and returns a small, bounded payload, letting an agent re-perceive its surroundings between actions without paying for full structured state or a vision stream.

---

## 6. Observation Contract

### 6.1 Sections and Filtering

Structured observations are organized into named **sections**. Agents request sections via `observe(Include=[...])`; dot notation addresses sub-sections (`entities.buildings`). Environments define their own sections but MUST document them in the `observe` tool description and SHOULD follow these conventional names where applicable:

`colonists` · `resources` · `entities` (with `entities.animals`, `entities.buildings`, `entities.items`, `entities.weapons`, `entities.hostiles`) · `terrain` · `rooms` · `research` · `zones` · `threats` · `factions` · `prisoners` · `traders` · `power` · `beds` · `alerts` · `actions` · `map` · `landmarks`

`Limit` bounds list-valued sections. Environments MUST NOT impose hidden caps that cannot be raised by `Limit` (a draft-02 requirement born of the landmark-starvation problem, Section 1.2).

### 6.2 `ValidActions` (REQ-OBS-01, Level 2)

Observations MUST include (in the `actions` section or top-level `ValidActions`) the set of action types currently executable, with enough parameter hints to act:

```json
{
  "ValidActions": [
    { "Type": "Draft", "Params": { "ColonistId": ["Lizzie", "Fox"] } },
    { "Type": "EstablishFarm", "Params": { "Near": "anchor", "Crop": ["Rice", "Potatoes"] } },
    { "Type": "DefendColony", "Params": {} }
  ]
}
```

Context-dependent actions (trade actions while a trader is present, `DefendColony` during a raid) MUST appear only when valid. This is the dynamic action space; RL wrappers can index it, LLM agents can read it.

### 6.3 `Alerts` (REQ-OBS-02, Level 2)

Observations MUST include an `Alerts` array of severity-ranked, human-readable conditions needing attention:

```json
{
  "Alerts": [
    { "Severity": 3, "Label": "Colonist starving", "Subject": "Lizzie" },
    { "Severity": 2, "Label": "Low food", "Detail": "2.1 days remaining" }
  ]
}
```

`Severity`: 0 = info, 1 = low, 2 = warning, 3 = critical. Alerts are the primary attention mechanism for LLM agents and a natural reward-shaping target.

### 6.4 `Landmarks` (REQ-OBS-03, required when `SpatialIntent: true`)

See Section 7.3.

### 6.5 Partial Observability

Observation scoping carried from draft-00: embodied agents receive sensor-local observations; systemic agents receive global state. Custom masks via registration config. Environments MUST NOT leak global information into embodied observations.

---

## 7. Spatial Intents

### 7.1 Motivation: the Center-Bias Case Study (Informative)

Spatial Intents let agents express *"build a bed near the stockpile"* without coordinates. The v1 design (a bare `Near` string) failed in practice — agents placed everything at the map center, for three compounding reasons observed in the RimWorld adapter:

1. The only anchor guaranteed to exist and be known to the agent was the literal `"MapCenter"`.
2. When an anchor name matched multiple instances, the resolver picked the instance *nearest the map center*.
3. When placement near the requested anchor was infeasible, the resolver *silently relocated* to the colony center — the agent's choice was overridden with no trace in the result, so policies could never learn that their anchor mattered.

The damage was not aesthetic. Silent re-anchoring breaks the causal link between the agent's spatial decision and the reward, making spatial credit assignment unlearnable. v2 fixes the contract: richer anchors (7.2), mandatory landmark discovery (7.3), and strict resolution integrity (7.4).

### 7.2 Anchor Grammar (REQ-SPA-01)

Wherever an intent takes a `Near` parameter, environments MUST accept:

| Form | Example | Resolution |
|------|---------|------------|
| Entity ID | `"Stockpile_4821"` | That entity's position |
| Zone label | `"Kitchen stockpile"` | Zone centroid |
| Type name | `"CookStove"` | Nearest instance to the Reference Point (7.4) |
| Landmark ID | `"Region_NE"`, `"FertileCluster_0"`, `"ColonyCenter"`, `"MapCenter"` | Landmark position (7.3) |
| Offset object | `{"Anchor": "ColonyCenter", "Direction": "N", "Distance": 15}` | Resolved anchor displaced `Distance` cells toward `Direction` (8-wind compass) |
| Coordinates | `{"X": 42, "Y": 13}` | Escape hatch; literal position |

Intents accept an optional `"AllowFallback": false|true` (default **false**) — see 7.4.

### 7.3 Landmark Discovery (REQ-SPA-04)

When the manifest declares `SpatialIntent: true`, `observe(Include=["landmarks"])` MUST return a `Landmarks` array, each entry carrying a stable `Id`, a `Kind`, and a `Position`:

```json
{
  "Landmarks": [
    { "Id": "ColonyCenter", "Kind": "Centroid", "Position": { "X": 118, "Y": 140 } },
    { "Id": "MapCenter", "Kind": "Centroid", "Position": { "X": 125, "Y": 125 } },
    { "Id": "Region_NE", "Kind": "Region", "Position": { "X": 187, "Y": 187 } },
    { "Id": "FertileCluster_0", "Kind": "Cluster", "Position": { "X": 201, "Y": 174 }, "CellCount": 190, "Fertility": 1.4 },
    { "Id": "Steam_Geyser_1", "Kind": "Feature", "Position": { "X": 96, "Y": 33 } }
  ]
}
```

**Minimum landmark set (MUST):** `ColonyCenter` (or `AgentCenter` for embodied agents), `MapCenter`, and the eight compass-region centroids `Region_N`, `Region_NE`, … `Region_NW`. **SHOULD additionally include:** resource clusters, fertile-terrain clusters, water/terrain features, and named zones/buildings.

Landmark count is governed only by the agent's `Limit` parameter, ordered by relevance (size, proximity); fixed internal caps that starve agents of anchors are non-conformant.

### 7.4 Resolution Integrity (REQ-SPA-02, REQ-SPA-03)

**REQ-SPA-02 — No silent substitution.** The environment MUST NOT resolve an intent against a different anchor than requested. If placement near the requested anchor is infeasible:

- With `AllowFallback: false` (default): fail with error `-32005`, listing at least one feasible alternative anchor when any feasible placement exists (`Alternatives` array, see 7.5).
- With `AllowFallback: true`: the environment MAY relocate, and MUST then set `FallbackApplied: true` and report the anchor actually used.

**REQ-SPA-03 — Reference Point.** When an anchor form is ambiguous (type names matching multiple instances) or a search origin is needed, the environment MUST use the agent's **Reference Point**: the activity centroid of the agent's controlled entities (colony centroid for a Controller; avatar position for embodied agents). The geometric map center MUST NOT be used as an implicit reference point or fallback target.

The Reference Point is the agent's **locus**, and the locus depends on the agent's agency scope (§4.2): a systemic/Controller agent's locus is its activity centroid (the centroid of the entities it controls), whereas an embodied agent's locus is its avatar position. Every locus-relative computation — compass-region centroids, `ColonyCenter`/`AgentCenter`, nearest-instance resolution, cluster ordering, and the `RenderMap` viewport (§5.9) — MUST be taken relative to that locus, never the geometric map center. Field-validated across two adapters — RimWorld (systemic, colony centroid) and Project Zomboid (embodied, survivor position) — confirming the rule generalizes across agency scopes.

**REQ-SPA-07 — Determinism.** Identical state + identical intent → identical resolution (ties broken deterministically).

### 7.5 Resolution Results (REQ-SPA-06)

Successful spatial intents MUST return a `ResolvedPlacement` (inline in the step result or as the `Feedback`/structured payload):

```json
{
  "Description": "Placed 3 Beds near Stockpile_4821 at (42,13), (44,13), (46,13)",
  "Count": 3,
  "AnchorRequested": "Stockpile",
  "AnchorResolved": "Stockpile_4821",
  "AnchorPosition": { "X": 42, "Y": 15 },
  "Positions": [ { "X": 42, "Y": 13 }, { "X": 44, "Y": 13 }, { "X": 46, "Y": 13 } ],
  "FallbackApplied": false
}
```

Failed resolution (error `-32005`) carries data:

```json
{
  "code": -32005,
  "message": "EstablishFarm near 'ColonyCenter': no fertile soil within search radius",
  "data": {
    "AnchorRequested": "ColonyCenter",
    "Alternatives": [
      { "Anchor": "FertileCluster_0", "Position": { "X": 201, "Y": 174 }, "Reason": "190 fertile cells, fertility 1.4" }
    ]
  }
}
```

### 7.6 `resolveSpatial` — Dry Run (REQ-SPA-05, SHOULD at Level 3)

Read-only preview: identical input to the corresponding intent, returns the `ResolvedPlacement` that *would* result (or the `-32005` error), without mutating state.

```json
{ "name": "resolveSpatial", "arguments": { "Intent": { "Type": "EstablishFarm", "Near": "Region_NE", "Size": 64 } } }
```

This lets planners compare candidate anchors before committing — and lets conformance suites verify resolution integrity cheaply.

### 7.7 Standard Intent Catalog (Informative)

Environments declare their spatial intents in the action space. The released adapters use: `PlaceBuildingNear {Building, Near, Count, Stuff?}` · `EstablishFarm {Near, Crop?, Size?}` · `EstablishStorage {Near, Size?}` · `DesignateMiningNear {Near, Count?}` · `DesignateClearNear {Near, Radius?}`. Games SHOULD reuse these names for analogous semantics so cross-game policies transfer.

### 7.8 Structured Construction & One-Shot Actions (REQ-BLD-01, Informative→Normative)

Where a game supports construction, it SHOULD expose a **structural primitive** — e.g. `BuildRoom` producing a walled, doored enclosure — rather than requiring the agent to place individual cells, walls, or doors one at a time. Cell-by-cell construction driven by an agent produces degenerate "blob" structures: coordinate spam yields unenclosed walls, missing doors, and rooms that fail their game-mechanical purpose.

Environments SHOULD further prefer **one-shot actions that cannot strand intermediate state**. A `BuildRoom` that accepts an inline `Furnish` parameter (so a room is never created empty) is preferable to a sequence of `BuildRoom` then `PlaceBed`, because the sequence has an observable intermediate state — an empty room — that a confused agent can leave behind indefinitely.

**REQ-BLD-01 (NORMATIVE FINDING).** Rule-driven conductors do not reliably read free-text error guidance, and do not reliably follow multi-branch prompt conditionals ("if the room is empty, next call Furnish"). Therefore sequencing and precondition constraints MUST be enforceable adapter-side — either as **structured loud errors that name the exact next action** to take (REQ-ERR-02), or as **one-shot actions** that perform the whole sequence atomically — and MUST NOT rely solely on agent prompting to be honored.

*Field evidence (informative):* an agent repeatedly invoked a build action and stranded empty structures — never advancing to furnishing — until the adapter refused further bare builds and returned an error naming the explicit furnish action as the required next step. The free-text guidance and prompt conditionals that preceded the structured refusal did not change the agent's behavior; the adapter-side enforcement did.

---

## 8. Rewards & Episodes

### 8.1 Rewards

- **REQ-RWD-01:** synchronous delivery in the causal `step` response (Section 5.2).
- `RewardComponents` decomposes the scalar; the manifest declares each component's `Name`, `Description`, `Range`.
- Per-agent reward shaping via `RewardShaping` in registration config (`Components` + `Weights`).
- Environments SHOULD expose components that measure *decision quality*, not just outcomes — e.g., a `farm_fertility_quality` component (mean fertility of farmed cells) makes spatial decisions directly learnable.

### 8.2 Episodes

`Done` signals terminal state; `Truncated` signals time-limit cuts; `TerminationReason` ∈ `success | failure | timeout | external`. After `Done=true`, further `step` calls MUST error `-32002` until `reset`.

`episodeSummary` (Section 5.6) is the canonical episode-boundary marker for downstream training infrastructure: orchestrators sum step rewards between summaries, and lesson-synthesis pipelines treat the summary as the consolidation trigger.

---

## 9. Determinism & Reproducibility

Carried from draft-00, renamed for the dialect:

- **REQ-DET-01 (Seeded Reset):** `reset(Seed=N)` + identical action sequence → identical observations and state hashes.
- **REQ-DET-02 (State Hashing):** `stateHash` consistent for identical states across runs.
- **REQ-DET-03 (RNG Isolation):** per-agent RNG streams independent; Agent A's actions MUST NOT perturb Agent B's stochastic observations except through in-game causality.

Verification protocol:

```
reset(Seed=s); h1 = [step(a).StateHash for a in actions]
reset(Seed=s); h2 = [step(a).StateHash for a in actions]
assert h1 == h2
```

Implementations MAY document allowed non-deterministic elements (GPU float variation; network features — must be disableable; real-time elements — must be mockable in training mode).

---

## 10. Multi-Agent Coordination

### 10.1 Stepping Modes

`batchStep` (Level 3) submits actions for several agents with a `SyncMode`:

```json
{
  "name": "batchStep",
  "arguments": {
    "Steps": [
      { "AgentId": "agent_1", "Action": { "Type": "Move", "Direction": "N" } },
      { "AgentId": "agent_2", "Action": "Wait" }
    ],
    "SyncMode": "barrier"
  }
}
```

`SyncMode` ∈ `barrier` (all submit, world advances once, all observe) | `sequential` (ordered turns; optional `Order` array) | `async` (live mode semantics).

### 10.2 Agent Communication

`sendMessage` (Level 3, MAY):

```json
{ "name": "sendMessage", "arguments": { "FromAgent": "npc_1", "ToAgent": "npc_2", "Channel": "dialogue", "Content": { "Text": "Hello!" } } }
```

Messages surface in the recipient's `Events`. Note: orchestrator-side meshes (e.g., Arkavo Edge agent-to-agent delegation) are a complementary, out-of-protocol channel; `sendMessage` is for *diegetic* communication the environment should know about.

### 10.3 Session Topology

Carried from draft-00 §9.4 in substance:

- **Exclusive Session** (default): one MCP connection spawns/owns the game process; training clock; game exits with the connection; multi-agent via orchestrator multiplexing.
- **Shared Session:** the game accepts multiple IPC connections; live clock; game outlives agents; event broadcasting via `notifications/event` with `EventType`, `Tick`, `Details`, `Visibility`.

First connection establishes the topology (stdio-spawn → exclusive; IPC-attach → shared); sessions cannot mix.

---

## 11. Vision Streams

Carried from draft-00 §7 with PascalCase descriptors. Streams provide pixel observations via shared memory (IOSurface / DXGI / POSIX shm + DMA-BUF; memory-mapped file + polling as fallback).

```json
{
  "StreamId": "rgb_0",
  "Width": 224, "Height": 224,
  "PixelFormat": "rgba8",
  "RingCount": 2,
  "Transport": { "Type": "iosurface", "SurfaceIds": [101, 102] },
  "Sync": { "Type": "metal_event", "Handle": 7 }
}
```

Frame synchronization: render to slot `N % RingCount`, signal sync primitive, include `FrameIds: {"rgb_0": N}` in the step result; agent waits, then reads.

---

## 12. Input Robustness

LLM-generated tool calls are imperfect; environments that hard-fail on benign formatting variance waste training time. Environments SHOULD implement a normalization layer:

- **REQ-ROB-01 (SHOULD):** accept common casing aliases of canonical field names (`agentId`/`agent_id` → `AgentId`) and common semantic aliases (`ColonistName` → `ColonistId`); unwrap redundantly nested `Action` objects.
- **REQ-ROB-02 (SHOULD):** fuzzy-match action type names within a small edit distance (≤ 3) of a valid action; e.g. `"EstablishFarms"` → `EstablishFarm`.
- **REQ-ROB-03 (MUST, when normalizing):** report every correction applied (in `Feedback` or a `Normalized` field) so the agent can converge on canonical forms. Silent normalization is permitted-input drift; reported normalization is teaching.
- **REQ-ROB-04 (MAY):** during a migration window, accept draft-00 snake_case tool names as aliases (`sim_step` → `step`, `register_agent` → `registerAgent`, `get_state_hash` → `stateHash`, `configure_streams` → `configureStreams`). Servers doing so SHOULD emit a deprecation notice in `Feedback`.

Normalization MUST NOT apply to ambiguous inputs (two valid actions within the edit-distance budget → error `-32602` listing both).

---

## 13. Platform Adaptation

### 13.1 Reference Implementations

| Platform | Repo | Dialect | Status |
|----------|------|---------|--------|
| Rust + .NET/Harmony (RimWorld), Lua (Project Zomboid), RCON (Factorio) | `game-rl` | draft-02 | Released (RimWorld, Zomboid); Factorio in development |
| Swift/Metal (GameOfMods) | `GameOfMods` | draft-00 → migrating (Appendix B) | Shipping |
| Reference environment + conformance harness | `game-rl` (`game-rl-reference`, `game-rl-conformance`) | draft-02 | This spec's companion artifacts |
| Orchestrator / RL consumer | `arkavo-edge` | draft-02 | Active |

### 13.2 .NET/Harmony Adaptation

Carried from draft-00 §10.2: Rust MCP server ↔ Unix socket/named pipe IPC ↔ Harmony-patched .NET game. Key hooks: `TickManager.DoSingleTick` (step sync), `Game.UpdatePlay` (frame sync), `Rand.Seed` setter (determinism), `Camera.Render` (vision). The IPC message protocol is documented in Appendix D.

### 13.3 Headless Operation & Process Lifecycle

Carried from draft-00: headless operation SHOULD be supported for training (REQUIRED at Level 3). **REQ-LIFE-01:** exclusive-session environments MUST terminate when the owning connection closes (stdin EOF). **REQ-LIFE-02:** SIGTERM/SIGINT → save trajectory if recording, notify agents, release shared resources, exit 0.

---

## 14. Conformance Requirements

### 14.1 Conformance Levels

**Level 1 — Minimal (an LLM agent can play):**
- MCP initialize handshake with `serverInfo.gameRlVersion`
- Tools: `manifest`, `registerAgent`, `deregisterAgent`, `step`, `observe`, `reset` with draft-02 names and casing
- Loud errors (REQ-ERR-01/02)
- Seeded reset determinism (REQ-DET-01 at reset granularity)

**Level 2 — Standard (an RL pipeline can train):**
- All Level 1, plus: `stateHash` with sequence determinism (REQ-DET-02), `episodeSummary`, `ValidActions` (REQ-OBS-01), `Alerts` (REQ-OBS-02), multi-agent support (≥ 4 agents), named scenarios

**Level 3 — Full:**
- All Level 2, plus: Spatial Intents v2 (REQ-SPA-01…07, when spatial actions exist), `configureStreams` with ≥ 1 profile, `saveTrajectory`/`loadTrajectory`, `batchStep` (barrier + sequential), domain randomization, headless operation

### 14.2 Conformance Testing

The conformance suite is a **runnable artifact**, not an aspiration: `game-rl-conformance` (Rust, in the `game-rl` repository) spawns any server command over stdio and emits a pass/fail report per check, mapped to the requirement IDs in this document:

| Check | Level | Verifies |
|-------|-------|----------|
| C-INIT | 1 | Handshake, `gameRlVersion` present |
| C-TOOLS | 1 | Tool surface present, camelCase names, valid `inputSchema` |
| C-MANIFEST | 1 | `manifest` tool returns `Name`, `GameRlVersion`, `Capabilities` |
| C-LIFECYCLE | 1 | register → observe → step → PascalCase StepResult fields |
| C-ERR-LOUD | 1 | Bogus action → error or non-null `Error` (REQ-ERR-01) |
| C-ERR-PARTIAL | 1 | Partial successes report the shortfall (REQ-ERR-03) |
| C-DET-RESET | 1 | Same seed → same initial hash (REQ-DET-01) |
| C-DET-SEQ | 2 | Same seed + actions → same hash sequence (REQ-DET-02) |
| C-EPISODE | 2 | `episodeSummary` returns `TotalReward`, `StepCount` |
| C-VALIDACTIONS | 2 | `ValidActions` present and context-sensitive (REQ-OBS-01) |
| C-ALERTS | 2 | `Alerts` present with severity (REQ-OBS-02) |
| C-RENDERMAP | 2 | `RenderMap` viewport present with coordinate rulers (REQ-VIS-01) |
| C-SPA-GRAMMAR | 3 | All anchor forms accepted (REQ-SPA-01) |
| C-SPA-LANDMARKS | 3 | Minimum landmark set exposed (REQ-SPA-04) |
| C-SPA-INTEGRITY | 3 | Infeasible anchor without `AllowFallback` → `-32005` with alternatives; never silent relocation (REQ-SPA-02/03) |
| C-SPA-AUDIT | 3 | `ResolvedPlacement` carries `AnchorRequested`/`AnchorResolved`/`Positions`/`FallbackApplied` (REQ-SPA-06) |

The suite runs against the reference environment (`game-rl-reference`) as its own validity check; the reference environment MUST achieve Level 3. The reference environment's `fertile-corner` scenario doubles as a **behavioral probe**: all fertile soil lies far from the map center, so center-biased agents measurably fail — orchestrators are encouraged to use it as a regression test for spatial policy quality.

The draft-03 conventions checked above (`RenderMap`, partial-success feedback, and the agency-scoped Reference Point of REQ-SPA-03) are **field-validated across two adapters** — RimWorld (systemic, colony centroid) and Project Zomboid (embodied, survivor position). They are therefore specified as proven rather than aspirational. Together, the **observation diet** (§5.2) and `RenderMap` (§5.9) form the efficiency story for LLM agents: a compact step response carries deltas and feedback, full structured state is pulled only on demand via `observe`, and `RenderMap` supplies cheap textual re-perception between actions — all without advancing simulation time.

### 14.3 Compliance Declaration

Manifests SHOULD declare conformance: `"Compliance": { "Level": 2, "Version": "2.0.0-draft02", "TestResultsUrl": "..." }`.

---

## 15. Security Considerations

Carried from draft-00: agent processes SHOULD be sandboxed; environments SHOULD enforce resource limits (agents per connection, actions/sec, payload sizes, stream memory); shared-memory vision streams require index validation, read-only agent mappings, and buffer clearing on disconnect. Authentication/authorization remain deferred to a future version.

---

## 16. References

**Normative:** [RFC 2119](https://tools.ietf.org/html/rfc2119) · [JSON-RPC 2.0](https://www.jsonrpc.org/specification) · [Model Context Protocol](https://modelcontextprotocol.io/)

**Informative:** [Gymnasium](https://gymnasium.farama.org/) · [Unity ML-Agents](https://unity.com/products/machine-learning-agents) · [Harmony](https://harmony.pardeike.net/) · [GameOfMods](https://github.com/arkavo-org/GameOfMods) · [Arkavo Edge](https://github.com/arkavo-org/arkavo-edge)

---

## Appendix A: Schema Definitions

JSON Schema (2020-12) definitions: `https://github.com/arkavo-org/specifications/schemas/game-rl/draft-02/`

| Schema | Description |
|--------|-------------|
| `game-rl.schema.json` | Core $defs (`AgentId`, `AgentType`, `Scope`, `ClockMode`, `Position`, `StateHash`, `Anchor`) |
| `manifest.schema.json` | `manifest` tool result |
| `register-agent.schema.json` | Registration request/response |
| `step.schema.json` | `step` request and StepResult |
| `observe.schema.json` | `observe` request |
| `reset.schema.json` | `reset` request |
| `spatial.schema.json` | Spatial intents, anchors, `ResolvedPlacement`, `-32005` error data |
| `episode-summary.schema.json` | `episodeSummary` result |
| `events.schema.json` | Event objects and broadcast notifications |
| `vision-stream.schema.json` | Stream descriptors |

## Appendix B: Migration from draft-00

### B.1 Tool Names

| draft-00 | draft-02 |
|----------|----------|
| `register_agent` | `registerAgent` |
| `deregister_agent` | `deregisterAgent` |
| `sim_step` | `step` |
| — (absent) | `observe` |
| `reset` | `reset` |
| `get_state_hash` | `stateHash` |
| — (absent) | `episodeSummary` |
| — (absent) | `manifest` (tool; `game://manifest` resource now optional) |
| — (absent) | `resolveSpatial` |
| `save_trajectory` / `load_trajectory` | `saveTrajectory` / `loadTrajectory` |
| `configure_streams` | `configureStreams` |
| `batch_step` | `batchStep` |
| `send_message` | `sendMessage` |

### B.2 Field Names

snake_case → PascalCase throughout payloads: `agent_id` → `AgentId`, `agent_type` → `AgentType`, `action` → `Action`, `ticks` → `Ticks`, `reward_components` → `RewardComponents`, `state_hash` → `StateHash`, `done` → `Done`, `truncated` → `Truncated`. Parameterized actions flatten `params`: `{"type": "move", "params": {"direction": "n"}}` → `{"Type": "Move", "Direction": "N"}`.

### B.3 Agent Types

| draft-00 archetype | draft-02 type |
|--------------------|---------------|
| `EntityBehavior` | `Entity` |
| `ColonyManager` | `Controller` |
| `WorldSimulation` | `System` |
| `GameMaster` | `Director` |
| `DialogueAgent` | `Entity` (conversation-scoped) |
| `CombatDirector` | `Director` |
| — | `Observer`, `Player` (new) |

### B.4 Implementation Guidance

- **GameOfMods (draft-00 dialect):** add camelCase tool aliases beside the existing snake_case registrations (REQ-ROB-04), add the `manifest` tool, then migrate payload field casing behind a version flag. The `replay_trajectory` tool maps to `loadTrajectory` with `PlaybackMode`.
- **draft-00 clients:** none known in production (Arkavo Edge already speaks the draft-02 dialect).

## Appendix C: Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0-draft (draft-00) | 2025-12-28 | Initial draft (snake_case dialect) |
| draft-01 | — | Designation intentionally unused: reserved to avoid confusion with interim internal revisions of draft-00. The numbering gap marks the dialect break. |
| 2.0.0-draft02 | 2026-06-11 | Canonical camelCase/PascalCase dialect (§1.5, §12); `observe`, `episodeSummary`, `manifest` tools added; Spatial Intents v2 with resolution-integrity requirements (§7); loud-error requirements (§3.4); `ValidActions`/`Alerts`/`Landmarks` observation contract (§6); agent types canonicalized to released set (§4.1, B.3); conformance levels rebuilt around a runnable suite (§14); `Ticks: 0` action-only steps; parameterized actions flattened |

## Appendix D: IPC Bridge Protocol (Informative)

The Rust↔game IPC layer used by Harmony-style adapters (newline-delimited JSON over Unix socket / named pipe, internally tagged with `"Type"`):

Server→Game: `RegisterAgent`, `DeregisterAgent`, `ExecuteAction {agent_id, action, ticks}`, `Reset {seed, scenario}`, `GetStateHash`, `Observe`, `ConfigureStreams`, `GetEpisodeSummary`, `ResolveSpatial {intent}`, `Shutdown`.

Game→Server: `Ready {name, version, capabilities}`, `AgentRegistered`, `StepResult`, `ResetComplete`, `StateHash`, `StreamsConfigured`, `EpisodeSummary`, `SpatialResult`, `Error {code, message}`.

This layer is an implementation detail of one adapter family, not part of the protocol's conformance surface; it is documented here so new adapters for .NET/Harmony games can reuse the existing bridge crates.

## Appendix E: Acknowledgments

This specification builds on work from the Farama Foundation (Gymnasium), Unity Technologies (ML-Agents), the Harmony community, and the Arkavo engineering team — and on the playtest transcripts of a great many simulated colonists who starved next to forbidden survival meals so that REQ-ERR-01 could exist.

---

*Copyright 2026 Arkavo AI. Licensed under Apache 2.0.*
