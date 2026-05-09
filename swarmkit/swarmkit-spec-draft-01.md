# SwarmKit Specification

|                  |                                             |
|------------------|---------------------------------------------|
| **Version**      | 0.1.0-draft                                 |
| **Status**       | Community Draft                             |
| **Authors**      | Arkavo Project Contributors                 |
| **License**      | Apache 2.0                                  |
| **Companion to** | Agent Runtime Policy (ARP) v0.1.0           |
| **Builds on**    | OpenTDF 1.0, A2A JSON-RPC 2.0               |

---

## Abstract

A **SwarmKit** is a portable, signed, TDF-encrypted package that defines coordinated work for a group of hyper-specialized AI agents. It is a superset of an agent skill: a single-role SwarmKit with no handoffs reduces to a skill. SwarmKits are decrypted by an orchestrator agent which then delegates role-scoped configuration—including per-role TDF Attribute Release Policies, skills, MCP tool grants, and an `agent_provisioning` block—to each specialist agent prior to execution. Execution of a SwarmKit is a **SwarmFlight**.

This document specifies the SwarmKit manifest schema, the `agent_provisioning` schema, the TDF encryption envelope, and the orchestrator decryption-and-delegation flow.

The `agent_provisioning` block defined here governs how a specialist is *provisioned* at SwarmFlight start. It is distinct from, and complementary to, the **Agent Runtime Policy (ARP)** specification (`agent-runtime-policy/arp-spec-draft-00.md`), which governs how a running agent *adapts* its operational behavior over time. SwarmKit provisions; ARP adapts.

---

## 1. Introduction

### 1.1 Motivation

A skill packages one capability for one agent. Real multi-agent work needs a packaging format that captures: the objective, expected deliverables, role decomposition, per-role runtime constraints, per-role data-access policies, coordination topology, evaluation rubric, and completion rules—as a single signed, transportable, encryptable artifact.

A SwarmKit is that artifact.

### 1.2 Relationship to Existing Concepts

| Concept | Role |
|---|---|
| **Skill** | One agent, one capability. Subset of SwarmKit (n=1, no coordination). |
| **Agent Runtime Policy (ARP)** | Companion specification (`agent-runtime-policy/arp-spec-draft-00.md`) that governs how a *running* specialist adapts its behavior — feedback loops, decay schedules, escalation. SwarmKit's `agent_provisioning` block hands off to ARP at SwarmFlight start: provisioning sets initial conditions; ARP runs the adaptation loop and emits the DecisionTrace per ARP §17. The two specs cover disjoint lifecycle stages. |
| **Task Contract** | (forthcoming) A future Arkavo specification under which a SwarmKit compiles to a Task Contract at SwarmFlight time. Until that spec lands, SwarmKits are executed directly by the orchestrator without an intermediate compilation step. |
| **SwarmFlight** | The runtime instance executing a SwarmKit. |
| **TDF / OpenTDF** | The wire format and encryption envelope for SwarmKit distribution. |
| **A2A JSON-RPC 2.0** | The inter-agent coordination protocol invoked by the orchestrator. |

A SwarmFlight runtime MUST instantiate exactly one Agent Runtime Policy (ARP) document per role declared in the manifest. Each role's ARP document MUST be either an explicit override supplied by the launcher or a default derived from `roles[].agent_provisioning`. Per-role ARP state (DecisionTrace, PolicyCache, AdaptationEngine) MUST be isolated from other roles in the same flight.

### 1.3 Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

---

## 2. Terminology

- **SwarmKit** — A signed, TDF-encrypted manifest defining coordinated multi-agent work.
- **SwarmKit Manifest** — The cleartext, canonical YAML/JSON document inside the TDF payload.
- **SwarmFlight** — A single execution instance of a SwarmKit.
- **Orchestrator** — The privileged agent that decrypts the SwarmKit and delegates configuration to specialists. Conceptually a Conductor in the existing five-role mesh; in this spec, "orchestrator" denotes the delegation responsibility specifically.
- **Specialist** — A hyper-specialized agent receiving a role-scoped delegation from the orchestrator.
- **TDF Attribute Release Policy** — OpenTDF policy that gates data decryption based on agent identity attributes. The unqualified abbreviation "ARP" in this document never refers to the TDF Attribute Release Policy; it refers exclusively to the companion **Agent Runtime Policy** specification.
- **Agent Runtime Policy (ARP)** — Companion specification (`agent-runtime-policy/arp-spec-draft-00.md`) governing how a running specialist adapts its operational behavior. Out of scope for this document; referenced from §1.2.
- **agent_provisioning** — Per-role block of the SwarmKit manifest governing how a specialist is provisioned at SwarmFlight start: model selection, inference parameters, budgets, tool/MCP allowlists, isolation, and failure handling. Distinct from both the TDF Attribute Release Policy (which governs data) and ARP (which governs runtime adaptation).
- **KAS** — Key Access Service (OpenTDF).

---

## 3. Architecture Overview

