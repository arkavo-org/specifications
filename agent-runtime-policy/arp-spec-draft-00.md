# Agent Runtime Policy (ARP) Specification
 
|                    |                                                              |
|--------------------|--------------------------------------------------------------|
| **Version**        | 0.1.0-draft                                                  |
| **Status**         | Community Draft                                               |
| **Authors**        | Arkavo Project Contributors                                   |
| **License**        | Apache 2.0                                                    |
| **Companion To**   | Agent Definition Language (ADL) v0.1.0                        |
 
---
 
## Part I: Core Specification & Integrity
 
### 1. Introduction
 
#### 1.1 Purpose
 
The Agent Runtime Policy (ARP) provides a standard format for describing how AI agents adapt their behavior at runtime. ARP documents are JSON objects that define the adaptation machinery, feedback loops, budget constraints, escalation rules, and observability requirements that govern an agent's operational evolution within its declared boundaries.
 
ADL describes what an agent *is* before deployment — identity, permissions, capabilities, and lifecycle. ARP describes how an agent *adapts* during operation — learning rates, decay schedules, escalation triggers, and the rules by which runtime policy evolves. Together, ADL and ARP form a complete, auditable description of both the static boundaries and the dynamic behavior of an autonomous agent.
 
ARP addresses a gap in the current standards landscape: no existing specification defines the rules by which agent runtime policy adapts. OPA/Rego governs static policy decisions (if X then allow/deny). ADL governs definition-time agent descriptions. MCP and A2A govern communication protocols. None describes the adaptation machinery — the feedback loops, exploration parameters, decay schedules, and cross-layer escalation rules that determine how an agent's operational behavior evolves over time. Every agent platform that performs runtime adaptation currently does so with proprietary configuration formats that are invisible to auditors, non-portable across runtimes, and impossible to compare across vendors.
 
ARP makes runtime adaptation auditable, portable, and vendor-comparable.
 
#### 1.2 Relationship to ADL
 
ADL defines **Constitutional policy** — the boundaries set by humans at definition time that the agent may never exceed. ARP defines **Operational policy** — the adaptation machinery that adjusts agent behavior within those boundaries based on runtime observations.
 
At startup, a conformant runtime loads both documents. The ADL document's permissions, data classifications, and resource limits are written into the PolicyCache as entries with `source: constitutional` and `decay: never`. The ARP document configures the adaptation engine that operates above that floor. An agent deployed with an ADL document but no ARP document operates under static policy only — no runtime adaptation occurs.
 
The `adl_ref` member in an ARP document cryptographically binds the ARP to its companion ADL document. Modifying either document without updating the binding invalidates the integrity signature, preventing privilege escalation via document tampering.
 
#### 1.3 Relationship to Other Specifications
 
ARP builds upon and interoperates with:
 
