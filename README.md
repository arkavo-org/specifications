# Arkavo Specifications

Technical specifications for the Arkavo ecosystem.

## Specifications

### game-rl/

**Game-RL Protocol: Multi-Agent AI Interface for Game Environments**

The Game-RL Protocol defines a standard interface for multi-agent AI systems to observe and interact with game environments, functioning either as embodied characters (APCs/NPCs) or systemic controllers (Game Masters). Built on MCP (Model Context Protocol), it provides:

- **Dual Agency Scopes**: Embodied agents (physics-bound) vs Systemic agents (god-mode)
- **Clock Modes**: Training (agent-owned, deterministic) vs Live (engine-owned, real-time)
- **Session Topologies**: Exclusive (stdio) vs Shared (IPC for Claude/ChatGPT Desktop)
- **Role-Based Permissions**: GameMaster can spawn/teleport; EntityBehavior can only move/interact
- **Vision Streams**: High-performance shared memory for pixel observations

Wire format: JSON-RPC 2.0 over MCP (stdio/IPC)

**Latest Draft**: [draft-arkavo-game-rl-00](game-rl/draft-arkavo-game-rl-00.md)

**JSON Schemas**: [schemas/game-rl/draft-00/](schemas/game-rl/draft-00/)

**Reference Implementation**: [GameOfMods](https://github.com/arkavo-org/GameOfMods)

---

### ntdf-rtmp/

**NTDF-RTMP: NanoTDF Manifest Transport over RTMP**

NTDF-RTMP specifies methods for transporting NanoTDF policy manifests over RTMP streaming connections, enabling real-time encrypted media with dynamic policy updates. It provides:

- **In-Band Transport**: Manifests via AMF onMetaData or RTMP Data Messages
- **Session State Machine**: AWAITING_MANIFEST → ACTIVE → REKEYING states
- **Receiver Protocol**: Buffering, manifest parsing, and key derivation
- **Compatibility**: Works with FFmpeg, OBS, Wirecast via standard onMetaData

Wire format: AMF-encoded manifest in RTMP metadata

**Latest Draft**: [draft-arkavo-ntdf-rtmp-02](ntdf-rtmp/draft-arkavo-ntdf-rtmp-02.md)

**Reference Implementation**: [arkavo-rs](https://github.com/arkavo-org/arkavo-rs)

---

### ntdf-token/

**NTDF Tokens: NanoTDF-based Authentication Tokens**

NTDF tokens are cryptographically-bound authentication tokens that replace traditional JWT Bearer tokens. Built on OpenTDF's NanoTDF specification, they provide:

- **Proof-of-Possession**: DPoP (RFC 9449) integration prevents token theft
- **Policy Binding**: Cryptographic GMAC binding enforces access control at the KAS
- **Confidentiality**: AES-256-GCM encrypted payload protects claim data
- **Provenance**: Ed25519 signatures ensure token integrity and origin

Wire format: `Authorization: NTDF <Z85-encoded-nanotdf>`

**Latest Draft**: [draft-arkavo-ntdf-token-00](ntdf-token/draft-arkavo-ntdf-token-00.md)

**Reference Implementation**: [arkavo-rs](https://github.com/arkavo-org/arkavo-rs)

---

### torg-decision/

**TØR-G: Token-Only Reasoner (Graph) Intermediate Representation**

TØR-G is a zero-parser, boolean-circuit Intermediate Representation designed for AI agent policy synthesis. It treats LLM output as a direct stream of graph construction instructions, providing:

- **No Text**: The language consists solely of atomic tokens
- **No Parsing**: The "compiler" is a deterministic state machine
- **Strict Booleanism**: All logic reduced to boolean decision combinatorics
- **Verifiability**: Every generated program is a finite DAG amenable to SAT/BDD verification

Wire format: Direct token stream with logit masking enforcement

**Latest Draft**: [draft-arkavo-torg-decision-00](torg-decision/draft-arkavo-torg-decision-00.md)

---

### tdf-json/

**TDF-JSON: Trusted Data Format — JSON Serialization**

TDF-JSON defines a JSON-based serialization for the Trusted Data Format, providing encrypted payloads with policy-bound key access.

**Latest Draft**: [draft-00](tdf-json/draft-00.md)

---

### tdf-cbor/

**TDF-CBOR: Trusted Data Format — CBOR Serialization**

TDF-CBOR defines a compact CBOR-based serialization for the Trusted Data Format, optimized for constrained environments and reduced wire size.

**Latest Draft**: [draft-00](tdf-cbor/draft-00.md)

---

### agent-runtime-policy/

**Agent Runtime Policy (ARP): Runtime Adaptation for AI Agents**

ARP provides a standard format for describing how AI agents adapt their behavior at runtime. A companion to the Agent Definition Language (ADL), ARP defines the adaptation machinery that governs an agent's operational evolution within its declared boundaries:

- **Adaptation Engine**: Thompson Sampling, UCB1, epsilon-greedy with Bayesian priors
- **Multi-Timescale Feedback**: Per-event quality gates, PolicyCache with temporal decay, peer gossip, offline consolidation
- **One-Way Ratchet**: Automated adaptation may only tighten policy; relaxation requires human approval
- **Cross-Layer Escalation**: Events in one layer (cognitive, execution, data, network) trigger policy tightening in others
- **Budget & Velocity Constraints**: Per-task ceilings, per-minute spend limits, degradation chains
- **Cryptographic Integrity**: Document signing, ADL binding, tamper-evident state storage

Wire format: `application/arp+json` (JSON canonical), `.arp.toml` (authoring)

**Latest Draft**: [arp-spec-draft-00](agent-runtime-policy/arp-spec-draft-00.md)

**JSON Schemas**: [schemas/agent-runtime-policy/draft-00/](schemas/agent-runtime-policy/draft-00/)

---

### swarmkit/

**SwarmKit: Multi-Agent Work Packaging Format**

A SwarmKit is a portable, signed, TDF-encrypted package that defines coordinated work for a group of hyper-specialized AI agents. A single-role SwarmKit with no handoffs reduces to a skill. SwarmKits are decrypted by an orchestrator agent which delegates role-scoped configuration — TDF Attribute Release Policies, skills, MCP tool grants, and an `agent_provisioning` block — to each specialist before execution. Execution is a SwarmFlight. SwarmKit's `agent_provisioning` block hands off to the companion [Agent Runtime Policy](agent-runtime-policy/arp-spec-draft-00.md) at SwarmFlight start: SwarmKit provisions, ARP adapts.

- **Manifest Schema**: Objective, roles, coordination topology, evaluation rubric, completion rules
- **TDF Encryption**: OpenTDF envelope with orchestrator-gated key release via KAS
- **Role-Scoped Delegation**: Per-role TDF ARPs, signed skills, scoped MCP grants
- **Coordination**: A2A JSON-RPC 2.0 over mesh / pipeline / hub-spoke / hierarchical topologies

Wire format: `application/vnd.arkavo.swarmkit+yaml` (manifest), `.swarmkit.tdf` (distributable)

**Latest Draft**: [swarmkit-spec-draft-00](swarmkit/swarmkit-spec-draft-00.md)

**JSON Schemas**: [schemas/swarmkit/draft-00/](schemas/swarmkit/draft-00/)

---

## JSON Schemas

Machine-readable schema definitions for protocol validation:

| Specification | Schema Directory |
|---------------|------------------|
| Game-RL | [schemas/game-rl/draft-00/](schemas/game-rl/draft-00/) |
| Agent Runtime Policy | [schemas/agent-runtime-policy/draft-00/](schemas/agent-runtime-policy/draft-00/) |
| SwarmKit | [schemas/swarmkit/draft-00/](schemas/swarmkit/draft-00/) |

## License

Apache 2.0 - See [LICENSE](LICENSE)