```
                ┌──────────────────────────────────────────┐
                │  SwarmKit.tdf  (signed + encrypted)      │
                │  ┌────────────────────────────────────┐  │
                │  │  manifest.yaml (canonical)         │  │
                │  │   ├─ objective                     │  │
                │  │   ├─ inputs / deliverables         │  │
                │  │   ├─ roles[]                       │  │
                │  │   │    ├─ agent_provisioning      │  │
                │  │   │    ├─ skills[]                 │  │
                │  │   │    ├─ mcp_tools[]              │  │
                │  │   │    └─ tdf_attribute_release    │  │
                │  │   ├─ coordination                  │  │
                │  │   ├─ constraints                   │  │
                │  │   ├─ evaluation                    │  │
                │  │   └─ completion                    │  │
                │  └────────────────────────────────────┘  │
                │  C2PA / CAWG assertions                  │
                └──────────────────┬───────────────────────┘
                                   │
                                   │ 1. Authenticate (DIF DID)
                                   │ 2. Request key from KAS
                                   ▼
                       ┌───────────────────────┐
                       │     Orchestrator       │
                       │  (privileged agent)    │
                       └────────┬───────────────┘
                                │ 3. Decrypt manifest
                                │ 4. Validate signatures
                                │ 5. For each role:
                                │      delegate {
                                │        agent_provisioning,
                                │        role-scoped TDF policy,
                                │        skills,
                                │        MCP tool grants
                                │      }
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │Specialist│            │Specialist│            │Specialist│
  │ (Scribe) │◀──A2A────▶ │(Planner) │◀──A2A────▶│ (Critic) │
  └──────────┘            └──────────┘            └──────────┘

           ──── SwarmFlight in progress ────
```

---

## 4. SwarmKit Manifest Schema

The manifest is the cleartext document inside the TDF payload. Implementations MUST canonicalize before hashing or signing (sorted keys, UTF-8, LF line endings, no trailing whitespace).

### 4.1 Top-Level Structure

```yaml
spec_version: "1.1.0"          # this spec version
kit:
  id: <content-hash>            # BLAKE3 of canonical manifest, set at build time
  name: string
  version: <semver>
  description: string
  authors:
    - did: "did:web:..."        # DIF DID
      name: string
  created: <RFC3339>
  expires: <RFC3339>            # OPTIONAL; SwarmFlights MUST refuse expired kits
  nonce: <base64url>            # 128-bit random; replay defense

objective:
  goal: string                  # one-sentence high-level goal
  success_criteria: [string]

inputs:        [<InputSpec>]
deliverables:  [<DeliverableSpec>]
roles:         [<RoleSpec>]     # MUST contain >= 1 role
coordination:  <CoordinationSpec>
constraints:   <ConstraintsSpec>
evaluation:    <EvaluationSpec>
completion:    <CompletionSpec>
provenance:    <ProvenanceSpec>
```

### 4.2 InputSpec / DeliverableSpec

```yaml
- name: string
  type: "text" | "json" | "binary" | "tdf-ref" | "iroh-ticket"
  schema: <json-schema | uri>   # OPTIONAL but RECOMMENDED
  required: bool                # inputs only
  classification: string        # OPTIONAL; e.g. "public", "internal", "confidential"
```