- **[ADL](https://www.adl-spec.org/)** — ARP is a companion specification to ADL. ARP documents reference ADL documents via `adl_ref`.
- **[JSON [RFC8259]](https://www.rfc-editor.org/rfc/rfc8259)** — ARP documents are valid JSON.
- **[JSON Schema](https://json-schema.org/draft/2020-12/json-schema-core)** — ARP documents are validated against JSON Schema.
- **[JCS [RFC8785]](https://www.rfc-editor.org/rfc/rfc8785)** — ARP documents use JCS for canonical serialization before signing.
- **[W3C DIDs](https://www.w3.org/TR/did-core/)** — ARP documents reference DIDs for agent identity and trust anchors.
- **[W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model-2.0/)** — ARP supports VCs for credential presentation in escalation and HITL workflows.
 
#### 1.4 Design Goals
 
- **Auditable:** Every aspect of runtime adaptation is declared in a machine-readable, human-reviewable document. Auditors can inspect the adaptation rules without reading source code.
- **Portable:** ARP documents describe adaptation machinery independent of any specific runtime, platform, or provider.
- **Cryptographically Verifiable:** ARP documents are signed and bound to their companion ADL documents, forming a tamper-evident pair.
- **One-Way Ratchet:** Automated adaptation may only tighten policy. Relaxation requires human approval. This is a foundational security invariant.
- **Vendor-Neutral:** ARP defines the adaptation interface, not the implementation. Thompson Sampling, UCB1, and epsilon-greedy are all expressible. Custom methods are extensible.
 
---
 
### 2. Requirements Language
 
The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.
 
---
 
### 3. Terminology
 
| Term | Definition |
|------|-----------|
| **ARP document** | A JSON object that conforms to this specification. |
| **Constitutional policy** | Static policy declared at definition time in an ADL document. Sets the floor that operational policy may never lower. |
| **Operational policy** | Dynamic policy adapted at runtime by the ARP adaptation engine. Operates above the Constitutional floor. |
| **PolicyCache** | A runtime store of learned policy entries with decay schedules, source attribution, and sensitivity-aware TTLs. |
| **Prior** | A probability distribution (typically Beta) representing the runtime's belief about an entity's quality, updated by Bayesian inference from outcome observations. |
| **Decay** | The process by which Operational policy entries lose influence over time, ensuring stale lessons do not permanently distort routing. |
| **Ratchet** | The invariant that automated adaptation may only tighten policy (increase restrictions). Relaxation (decreased restrictions) requires human approval. |
| **Quarantine** | A temporary or sticky restriction placed on an entity (tool, model, peer, data source) based on observed failures or threats. |
| **Consolidation** | An offline process that deduplicates lessons, extracts patterns, detects contradictions, and produces axioms from accumulated PolicyCache entries. |
| **Gossip** | The mechanism by which agents share learned policy updates with peers in a mesh, subject to quorum rules and trust verification. |
| **DecisionTrace** | An append-only audit log recording every policy enforcement event with full attribution of the routing decision, outcome, and resulting feedback. |
| **Entity** | A model, tool, peer agent, data source, or endpoint — any object whose quality is tracked by the adaptation engine. |
| **Feasible set** | The set of entities currently available for selection. Availability failures remove entities from the feasible set without updating quality priors. |
| **Quality signal** | An outcome observation that reflects the quality of an entity's output. Updates Bayesian priors. |
| **Availability signal** | An outcome observation that reflects whether an entity was reachable. Updates the feasible set only, never priors. |
| **Cold start** | The initial state when no prior observations exist for an entity. Governed by the cold start policy in §6.2. |
| **HITL** | Human-In-The-Loop. A human operator who approves relaxation, reviews escalations, and provides teaching inputs. |
 
---
 
### 4. Document Structure
 
#### 4.1 Media Type
 
- ARP documents use the media type **`application/arp+json`**.
- ARP documents **MUST** be encoded in UTF-8.
- ARP documents **MUST** be valid JSON [RFC8259].
- Member names **MUST** use **snake_case** (lowercase with underscores).
- All timestamps **MUST** be ISO 8601 strings with timezone.
- All URIs **MUST** conform to [RFC3986].
 
#### 4.2 Authoring Format
 
TOML is the **RECOMMENDED** authoring format for human-edited ARP documents. JSON is the canonical wire format. When an ARP document is authored in TOML, implementations **MUST** convert it to JSON for processing, validation, and signing. The media type `application/arp+json` applies to the JSON canonical form only.
 
Rationale: TOML's explicit typing prevents the class of implicit coercion bugs common in YAML (boolean coercion of country codes, float collapse of version strings). TOML's lack of a canonical serialization form is irrelevant because signing and validation operate on the JSON canonical form.
 
File extensions: `.arp.json` (canonical), `.arp.toml` (authoring).
 
#### 4.3 Top-Level Object
 
An ARP document **MUST** be a single JSON object.
 
**Required members:**
 
- `arp_spec` — Version of the ARP specification (semantic versioning).
- `adl_ref` — URI or hash reference to the companion ADL document.
- `adaptation` — Adaptation engine configuration (§6).
- `feedback_loops` — Feedback loop configuration (§7).
- `budget` — Budget constraints (§13).
 
**Optional members:**
 
- `integrity` (§5), `precedence` (§8), `cognitive` (§9), `execution` (§10), `data_sovereignty` (§11), `network` (§12), `escalation` (§14), `quarantine` (§14), `hitl` (§15), `session` (§15), `state_storage` (§16), `observability` (§17), `metadata`
 
#### 4.4 Extension Mechanism
 
ARP uses the same extension mechanism as ADL. Custom members **MUST** be prefixed with `x_` followed by a namespace identifier (e.g., `x_arkavo_consolidation_model`). Implementations **MUST** preserve extension members when processing but **MAY** ignore their contents.
 
#### 4.5 ADL Document Binding
 
The `adl_ref` member **MUST** be present and **MUST** be an object containing at minimum one of:
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `uri` | string | CONDITIONAL | URI of the companion ADL document |
| `document_hash` | string | CONDITIONAL | Cryptographic hash of the ADL document (`algorithm:hex_digest`) |
 
At least one of `uri` or `document_hash` **MUST** be present. When both are present, implementations **MUST** verify that the document retrieved from `uri` matches `document_hash`. When `integrity` (§5) is present, `document_hash` is **REQUIRED** — the ARP signature covers the ARP document contents including the ADL hash, creating a tamper-evident pair.
 
Example:
 
```json
{
  "adl_ref": {
    "uri": "https://acme.example.com/agents/claims-processor/adl.json",
    "document_hash": "sha256:a1b2c3d4e5f6..."
  }
}
```
 
---
 
### 5. Cryptographic Integrity & Trust Binding
 
The `integrity` member declares the cryptographic binding of the ARP document to a governing authority. **OPTIONAL.** When present, it ensures that the adaptation machinery cannot be tampered with — an attacker who modifies the ARP (e.g., disabling the one-way ratchet or setting `exploration_floor` to 1.0) will invalidate the signature.
 
When `integrity` is present, value **MUST** be an object:
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `signed_by` | string | REQUIRED | DID of the signing authority |
| `algorithm` | string | REQUIRED | Signature algorithm (Ed25519 RECOMMENDED) |
| `signature` | string | REQUIRED | Base64url-encoded signature |
| `signed_content` | string | REQUIRED | `"canonical"` or `"digest"` |
| `digest_algorithm` | string | CONDITIONAL | Required when `signed_content` is `"digest"` |
| `digest_value` | string | CONDITIONAL | Required when `signed_content` is `"digest"` |
| `issued_at` | string | OPTIONAL | ISO 8601 timestamp |
| `expires_at` | string | OPTIONAL | ISO 8601 timestamp |
 
**Verification procedure:**
 
1. Remove the `integrity` member from the document.
2. Serialize the remaining document using JCS [RFC8785].
3. If `signed_content` is `"digest"`, compute the digest using `digest_algorithm` and verify it matches `digest_value`.
4. Resolve the public key from the DID in `signed_by`.
5. Verify the signature against the canonical byte sequence (or digest).
6. Verify that `adl_ref.document_hash` is present and that the referenced ADL document matches the declared hash.
 
Implementations **MUST** reject ARP documents whose `integrity` signature does not verify. Implementations **SHOULD** warn when `expires_at` is in the past or within 30 days.
 
**Trust registry binding:** The `signed_by` DID **SHOULD** be resolvable to a DID Document that includes the signing public key. For mesh deployments, the signing authority **SHOULD** be a governance DID (e.g., `did:web:acme.example.com:governance`) rather than an individual agent DID, ensuring that ARP documents are authorized by organizational policy, not self-signed by the agent they govern.
 
Example:
 
```json
{
  "integrity": {
    "signed_by": "did:web:acme.example.com:governance",
    "algorithm": "Ed25519",
    "signature": "base64url_encoded_signature",
    "signed_content": "canonical",
    "issued_at": "2026-03-01T00:00:00Z",
    "expires_at": "2027-03-01T00:00:00Z"
  }
}
```
 
---
 
## Part II: The Adaptation Engine
 
### 6. Adaptation Machinery
 
The `adaptation` member configures the core learning mechanism that governs how the runtime selects among competing entities (models, tools, peers, data sources). **REQUIRED.**
 
#### 6.1 Method Selection
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `method` | string | REQUIRED | Adaptation method identifier |
| `parameters` | object | OPTIONAL | Method-specific parameters |
 
`method` **MUST** be one of:
 
| Value | Description |
|-------|-------------|
| `thompson_sampling` | Bayesian bandit with Beta distribution priors. **RECOMMENDED.** |
| `epsilon_greedy` | Random exploration with probability epsilon. |
| `ucb1` | Upper Confidence Bound algorithm. |
| `static` | No adaptation. All routing decisions are deterministic based on Constitutional policy. |
 
When `method` is `thompson_sampling`, `parameters` **MAY** contain:
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `exploration_floor` | number | 0.05 | Minimum probability of exploring non-preferred entities (0.0–1.0) |
 
When `method` is `epsilon_greedy`, `parameters` **MUST** contain:
 
| Member | Type | Description |
|--------|------|-------------|
| `epsilon` | number | Exploration probability (0.0–1.0) |
| `epsilon_decay` | number | Per-episode decay factor for epsilon (0.0–1.0) |
| `epsilon_min` | number | Minimum epsilon value after decay |
 
Implementations **MAY** support additional methods via the extension mechanism (`x_` prefix).
 
#### 6.2 Cold Start Policy
 
The `cold_start` member defines agent behavior when no prior observations exist for an entity. **OPTIONAL.** When omitted, implementations **MUST** use the defaults specified below.
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `strategy` | string | `"optimistic"` | Cold start behavior |
| `initial_prior` | object | `{"alpha": 2, "beta": 1}` | Initial Beta distribution parameters |
| `warmup_period` | integer | 5 | Minimum observations before prior is considered reliable |
| `warmup_behavior` | string | `"constitutional_only"` | Behavior during warmup |
 
`strategy` values:
 
| Value | Description |
|-------|-------------|
| `optimistic` | New entities start with mildly optimistic priors (`alpha > beta`). Encourages exploration. **RECOMMENDED.** |
| `pessimistic` | New entities start with pessimistic priors (`alpha < beta`). Requires evidence before trust. |
| `neutral` | New entities start with uniform priors (`alpha = beta = 1`). No assumption. |
| `constitutional_only` | New entities are only used when Constitutional policy (ADL) explicitly permits them. No exploration of unknown entities. |
 
`warmup_behavior` governs decision-making during the warmup period (fewer than `warmup_period` observations):
 
| Value | Description |
|-------|-------------|
| `constitutional_only` | During warmup, route only according to ADL-declared preferences. No Bayesian selection. |
| `explore_with_guard` | During warmup, allow Bayesian selection but apply the quality gate with a tightened threshold (default: `threshold * 1.5`). |
| `normal` | During warmup, apply normal adaptation. Not recommended for safety-critical deployments. |
 
#### 6.3 Prior Management
 
The `prior_management` member defines how priors are bound to entity versions and when they reset. **OPTIONAL.**
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `version_binding` | string | `"entity_hash"` | What identity priors are bound to |
| `reset_on_version_change` | boolean | true | Whether to reset priors when a new version is detected |
| `reset_state` | object | `{"alpha": 2, "beta": 1}` | Prior state after reset |
 
`version_binding` values:
 
| Value | Description |
|-------|-------------|
| `entity_hash` | Priors are bound to the hash of the entity's binary or configuration (tool AIBOM hash, model checkpoint hash). A new hash triggers reset. **RECOMMENDED.** |
| `semantic_version` | Priors are bound to the entity's declared semantic version. A new MAJOR or MINOR version triggers reset; PATCH does not. |
| `name_only` | Priors are bound to the entity name only. No automatic reset. Not recommended. |
 
Rationale: When a tool is patched (v1.1 → v1.2), the runtime's accumulated negative priors from v1.1 must not prevent exploration of the potentially-fixed v1.2. Binding priors to the entity hash ensures that a new deployment is evaluated on its own merits.
 
#### 6.4 Availability vs. Quality Separation
 
The `signal_separation` member defines how the adaptation engine distinguishes availability failures from quality failures. **OPTIONAL.** When omitted, implementations **MUST** apply the defaults.
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `availability_failures_update_priors` | boolean | `false` | Whether availability failures (timeout, connection refused, rate limit) update Beta priors |
| `availability_cooldown_sec` | integer | 300 | Duration to remove an entity from the feasible set after an availability failure |
| `availability_failure_types` | array | `["timeout", "connection_refused", "rate_limited", "service_unavailable"]` | Error types classified as availability failures |
 
**Invariant:** When `availability_failures_update_priors` is `false` (the default and **RECOMMENDED** value), an entity that is temporarily unreachable will be removed from the feasible candidate set for `availability_cooldown_sec` but its Beta prior will remain unchanged. This prevents temporary outages from permanently poisoning learned beliefs about entity quality.
 
Implementations **MUST** classify each failure against `availability_failure_types` before deciding whether to update priors. Unrecognized error types **MUST** be treated as quality failures (the conservative default).
 
---
 
### 7. Feedback Loops & Temporal Decay
 
The `feedback_loops` member configures the multi-timescale learning system. **REQUIRED.** Each timescale operates independently but feeds into the same PolicyCache and prior store.
 
#### 7.1 Immediate Feedback (Per-Event)
 
The `immediate` member configures per-event feedback. **REQUIRED.**
 
##### 7.1.1 Quality Gate
 
The `quality_gate` member defines how individual outputs are scored. **REQUIRED.**
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `threshold_default` | number | REQUIRED | Default minimum quality score (0.0–1.0) |
| `metric` | string | REQUIRED | Quality measurement method |
| `on_failure` | string | REQUIRED | Action when quality is below threshold |
| `threshold_overrides` | array | OPTIONAL | Per-(task_class, entity) threshold overrides |
 
`metric` values:
 
| Value | Description |
|-------|-------------|
| `cosine_similarity` | Embedding distance between task prompt and completion |
| `llm_judge` | Secondary LLM evaluates output quality |
| `regex_match` | Output matches expected structural patterns |
| `composite` | Weighted combination of multiple metrics |
 
`on_failure` values:
 
| Value | Description |
|-------|-------------|
| `update_prior_and_log` | Update the entity's Beta prior (failures += 1) and write a DecisionTrace entry. **RECOMMENDED.** |
| `update_prior_and_retry` | Update prior, log, and retry with a different entity. |
| `log_only` | Write a DecisionTrace entry but do not update priors. For observation-only deployments. |
 
`threshold_overrides` entries:
 
| Member | Type | Description |
|--------|------|-------------|
| `task_class` | string | Task class identifier (e.g., `"code_generation"`, `"summarization"`) |
| `entity` | string | Entity identifier (optional — if omitted, applies to all entities for this task class) |
| `threshold` | number | Override threshold for this (task_class, entity) pair |
 
Example:
 
```json
{
  "quality_gate": {
    "threshold_default": 0.7,
    "metric": "composite",
    "on_failure": "update_prior_and_retry",
    "threshold_overrides": [
      { "task_class": "code_generation", "threshold": 0.85 },
      { "task_class": "summarization", "threshold": 0.6 },
      { "task_class": "code_generation", "entity": "model:deepseek-r1", "threshold": 0.9 }
    ]
  }
}
```
 
#### 7.2 Short-Term Feedback (Per-Task)
 
The `short_term` member configures PolicyCache behavior. **REQUIRED.**
 
##### 7.2.1 PolicyCache Configuration
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `default_ttl_sec` | integer | REQUIRED | Default time-to-live for PolicyCache entries |
| `decay_strategy` | string | REQUIRED | How entries lose influence over time |
| `decay_half_life_sec` | integer | CONDITIONAL | Required for `exponential` decay |
| `human_source_exempt_from_decay` | boolean | OPTIONAL | Whether entries with `source: human` bypass decay. Default: `true` |
| `incident_source_quarantine_sec` | integer | OPTIONAL | Duration that `source: incident_response` entries bypass decay |
 
`decay_strategy` values:
 
| Value | Description |
|-------|-------------|
| `exponential` | Entry influence halves every `decay_half_life_sec`. **RECOMMENDED.** |
| `linear` | Entry influence decreases linearly over `default_ttl_sec`. |
| `step` | Entry has full influence until `default_ttl_sec`, then drops to zero. |
| `none` | Entries never decay. Not recommended for production. |
 
#### 7.3 Medium-Term Feedback (Gossip)
 
The `gossip` member configures peer-to-peer policy propagation across the agent mesh. **OPTIONAL.** When omitted, agents operate in isolation with no shared learning.
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `propagation_interval_sec` | integer | REQUIRED | Minimum interval between gossip broadcasts |
| `peer_discount_factor` | number | REQUIRED | Multiplier applied to peer-reported lessons (0.0–1.0). Lower values = less trust in peer observations. |
| `require_did_signature` | boolean | REQUIRED | Whether gossip messages must be DID-signed |
 
##### 7.3.1 Sybil Resistance
 
The `quorum` member defines Byzantine fault tolerance for gossip. **REQUIRED** when `gossip` is present.
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `min_trusted_peers` | integer | REQUIRED | Minimum corroborating signals before a gossip lesson is accepted |
| `trusted_registry` | string | REQUIRED | DID or URI of the trusted peer registry |
| `untrusted_did_weight` | number | REQUIRED | Weight assigned to gossip from DIDs not in the trusted registry (0.0 RECOMMENDED) |
| `new_did_probation_period_sec` | integer | REQUIRED | Duration before a new DID can contribute to quorum |
 
An agent **MUST NOT** update its own PolicyCache based on peer gossip unless it receives corroborating signals from at least `min_trusted_peers` distinct DIDs listed in `trusted_registry`. DIDs not in the trusted registry receive `untrusted_did_weight` (typically 0.0, meaning they are ignored). Newly registered DIDs **MUST** complete `new_did_probation_period_sec` before their gossip counts toward quorum.
 
#### 7.4 Long-Term Feedback (Consolidation)
 
The `consolidation` member configures the offline learning phase that processes accumulated PolicyCache entries into stable axioms. **OPTIONAL.**
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `schedule` | string | REQUIRED | Cron-style schedule or interval (e.g., `"daily_02:00Z"`, `"every_6h"`) |
| `runner` | string | REQUIRED | Where consolidation executes (`"conductor"`, `"dedicated_node"`) |
| `model_preference` | string | OPTIONAL | Model selection strategy for consolidation (`"cheapest_available"` RECOMMENDED) |
| `operations` | array | REQUIRED | Ordered list of consolidation operations |
| `axiom_store` | object | REQUIRED | Axiom store configuration |
| `require_signature` | boolean | OPTIONAL | Whether consolidation outputs must be signed by the runner's DID. Default: `true` |
 
`operations` values (executed in declared order):
 
| Value | Description |
|-------|-------------|
| `lesson_deduplication` | Merge PolicyCache entries that describe the same underlying observation |
| `pattern_extraction` | Identify recurring patterns across deduplicated lessons |
| `contradiction_detection` | Find axioms or lessons that contradict each other |
| `axiom_falsifiability_check` | Test existing axioms against recent observations; invalidate falsified axioms |
| `curiosity_task_generation` | Generate exploratory tasks to probe gaps in learned knowledge |
 
`axiom_store` configuration:
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `max_axioms` | integer | REQUIRED | Maximum number of active axioms |
| `falsification_window_days` | integer | REQUIRED | Duration over which an axiom must survive contradiction checks |
| `require_consolidation_signature` | boolean | OPTIONAL | Whether axiom writes must be signed. Default: `true` |
 
#### 7.5 Retry & Circuit Breaker Policy
 
The `resilience` member configures retry behavior and circuit breakers across all entity types. **OPTIONAL.** When omitted, implementations **MUST** use reasonable defaults (max 3 retries, exponential backoff, 5-failure circuit breaker).
 
##### 7.5.1 Retry Policy
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `max_retries_default` | integer | 3 | Default maximum retry attempts |
| `backoff_strategy` | string | `"exponential"` | `"fixed"`, `"linear"`, `"exponential"` |
| `initial_delay_ms` | integer | 500 | First retry delay |
| `max_delay_ms` | integer | 30000 | Maximum delay between retries |
| `retry_budget_per_minute` | integer | 20 | Maximum total retries across all entities per minute. Prevents retry storms. |
| `entity_overrides` | array | OPTIONAL | Per-entity retry configuration |
 
##### 7.5.2 Circuit Breaker
 
| Member | Type | Default | Description |
|--------|------|---------|-------------|
| `failure_threshold` | integer | 5 | Consecutive failures before circuit opens |
| `recovery_timeout_sec` | integer | 60 | Time before half-open probe |
| `success_threshold` | integer | 2 | Consecutive successes in half-open state to close circuit |
| `entity_overrides` | array | OPTIONAL | Per-entity circuit breaker configuration |
| `report_to_gossip` | boolean | `true` | Whether circuit breaker state changes are gossip-propagated |
 
Circuit breaker state transitions are first-class DecisionTrace events. When a circuit opens, the entity is removed from the feasible set (availability signal, not quality signal — priors are not updated unless `signal_separation.availability_failures_update_priors` is `true`).
 
---
 
### 8. Order of Precedence & Conflict Resolution
 
When multiple policy constraints apply simultaneously, implementations **MUST** resolve conflicts according to the following precedence order (highest priority first):
 
1. **Cryptographic / Authentication blocks** — If an entity fails signature verification, credential validation, or DID resolution, it is rejected regardless of all other policy.
2. **Budget constraints** — If a budget ceiling or velocity limit is exceeded, the action is halted regardless of retry policy, circuit breaker state, or quality scores.
3. **Escalation actions** — Active escalation rules (§14) override layer-specific defaults.
4. **Quarantine** — Quarantined entities are excluded from the feasible set regardless of their prior quality scores.
5. **Sandbox / Resource limits** — ADL-declared resource limits (Constitutional) take precedence over ARP-declared adaptive limits (Operational). ARP may tighten but never loosen.
6. **Circuit breaker** — An open circuit prevents invocation regardless of retry policy.
7. **Retry policy** — Governs re-attempts within the constraints set by all higher-precedence rules.
8. **Adaptation engine** — Thompson Sampling / UCB1 / epsilon-greedy selection operates only on the feasible set remaining after all higher-precedence filters.
 
**The One-Way Ratchet Rule:** At every precedence level, automated adaptation **MUST** only move policy in the restrictive direction. An automated process **MUST NOT** relax a budget constraint, lift a quarantine, open a sandbox path, or lower a classification level. Any such relaxation **REQUIRES** human approval via the HITL interface (§15) and **MUST** be logged as a manual event in the DecisionTrace.
 
**Cross-constraint conflict example:** A circuit breaker says "retry tool in 30 seconds." The budget layer says "velocity constraint exceeded." Resolution: budget (precedence 2) overrides circuit breaker (precedence 6). The tool is not retried; the task is halted or degraded per the budget exhaustion policy.
 
---
 
## Part III: Layer-Specific Adaptation Rules
 
Each section in Part III corresponds to one layer of the 4-Layer Agentic Policy Stack. All members in Part III are **OPTIONAL.** When a layer section is omitted, the runtime applies Constitutional policy (ADL) for that layer with no runtime adaptation.
 
### 9. Layer 1: Cognitive & Intent Adaptation
 
The `cognitive` member configures runtime adaptation of the agent's reasoning loop, goal adherence, and semantic boundaries.
 
#### 9.1 Context Inspection & Sanitization
 
The `context_inspection` member defines runtime content analysis rules. These rules operate *in addition to* any Constitutional content policies declared in the ADL document.
 
##### 9.1.1 PII Detection
 
| Member | Type | Description |
|--------|------|-------------|
| `pattern_registry` | array | Named patterns with regex, sensitivity level, and action |
| `jurisdiction_profiles` | array | Pre-defined pattern sets for jurisdictions (e.g., `"us"`, `"eu"`, `"au"`) |
| `runtime_teachable` | boolean | Whether new patterns can be added via the HITL teaching interface (§15.2). Default: `true` |
 
Each pattern entry:
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `name` | string | REQUIRED | Human-readable pattern identifier |
| `regex` | string | REQUIRED | Regular expression for detection |
| `sensitivity` | string | REQUIRED | Data classification level triggered on match (`public`, `internal`, `confidential`, `restricted`) |
| `action` | string | REQUIRED | `"block"` (halt and report), `"redact"` (mask and continue), `"log"` (record and continue) |
 
##### 9.1.2 Injection Detection
 
| Member | Type | Description |
|--------|------|-------------|
| `sql_keywords` | array | SQL keywords to detect in agent I/O. Configurable per database type. |
| `shell_commands` | array | Shell commands to detect. Allow/deny semantics. |
| `base64_detection_threshold` | integer | Minimum length of base64-encoded strings to flag. Default: 20. |
| `custom_injection_patterns` | array | Organization-specific injection detection patterns |
 
##### 9.1.3 Input/Output Length Constraints
 
| Member | Type | Description |
|--------|------|-------------|
| `max_input_chars` | integer | Maximum input length in characters |
| `max_output_chars` | integer | Maximum output length in characters |
 
#### 9.2 Drift Detection
 
| Member | Type | Description |
|--------|------|-------------|
| `metric` | string | How semantic drift is measured (`"cosine_distance"`, `"embedding_divergence"`) |
| `threshold` | number | Maximum allowed drift before halt (0.0–1.0) |
| `sample_interval_tokens` | integer | How frequently drift is sampled during generation |
| `on_drift_exceeded` | string | `"halt_and_report"`, `"request_reanchor"`, `"escalate_to_hitl"` |
 
The `threshold` value is a PolicyCache entry, not a hardcoded constant. It **MAY** be tightened by the feedback loop (e.g., when cross-layer escalation from Layer 2 triggers drift threshold tightening) but **MUST NOT** be loosened without HITL approval.
 
---
 
### 10. Layer 2: Execution & Tooling Adaptation
 
The `execution` member configures runtime adaptation of tool invocation, sandbox boundaries, and supply chain integrity.
 
#### 10.1 Sandbox Resource Adaptation
 
| Member | Type | Description |
|--------|------|-------------|
| `default_limits` | object | Default sandbox limits (memory_limit_mb, cpu_limit_percent, timeout_sec) |
| `entity_overrides` | array | Per-(tool_name, tool_version) limit overrides |
| `adaptive_limits` | object | Feedback-driven limit adjustment rules |
 
`adaptive_limits` configuration:
 
| Member | Type | Description |
|--------|------|-------------|
| `ram_highwater_action` | object | Action when RAM usage exceeds a percentage of the limit |
| `cpu_spike_action` | object | Action when CPU usage exceeds a percentage of the limit |
| `duration_trending_action` | object | Action when execution duration trends upward |
 
Each action object:
 
| Member | Type | Description |
|--------|------|-------------|
| `trigger_threshold_pct` | number | Percentage of limit that triggers action (e.g., 0.9 for 90%) |
| `action` | string | `"tighten_limit"`, `"log_warning"`, `"quarantine_tool"` |
| `tighten_factor` | number | Factor by which to reduce the limit (e.g., 0.8 = reduce to 80%) |
 
Adaptive limits **MUST** only tighten. A tool whose RAM limit was tightened from 512MB to 410MB by the feedback loop cannot have its limit raised back to 512MB without HITL approval. The Constitutional limit declared in the ADL document remains the upper bound.
 
#### 10.2 Tool Risk Governance
 
| Member | Type | Description |
|--------|------|-------------|
| `risk_levels` | array | Defined risk levels with names and ordinal ranking |
| `classifications` | array | Per-tool-pattern risk assignments |
| `obligations` | object | Per-risk-level obligation requirements |
| `runtime_promotion` | object | Rules for feedback-driven risk promotion |
 
Default `risk_levels`:
 
| Level | Ordinal | Description |
|-------|---------|-------------|
| `low` | 0 | Read-only operations |
| `medium` | 1 | Build, test, package operations |
| `high` | 2 | Shell execution, file writes, network access |
| `critical` | 3 | System modification, privilege escalation |
 
Each `classifications` entry:
 
| Member | Type | Description |
|--------|------|-------------|
| `tool_pattern` | string | Glob pattern matching tool names |
| `risk` | string | Assigned risk level |
| `requires_hitl` | boolean | Whether this tool requires human confirmation before invocation |
| `version_binding` | string | How risk classification binds to tool version (same values as §6.3) |
 
Each `obligations` entry (keyed by risk level):
 
| Member | Type | Description |
|--------|------|-------------|
| `sandbox` | boolean | Whether tool must run in sandbox |
| `audit_log` | boolean | Whether invocation must be logged to DecisionTrace |
| `notify_conductor` | boolean | Whether conductor must be notified |
| `require_hitl` | boolean | Whether human confirmation is required |
 
`runtime_promotion` configuration:
 
| Member | Type | Description |
|--------|------|-------------|
| `sandbox_violation_promotes_to` | string | Risk level assigned when a tool triggers a sandbox violation |
| `consecutive_failure_threshold` | integer | Failures before risk promotion |
| `promotes_by` | integer | How many ordinal levels to promote on threshold breach (default: 1) |
 
Runtime promotion is one-way. A tool promoted from `medium` to `high` by the feedback loop cannot be demoted without HITL approval.
 
#### 10.3 Allowed-Path Adaptation
 
| Member | Type | Description |
|--------|------|-------------|
| `path_removal_on_violation` | boolean | Whether a path is removed from allowed set when a tool accesses it in a policy-violating way. Default: `true` |
| `path_addition_requires` | string | `"hitl_approval"` (RECOMMENDED), `"constitutional_only"` (paths can only come from ADL) |
 
Paths are removable by feedback but **MUST NOT** be addable by automated processes. Only HITL or Constitutional policy can add paths.
 
---
 
### 11. Layer 3: Data Sovereignty Adaptation
 
The `data_sovereignty` member configures runtime adaptation of data access, classification, and retention.
 
#### 11.1 Authorization Caching
 
| Member | Type | Description |
|--------|------|-------------|
| `default_ttl_sec` | integer | Default cache TTL for authorization decisions |
| `sensitivity_multipliers` | object | Multipliers applied to TTL based on data sensitivity |
| `max_cache_entries` | integer | Maximum cache size |
| `invalidation_triggers` | array | Events that force cache invalidation |
 
`sensitivity_multipliers` defines shorter TTLs for more sensitive data:
 
```json
{
  "sensitivity_multipliers": {
    "public": 1.0,
    "internal": 0.5,
    "confidential": 0.25,
    "restricted": 0.1
  }
}
```
 
Effective TTL = `default_ttl_sec * sensitivity_multipliers[level]`. For `default_ttl_sec: 60` and sensitivity `restricted`, effective TTL = 6 seconds.
 
`invalidation_triggers` values:
 
| Value | Description |
|-------|-------------|
| `classification_change` | A data source's classification level changed |
| `escalation_event` | A cross-layer escalation was triggered |
| `credential_revocation` | A Verifiable Credential was revoked |
| `quarantine_event` | A data source was quarantined |
 
#### 11.2 Adaptive Data Reclassification
 
| Member | Type | Description |
|--------|------|-------------|
| `pii_frequency_escalation` | object | Rules for escalating classification when PII is detected above a threshold |
| `context_poisoning_detection` | object | Rules for detecting output quality degradation correlated with data source ingestion |
| `reclassification_scope` | string | `"per_task_class"`, `"per_data_source"`, `"per_agent"` |
 
`pii_frequency_escalation`:
 
| Member | Type | Description |
|--------|------|-------------|
| `window_sec` | integer | Observation window for PII detection frequency |
| `threshold_count` | integer | Number of PII detections in window before escalation |
| `escalation_direction` | string | **MUST** be `"up_one_level"`. Automated reclassification is always upward. |
| `requires_hitl_above` | string | Sensitivity level above which escalation requires HITL confirmation. Default: `"confidential"` |
 
Reclassification is one-way. A data source escalated from `internal` to `confidential` by the feedback loop cannot be demoted without HITL approval.
 
---
 
### 12. Layer 4: Network & Mesh Adaptation
 
The `network` member configures runtime adaptation of inter-agent communication, egress filtering, and delegation routing.
 
#### 12.1 Adaptive Egress Filtering
 
| Member | Type | Description |
|--------|------|-------------|
| `dynamic_deny_entries` | object | Rules for adding deny entries from feedback |
| `deny_takes_precedence` | boolean | Whether deny rules override allow rules. **MUST** be `true`. |
| `dynamic_deny_ttl_sec` | integer | Default TTL for feedback-generated deny entries |
 
`dynamic_deny_entries` triggers:
 
| Member | Type | Description |
|--------|------|-------------|
| `on_connection_failure_count` | integer | Consecutive connection failures before adding deny entry |
| `on_tls_error` | boolean | Whether TLS errors trigger a deny entry |
| `on_suspicious_response` | boolean | Whether responses matching injection patterns trigger a deny entry |
 
Dynamic deny entries are removable by TTL expiration or HITL approval. Static deny entries (from ADL) are never removable.
 
#### 12.2 Peer Routing & Delegation
 
| Member | Type | Description |
|--------|------|-------------|
| `peer_adaptation_method` | string | References §6.1 method. Default: same as top-level `adaptation.method` |
| `delegation_roi_tracking` | boolean | Whether to track cost-vs-quality ROI for delegation decisions |
| `delegation_roi_window_sec` | integer | Observation window for ROI calculation |
| `prefer_local_threshold` | number | ROI below which delegation is abandoned in favor of local execution with a cheaper model |
 
---
 
## Part IV: Constraints, Overrides & Observability
 
### 13. Budget & Throughput Constraints
 
The `budget` member defines monetary and throughput limits. **REQUIRED.** Budget constraints operate at precedence level 2 (§8), overriding all adaptation decisions except cryptographic/authentication blocks.
 
#### 13.1 Task-Level Budget
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `task_ceiling_usd` | number | REQUIRED | Maximum total spend per task |
| `on_exhaustion` | string | REQUIRED | Behavior when task budget is exhausted |
| `degradation_chain` | array | OPTIONAL | Ordered fallback sequence before final exhaustion action |
| `alert_threshold_pct` | number | OPTIONAL | Percentage of budget that triggers an alert (default: 80) |
 
`on_exhaustion` values:
 
| Value | Description |
|-------|-------------|
| `halt_and_report` | Stop the task, write a DecisionTrace entry with full cost breakdown. **RECOMMENDED.** |
| `degrade_to_cheapest_model` | Switch to the lowest-cost model available, continue task. |
| `queue_for_human_approval` | Pause the task, request HITL approval to continue with additional budget. |
 
`degradation_chain` is an ordered sequence of fallback actions. The runtime executes them in order until one succeeds or the chain is exhausted:
 
```json
{
  "degradation_chain": [
    "degrade_to_cheapest_model",
    "disable_optional_tools",
    "halt_and_report"
  ]
}
```
 
#### 13.2 Velocity Constraints
 
| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `max_spend_per_minute_usd` | number | REQUIRED | Maximum monetary spend rate |
| `max_tool_calls_per_minute` | integer | OPTIONAL | Maximum tool invocations per minute |
| `max_tokens_per_minute` | integer | OPTIONAL | Maximum token consumption per minute |
 
Velocity constraints act as circuit breakers for runaway agent behavior. A burst of 10,000 cheap API calls in 10 seconds may individually pass the per-call budget but collectively trigger the velocity constraint, halting the agent before catastrophic spend or data exfiltration.
 
#### 13.3 Per-Layer Limits
 
| Member | Type | Description |
|--------|------|-------------|
| `cognitive` | object | `per_completion_limit_usd` |
| `execution` | object | `per_tool_call_limit_usd` |
| `data` | object | `per_retrieval_limit_usd` |
| `network` | object | `per_delegation_limit_usd` |
 
#### 13.4 Rate Limiting
 
| Member | Type | Description |
|--------|------|-------------|
| `global` | object | Global rate limits (requests_per_second, burst_size) |
| `per_endpoint` | array | Per-endpoint rate limit overrides |
| `per_tool` | array | Per-tool rate limit overrides |
| `max_tracking_entries` | integer | Maximum number of tracked rate limit entries (IP, session, etc.) |
| `tracking_entry_ttl_sec` | integer | TTL for rate limit tracking entries |
| `cleanup_interval_sec` | integer | Interval for purging expired tracking entries |
 
Each rate limit entry:
 
| Member | Type | Description |
|--------|------|-------------|
| `requests_per_second` | number | Maximum request rate |
| `burst_size` | integer | Maximum burst above steady-state rate |
 
#### 13.5 Accounting
 
| Member | Type | Description |
|--------|------|-------------|
| `trace_id_ref` | string | JSON Pointer to the DecisionTrace trace_id for cost attribution |
| `attribution` | string | `"per_agent"` (each agent tracks its own spend) or `"per_task"` (spend tracked at task level) |
| `rollup_window_sec` | integer | Aggregation window for spend reporting |
 
---
 
### 14. Escalation & Quarantine
 
#### 14.1 Cross-Layer Escalation
 
The `escalation` member defines rules that trigger policy tightening across layer boundaries. **OPTIONAL.**
 
| Member | Type | Description |
|--------|------|-------------|
| `cross_layer_rules` | array | Escalation trigger-action pairs |
| `ratchet_direction` | string | **MUST** be `"tighten_only"` |
| `relaxation_requires` | string | **MUST** be `"human_approval"` |
 
Each escalation rule:
 
| Member | Type | Description |
|--------|------|-------------|
| `trigger` | object | `{ "layer": "...", "event": "..." }` |
| `actions` | array | `[{ "target_layer": "...", "action": "...", "parameters": {...} }]` |
| `confidence_threshold` | number | Minimum confidence of the triggering signal for this rule to fire (0.0–1.0) |
 
Standard escalation events:
 
| Source Layer | Event | Example Target Actions |
|-------------|-------|----------------------|
| execution | `sandbox_violation` | cognitive: tighten drift threshold; network: add egress deny for tool process |
| data_sovereignty | `pii_frequency_exceeded` | data_sovereignty: escalate classification; cognitive: inject redaction reminder |
| network | `peer_returns_manipulative_content` | cognitive: quarantine peer; data_sovereignty: run context-poisoning check |
| cognitive | `drift_threshold_exceeded` | execution: suspend tool invocations until reanchored |
| budget | `budget_exhausted` | all layers: halt task, report cost breakdown |
| cognitive | `repeated_quality_failures` | network: quarantine delegation source if failures correlate with delegated context |
 
Escalation events are first-class DecisionTrace entries with `event_type: "escalation"` and references to both source and target layers.
 
#### 14.2 Quarantine
 
| Member | Type | Description |
|--------|------|-------------|
| `default_ttl_sec` | integer | Default quarantine duration |
| `confidence_threshold_for_sticky` | number | Signal confidence above which quarantine is sticky (no TTL expiration). Default: 0.7 |
| `ttl_formula` | string | Formula for confidence-scaled TTL. Default: `"default_ttl * (1 - confidence)"` |
| `sticky_requires` | string | What releases a sticky quarantine. **MUST** be `"human_review"` |
| `scopes` | array | Entity types that can be quarantined: `"tool"`, `"model"`, `"peer"`, `"data_source"`, `"endpoint"` |
 
Quarantine is an availability-level exclusion. A quarantined entity is removed from the feasible set. Whether quarantine also updates quality priors depends on `signal_separation.availability_failures_update_priors` (§6.4).
 
---
 
### 15. Human-In-The-Loop (HITL) & Session Management
 
HITL is the ultimate failsafe for the one-way ratchet and the cornerstone of compliance with the EU AI Act's Article 14 (Human Oversight). This section defines when and how human intervention occurs.
 
#### 15.1 Ratchet Relaxation
 
| Member | Type | Description |
|--------|------|-------------|
| `relaxation_conditions` | array | Conditions under which HITL relaxation is permitted |
| `approval_mechanism` | string | How HITL approval is captured (`"credential_presentation"`, `"signed_command"`, `"interactive_confirmation"`) |
| `audit_requirement` | string | `"always_logged"` (REQUIRED — every relaxation is a DecisionTrace event) |
 
Each `relaxation_conditions` entry:
 
| Member | Type | Description |
|--------|------|-------------|
| `scope` | string | What can be relaxed (`"quarantine"`, `"classification"`, `"budget_ceiling"`, `"risk_level"`, `"drift_threshold"`) |
| `required_credential` | string | Verifiable Credential type required for approval |
| `required_role` | string | Organizational role required (e.g., `"security_officer"`, `"data_privacy_officer"`) |
 
#### 15.2 Teaching Interface
 
| Member | Type | Description |
|--------|------|-------------|
| `enabled` | boolean | Whether runtime teaching is active. Default: `true` |
| `intent_classification` | array | Recognized teaching intents |
| `policy_cache_write` | object | How teaching inputs are written to PolicyCache |
 
`intent_classification` values:
 
| Intent | Description | PolicyCache Action |
|--------|-------------|-------------------|
| `instruction` | "Always prefer model X for task Y" | Write entry with `source: human`, exempt from decay |
| `correction` | "That last response was wrong because..." | Update prior for the entity that produced the response |
| `reinforcement` | "That was a good response" | Increment success count on the entity's prior |
| `pattern` | "When you see format X, treat it as PII" | Add pattern to context inspection registry |
 
Teaching inputs **MUST** be written to PolicyCache with `source: human` and bypass normal decay/demotion logic (per §7.2.1 `human_source_exempt_from_decay`).
 
#### 15.3 Session Management
 
| Member | Type | Description |
|--------|------|-------------|
| `timeout_bounds` | object | Min/max constraints on session timeouts |
| `defaults` | object | Default timeout values |
| `sensitivity_proportional` | object | Rules for adjusting timeouts based on data sensitivity |
 
`timeout_bounds`:
 
| Member | Type | Description |
|--------|------|-------------|
| `absolute_timeout_min_sec` | integer | Minimum allowed absolute timeout |
| `absolute_timeout_max_sec` | integer | Maximum allowed absolute timeout |
| `idle_timeout_max_sec` | integer | Maximum allowed idle timeout |
 
`sensitivity_proportional`:
 
| Member | Type | Description |
|--------|------|-------------|
| `enabled` | boolean | Whether session timeouts scale with data sensitivity |
| `idle_timeout_multipliers` | object | Per-sensitivity-level multipliers applied to default idle timeout |
 
```json
{
  "idle_timeout_multipliers": {
    "public": 1.0,
    "internal": 1.0,
    "confidential": 0.5,
    "restricted": 0.25
  }
}
```
 
For `restricted` data with a default idle timeout of 900 seconds, effective idle timeout = 225 seconds.
 
---
 
### 16. State Storage
 
The `state_storage` member declares where learned state (PolicyCache entries, prior distributions, axioms) is persisted. **OPTIONAL.** When present, this enables auditors to verify that learned state has not been tampered with outside the ARP's rules.
 
#### 16.1 PolicyCache Backend
 
| Member | Type | Description |
|--------|------|-------------|
| `backend` | string | Storage type (`"iroh_content_addressed"`, `"local_encrypted"`, `"external_kv"`) |
| `integrity` | string | Tamper detection mechanism (`"hash_chain"`, `"merkle_verified"`, `"none"`) |
| `encryption_at_rest` | boolean | Whether stored entries are encrypted |
 
#### 16.2 Prior Distributions
 
| Member | Type | Description |
|--------|------|-------------|
| `backend` | string | Storage type |
| `snapshot_interval_sec` | integer | How frequently prior state is snapshotted |
| `snapshot_signed_by` | string | DID that signs snapshots (typically the agent's own DID) |
 
#### 16.3 Axiom Store
 
| Member | Type | Description |
|--------|------|-------------|
| `backend` | string | Storage type |
| `requires_consolidation_signature` | boolean | Whether axiom writes must be signed by the consolidation runner |
 
---
 
### 17. Observability & Auditability
 
#### 17.1 DecisionTrace Schema
 
The `decision_trace` member configures the append-only audit log. **OPTIONAL** (but **STRONGLY RECOMMENDED** for any deployment with `data_classification.sensitivity >= internal`).
 
| Member | Type | Description |
|--------|------|-------------|
| `enabled` | boolean | Whether DecisionTrace logging is active |
| `storage` | string | **MUST** be `"append_only"` |
| `retention_days` | integer | Minimum retention period for trace entries |
 
##### 17.1.1 Trace Entry Schema
 
Every policy enforcement event writes a trace entry with the following structure:
 
```json
{
  "trace_id": "uuid",
  "timestamp": "ISO8601",
  "task_id": "uuid",
  "agent_id": "did:web:...",
  "layer": "cognitive | execution | data_sovereignty | network",
  "event_type": "routing_decision | quality_gate | tool_invocation | data_access | delegation | escalation | budget_event | quarantine | hitl_action",
  "decision": {
    "chosen": "entity identifier",
    "alternatives_considered": ["..."],
    "selection_method": "thompson_sampling | policy_cache | static_rule | hitl_override",
    "prior_state": { "alpha": 0.0, "beta": 0.0 },
    "posterior_state": { "alpha": 0.0, "beta": 0.0 }
  },
  "outcome": {
    "success": true,
    "quality_score": 0.0,
    "latency_ms": 0,
    "cost_usd": 0.00,
    "error_type": null
  },
  "budget": {
    "remaining_task_budget_usd": 0.00,
    "spend_this_event_usd": 0.00,
    "layer_spend_cumulative_usd": 0.00,
    "velocity_spend_this_minute_usd": 0.00
  },
  "escalation": {
    "triggered": false,
    "source_layer": null,
    "target_layers": [],
    "action_taken": null,
    "confidence": null
  },
  "feedback": {
    "prior_updated": true,
    "policy_cache_written": true,
    "gossip_propagated": false,
    "lesson_key": null
  }
}
```
 
**Credit assignment rule:** The `decision` block records who chose what and why. The `outcome` block records what happened. The `feedback` block records how the system learned from it. These three are independently attributable — routing quality and execution quality are never conflated.
 
##### 17.1.2 Cryptographic Signing
 
| Member | Type | Description |
|--------|------|-------------|
| `signing_required_above_sensitivity` | string | Sensitivity level above which trace entries must be signed. Default: `"confidential"` |
| `algorithm` | string | Signature algorithm. Ed25519 RECOMMENDED. |
| `signed_by` | string | DID that signs trace entries (typically the agent's own DID) |
 
When signing is required, each trace entry **MUST** include:
 
```json
{
  "cryptographic_proofs": {
    "agent_signature": "ed25519_sig_base64url",
    "vc_presented": "did:web:.../credentials/123",
    "human_approval_signature": null
  }
}
```
 
`human_approval_signature` is populated when the event is a HITL action (relaxation, teaching input, manual override).
 
#### 17.2 Export Formats
 
| Member | Type | Description |
|--------|------|-------------|
| `siem_export` | object | Configuration for SIEM integration (format, endpoint, frequency) |
| `grc_export` | object | Configuration for GRC/compliance reporting (format, mapping) |
| `regulatory_export` | object | Configuration for regulatory consumption (EU AI Act Article 14 transparency reports) |
 
Each export configuration:
 
| Member | Type | Description |
|--------|------|-------------|
| `format` | string | `"json_lines"`, `"csv"`, `"otel_traces"`, `"custom"` |
| `endpoint` | string | URI for export delivery |
| `frequency` | string | Export frequency (`"realtime"`, `"hourly"`, `"daily"`) |
| `filter` | object | Which trace entries to export (by layer, event_type, sensitivity) |
 
---
 
## Part V: Compliance, Security & Administration
 
### 18. Compliance Mapping
 
This section maps ARP members to specific compliance framework controls. Implementations **SHOULD** reference these mappings when generating compliance documentation.
 
#### 18.1 EU AI Act
 
| Article | ARP Coverage |
|---------|-------------|
| Article 9 (Risk Management) | Tool risk governance (§10.2), cross-layer escalation (§14.1), quality gate (§7.1.1) |
| Article 14 (Human Oversight) | HITL (§15), ratchet relaxation (§15.1), degradation chain (§13.1) as fallback mechanism |
| Article 15 (Accuracy, Robustness, Cybersecurity) | Adaptation engine (§6), quality gate (§7.1.1), integrity (§5), state storage tamper detection (§16) |
| Article 52 (Transparency) | DecisionTrace (§17.1), export formats (§17.2), drift detection (§9.2) as explainability substrate |
 
#### 18.2 NIST AI RMF
 
| Function | ARP Coverage |
|----------|-------------|
| GOVERN | HITL (§15), precedence rules (§8), integrity (§5) |
| MAP | DecisionTrace (§17.1) — records all routing decisions with alternatives considered |
| MEASURE | Quality gate (§7.1.1) as continuous runtime evaluation, feedback loops (§7) as measurement system |
| MANAGE | Escalation (§14), quarantine (§14.2), budget constraints (§13), degradation chains (§13.1) |
 
#### 18.3 NIST 800-53
 
| Control Family | ARP Coverage |
|---------------|-------------|
| AC (Access Control) | Authorization caching (§11.1), data reclassification (§11.2), session management (§15.3) |
| AU (Audit) | DecisionTrace (§17.1), cryptographic signing (§17.1.2), export formats (§17.2) |
| CM (Configuration Management) | ARP document integrity (§5), ADL binding (§4.5), version binding (§6.3) |
| IR (Incident Response) | Cross-layer escalation (§14.1), quarantine (§14.2), HITL (§15) |
| RA (Risk Assessment) | Tool risk governance (§10.2), runtime risk promotion (§10.2) |
| SC (System & Communications Protection) | Egress filtering (§12.1), peer routing (§12.2), budget velocity (§13.2) |
| SI (System & Information Integrity) | Context inspection (§9.1), PII detection (§9.1.1), drift detection (§9.2) |
 
#### 18.4 SOC 2
 
| Criteria | ARP Coverage |
|----------|-------------|
| CC6 (Logical and Physical Access) | Authorization caching (§11.1), sensitivity-proportional TTLs (§11.1) |
| CC7 (System Operations) | Feedback loops (§7), circuit breakers (§7.5.2), rate limiting (§13.4) |
| CC8 (Change Management) | Version binding (§6.3), prior reset on version change (§6.3), ARP integrity (§5) |
 
#### 18.5 ISO 42001
 
ARP as a complete document serves as a reference implementation for ISO 42001's requirement for an AI management system. The adaptation engine (§6) maps to the AI lifecycle management requirement. The feedback loops (§7) map to the continuous improvement requirement. The HITL interface (§15) maps to the human oversight requirement. The DecisionTrace (§17) maps to the monitoring and measurement requirement.
 
---
 
### 19. Security Considerations
 
#### 19.1 ARP Document Tampering
 
An attacker who modifies an ARP document can disable the one-way ratchet, set `exploration_floor` to 1.0 (forcing random entity selection), disable quality gates, or set budget ceilings to unlimited values. Implementations **MUST** verify the `integrity` signature (§5) before loading an ARP document in production environments. ARP documents without `integrity` **SHOULD** only be used in development or testing environments. Runtimes **MUST** reject ARP documents whose `integrity` signature does not verify.
 
#### 19.2 State Storage Attacks
 
An attacker with access to the PolicyCache backend could modify learned priors (making a compromised tool appear highly reliable), delete quarantine entries, or inject false axioms. Implementations **MUST** use the tamper detection mechanism declared in `state_storage` (§16). Hash-chained PolicyCache entries ensure that any external modification breaks the chain and is detectable. Implementations **SHOULD** alert on hash chain breaks.
 
#### 19.3 Gossip Poisoning
 
A compromised agent could broadcast false gossip (e.g., "peer B has 0% quality") to manipulate the mesh's routing table. The quorum rules (§7.3.1) require corroboration from multiple trusted DIDs before gossip is accepted. The `untrusted_did_weight` of 0.0 and `new_did_probation_period_sec` prevent Sybil attacks via cheap DID generation. Implementations **SHOULD** monitor for gossip patterns that suggest coordinated reputation manipulation.
 
#### 19.4 Adaptation Gaming
 
An adversary who controls an entity (tool, model, or data source) could manipulate its responses to game the adaptation engine — for example, returning high-quality results during an initial warmup period to build a strong prior, then degrading quality once trusted. The quality gate's `threshold_overrides` (§7.1.1) and the consolidation phase's `contradiction_detection` (§7.4) provide partial defense. Implementations **SHOULD** implement anomaly detection on prior trajectories — a sudden quality drop after a long period of high quality should trigger an escalation.
 
#### 19.5 Budget Exhaustion Attacks
 
A compromised tool or malicious peer could cause rapid budget consumption through expensive operations. The velocity constraints (§13.2) act as a first line of defense, halting spend spikes before the task budget is exhausted. The per-layer limits (§13.3) ensure that a single layer cannot consume the entire budget. Implementations **SHOULD** alert on velocity constraint triggers as potential indicators of compromise.
 
#### 19.6 Quarantine Evasion
 
An entity could evade quarantine by changing its identity (new DID, new tool name). The version binding mechanism (§6.3) mitigates this for tools — priors are bound to entity hash, so a "new" tool with the same binary gets the same priors. For peer agents, the gossip trust registry (§7.3.1) ensures that new DIDs start with zero weight and must complete a probation period. Implementations **SHOULD** track entity lineage (predecessor/successor relationships) to detect identity rotation.
 
---
 
### 20. IANA Considerations
 
#### 20.1 Media Type Registration
 
This document requests IANA to register the `application/arp+json` media type.
 
- **Type name:** application
- **Subtype name:** arp+json
- **Required parameters:** None
- **Encoding considerations:** binary (UTF-8 JSON per [RFC8259])
- **Security considerations:** See §19
- **Published specification:** [this document]
- **File extension(s):** `.arp.json`, `.arp.toml` (authoring only)
 
#### 20.2 ADL Profile Registration
 
This document requests registration of the following profiles in the ADL Profile Registry (ADL §17.2):
 
| Identifier | Name | Version |
|-----------|------|---------|
| `urn:adl:profile:budget:1.0` | ADL Budget Profile | 1.0.0 |
| `urn:adl:profile:data-sovereignty:1.0` | ADL Data Sovereignty Profile | 1.0.0 |
| `urn:adl:profile:tool-governance:1.0` | ADL Tool Governance Profile | 1.0.0 |
 
These profiles define ADL members for budget constraints, PII detection patterns, and tool risk classification that serve as Constitutional policy complements to ARP's Operational policy.
 
---
 
## Appendix A. JSON Schema
 
The normative JSON Schema for ARP 0.1.0 will be published at `https://arp-spec.org/0.1/schema.json` (JSON Schema Draft 2020-12).
 
---
 
## Appendix B. TOML Authoring Examples
 
### B.1 Minimal ARP Document
 
```toml
[document]
arp_spec = "0.1.0"
 
[adl_ref]
uri = "https://acme.example.com/agents/claims-processor/adl.json"
document_hash = "sha256:a1b2c3d4e5f6..."
 
[adaptation]
method = "thompson_sampling"
 
[adaptation.parameters]
exploration_floor = 0.05
 
[feedback_loops.immediate.quality_gate]
threshold_default = 0.7
metric = "cosine_similarity"
on_failure = "update_prior_and_log"
 
[feedback_loops.short_term.policy_cache]
default_ttl_sec = 3600
decay_strategy = "exponential"
decay_half_life_sec = 86400
 
[budget]
task_ceiling_usd = 2.50
on_exhaustion = "halt_and_report"
 
[budget.velocity]
max_spend_per_minute_usd = 0.50
```
 
### B.2 Production ARP Document with All Layers
 
```toml
[document]
arp_spec = "0.1.0"
 
[adl_ref]
uri = "https://acme.example.com/agents/claims-processor/adl.json"
document_hash = "sha256:a1b2c3d4e5f6..."
 
[integrity]
signed_by = "did:web:acme.example.com:governance"
algorithm = "Ed25519"
signature = "base64url_sig"
signed_content = "canonical"
 
# Part II: Adaptation Engine
[adaptation]
method = "thompson_sampling"
 
[adaptation.parameters]
exploration_floor = 0.05
 
[adaptation.cold_start]
strategy = "optimistic"
warmup_period = 5
warmup_behavior = "constitutional_only"
 
[adaptation.cold_start.initial_prior]
alpha = 2
beta = 1
 
[adaptation.prior_management]
version_binding = "entity_hash"
reset_on_version_change = true
 
[adaptation.prior_management.reset_state]
alpha = 2
beta = 1
 
[adaptation.signal_separation]
availability_failures_update_priors = false
availability_cooldown_sec = 300
 
# Feedback Loops
[feedback_loops.immediate.quality_gate]
threshold_default = 0.7
metric = "composite"
on_failure = "update_prior_and_retry"
 
[[feedback_loops.immediate.quality_gate.threshold_overrides]]
task_class = "code_generation"
threshold = 0.85
 
[[feedback_loops.immediate.quality_gate.threshold_overrides]]
task_class = "summarization"
threshold = 0.6
 
[feedback_loops.short_term.policy_cache]
default_ttl_sec = 3600
decay_strategy = "exponential"
decay_half_life_sec = 86400
human_source_exempt_from_decay = true
incident_source_quarantine_sec = 604800
 
[feedback_loops.gossip]
propagation_interval_sec = 300
peer_discount_factor = 0.7
require_did_signature = true
 
[feedback_loops.gossip.quorum]
min_trusted_peers = 3
trusted_registry = "did:web:acme.example.com:mesh:trusted-peers"
untrusted_did_weight = 0.0
new_did_probation_period_sec = 86400
 
[feedback_loops.consolidation]
schedule = "daily_02:00Z"
runner = "conductor"
model_preference = "cheapest_available"
operations = ["lesson_deduplication", "pattern_extraction", "contradiction_detection", "axiom_falsifiability_check"]
require_signature = true
 
[feedback_loops.consolidation.axiom_store]
max_axioms = 1000
falsification_window_days = 30
 
[feedback_loops.resilience.retry]
max_retries_default = 3
backoff_strategy = "exponential"
initial_delay_ms = 500
max_delay_ms = 30000
retry_budget_per_minute = 20
 
[feedback_loops.resilience.circuit_breaker]
failure_threshold = 5
recovery_timeout_sec = 60
success_threshold = 2
report_to_gossip = true
 
# Part III: Layer-Specific Rules
[cognitive.context_inspection]
max_input_chars = 100000
max_output_chars = 50000
 
[[cognitive.context_inspection.pattern_registry]]
name = "ssn"
regex = '\b\d{3}-\d{2}-\d{4}\b'
sensitivity = "restricted"
action = "block"
 
[[cognitive.context_inspection.pattern_registry]]
name = "credit_card"
regex = '\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'
sensitivity = "confidential"
action = "redact"
 
[cognitive.context_inspection.injection_detection]
sql_keywords = ["SELECT", "DROP", "INSERT", "UPDATE", "DELETE", "TRUNCATE", "ALTER"]
shell_commands = ["rm", "sudo", "chmod", "curl", "wget"]
base64_detection_threshold = 20
 
[cognitive.drift_detection]
metric = "cosine_distance"
threshold = 0.15
sample_interval_tokens = 500
on_drift_exceeded = "halt_and_report"
 
[execution.sandbox.default_limits]
memory_limit_mb = 512
cpu_limit_percent = 50
timeout_sec = 60
 
[[execution.sandbox.entity_overrides]]
tool_pattern = "data_processor"
memory_limit_mb = 2048
allowed_paths = ["/data", "/tmp"]
 
[execution.sandbox.adaptive_limits.ram_highwater_action]
trigger_threshold_pct = 0.9
action = "tighten_limit"
tighten_factor = 0.8
 
[[execution.tool_risk.classifications]]
tool_pattern = "file_read*"
risk = "low"
requires_hitl = false
 
[[execution.tool_risk.classifications]]
tool_pattern = "shell_exec*"
risk = "critical"
requires_hitl = true
 
[execution.tool_risk.obligations.high]
sandbox = true
audit_log = true
notify_conductor = true
require_hitl = false
 
[execution.tool_risk.runtime_promotion]
sandbox_violation_promotes_to = "critical"
consecutive_failure_threshold = 5
promotes_by = 1
 
[data_sovereignty.authorization_cache]
default_ttl_sec = 60
max_cache_entries = 1000
 
[data_sovereignty.authorization_cache.sensitivity_multipliers]
public = 1.0
internal = 0.5
confidential = 0.25
restricted = 0.1
 
[data_sovereignty.authorization_cache.invalidation_triggers]
triggers = ["classification_change", "escalation_event", "credential_revocation"]
 
[data_sovereignty.reclassification.pii_frequency_escalation]
window_sec = 300
threshold_count = 5
escalation_direction = "up_one_level"
requires_hitl_above = "confidential"
 
[network.egress.dynamic_deny]
on_connection_failure_count = 3
on_tls_error = true
on_suspicious_response = true
 
[network.egress]
deny_takes_precedence = true
dynamic_deny_ttl_sec = 3600
 
[network.peer_routing]
delegation_roi_tracking = true
delegation_roi_window_sec = 3600
prefer_local_threshold = 0.5
 
# Part IV: Constraints & Overrides
[budget]
task_ceiling_usd = 2.50
on_exhaustion = "halt_and_report"
alert_threshold_pct = 80
degradation_chain = ["degrade_to_cheapest_model", "disable_optional_tools", "halt_and_report"]
 
[budget.velocity]
max_spend_per_minute_usd = 0.50
max_tool_calls_per_minute = 60
max_tokens_per_minute = 50000
 
[budget.per_layer.cognitive]
per_completion_limit_usd = 0.40
 
[budget.per_layer.execution]
per_tool_call_limit_usd = 0.25
 
[budget.per_layer.data]
per_retrieval_limit_usd = 0.10
 
[budget.per_layer.network]
per_delegation_limit_usd = 1.00
 
[budget.rate_limiting.global]
requests_per_second = 100
burst_size = 10
 
[budget.accounting]
attribution = "per_agent"
rollup_window_sec = 3600
 
[[escalation.cross_layer_rules]]
trigger = { layer = "execution", event = "sandbox_violation" }
confidence_threshold = 0.5
actions = [
  { target_layer = "cognitive", action = "tighten_drift_threshold", parameters = { factor = 0.5 } },
  { target_layer = "network", action = "add_egress_deny", parameters = { scope = "tool_process" } }
]
 
[escalation]
ratchet_direction = "tighten_only"
relaxation_requires = "human_approval"
 
[quarantine]
default_ttl_sec = 3600
confidence_threshold_for_sticky = 0.7
ttl_formula = "default_ttl * (1 - confidence)"
sticky_requires = "human_review"
scopes = ["tool", "model", "peer", "data_source", "endpoint"]
 
[hitl.relaxation]
audit_requirement = "always_logged"
 
[hitl.teaching]
enabled = true
 
[session.timeout_bounds]
absolute_timeout_min_sec = 60
absolute_timeout_max_sec = 86400
idle_timeout_max_sec = 14400
 
[session.defaults]
absolute_timeout_sec = 3600
idle_timeout_sec = 900
 
[session.sensitivity_proportional]
enabled = true
 
[session.sensitivity_proportional.idle_timeout_multipliers]
public = 1.0
internal = 1.0
confidential = 0.5
restricted = 0.25
 
[state_storage.policy_cache]
backend = "iroh_content_addressed"
integrity = "hash_chain"
encryption_at_rest = true
 
[state_storage.prior_distributions]
backend = "local_encrypted"
snapshot_interval_sec = 300
snapshot_signed_by = "agent_did"
 
[state_storage.axiom_store]
backend = "iroh_content_addressed"
requires_consolidation_signature = true
 
[observability.decision_trace]
enabled = true
storage = "append_only"
retention_days = 90
 
[observability.decision_trace.cryptographic_signing]
signing_required_above_sensitivity = "confidential"
algorithm = "Ed25519"
signed_by = "agent_did"
```
 
---
 
## Appendix C. ADL ↔ ARP Boundary Table
 
| Concern | ADL (Constitutional) | ARP (Operational) |
|---------|---------------------|-------------------|
| Network permissions | `permissions.network.allowed_hosts` | `network.egress.dynamic_deny` |
| Resource limits | `permissions.resource_limits` | `execution.sandbox.adaptive_limits` |
| Data classification | `data_classification.sensitivity` | `data_sovereignty.reclassification` |
| Tool definitions | `tools[*]` with `requires_confirmation` | `execution.tool_risk` with runtime promotion |
| Agent identity | `cryptographic_identity`, `id` | `state_storage` (prior signing by agent DID) |
| Lifecycle | `lifecycle.status`, `sunset_date` | Not in scope (ADL only) |
| Model config | `model.provider`, `model.temperature` | `adaptation` (which model to route to, based on learned priors) |
| System prompt | `system_prompt.template` | `cognitive.drift_detection` (monitoring drift from system prompt intent) |
| Authentication | `security.authentication` | `data_sovereignty.authorization_cache` (caching auth decisions with sensitivity-aware TTLs) |
| Attestation | `security.attestation` | `integrity` (ARP document signing and verification) |
| Budget | NOT IN ADL (proposed profile) | `budget` (full task/velocity/per-layer constraints) |
| Rate limiting | NOT IN ADL | `budget.rate_limiting` |
| Session timeouts | Partially in `runtime` | `session` (adaptive, sensitivity-proportional) |
| Tool risk classification | NOT IN ADL (proposed profile) | `execution.tool_risk` |
| PII patterns | NOT IN ADL (proposed profile) | `cognitive.context_inspection.pattern_registry` |
| Feedback loops | NOT IN ADL (not applicable) | `feedback_loops` (all timescales) |
| Gossip / peer learning | NOT IN ADL (not applicable) | `feedback_loops.gossip` |
| Escalation | NOT IN ADL (not applicable) | `escalation` |
| DecisionTrace | NOT IN ADL (not applicable) | `observability.decision_trace` |
 
---
 
## Appendix D. Code Audit Migration Table
 
This appendix maps each finding from the Arkavo codebase security audit to its home in the ADL + ARP document pair, with migration steps.
 
| # | Audit Finding | Source Location | ADL Home | ARP Home | Migration |
|---|--------------|----------------|----------|----------|-----------|
| 1 | Preflight PII patterns (SSN, credit card regex) | `arkavo-router/src/preflight/features.rs` | `data_classification.categories` (declares PII handling) | §9.1.1 `cognitive.context_inspection.pattern_registry` | Extract regexes to `.arp.toml`. Load at startup. Support runtime addition via HITL teaching (§15.2). |
| 2 | Duplicate rate limit defaults | `arkavo-protocol/src/rate_limit.rs`, `arkavo-security/src/rate_limit.rs` | — | §13.4 `budget.rate_limiting` | Delete duplicate. Single source in ARP. Re-export from one crate. |
| 3 | Session timeout bounds (1min–24hr hardcoded) | `arkavo-session/src/timeout.rs` | `runtime` (partial) | §15.3 `session.timeout_bounds` | Extract min/max/default to `.arp.toml`. Add sensitivity-proportional idle timeout. |
| 4 | Sandbox resource limits (512MB, 50% CPU) | `arkavo-dataflow/src/engine/sandbox.rs` | `permissions.resource_limits` | §10.1 `execution.sandbox.default_limits` + `adaptive_limits` | ADL declares ceiling. ARP declares defaults and adaptive tightening rules. |
| 5 | Authorization cache TTL (60s flat) | `arkavo-authorization/src/cache.rs` | — | §11.1 `data_sovereignty.authorization_cache` | Extract TTL. Add sensitivity multipliers. Add invalidation triggers. |
| 6 | Egress filter blocked CIDRs | `arkavo-validation/src/url.rs` | `permissions.network` (allowed_hosts, deny_private) | §12.1 `network.egress.dynamic_deny` | ADL declares static allow/deny. ARP declares adaptive deny with TTL. Composable with deny-takes-precedence. |
| 7 | DLP sensitivity mappings | `arkavo-security/src/data_classification.rs` | `data_classification` (sensitivity, categories) | §11.2 `data_sovereignty.reclassification` | ADL declares baseline mappings. ARP declares one-way ratchet escalation rules. |
| 8 | Scattered retry/timeout values | Multiple crates | `runtime` (partial) | §7.5 `feedback_loops.resilience` | Consolidate all retry/timeout/circuit breaker config into ARP. Per-entity overrides. |
| 9 | Quality gate thresholds (MAX_RETRIES=3) | `arkavo-router/src/quality_gate.rs` | — | §7.1.1 `feedback_loops.immediate.quality_gate` | Extract. Add per-(model, task_class) threshold overrides. |
| 10 | Tool risk classification (hardcoded enum) | `arkavo-orchestrator/src/task_policy_manager.rs` | — (proposed `tool-governance` profile) | §10.2 `execution.tool_risk` | Extract risk levels, classifications, and obligation assembly to `.arp.toml`. Add runtime promotion rules. |
 
---
 
## References
 
### Normative References
 
- **[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119.
- **[RFC8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174.
- **[RFC8259]** Bray, T., Ed., "The JavaScript Object Notation (JSON) Data Interchange Format", STD 90, RFC 8259.
- **[RFC8785]** Rundgren, A., Jordan, B., and S. Erdtman, "JSON Canonicalization Scheme (JCS)", RFC 8785.
 
### Informative References
 
- **[ADL]** Agent Definition Language Specification, v0.1.0, https://www.adl-spec.org/spec
- **[A2A]** A2A Protocol Working Group, "Agent-to-Agent Protocol Specification", https://a2a-protocol.org/specification
- **[MCP]** Anthropic, "Model Context Protocol Specification", https://modelcontextprotocol.io/specification
- **[W3C.DID]** Sporny, M., et al., "Decentralized Identifiers (DIDs) v1.0", W3C Recommendation.
- **[W3C.VC]** Sporny, M., et al., "Verifiable Credentials Data Model v2.0", W3C Recommendation.
- **[OWASP-ASI]** OWASP Agentic Security Initiative, v1.0, 2025.
- **[NIST-AI-RMF]** NIST, "Artificial Intelligence Risk Management Framework", AI 100-1, January 2023.
- **[EU-AI-ACT]** European Parliament, "Regulation (EU) 2024/1689 laying down harmonised rules on artificial intelligence", 2024.