`tdf-ref` and `iroh-ticket` allow large or sensitive payloads to be passed by reference rather than inline (compatible with the MapReduce pattern in arkavo-edge issue #432).

### 4.3 RoleSpec

```yaml
- id: string                    # unique within kit; e.g. "planner-1"
  role_type: string             # free-form; SHOULD use a value from Appendix C
  description: string
  agent_provisioning: <AgentProvisioning>   # see §5
  skills:
    - id: string                # skill content hash or registry ID
      version: <semver>
      source: "inline" | "registry" | "tdf-ref"
      payload: <inline body or reference>
  mcp_tools:
    - server: <uri>             # MCP server endpoint
      tools: [string]           # explicit tool allowlist; "*" SHOULD NOT be used
      auth: "delegated" | "passthrough" | "none"
  tdf_attribute_release_policy:  # role-scoped OpenTDF attribute release policy
    attributes:
      - "https://attr.arkavo.com/role/planner"
      - "https://attr.arkavo.com/clearance/internal"
    rule: "allOf" | "anyOf" | "hierarchy"
  handoffs:
    - to: <role-id>
      on: <condition-expr>      # e.g. "result.status == 'needs_review'"
  context_scope:
    can_read: [<role-id>]       # which other roles' outputs this role may read
    can_write: ["self"]
```

`role_type` is an opaque string. Implementations SHOULD use a value from the recommended vocabulary in Appendix C, but MAY define their own values to express domain-specific roles (e.g. `asset_analyst`, `campaign_lead`). Orchestrators MUST NOT reject a manifest solely because `role_type` is outside the recommended vocabulary.

### 4.4 CoordinationSpec

```yaml
topology: "mesh" | "pipeline" | "hub-spoke" | "hierarchical"
protocol: "a2a-jsonrpc-2.0"
routing:
  strategy: "thompson_sampling" | "round_robin" | "critic_directed" | "static"
  parameters: <opaque>          # strategy-specific
context_sharing:
  store: "shared" | "per-role"
  compaction: <CompactionSpec>  # tool-result compaction strategy
```

### 4.5 ConstraintsSpec

```yaml
global_budget:
  max_wallclock_seconds: int
  max_total_tokens: int
  max_cost_usd: number
data_classifications: [string]  # e.g. ["public"], ["internal","confidential"]
jurisdiction: [string]          # OPTIONAL; e.g. ISO 3166 codes
network:
  egress_allowed: bool
  egress_allowlist: [string]    # domain allowlist when egress_allowed
```

### 4.6 EvaluationSpec

```yaml
rubric:
  reference: <uri-or-fragment>  # OPTIONAL; pointer to an external rubric definition
  dimensions:
    - name: string              # producer-defined; see Appendix D for a recommended baseline
      weight: number             # 0..1
      threshold: number          # 0..1
critic_role: <role-id>          # which role evaluates
sample_size: int                # OPTIONAL; for Thompson Sampling routing
```

`dimensions` is producer-defined. The set of dimension names, weights, and thresholds is open: implementations MAY adopt any rubric that fits their domain. The weight of each dimension MUST be in `[0, 1]`, and the sum of `weight` across `dimensions` MUST equal 1.0 (within floating-point tolerance ±1e-6). Validators MUST reject manifests that violate either constraint. A recommended baseline rubric is given in Appendix D for kits that have no domain-specific evaluation criteria.

### 4.7 CompletionSpec

```yaml
rules:
  - "all deliverables present"
  - "rubric.weighted_score >= 0.75"
  - "no role in error state"
on_failure: "retry" | "abort" | "escalate" | "partial"
max_retries: int
```

### 4.8 ProvenanceSpec

```yaml
c2pa_assertions:                # CAWG identity + provenance
  - assertion_uri: "..."
    digest: <multihash>
signatures:
  - signer_did: "did:web:..."
    algorithm: "ed25519"
    signature: <base64url>      # over canonical manifest excluding this block
```

---

## 5. agent_provisioning

The `agent_provisioning` block is per-role and governs how the specialist agent is provisioned at SwarmFlight start: model selection, inference parameters, budgets, tool/MCP allowlists, isolation, and failure handling.

This block is distinct from two adjacent concepts:

- It is **not** the TDF Attribute Release Policy (§4.3 `tdf_attribute_release_policy`), which governs *data* access.
- It is **not** the Agent Runtime Policy (ARP) defined in `agent-runtime-policy/arp-spec-draft-00.md`, which governs how a *running* specialist *adapts* — feedback loops, decay schedules, escalation. SwarmKit's `agent_provisioning` sets the initial conditions; ARP runs the adaptation loop after provisioning completes.

```yaml
agent_provisioning:
  model:
    family: "qwen3" | "ministral-3" | "gemma-4" | "llama-3" | "custom"
    size: string                # "3B", "7B", "26B-MoE", etc.
    quantization: string        # "Q4_K_M", "F16", "MXFP4", etc.
    backend: "llama.cpp" | "mlx" | "vllm" | "custom"
    fallback:                   # OPTIONAL; if primary unavailable
      family: string
      size: string

  inference:
    max_tokens: int             # per-call ceiling
    temperature: number
    top_p: number
    top_k: int                  # OPTIONAL
    thinking: bool              # extended thinking on/off
    stop_sequences: [string]
    seed: int                   # OPTIONAL; for determinism

  budget:
    max_inference_calls: int    # per SwarmFlight
    max_wallclock_ms: int
    max_total_tokens: int       # input + output cumulative

  tool_use:
    max_calls: int
    max_parallel: int
    on_error: "retry" | "skip" | "abort"
    retry_policy:
      max_attempts: int
      backoff_ms: int

  context:
    max_context_tokens: int
    kv_cache_id: string         # OPTIONAL; arkavo-kv-cache slot reference
    compaction_strategy: "tool_result" | "summary" | "none"
    persistence: "ephemeral" | "session" | "kit"

  observability:
    trace_required: bool
    log_level: "trace" | "debug" | "info" | "warn" | "error"
    metrics_endpoint: <uri>     # OPTIONAL

  isolation:
    sandbox: "process" | "container" | "vm" | "none"
    fs_writable: [path]
    network_egress: bool        # overridable downward by global constraint

  failure:
    on_timeout: "retry" | "abort" | "fallback"
    fallback_role: <role-id>    # OPTIONAL
    circuit_breaker:
      threshold: int
      window_ms: int
```

### 5.1 Validation Rules

- `inference.max_tokens` MUST NOT exceed model context window minus prompt overhead.
- `budget.*` MUST be ≤ corresponding `constraints.global_budget.*`.
- `model.family` MUST be in the orchestrator's supported set; orchestrator MUST refuse provisioning otherwise.
- Format compatibility with downstream tooling (e.g. TØR-G constrained decoding) is the orchestrator's responsibility to verify per model family.
- `network_egress: true` MUST be denied if `constraints.network.egress_allowed: false`.

### 5.2 Defaults

When fields are omitted, the orchestrator applies defaults from its policy. Implementations SHOULD log every defaulted field for audit.

---

## 6. TDF Encryption

A SwarmKit on disk or in transit is a `.tdf` (or `.ztdf`) file conforming to OpenTDF.

### 6.1 Envelope

```
SwarmKit.tdf
├── manifest.json              # OpenTDF manifest (camelCase per spec)
│   ├── encryptionInformation
│   │   ├── keyAccess[]
│   │   │   └── policyBinding  # binds policy to wrapped key
│   │   └── method (AES-256-GCM)
│   ├── policy (base64url)     # the SwarmKit-level ARP
│   └── integrityInformation
└── 0.payload                  # encrypted SwarmKit manifest (YAML/JSON canonical)
```

### 6.2 Payload Format

The payload `0.payload` is the AES-256-GCM ciphertext of the canonical SwarmKit manifest (§4). MIME type: `application/vnd.arkavo.swarmkit+yaml` or `+json`.

### 6.3 SwarmKit-Level TDF Attribute Release Policy (Orchestrator Gate)

The TDF policy embedded in `encryptionInformation.policy` gates orchestrator decryption. Example:

```json
{
  "uuid": "...",
  "body": {
    "dataAttributes": [
      {"attribute": "https://attr.arkavo.com/role/orchestrator"},
      {"attribute": "https://attr.arkavo.com/clearance/internal"}
    ],
    "dissem": ["did:web:orchestrator.arkavo.net"]
  }
}
```

Only an agent presenting both attributes to KAS receives the wrapped key.

### 6.4 Per-Role TDF Policies (Inside the Manifest)

The `roles[].tdf_attribute_release_policy` blocks are **not** the SwarmKit-level policy from §6.3. They describe the per-role TDF Attribute Release Policies the orchestrator MUST construct and bind to data objects passed to each specialist. Specialists therefore receive only the data their role-scoped policy permits, even if the orchestrator decrypts everything.

This is the core delegation primitive: **orchestrator unwraps the kit; orchestrator re-wraps role-scoped data with role-scoped attribute release policies**.

### 6.5 Cryptographic Profile

Per opentdf-rs / OpenTDF spec:

- Symmetric: AES-256-GCM
- Key wrap (Standard TDF): RSA-2048 OAEP. Note SHA-1 OAEP padding is retained for Go SDK interop; SwarmKit producers SHOULD set `oaepPadding: "SHA-256"` once the platform supports it.
- Key agreement (NanoTDF variant, for small kits): ECDH P-256 + HKDF-SHA256

Field names MUST be camelCase (opentdf-rs / Go convention). snake_case (legacy OpenTDFKit Swift) is non-conforming for SwarmKit.

---

## 7. Orchestrator Decryption and Delegation Flow

### 7.1 Sequence

```
1.  Orchestrator receives SwarmKit.tdf
2.  Orchestrator authenticates to KAS using DIF DID
3.  KAS evaluates SwarmKit-level ARP against orchestrator attributes
4.  KAS releases wrapped key; orchestrator unwraps payload key
5.  Orchestrator decrypts payload → canonical manifest
6.  Orchestrator verifies:
       - manifest.kit.id == BLAKE3(canonical_manifest)
       - C2PA / CAWG signatures over manifest
       - manifest.kit.expires > now()
       - nonce not in replay cache
7.  Orchestrator validates schema (§4) and agent_provisioning (§5.1)
8.  For each role in manifest.roles:
       a. Provision specialist agent per agent_provisioning (see §7.1.1)
       b. Construct role-scoped TDF attribute release policy for inputs assigned to this role
       c. Bundle role's skills (verifying skill signatures)
       d. Issue MCP tool grants:
            - "delegated"   → orchestrator holds OAuth token, brokers calls
            - "passthrough" → orchestrator forwards specialist's own credentials
            - "none"        → no auth, public MCP servers only
       e. Open A2A JSON-RPC 2.0 channel to specialist
       f. Send delegation envelope (§7.2) over channel
9.  Orchestrator emits SwarmFlight start event to its lineage stream (sequence-integrity specification forthcoming)
10. SwarmFlight begins
```

### 7.1.1 Specialist-Process Sharing

Two roles MAY share a single specialist process iff their `agent_provisioning` blocks are byte-identical after canonicalization. When shared, the orchestrator MUST still issue separate delegation envelopes per role, so each role retains its own role-scoped TDF policy, MCP grants, and skill set; sharing is a provisioning-layer optimization only.

Roles MUST NOT share a process when:

- their `agent_provisioning.isolation` blocks differ in any field, or
- their `agent_provisioning.budget.*` ceilings differ (a shared process cannot enforce per-role budget ceilings without per-role accounting that defeats the purpose of sharing), or
- their `tdf_attribute_release_policy.attributes` sets are not equal (shared decrypted state would leak across roles).

Orchestrators that share specialist processes MUST keep per-role accounting for budget, tool-call counts, and DecisionTrace events as if each role had its own process.

### 7.2 Delegation Envelope

The delegation envelope is a JSON object sent over the A2A JSON-RPC 2.0 channel established in §7.1 step 8e. The wire encoding is JSON (not YAML) to match the A2A transport. Field names use snake_case to match the SwarmKit manifest. The envelope MAY itself be TDF-wrapped if the transport is untrusted.

```json
{
  "delegation": {
    "flight_id": "<uuid>",
    "kit_id": "<content-hash>",
    "role_id": "<role-id>",
    "agent_provisioning": { },
    "skills": [],
    "mcp_grants": [
      {
        "server": "<uri>",
        "tools": ["<tool-name>"],
        "bearer_token": "<opaque>",
        "expires": "<RFC3339>"
      }
    ],
    "tdf_policy_role_scoped": { },
    "context": {
      "initial_inputs": [],
      "peer_roles": ["<role-id>"]
    },
    "signing": {
      "delegation_signature": "<base64url>"
    }
  }
}
```

`delegation_signature` is computed over the canonical form of the envelope with the `signing` block excluded. Canonical form is JCS [RFC8785]: keys sorted lexicographically, no insignificant whitespace, UTF-8, RFC 8785 number serialization. Specialists MUST canonicalize the received envelope identically before signature verification; differing canonicalizations across implementations are a specification bug, not a degree of freedom.

### 7.3 Specialist Acceptance

A specialist MUST:

1. Verify the orchestrator's signature on the delegation envelope.
2. Verify each skill's signature independently.
3. Refuse if any field of `agent_provisioning` violates its own host policy (e.g. `network_egress: true` on an offline host).
4. Acknowledge with a `ready` message including the BLAKE3 of the received envelope.

### 7.4 Revocation

The orchestrator MAY revoke delegation mid-flight by:

- Closing the A2A channel.
- Issuing a revocation event to KAS, invalidating role-scoped wrapped keys.
- Logging the event in the orchestrator's lineage stream (sequence-integrity specification forthcoming).

---

## 8. Skills and MCP Tool Distribution

### 8.1 Skills

A SwarmKit MAY embed skills inline (`source: inline`), reference them by content hash (`source: registry`), or reference them as TDF blobs (`source: tdf-ref`).

A skill is the existing SKILL.md pattern: name, description, instructions, optional bundled resources. A SwarmKit with `roles.length == 1`, no handoffs, and exactly one skill is semantically equivalent to that skill—this is the superset reduction.

#### 8.1.1 SkillContent JSON Schema

A skill's content MUST be a JSON object with the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Producer-defined skill name. |
| `description` | string | yes | One-sentence description of the skill's purpose. |
| `instructions` | string | yes | The instruction text the specialist follows when invoking the skill. |
| `resources` | array of SkillResource | no | Optional bundled resources. Default: empty array. |

A `SkillResource` MUST be a JSON object with:

| Field | Type | Description |
|---|---|---|
| `name` | string | Resource identifier within the skill. |
| `mime` | string | Media type per RFC 6838. |
| `bytes_base64` | string | Base64url-encoded resource bytes (no padding, per RFC 4648 §5). |

#### 8.1.2 Skill Signing

Skill signatures MUST use Ed25519 over the BLAKE3 digest of the JCS-canonical (RFC 8785) JSON encoding of the SkillContent object. Specifically:

```
canonical_bytes = JCS_canonical_json(SkillContent)
digest          = BLAKE3(canonical_bytes)        // 32 bytes
signature       = Ed25519_sign(signer_private_key, digest)
```

The signature MUST be encoded as base64url without padding (per RFC 4648 §5). Verifiers MUST use the same canonicalization and digest functions.

#### 8.1.3 Registry Cache Layout

When `Skill.source` is `registry`, the runtime MUST resolve the skill content from a content-addressed local cache. The cache layout is:

- `<registry_cache>/<blake3-hex>.skill.json` — JCS-canonical SkillContent JSON. Filename is the BLAKE3 hex digest of the file contents.
- `<registry_cache>/<blake3-hex>.sig.json` (RESERVED) — sidecar signature file. Reserved for future use; runtimes MUST NOT depend on its presence in this draft.

Producers MAY populate the cache via any mechanism (manual placement, bundled fixtures, future fetch-from-network). The `registry_cache` path is implementation-defined; reference implementations use `$XDG_CACHE_HOME/arkavo/skills` or `~/.cache/arkavo/skills`.

### 8.2 MCP Tool Grants

Per §7.2, MCP grants are scoped to:

- a specific MCP server URI,
- an explicit tool allowlist (wildcards SHOULD NOT be used),
- an auth mode (`delegated`, `passthrough`, or `none`),
- a bearer token bound to the flight ID with explicit expiry.

Specialists MUST NOT cache MCP tokens beyond `expires`. Orchestrators SHOULD rotate tokens on long-running flights.

---

## 9. Versioning and Identity

### 9.1 Content Addressing

`kit.id` MUST be `BLAKE3(canonical_manifest)`, where canonicalization sorts keys, normalizes line endings, and excludes the `kit.id` and `provenance.signatures` fields themselves. Producers MUST recompute `kit.id` after any manifest change. Validators MUST reject manifests where the declared `kit.id` does not match the recomputed value.

### 9.2 Semver

`kit.version` follows semver. Changes to `roles[].agent_provisioning.model.family` or to the `objective` are MAJOR. Adding optional fields is MINOR. Documentation-only changes are PATCH.

### 9.3 DID and C2PA

`kit.authors[].did` MUST be a resolvable DIF DID. `provenance.c2pa_assertions` MUST conform to CAWG identity assertion v1.x.

### 9.4 Spec Version Compatibility

This document defines `spec_version: "1.1.0"`. Orchestrators MUST refuse manifests where the major spec_version differs from their supported set.

---

## 10. Security Considerations

### 10.1 Threat Model

- **Tampered manifest:** Mitigated by C2PA/CAWG signatures and content-addressing.
- **Replay:** Mitigated by `kit.nonce` and `kit.expires`. Orchestrators MUST maintain a nonce cache for the longest active `expires` value. To bound the resulting cache footprint and deny far-future replay-cache exhaustion, orchestrators MUST cap accepted manifests at `expires - created <= 1 year`, and SHOULD cap at `<= 90 days` unless an operational requirement demands a longer horizon. Manifests exceeding the cap MUST be rejected before any decryption.
- **Privilege escalation by specialist:** A specialist receives only its role-scoped delegation envelope. It does not receive the SwarmKit-level wrapped key. It cannot decrypt data outside its role-scoped TDF attribute release policy.
- **Compromised orchestrator:** Catastrophic by design. Mitigations: short-lived KAS tokens, strict orchestrator clearance attributes, audit via the orchestrator's lineage stream.
- **Tool capability creep:** `mcp_tools[].tools` is an explicit allowlist; specialists that attempt out-of-scope MCP calls fail at the orchestrator's MCP broker (when `auth: delegated`).
- **Skill substitution:** Skills are individually signed; specialists verify each.
- **Cross-flight contamination:** `kv_cache_id` slots MUST be flight-scoped unless explicitly marked persistent.
- **Self-evaluation laundering:** A single-role kit (or any kit where `evaluation.critic_role` resolves to a role evaluating its own outputs) produces an unverified rubric score. Orchestrators MUST tag `evaluation` results from such kits as `self_evaluated: true` in the DecisionTrace, and downstream consumers (gateways, registries, marketplaces) MUST treat self-evaluated scores as unverified for the purpose of trust signals, ranking, or quality-gated routing.
- **Privilege escalation by sibling role:** A role with broader TDF attributes (e.g., an `auditor` role carrying an `audit_authority/true` attribute) MUST receive its own role-scoped TDF policy at orchestrator-decryption time. Producers MAY add per-role attributes to differentiate access — sibling roles in the same SwarmKit can carry distinct attribute sets, and the KAS enforces the boundary. Reference: the compliance-kit example (https://github.com/arkavo-org/arkavo-edge/blob/main/examples/compliance-kit/) demonstrates this pattern with `pii_classifier`, `policy_enforcer`, and `auditor` roles where only the auditor's TDF policy carries `audit_authority/true`.

### 10.2 Decomposition Attacks

A SwarmKit attacker may attempt to construct multi-role kits where each role appears innocuous but composition is harmful. When a sequence-integrity specification with cross-action taint propagation rules is available, implementations SHOULD apply those rules during manifest validation as well as at runtime. Until then, orchestrators SHOULD inspect role-to-role handoffs and the union of MCP grants for capability combinations that exceed any single role's authorization.

### 10.3 Injection in Delegation Envelopes

Envelopes pass through orchestrator-controlled channels. Specialists MUST treat all envelope content as data, not instruction, except for the explicit `agent_provisioning` and `skills` fields whose semantics are defined here.

---

## 11. Conformance

A conforming **SwarmKit producer** MUST:

- C-P1. Produce TDF envelopes per §6.
- C-P2. Sign manifests with at least one DID-resolvable identity.
- C-P3. Emit canonical manifests (§9.1).
- C-P4. Set `kit.expires` for kits intended for distribution.

A conforming **orchestrator** MUST:

- C-O1. Reject expired or replay-detected kits.
- C-O2. Verify all signatures before any delegation.
- C-O3. Construct role-scoped TDF attribute release policies and never share the SwarmKit-level wrapped key with specialists.
- C-O4. Enforce `agent_provisioning` validation per §5.1 before provisioning.
- C-O5. Issue MCP grants with explicit allowlists and expiries.
- C-O6. Emit a lineage event on every delegation and revocation, written to whichever sequence-integrity stream the orchestrator's deployment uses.

A conforming **specialist** MUST:

- C-S1. Verify orchestrator signature on the delegation envelope.
- C-S2. Verify each skill signature independently.
- C-S3. Refuse policies that violate its host environment.
- C-S4. Honor `mcp_grants[].expires` and not cache tokens beyond it.

---

## 12. Examples

### 12.1 Single-Role SwarmKit (Skill Equivalence)

The example values below — `kit.id`, `kit.nonce`, and `provenance.signatures[].signature` — are realistic byte lengths (BLAKE3-256, 16-byte nonce, ed25519 signature), suitable for self-validation against the schemas in `schemas/swarmkit/draft-00/`. They are not derived from the manifest content and are not verifiable.

```yaml
spec_version: "1.1.0"
kit:
  id: "blake3:3kW9D3RPLdzL6UYJgCsjxn5gV-AFQxPd4w88aiEmBck"
  name: "Markdown Linter"
  version: "1.0.0"
  description: "Lint markdown for style violations"
  authors: [{did: "did:web:arkavo.net", name: "Arkavo"}]
  created: "2026-04-29T12:00:00Z"
  expires: "2026-07-28T12:00:00Z"
  nonce: "thz1Cz8aWOUURbyQQfvA0Q"
objective:
  goal: "Identify style violations in a markdown file"
  success_criteria: ["all violations enumerated", "no false positives in test set"]
inputs:
  - {name: "doc", type: "text", required: true}
deliverables:
  - {name: "violations", type: "json"}
roles:
  - id: "linter"
    role_type: "specialist"
    description: "Apply lint rules"
    agent_provisioning:
      model: {family: "qwen3", size: "3B", quantization: "Q4_K_M", backend: "llama.cpp"}
      inference: {max_tokens: 512, temperature: 0.1, thinking: false}
      budget: {max_inference_calls: 4, max_wallclock_ms: 30000, max_total_tokens: 8000}
      context: {max_context_tokens: 16000, persistence: "ephemeral"}
    skills:
      - {id: "skill:markdown-lint", version: "2.1.0", source: "registry"}
    mcp_tools: []
    tdf_attribute_release_policy:
      attributes: ["https://attr.arkavo.com/clearance/public"]
      rule: "allOf"
    handoffs: []
coordination:
  topology: "hub-spoke"
  protocol: "a2a-jsonrpc-2.0"
  routing: {strategy: "static"}
constraints:
  global_budget: {max_wallclock_seconds: 60, max_total_tokens: 8000, max_cost_usd: 0.01}
  data_classifications: ["public"]
  network: {egress_allowed: false, egress_allowlist: []}
evaluation:
  rubric:
    dimensions:
      - {name: "accuracy", weight: 1.0, threshold: 0.8}
  critic_role: "linter"  # self-evaluation; per §10.1 results are unverified
completion:
  rules: ["all deliverables present"]
  on_failure: "abort"
  max_retries: 0
provenance:
  signatures:
    - signer_did: "did:web:arkavo.net"
      algorithm: "ed25519"
      signature: "-E3rwWzQ_pl92kJP9ZTnQv1fEohUfmhU2d4SE1bJrMXsjli6z6gNs4ZaMAtL0X2qxRDKSgJWZEgNen9ikweYRw"
```

### 12.2 Five-Role Mesh SwarmKit (abbreviated)

```yaml
roles:
  - id: "scribe-1"
    role_type: "scribe"
    agent_provisioning:
      model: {family: "ministral-3", size: "3B", backend: "llama.cpp"}
      inference: {max_tokens: 200, temperature: 0.1, thinking: false}
    skills: [{id: "skill:transcribe-events", version: "1.0.0", source: "registry"}]
    handoffs: [{to: "historian-1", on: "always"}]

  - id: "historian-1"
    role_type: "historian"
    agent_provisioning:
      model: {family: "qwen3", size: "7B", backend: "llama.cpp"}
      context: {max_context_tokens: 32000, kv_cache_id: "historian-slot-1"}
    skills: [{id: "skill:context-pool", version: "1.2.0", source: "registry"}]
    handoffs: [{to: "planner-1", on: "context_ready"}]

  - id: "planner-1"
    role_type: "planner"
    agent_provisioning:
      model: {family: "gemma-4", size: "26B-MoE", backend: "llama.cpp"}
      inference: {max_tokens: 2000, temperature: 0.3, thinking: true}
    handoffs:
      - {to: "operator-1", on: "plan_ready"}
      - {to: "critic-1", on: "plan_ready"}

  - id: "critic-1"
    role_type: "critic"
    agent_provisioning:
      model: {family: "qwen3", size: "7B"}
      inference: {max_tokens: 500, temperature: 0.0, thinking: false}

  - id: "operator-1"
    role_type: "operator"
    agent_provisioning:
      model: {family: "ministral-3", size: "3B"}
      inference: {max_tokens: 200, temperature: 0.1, thinking: false}
    mcp_tools:
      - server: "https://game-rl.arkavo.net/mcp"
        tools: ["observe_state", "submit_action", "query_inventory"]
        auth: "delegated"

coordination:
  topology: "mesh"
  protocol: "a2a-jsonrpc-2.0"
  routing: {strategy: "thompson_sampling"}
```

---

## 13. Open Questions

1. **MCP grant rotation cadence.** Should the orchestrator define a default rotation interval, or leave it to per-server policy?
2. **`tdf-ref` and `iroh-ticket` interaction.** Should SwarmKit specify a preference order or leave it to the orchestrator?
3. **Specialist-to-specialist delegation.** Out of scope for v1.0. Hierarchical SwarmKits (a role launches a sub-flight) deferred to a future draft.

---

## 14. References

### 14.1 Normative

- **Agent Runtime Policy (ARP)** — `agent-runtime-policy/arp-spec-draft-00.md` (this repo). Companion specification governing runtime adaptation of provisioned specialists.
- **OpenTDF** — https://opentdf.io
- **opentdf-rs** — https://github.com/arkavo-org/opentdf-rs
- **JSON-RPC 2.0** — https://www.jsonrpc.org/specification
- **JSON Canonicalization Scheme (JCS)** — RFC 8785
- **BCP 14** — RFC 2119, RFC 8174 (requirements language)
- **C2PA / CAWG** — Coalition for Content Provenance and Authenticity, Creator Assertions Working Group
- **DIF did:web** — https://w3c-ccg.github.io/did-method-web/
- **BLAKE3** — https://github.com/BLAKE3-team/BLAKE3-specs

### 14.2 Forthcoming

The following specifications are referenced from this document but have not yet been published. Implementations conforming to this draft MUST NOT depend on the unwritten specifications; their reference points in this draft are advisory until published.

- **Task Contract Protocol** — A future Arkavo specification under which a SwarmKit will compile to a Task Contract at SwarmFlight time. Referenced from §1.2.
- **Sequence Integrity Specification** — A future Arkavo specification defining lineage events and cross-action taint propagation. Referenced from §7.1, §7.4, §10.2, §11.

---

## Appendix A: Reduction to Skill

A SwarmKit reduces to a skill iff:

- `roles.length == 1`
- `roles[0].handoffs == []`
- `roles[0].skills.length == 1`
- `coordination.topology == "hub-spoke"` (degenerate)
- `evaluation.critic_role == roles[0].id` (self-evaluation) OR `evaluation` omitted

In this case the SwarmFlight is equivalent to invoking the skill directly. Orchestrators MAY short-circuit such kits to a direct skill invocation as an optimization, provided audit lineage is preserved.

## Appendix B: Filename Conventions

- Cleartext manifest (development only): `<name>.swarmkit.yaml` or `.json`
- Encrypted distributable: `<name>.swarmkit.tdf`
- NanoTDF variant (small kits, fits in single message): `<name>.swarmkit.ntdf`

## Appendix C: Recommended `role_type` Vocabulary

`role_type` (§4.3) is a free-form string. The values below are the recommended vocabulary; producers SHOULD use them when the role's behavior matches one of these archetypes, and MAY define their own values for domain-specific roles.

| Value | Archetype |
|---|---|
| `scribe` | Records and transcribes events into structured text. |
| `historian` | Maintains long-context memory; aggregates prior outputs. |
| `planner` | Decomposes objective into ordered subtasks. |
| `critic` | Scores outputs against the rubric defined in `evaluation`. |
| `operator` | Executes tools; submits actions to external systems. |
| `specialist` | Generic specialist; use when no other archetype fits. |

Domain-specific values are unrestricted. For example, a Campaign SwarmKit might use `campaign_lead`, `asset_analyst`, `audience_strategist`, `platform_copy`, `brand_safety`, etc. Orchestrators MUST NOT reject a manifest solely because `role_type` is outside this list.

### C.1 Domain-Specific Examples

Producers SHOULD use the recommended vocabulary above when their role's behavior matches an archetype. For domain-specific roles, free-form values are permitted. The following examples from shipped SwarmKit reference implementations illustrate the extension contract:

| Domain | role_type values |
|---|---|
| Marketing | `asset_analyst`, `platform_copy`, `critic` |
| Developer | `code_reviewer`, `security_auditor`, `test_author` |
| Creative | `prompt_designer`, `vrm_assembler`, `vrm_validator` |
| Regulated | `pii_classifier`, `policy_enforcer`, `auditor` |

Conformance test SK-006 verifies that orchestrators MUST NOT reject manifests solely because `role_type` is outside the recommended vocabulary. Conformance tests SK-007 through SK-017 verify the same extension contract for `evaluation.rubric.dimensions[].name`, `tdf_attribute_release_policy.attributes`, `data_classifications`, `mcp_tools[].tools`, and `kit.authors[].did`.

## Appendix D: Recommended Baseline Evaluation Rubric

For kits with no domain-specific evaluation criteria, the following rubric is recommended as a starting point. Producers MAY adopt, modify, or replace it.

| Dimension | Weight | Threshold |
|---|---|---|
| `accuracy` | 0.30 | 0.7 |
| `completeness` | 0.25 | 0.7 |
| `safety` | 0.25 | 0.9 |
| `alignment` | 0.20 | 0.8 |

Weights sum to 1.0; thresholds are minimum acceptable per-dimension scores. Producers that adopt this rubric should reference it as `rubric.reference: "swarmkit-spec-draft-00#appendix-d"`.
