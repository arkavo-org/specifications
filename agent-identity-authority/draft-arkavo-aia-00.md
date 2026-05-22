# Agent Identity Authority (AIA) Specification

|                  |                                                                  |
|------------------|------------------------------------------------------------------|
| **Version**      | 0.1.0-draft                                                      |
| **Status**       | Community Draft                                                  |
| **Authors**      | Arkavo Project Contributors                                      |
| **License**      | Apache 2.0                                                       |
| **Companion to** | SwarmKit v1.1.0, Agent Runtime Policy (ARP) v0.1.0               |
| **Builds on**    | OpenTDF 1.0, WIMSE, RFC 9334 (RATS), RFC 8693, C2PA / CAWG       |

---

## Abstract

**Agent Identity Authority (AIA)** specifies a dedicated, upstream authority for
AI agent identity. Its only responsibility is to issue and attest agent
identity. It is not an orchestrator and it is not a policy
decision point. It does not assign work, grant resource access, make ABAC
decisions, delegate authority, or release TDF keys.

AIA defines the **Agent Identity Document (AID)** — a signed, content-addressed
record that binds an agent to (a) an attested model and runtime, (b) an owner
principal, and (c) a public key for proof-of-possession — and the compact
**Agent Identity Credential** projected from it. It defines a SPIFFE/SPIRE-
compatible **Model Attestation Profile**, the issuance and lifecycle flow, the
boundary between identity-originated and orchestrator-issued OpenTDF attributes,
WIMSE interoperability mappings, and a CAWG **agent identity assertion** for
C2PA content provenance.

AIA closes a structural gap in SwarmKit AE-2026-004: in the five-role mesh,
identity issuance is folded into orchestration implicitly — the agent that
decrypts a SwarmKit and signs delegation envelopes is also, in effect, the
authority that mints each agent's identity. AIA places identity issuance in an
**upstream** authority so that the orchestrator can never become a super-agent
that both creates identities and grants access.

---

## 1. Introduction

### 1.1 Motivation

SwarmKit (`swarmkit/swarmkit-spec-draft-01.md`) decrypts a signed work package
and delegates role-scoped configuration to specialist agents. The orchestrator
signs each delegation envelope (SwarmKit §7.2) with a `delegation_signature`.
Specialists, peers, Key Access Services (KAS), and MCP brokers currently treat
the orchestrator's signature as the sole evidence of *who an agent is*.

This conflates two distinct questions:

1. **Authorization** — *what may this agent do?* What work it is assigned (the
   Planner and the orchestrator), what resources it may reach (the
   orchestrator's capability grant), what keys it may unwrap (the KAS).
2. **Identity** — *who is this agent, what model is it running, and on whose
   behalf does it act?* Currently answered by nothing — it is inferred from
   the same orchestrator signature that answers question 1.

Folding identity into orchestration creates a super-agent: an orchestrator that
can both **create identities** and **grant access**. A compromised orchestrator
then does not merely mis-route work (SwarmKit §10.1 already classifies this as
"catastrophic by design"); it can fabricate an agent with an arbitrary model
claim, an arbitrary owner, and arbitrary trust level, because nothing else
vouches for those facts.

AIA removes identity issuance from the orchestrator entirely and places it in a
dedicated, **upstream** authority. The orchestrator becomes a downstream agent
that itself holds an AIA-issued identity. An agent authenticates to KAS, MCP
brokers, and peers using a credential signed by the Agent Identity Authority and
bound to attestation evidence the orchestrator does not control. Separating the
two roles prevents the orchestrator from becoming an agent that can both create
identities and grant access.

### 1.2 Relationship to Existing Specifications

AIA is deliberately narrow. It issues and attests identity. It does not assign
work, gate actions, score outputs, adapt behavior, grant access, or release
keys. The following table fixes the boundaries.

| Specification / Role | Concern | AIA overlap |
|---|---|---|
| **SwarmKit** (`swarmkit/`) | Work packaging; orchestrator decryption and delegation. | AIA adds an upstream identity tier. It does **not** change the SwarmKit manifest schema or the orchestrator's decryption flow; it adds an `aid_ref` to the delegation envelope (§6.3). |
| **Planner role** | Decomposes the objective into ordered subtasks; assigns work. | None. AIA never assigns, orders, or routes work. The Planner answers *what work*; AIA answers *who*. |
| **Orchestrator** | Decrypts the SwarmKit; delegates work; issues capability grants. | AIA strips identity-minting from the orchestrator. The orchestrator continues to delegate work and issue capability grants (authorization); AIA signs identity credentials (identity). The orchestrator itself holds an AIA credential (§6.2). |
| **OpenTDF KAS** | Policy enforcement; key release (a policy decision point). | None. AIA never makes an ABAC decision and never releases a key. The KAS *consumes* identity attributes that originate from an AID (§3.5); it does not *produce* them. |
| **TØR-G** (`torg-decision/`) | Boolean-circuit decision IR; gates whether a requested action executes. | None. AIA never gates an action. A verifier MAY feed an AID verification result into a TØR-G input; AIA itself emits no decision. |
| **Critic role** | Scores outputs against the evaluation rubric. | None. AIA attests identity, not quality. |
| **Agent Runtime Policy (ARP)** (`agent-runtime-policy/`) | Runtime adaptation of a *running* agent. | None. An AID is an input to ARP's `integrity` binding, not a product of it. |

A one-line summary: **Planner assigns, the orchestrator delegates and grants
capabilities, the KAS enforces and releases keys, TØR-G gates, the Critic
scores, ARP adapts — AIA only issues and attests identity.**

### 1.3 The Sixth Role and the Identity Tier

SwarmKit Appendix C defines a five-archetype mesh: `scribe`, `historian`,
`planner`, `critic`, `operator`. AIA adds a sixth role, `agent_type`/
`role_type` value `identity_authority` — but it is not merely a sixth peer in
the mesh. It sits in an **identity tier above the orchestrator** (§3.1). Every
other agent, the orchestrator included, is issued its identity by an Agent
Identity Authority before it can participate.

### 1.4 Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14
[RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown
here.

---

## 2. Terminology

- **AIA** — Agent Identity Authority; the abbreviation names both this
  specification and the role it defines.
- **Agent Identity Authority** — The dedicated role that issues and attests
  agent identity. `agent_type` value: `identity_authority`. Abbreviated "the
  Authority" below. Also termed the **Identity Agent** when described as an
  agent among agents.
- **Root Identity Agent** — The trust anchor of an AIA deployment; an offline or
  hardware-protected root that certifies one or more Agent Identity Authorities.
  Issues no agent credentials directly. See §3.4.
- **Swarm Identity Agent** — An Agent Identity Authority that issues identities
  to the agents of one swarm or deployment; certified by a Root Identity Agent.
- **Agent Identity Document (AID)** — A signed, content-addressed JSON document
  that binds an agent to an attested model/runtime, an owner, and a key.
  Defined in §4.
- **Agent Identity Credential** — The compact, JWT-shaped token projected from
  an AID and carried by an agent for presentation. Defined in §7.1.
- **Subject** — The agent an AID describes; identified by a DID and/or a SPIFFE
  ID.
- **Owner** — The principal (person, device, organization, or policy engine) on
  whose behalf the subject acts.
- **Trust level** — The strength of attestation behind a credential: `attested`,
  `asserted`, or `unverified` (§4.3).
- **Trust chain** — The published chain Root Identity Agent → Agent Identity
  Authority → AID, by which a verifier validates a credential (§3.4).
- **Model Attestation** — Evidence and verified claims about the model weights
  and inference runtime a subject is executing. Defined in §5.
- **Model Attestor** — The component that gathers Model Attestation evidence; in
  a SPIRE deployment, a SPIRE workload attestor plugin (§5.6). A sub-component
  of, or input to, the Agent Identity Authority — not itself a role.
- **Capability grant** — The orchestrator-issued artifact that authorizes an
  agent's access to specific resources, tools, and execution modes. Distinct
  from an AID (§3.5). Out of scope for this document.
- **Identity-originated attribute** — An OpenTDF attribute in the
  `agent.identity.*` namespace whose value derives from an AID (§3.5).
- **WIT / WIC** — WIMSE Workload Identity Token / Workload Identity Certificate
  [WIMSE-CREDS].
- **Proof-of-Possession (PoP)** — Demonstration that a presenter holds the
  private key bound to a credential. A credential is not a bearer token (§4.6).
- **SwarmFlight** — A single execution instance of a SwarmKit (SwarmKit §2).

---

## 3. The Agent Identity Authority Role

### 3.1 Position in the Agent Ecosystem

The Agent Identity Authority is **upstream** of the orchestrator, not a peer the
orchestrator provisions. Identity is issued before work is assigned.

```
        ┌───────────────────────────┐
        │   Root Identity Agent     │  trust anchor — offline / HSM root;
        │   (trust anchor)          │  certifies Authorities, issues no
        └─────────────┬─────────────┘  agent credentials directly
                      │ certifies
                      ▼
        ┌───────────────────────────┐
        │  Swarm Identity Agent     │  the Agent Identity Authority (AIA):
        │  (Agent Identity          │  creates identities, issues credentials,
        │   Authority)              │  rotates keys, attests runtime, revokes
        └─────────────┬─────────────┘
                      │ issues an Agent Identity Credential to EVERY agent below
        ┌─────────────┼───────────────────────────┐
        ▼             ▼                           ▼
 ┌─────────────┐  ┌──────────┐              ┌──────────┐
 │ Orchestrator│  │  Worker  │   ... A2A ...│  Worker  │
 │   Agent     │  │  Agent   │◀────────────▶│  Agent   │
 └──────┬──────┘  └────┬─────┘              └──────────┘
        │ capability   │ presents its identity credential
        │ grant        │ with an A2A request
        ▼              │
 ┌─────────────────────▼─────┐
 │   OpenTDF Auth / KAS      │  policy enforcement + key release;
 │                           │  consumes identity-originated attributes
 └───────────────────────────┘
```

The ecosystem flow, end to end:

```
Agent Identity Authority  ──issues──▶  identity credential
        ▼
Worker Agent  ──A2A request (presents credential)──▶  Orchestrator Agent
        ▼
Orchestrator Agent  ──capability grant──▶  OpenTDF Auth / KAS
```

The role separation maps onto familiar infrastructure:

| AIA ecosystem role | Closest analogue |
|---|---|
| Agent Identity Authority | OIDC Identity Provider **+** SPIFFE CA |
| Orchestrator | Security Token Service (STS) / capability issuer |
| OpenTDF KAS | Policy enforcement point + key release |

### 3.2 Responsibilities

The Agent Identity Authority has exactly one purpose: issue and attest agent
identity. Its responsibilities, and its non-responsibilities, are both
normative.

**An Agent Identity Authority does (REQUIRED):**

| | Responsibility | Section |
|---|---|---|
| ✓ R1 | Create agent identities | §6.1 |
| ✓ R2 | Issue agent credentials | §4, §7.1 |
| ✓ R3 | Rotate agent keys | §6.5, §9.2 |
| ✓ R4 | Attest runtime / platform identity | §4.8, §5 |
| ✓ R5 | Publish the trust chain | §3.4 |
| ✓ R6 | Revoke compromised agents | §9.3 |

**An Agent Identity Authority does not (MUST NOT):**

| | Non-responsibility | Belongs to |
|---|---|---|
| ✗ N1 | Assign work | Planner, orchestrator |
| ✗ N2 | Grant resource access | Orchestrator (capability grant) |
| ✗ N3 | Make ABAC decisions | OpenTDF KAS / a policy decision point |
| ✗ N4 | Delegate authority | Owner, orchestrator |
| ✗ N5 | Release TDF keys | OpenTDF KAS |

N1–N5 are absolute. An AID **MUST NOT** carry a work assignment, a resource
scope, a tool grant, an execution-mode permission, an ABAC rule, or a wrapped
key. An AID states *facts about an agent*; every form of authorization is a
separate artifact issued by a separate role. A verifier that needs both
identity and authorization checks two artifacts signed by two authorities.

Constraint N4 deserves a precise reading. AIA *attests* delegation chains — it
records, in an AID, that agent B acts on behalf of agent A on behalf of owner O
(§4.9). Recording who-acts-for-whom is identity provenance. *Delegating
authority* — conferring on B the permission to do something — is a different
act, and AIA never performs it. The delegation chain in an AID is a description,
never a grant.

### 3.3 Separation from Orchestration

Before AIA, the chain of trust for an agent's identity is:

```
agent identity  ⇐  orchestrator's delegation_signature  ⇐  orchestrator key
```

The orchestrator is the sole authority. There is no second signature to check,
and the orchestrator is therefore a super-agent: it creates identity and grants
access with the same key.

After AIA:

```
agent identity  ⇐  Agent Identity Credential  ⇐  Authority key  ⇐  Root Identity Agent
agent's work    ⇐  delegation envelope        ⇐  orchestrator key
agent's access  ⇐  capability grant           ⇐  orchestrator key
key release     ⇐  KAS decision               ⇐  KAS, over identity + capability attributes
```

A downstream verifier authenticates the agent via its credential and the
Authority trust chain, and authorizes the agent's request via the orchestrator's
delegation envelope and capability grant. The checks are independent and signed
by independent keys.

Consequently, a compromised orchestrator:

- **can** still mis-assign work, forge delegation envelopes, and issue capability
  grants within its own clearance (unchanged from SwarmKit §10.1);
- **cannot** forge an agent's attested identity, because it does not hold the
  Authority key and cannot produce Model Attestation evidence (§5.4) a verifier
  will accept.

This does not make a compromised orchestrator safe. It bounds the blast radius:
identity-dependent trust signals (KAS identity attributes, MCP broker decisions,
A2A peer authentication, content provenance) remain sound even when the
orchestrator is hostile.

### 3.4 Trust Anchor and Trust Chain

AIA deployments are two-tier:

- A **Root Identity Agent** is the trust anchor. It is offline or
  hardware-protected, certifies one or more Agent Identity Authorities, and
  issues *no* agent credentials directly. Compromise of an Authority is
  contained by rotating its certification at the root; compromise of the root
  is the catastrophic case and is why the root does no day-to-day work.
- A **Swarm Identity Agent** is an Agent Identity Authority that issues
  identities to the agents of one swarm or deployment. Its signing key is
  certified by a Root Identity Agent.

The Authority key MUST be rooted in a trust anchor that is **not** the
orchestrator's key and **not** derived from the SwarmKit-level wrapped key.
Acceptable Root Identity Agent anchors, in decreasing order of assurance:

1. A hardware-backed key whose attestation chains to a manufacturer or platform
   root (TPM, TEE, secure element).
2. An organizational root DID (`did:web`) published out-of-band.
3. The SwarmKit author's DID (`kit.authors[].did`, SwarmKit §4.1), when author
   and deploying organization are the same trust domain.

**Publishing the trust chain (R5).** The Authority MUST publish, at a stable and
verifier-resolvable location, the chain Root Identity Agent → Authority key →
(per-AID) signature, together with the current revocation list (§9.3). A
verifier MUST be able to validate any AID against this published chain
*independently of any artifact the orchestrator produces*. An Authority whose
key chains only to the orchestrator provides no separation and MUST be treated
by verifiers as equivalent to no AIA at all.

### 3.5 Identity-Originated Attributes and the OpenTDF Boundary

OpenTDF attribute-release and KAS decisions are ABAC: they evaluate attributes
about the requesting agent. Those attributes have two distinct origins, and AIA
fixes the boundary.

**Attributes that originate from identity** — the Authority MAY originate these;
they are populated from an AID; they live in the `agent.identity.*` namespace:

| OpenTDF attribute | AID source field |
|---|---|
| `agent.identity.type` | `subject.agent_type` (§4.3) |
| `agent.identity.trust_level` | `subject.trust_level` (§4.3) |
| `agent.identity.runtime` | `subject.runtime` (§4.3) |
| `agent.identity.owner` | `owner.did` (§4.4) |
| `agent.identity.organization` | `subject.organization` (§4.3) |

**Attributes that MUST NOT originate from identity** — these are authorization,
not identity; they are issued by the orchestrator as part of the capability
grant; an Authority MUST NOT originate them and an AID MUST NOT carry them:

- `agent.tool_access`
- `agent.code_authority`
- `agent.resource_scope`
- `agent.execution_mode`

Normative rules:

- An Authority MUST confine the attributes it originates to the
  `agent.identity.*` namespace.
- An AID MUST NOT carry, in any field, an attribute outside `agent.identity.*`.
- A KAS that derives attributes from an AID MUST accept only `agent.identity.*`
  attributes from that source, and MUST obtain `agent.tool_access`,
  `agent.code_authority`, `agent.resource_scope`, and `agent.execution_mode`
  from the orchestrator's capability grant, never from a credential.

This boundary is what makes the separation enforceable at the KAS: even a KAS
that fully trusts an AID learns only *who the agent is* from it, never *what the
agent may do*.

---

## 4. The Agent Identity Document (AID)

### 4.1 Media Type and Encoding

- AID documents use the media type **`application/vnd.arkavo.aid+json`**.
- An AID MUST be encoded in UTF-8 and MUST be valid JSON [RFC8259].
- Member names MUST use **snake_case**, consistent with the SwarmKit delegation
  envelope.
- All timestamps MUST be RFC 3339 strings with an explicit timezone.
- All URIs MUST conform to [RFC3986].
- Binary values MUST be base64url without padding, per [RFC4648] §5.

File extension: `.aid.json`. The compact credential form (§7.1) is a JWT.

### 4.2 Top-Level Structure

```json
{
  "aia_spec": "0.1.0",
  "aid": {
    "id": "blake3:<base64url>",
    "issued": "<RFC3339>",
    "expires": "<RFC3339>",
    "nonce": "<base64url>"
  },
  "subject":            { },
  "owner":              { },
  "flight_binding":     { },
  "key":                { },
  "model_attestation":  { },
  "runtime_attestation":{ },
  "delegation":         { },
  "assertions":         [ ],
  "signature":          { }
}
```

| Member | Required | Section |
|---|---|---|
| `aia_spec` | yes | semver of this spec the AID conforms to |
| `aid` | yes | §4.12 — identity, validity window, replay nonce |
| `subject` | yes | §4.3 |
| `owner` | yes | §4.4 |
| `flight_binding` | no | §4.5 — present only for ephemeral flight-scoped credentials |
| `key` | yes | §4.6 |
| `model_attestation` | yes | §5 |
| `runtime_attestation` | yes | §4.8 |
| `delegation` | no | §4.9 — present iff the subject acts in a delegation chain |
| `assertions` | no | §4.10 — default `[]` |
| `signature` | yes | §4.11 |

An AID identifies a **durable agent** by default — an agent that persists across
flights, with a validity window on the order of days (§9.2). A short-lived
agent created for a single SwarmFlight MAY additionally carry `flight_binding`
(§4.5). Identity is durable; flight scoping is the orchestrator's concern.

### 4.3 subject

The agent the AID describes.

```json
"subject": {
  "did": "did:web:agent.arkavo.net:worker-17",
  "spiffe_id": "spiffe://arkavo.net/agent/worker-17",
  "agent_type": "coding",
  "role_type": "operator",
  "trust_level": "attested",
  "runtime": "local",
  "organization": "did:web:arkavo.net",
  "capabilities": ["a2a", "mcp"],
  "display_name": "worker-17"
}
```

- `did` — REQUIRED. A resolvable DIF DID identifying the subject.
- `spiffe_id` — OPTIONAL but RECOMMENDED. A SPIFFE ID for the subject,
  consistent with the WIMSE mapping in §7.3.
- `agent_type` — REQUIRED. A coarse functional class of the agent, e.g.
  `coding`, `orchestrator`, `identity_authority`, `research`. Populates the
  OpenTDF attribute `agent.identity.type` (§3.5).
- `role_type` — OPTIONAL. The SwarmKit mesh archetype, when the agent acts in a
  SwarmKit mesh (`scribe`, `historian`, `planner`, `critic`, `operator`,
  `identity_authority`). Distinct from `agent_type`: `agent_type` is the
  agent's functional class, `role_type` is its position in a specific mesh.
- `trust_level` — REQUIRED. The strength of attestation behind this identity:
  - `attested` — backed by hardware-rooted Model Attestation evidence
    (`rats-hardware`, §5.4) that the Authority verified.
  - `asserted` — backed by software-only evidence (`rats-software`) the
    Authority accepted on the integrity of the attesting host.
  - `unverified` — issued without acceptable attestation evidence. An Authority
    SHOULD NOT issue at this level; verifiers SHOULD treat it as no AIA.
  Populates `agent.identity.trust_level`.
- `runtime` — REQUIRED. A coarse runtime locus: `local`, `remote`, `cloud`, or
  `tee`. Distinct from `runtime_attestation.backend` (the inference engine).
  Populates `agent.identity.runtime`.
- `organization` — OPTIONAL. A resolvable DID of the organization the agent
  belongs to. Populates `agent.identity.organization`.
- `capabilities` — REQUIRED. A list of coarse protocol/interface capabilities
  the agent speaks, e.g. `a2a`, `mcp`. This is **descriptive identity**, not
  authorization: it states *what protocols the agent can speak*, never *what
  resources it may reach*. `capabilities` MUST NOT be interpreted as a tool
  grant or resource scope (§3.5).
- `display_name` — OPTIONAL human-readable label.

An AID describes exactly one subject.

### 4.4 owner

The principal on whose behalf the subject acts. An agent's credential
cryptographically references its owner so that downstream services can evaluate
provenance of authority, not merely identity of process.

```json
"owner": {
  "did": "did:web:arkavo.net",
  "owner_type": "organization",
  "binding": "delegated",
  "owner_signature": "<base64url>"
}
```

- `did` — REQUIRED. Resolvable DID of the owner. Populates
  `agent.identity.owner` (§3.5).
- `owner_type` — REQUIRED. One of `person`, `device`, `organization`,
  `policy_engine`.
- `binding` — REQUIRED. One of:
  - `delegated` — the owner authorized this agent's class of work in advance;
    the Authority vouches for the link.
  - `owner_signed` — the owner directly signed this AID's owner block
    (`owner_signature` REQUIRED); strongest binding.
  - `self` — the subject is its own owner (autonomous agent with no external
    principal).
- `owner_signature` — REQUIRED iff `binding` is `owner_signed`; an Ed25519
  signature by the owner key over the canonical `subject` + `owner` blocks
  (excluding `owner_signature` itself).

An autonomous action — one the subject takes that is not traceable to an owner
delegation — MUST be issued under `binding: self` or under a `delegation` block
(§4.9) whose chain terminates in `self`. Autonomous actions MUST be
distinguishable from delegated ones.

### 4.5 flight_binding

OPTIONAL. Present **only** for an ephemeral agent created for a single
SwarmFlight; absent for a durable agent identity (the default, §4.2).

```json
"flight_binding": {
  "flight_id": "<uuid>",
  "kit_id": "blake3:<base64url>",
  "role_id": "planner-1"
}
```

When present, all three members are REQUIRED, and `kit_id` MUST equal the
`kit.id` of the SwarmKit being executed (SwarmKit §9.1). A verifier presented an
AID that carries `flight_binding` MUST reject it if the binding does not match
the flight of presentation.

`flight_binding` scopes *identity* to a flight; it does not scope
*authorization*. Per-flight authorization is the orchestrator's capability grant
(§3.5), which is issued per flight regardless of whether the AID is durable or
flight-bound.

### 4.6 key

The subject's public key, for proof-of-possession. A credential is **not** a
bearer token: a presenter MUST prove possession of the corresponding private
key (§7.1, [WIMSE-CREDS] §3.1).

```json
"key": {
  "cnf": {
    "jwk": { "kty": "OKP", "crv": "Ed25519", "x": "<base64url>", "alg": "EdDSA" }
  }
}
```

The `cnf` member follows [RFC7800]. The `jwk` MUST include an `alg`; the value
`none` MUST NOT be used and symmetric algorithms MUST NOT be used. A verifier
that accepts a credential without a successful PoP challenge is non-conforming
(§11 C-V3).

### 4.7 model_attestation

The Model Attestation block. Its schema and semantics are defined in §5. It is
REQUIRED in every AID; the `trust_level` (§4.3) is derived from its evidence.

### 4.8 runtime_attestation

Claims about the execution environment, distinct from the model.

```json
"runtime_attestation": {
  "backend": "llama.cpp",
  "backend_version": "b4567",
  "backend_digest": "blake3:<base64url>",
  "host": { "isolation": "container", "tee": "sev-snp", "measured": true },
  "evidence_ref": "rats:<uri-or-content-hash>"
}
```

- `backend` — REQUIRED. The inference runtime: `llama.cpp`, `mlx`, `vllm`, or
  `custom`. Mirrors SwarmKit `agent_provisioning.model.backend`.
- `backend_version` — REQUIRED. Version string of the runtime build.
- `backend_digest` — RECOMMENDED. BLAKE3 of the runtime binary.
- `host.isolation` — REQUIRED. One of `process`, `container`, `vm`, `none`.
- `host.tee` — REQUIRED. The trusted execution environment, or `none`. Values:
  `none`, `sev-snp`, `tdx`, `sgx`, `cca`, `nitro`.
- `host.measured` — REQUIRED boolean. Whether `evidence_ref` carries a hardware
  measurement (versus a software-only self-report).
- `evidence_ref` — OPTIONAL. A reference to RATS evidence for the runtime.

### 4.9 delegation

Present iff the subject operates inside a delegation chain. This block
**attests** a delegation chain; it does not create one. The chain is established
by owners and orchestrators; AIA records it and binds it to identity (§3.2 N4).

```json
"delegation": {
  "chain": [
    { "actor": "did:web:arkavo.net",                  "kind": "owner" },
    { "actor": "did:web:agent.arkavo.net:planner-1",   "kind": "agent" }
  ],
  "act": "did:web:agent.arkavo.net:worker-17",
  "may_act": {
    "audience": ["spiffe://arkavo.net/agent"],
    "max_depth": 4,
    "scope_reduction_only": true
  }
}
```

- `chain` — REQUIRED. The ordered delegation chain, root principal first, the
  immediate delegator last. Each entry has an `actor` DID and a `kind`
  (`owner` or `agent`). Models the RFC 8693 delegation pattern.
- `act` — REQUIRED. The DID of the subject; the current actor, per RFC 8693
  `act`.
- `may_act` — REQUIRED. Constraints the subject MUST observe if its identity is
  re-bound for a further hop (§6.4):
  - `audience` — the trust domains within which identity may be re-bound.
  - `max_depth` — the maximum total chain length.
  - `scope_reduction_only` — when `true`, a re-bound downstream AID MUST carry a
    `model_attestation`, `trust_level`, and `flight_binding` no broader than
    this AID's, and MUST NOT introduce a new owner.

`may_act` constrains how identity is *re-attested* down a chain. It is not a
grant of authority — see §3.2 N4.

### 4.10 assertions

References to CAWG agent identity assertions (§8) emitted for content this
subject produces. Each entry:

```json
{ "assertion_label": "cawg.agent_identity", "assertion_digest": "blake3:<base64url>" }
```

### 4.11 signature

The Authority's signature over the AID.

```json
"signature": {
  "authority_did": "did:web:swarm-id.arkavo.net",
  "trust_anchor": "did:web:root-id.arkavo.net",
  "algorithm": "ed25519",
  "value": "<base64url>"
}
```

- `authority_did` — REQUIRED. DID of the issuing Agent Identity Authority.
- `trust_anchor` — REQUIRED. The Root Identity Agent the Authority key chains to
  (§3.4). A verifier MUST confirm this anchor is one it is configured to trust
  and is independent of the orchestrator.
- `algorithm` — REQUIRED. `ed25519`.
- `value` — REQUIRED. The signature, computed per §4.12.

### 4.12 Content Addressing, Canonicalization, and Signing

`aid.id` MUST be `blake3:` followed by the base64url (no padding) BLAKE3-256
digest of the JCS-canonical [RFC8785] encoding of the AID with the `aid.id` and
`signature` members **excluded**.

```
canonical_bytes = JCS_canonical_json( AID without aid.id and without signature )
aid.id          = "blake3:" || base64url( BLAKE3_256( canonical_bytes ) )
signature.value = base64url( Ed25519_sign( authority_private_key,
                                           BLAKE3_256( canonical_bytes ) ) )
```

The Authority signs the BLAKE3 digest of the same canonical bytes that produce
`aid.id`. A verifier MUST recompute `aid.id`, MUST reject the AID if the
recomputed value differs from the declared value, and MUST verify
`signature.value` against the recomputed digest using the Authority key resolved
from `signature.authority_did` and validated up the published trust chain
(§3.4).

This is the same canonicalization-and-digest discipline SwarmKit uses for
`kit.id` (§9.1) and skill signing (§8.1.2): BLAKE3 over JCS-canonical JSON.

---

## 5. Model Attestation Profile

### 5.1 Overview

Model Attestation answers a question no current agent-identity scheme answers
directly: *is this process actually running model X (weight digest Y) on
runtime Z?* It is the missing primitive — without it, a `model.family` claim in
a SwarmKit `agent_provisioning` block is an unverified assertion. The
`subject.trust_level` of a credential (§4.3) is derived from the strength of
this attestation.

The profile is designed for **SPIFFE/SPIRE compatibility**. A SPIRE workload
attestor plugin produces selectors (§5.5) that a SPIRE server matches to a
SPIFFE ID; AIA reuses the same evidence to populate the `model_attestation`
block of an AID. A deployment MAY run AIA with SPIRE (selectors drive SVID
issuance) or standalone (the Authority consumes evidence directly) — the
evidence and claims are identical.

### 5.2 Weight Digest

```json
"model_attestation": {
  "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
  "weights": {
    "digest_algorithm": "blake3",
    "digest": "blake3:<base64url>",
    "shard_layout": "single",
    "merkle_root": null,
    "artifact_count": 1
  }
}
```

- `model.family` / `size` / `quantization` — REQUIRED. Mirror the SwarmKit
  `agent_provisioning.model` fields so an AID can be checked against the kit.
- `weights.digest_algorithm` — REQUIRED. `blake3`. AIA reuses SwarmKit's BLAKE3
  content-addressing pipeline; no new hash primitive is introduced.
- `weights.digest` — REQUIRED. The weight content address (§5.2.1).
- `weights.shard_layout` — REQUIRED. `single` or `sharded`.
- `weights.merkle_root` — REQUIRED (non-null) when `shard_layout` is `sharded`;
  the BLAKE3 Merkle root over per-shard digests, sorted by shard index. `null`
  otherwise.
- `weights.artifact_count` — REQUIRED. The number of weight artifacts hashed.

#### 5.2.1 Computing the Weight Digest

For `shard_layout: single`, the digest is `BLAKE3_256` of the single weight
artifact's bytes.

For `shard_layout: sharded`, the attestor computes `BLAKE3_256` of each shard,
sorts the per-shard digests by ascending shard index, concatenates them, and
takes `BLAKE3_256` of the concatenation as `merkle_root`; `weights.digest` is
set equal to `merkle_root`. Sharded layout lets a large model be hashed
incrementally as shards load, and lets a verifier check a model against a
published per-shard manifest without re-downloading it.

The digest MUST be computed over the weight artifacts **as loaded into the
inference process**, not over a packaging container.

### 5.3 Runtime Measurement

`model_attestation` records *what model*; `runtime_attestation` (§4.8) records
*what runtime*. The two are separated so a verifier can pin a model digest
across runtime upgrades, or a runtime across model swaps, independently.

### 5.4 Attestation Evidence

```json
"evidence": {
  "format": "rats-software",
  "produced_by": "arkavo-model-attestor/0.1.0",
  "produced_at": "<RFC3339>",
  "nonce": "<base64url>",
  "claims_digest": "blake3:<base64url>",
  "hardware_evidence": null
}
```

- `format` — REQUIRED. One of:
  - `rats-software` — a software-only measurement: the attestor process read
    the weight bytes and runtime binary and hashed them. Yields at most
    `trust_level: asserted` (§4.3).
  - `rats-hardware` — the measurement chains to a hardware root of trust (TPM
    quote, TEE attestation report); `hardware_evidence` REQUIRED. Required for
    `trust_level: attested`.
- `produced_by` — REQUIRED. Identifier and version of the Model Attestor.
- `produced_at` — REQUIRED. When evidence was gathered.
- `nonce` — REQUIRED. The freshness nonce; binds the evidence to a specific
  issuance and defeats replay (§10.4).
- `claims_digest` — REQUIRED. BLAKE3 of the JCS-canonical `model` + `weights`
  blocks; lets evidence be transported separately from the AID.
- `hardware_evidence` — REQUIRED (non-null) when `format` is `rats-hardware`. An
  opaque, base64url-encoded RATS evidence blob. `null` otherwise.

This block aligns with the RATS architecture [RFC9334]: the Model Attestor is
the *Attester*, the Agent Identity Authority (or SPIRE server) is the
*Verifier*, and the AID is a fragment of the resulting *Attestation Result*.
Hardware-backed evidence is RECOMMENDED for any flight whose SwarmKit declares a
data classification above `public`.

### 5.5 SPIRE Workload Selectors

A Model Attestor running as a SPIRE workload attestor plugin returns selectors
of type `arkavo`:

| Selector | Meaning |
|---|---|
| `arkavo:model-family:<family>` | Model family, e.g. `gemma-4`. |
| `arkavo:model-size:<size>` | Parameter size, e.g. `26B-MoE`. |
| `arkavo:model-quant:<quant>` | Quantization, e.g. `Q4_K_M`. |
| `arkavo:weights:<algo>:<hex>` | Weight digest, e.g. `arkavo:weights:blake3:9f8e...`. |
| `arkavo:runtime:<backend>` | Inference backend, e.g. `llama.cpp`. |
| `arkavo:runtime-version:<v>` | Backend build version. |
| `arkavo:tee:<type>` | TEE type, or `none`. |
| `arkavo:evidence:<format>` | `rats-software` or `rats-hardware`. |

A SPIRE registration entry that issues an SVID for a Gemma-4 agent pins the
selector set `{arkavo:model-family:gemma-4, arkavo:weights:blake3:9f8e...,
arkavo:runtime:llama.cpp, arkavo:tee:none}`. SPIRE issues the SVID only to a
workload the plugin attests with all pinned selectors. The same selector set,
expanded to full values, populates the AID `model_attestation` block.

Selector values MUST be lowercase except for digest hex and version strings,
which are case-preserved. A value containing a colon MUST be percent-encoded per
[RFC3986].

### 5.6 The Model Attestor

The Model Attestor produces §5.4 evidence. As a SPIRE workload attestor plugin
(`arkavo-model-attestor`) it:

1. Identifies the inference process (by PID handed to the plugin by the SPIRE
   agent).
2. Enumerates the weight artifacts that process has mapped or opened, and
   computes the weight digest (§5.2.1).
3. Identifies the runtime backend and version, and OPTIONALLY hashes the
   runtime binary.
4. Reads TEE attestation evidence if a TEE is present.
5. Returns the §5.5 selectors to the SPIRE agent.

A standalone (non-SPIRE) Model Attestor performs steps 1–4 and hands the §5.4
evidence block directly to the Agent Identity Authority. The Model Attestor is a
sub-component of, or input to, the Authority — it is not itself a role and never
issues credentials.

This specification defines the *profile* — the evidence format, the digest
discipline, the selector vocabulary. A reference `arkavo-model-attestor`
implementation for `llama.cpp` and `mlx` is a separate engineering deliverable
and is out of scope here.

### 5.7 Verification

A verifier of `model_attestation` MUST:

- **V-M1.** Recompute `weights.digest` against an expected value when one is
  known (a SwarmKit-pinned digest, a SPIRE registration selector, or a
  published model manifest), and reject on mismatch.
- **V-M2.** Confirm `evidence.nonce` matches the nonce expected for this
  issuance (§10.4).
- **V-M3.** When `evidence.format` is `rats-hardware`, verify
  `hardware_evidence` against the appropriate hardware root of trust.
- **V-M4.** Treat `rats-software` evidence as asserting only that *the attestor
  host* observed these bytes — no stronger than that host's own integrity, and
  never sufficient for `trust_level: attested`.

---

## 6. Identity Issuance and Lifecycle Flow

### 6.1 Issuance

An Agent Identity Authority issues an identity to an agent when that agent is
created or deployed — **before, and independently of, any work assignment**.
Issuance is not driven by an orchestrator and does not require a SwarmFlight.

```
1.  An agent is created (a process is started, weights are loaded).
2.  A Model Attestor gathers §5 evidence for the agent's process —
    directly, or by the agent obtaining a SPIRE SVID.
3.  The Authority verifies the evidence (§5.7), resolves the owner
    binding (§4.4), and derives subject.trust_level from the evidence
    strength (§4.3).
4.  The Authority constructs, content-addresses, and signs the AID (§4.12).
5.  The Authority issues the Agent Identity Credential (§7.1) — the compact
    form the agent carries — and records the AID.
6.  The agent presents its credential, with proof-of-possession, whenever
    it authenticates: to peers over A2A, to the orchestrator, to a KAS.
```

The credential's validity window (`aid.expires`) is durable — on the order of
days — and the agent is rotated (§6.5, §9.2) rather than re-issued per flight.

### 6.2 The Orchestrator's Own Identity

The orchestrator is itself an agent and MUST hold an Agent Identity Credential
issued by an Agent Identity Authority (`subject.agent_type: orchestrator`). It
obtains that credential exactly as any other agent does (§6.1). There is no
circularity: the Authority is upstream of the orchestrator (§3.1); the
orchestrator never issues, and never co-signs, an identity — its own included.

### 6.3 Joining a SwarmFlight

When an agent joins a SwarmFlight:

1. The agent presents its (durable) Agent Identity Credential to the
   orchestrator over A2A, with proof-of-possession.
2. The orchestrator verifies the credential against the published trust chain
   (§3.4) — it does not, and cannot, issue identity.
3. The orchestrator sends the SwarmKit delegation envelope, adding an `aid_ref`
   member: the `aid.id` of the agent's credential.
4. The orchestrator separately issues the capability grant — the authorization
   artifact (§3.5) — scoped to this flight.
5. The agent, before acting, verifies both its delegation envelope
   (orchestrator signature, SwarmKit §7.3) and the orchestrator's own credential.

`aid_ref` is the only addition AIA makes to the SwarmKit delegation envelope. It
is OPTIONAL in the SwarmKit schema (a non-AIA flight omits it) and REQUIRED in
an AIA flight. A specialist that receives a delegation envelope with no
`aid_ref`, in a flight that uses AIA, MUST refuse the delegation (§11 C-S2).

For a short-lived agent created solely for one flight, the Authority MAY issue a
flight-scoped credential carrying `flight_binding` (§4.5); this is an issuance
choice, and does not change the rule that authorization is the orchestrator's
capability grant.

### 6.4 Delegation-Chain Attestation and Re-binding

When an agent delegates work to a sub-agent (a hierarchical SwarmKit, or an A2A
hand-off across a trust boundary), the sub-agent MUST receive a fresh credential
from an Agent Identity Authority — not a copy of the delegator's. The Authority
issues the downstream AID with:

- `delegation.chain` extended by the delegator;
- `delegation.act` set to the sub-agent;
- `delegation.may_act` no broader than the delegator's (§4.9);
- `model_attestation`, `trust_level`, and `flight_binding` no broader than the
  delegator's when `scope_reduction_only` is `true`.

This re-attests *identity* down the chain so a downstream verifier can evaluate
provenance; it does not delegate *authority* (§3.2 N4). A credential MUST NOT be
re-bound to a trust domain outside `delegation.may_act.audience`.

### 6.5 Key Rotation

The Authority rotates an agent's key by issuing a new AID for the same subject
with a fresh `aid.id`, a fresh `aid.nonce`, a later `aid.issued`, an updated
`key.cnf`, and a metadata note superseding the prior `aid.id`. Rotation is
REQUIRED when the subject's private key changes or is suspected compromised, and
RECOMMENDED on a fixed schedule no longer than `aid.expires - aid.issued`.
WIMSE-style short windows ([WIMSE-CREDS] §3.1) are RECOMMENDED.

---

## 7. WIMSE Interoperability

AIA is intentionally close to the IETF WIMSE work so an Arkavo agent ecosystem
can interoperate with WIMSE-native infrastructure. This section maps AIA
constructs to [WIMSE-ARCH] and [WIMSE-CREDS]. The mapping is informative for AIA
conformance but normative for any implementation claiming WIMSE interop.

### 7.1 The Agent Identity Credential (AID projected to a WIT)

The **Agent Identity Credential** is the compact, JWT-shaped token an agent
carries and presents. It is a projection of an AID and a superset of a WIMSE
WIT. A conforming implementation MUST be able to project an AID onto this form:

```json
{
  "iss": "did:web:swarm-id.arkavo.net",
  "sub": "did:web:agent.arkavo.net:worker-17",
  "agent_type": "coding",
  "trust_level": "attested",
  "runtime": "local",
  "model": "gemma-4",
  "capabilities": ["a2a", "mcp"],
  "cnf": { "jwk": { "kty": "OKP", "crv": "Ed25519", "x": "<base64url>", "alg": "EdDSA" } },
  "exp": "2026-06-01T00:00:00Z",
  "aid_id": "blake3:<base64url>"
}
```

| WIT claim ([WIMSE-CREDS] §3.1) | Agent Identity Credential source |
|---|---|
| `iss` (optional, URI) | `signature.authority_did` |
| `sub` (required, URI) | `subject.did` (or `subject.spiffe_id`) |
| `exp` (required) | `aid.expires` |
| `cnf` (required, `jwk`) | `key.cnf` verbatim |
| `jti` (optional) | `aid.id` (carried as `aid_id`) |
| header `typ` | `wit+jwt` |
| header `alg` | `EdDSA` |

`agent_type`, `trust_level`, `runtime`, `model`, and `capabilities` are carried
as private claims; `model` is `model_attestation.model.family`. The richer AID
blocks — `owner`, `flight_binding`, full `model_attestation`,
`runtime_attestation`, `delegation`, `assertions` — are resolved from the full
AID by `aid_id`, or carried under an `arkavo_aid` private claim. Like a WIT, the
credential MUST NOT be used as a bearer token: a presenter proves possession of
the `cnf` private key.

> Note on identifiers. `iss`/`sub` SHOULD be resolvable DIDs or SPIFFE IDs in
> production so the trust chain (§3.4) can be validated. A short form such as
> `agent:worker-17` MAY appear as `subject.display_name`, but is not a
> resolvable identifier and MUST NOT be the sole value of `sub`.

### 7.2 AID and the Workload Identity Certificate (WIC)

When an agent authenticates at the transport layer, it MAY present a WIC
([WIMSE-CREDS] §4.1) — an X.509 certificate with the subject identity in a
single URI `SubjectAltName`. AIA places `subject.spiffe_id` in that SAN, making
the WIC a valid X.509-SVID. The AID carries the attestation context the bare
certificate cannot; the two are linked by the SPIFFE ID.

### 7.3 did:web and spiffe://

A subject has both a `did:web` and a `spiffe://` ID. They MUST be
deterministically related:

```
subject.did        = did:web:<authority>:agent:<agent-name>
subject.spiffe_id  = spiffe://<authority>/agent/<agent-name>
```

For a flight-scoped credential the agent path component MAY instead encode the
flight, e.g. `did:web:<authority>:flight:<flight-id>:<role-id>`. A verifier MUST
reject an AID whose `did` and `spiffe_id` do not share the same authority and
agent (or flight/role) components.

### 7.4 Owner Binding

WIMSE's AI-agent identity model [WIMSE-AI] introduces the *owner* — the person,
device, or policy engine an agent acts for — and *dual-identity credentials*
binding agent to owner. AIA's `owner` block (§4.4) is exactly this binding. The
`owner.binding` values correspond to the WIMSE operational models: `delegated` ≈
agent-mediated, `owner_signed` ≈ owner-mediated; a server-mediated challenge
maps to an Authority that contacts the owner before setting `binding:
delegated`.

### 7.5 Delegation Chains

AIA `delegation` (§4.9) records the RFC 8693 delegation pattern that
[WIMSE-ARCH] §3.3.6 builds on, and the per-hop re-binding §3.3.9.4 requires.
`delegation.act` is RFC 8693 `act`; `delegation.may_act` constrains how identity
is re-attested onward as RFC 8693 `may_act` constrains onward delegation. AIA
does not adopt RFC 8693 token *exchange* — the OAuth 2.0 Token Exchange profiling
of the Arkavo Task Contract Protocol is tracked separately and deferred (§14.3).

---

## 8. CAWG Agent Identity Assertion

### 8.1 Assertion Label and Purpose

SwarmKit `provenance.c2pa_assertions` already requires CAWG identity assertion
v1.x conformance — but the CAWG identity assertion describes a *human creator*.
Content produced by an attested agent needs an assertion that says "this output
was produced by attested agent identity X." AIA defines that assertion type.

- **Assertion label:** `cawg.agent_identity`
- **Assertion kind:** a C2PA assertion, embeddable in a C2PA Manifest, composing
  with — not replacing — the CAWG (human) identity assertion.

### 8.2 Assertion Payload

The payload is a compact projection of the AID.

```json
{
  "label": "cawg.agent_identity",
  "data": {
    "aia_spec": "0.1.0",
    "aid_id": "blake3:<base64url>",
    "subject_did": "did:web:agent.arkavo.net:worker-17",
    "agent_type": "coding",
    "trust_level": "attested",
    "owner_did": "did:web:arkavo.net",
    "owner_binding": "delegated",
    "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
    "weights_digest": "blake3:<base64url>",
    "attestation_format": "rats-hardware",
    "authority_did": "did:web:swarm-id.arkavo.net",
    "aid_signature": "<base64url>"
  }
}
```

Every `data` member is copied verbatim from the AID it projects. `aid_signature`
is the AID's `signature.value`; a consumer holding the full AID verifies the
assertion exactly as it verifies the AID (§4.12). A consumer holding only the
assertion can still establish the chain authority → subject → model, and resolve
the full AID by `aid_id`.

### 8.3 Binding to a C2PA Manifest

The `cawg.agent_identity` assertion is hashed and referenced by the C2PA
Manifest's claim like any other assertion; the C2PA claim signature covers it.
When both a `cawg.agent_identity` and a (human) CAWG identity assertion are
present, they compose: the human assertion attributes the work to a creator, the
agent assertion attributes the generation to an attested agent acting under that
creator's ownership. The agent assertion's `owner_did` SHOULD resolve to the
same principal as the human identity assertion.

### 8.4 Verification

A verifier of a `cawg.agent_identity` assertion MUST:

- **V-A1.** Verify the enclosing C2PA Manifest's claim signature.
- **V-A2.** Resolve `authority_did` and confirm its key chains to a trusted Root
  Identity Agent independent of the content's signer (§3.4).
- **V-A3.** When the full AID is available, verify it per §4.12 and confirm its
  `aid.id` equals the assertion's `aid_id`.
- **V-A4.** Treat the assertion as evidence of *generation provenance* only — it
  does not assert correctness, safety, or quality of the content.

### 8.5 Upstream Intent

`cawg.agent_identity` is specified here so the Arkavo ecosystem can use it
immediately. It is structured as a candidate contribution to the Creator
Assertions Working Group: the label is namespaced `cawg.` in anticipation of
adoption, and the payload is a strict projection of an AID. Until CAWG adopts an
agent identity assertion, implementations MUST treat `cawg.agent_identity` as an
Arkavo extension assertion.

---

## 9. Lifecycle and Revocation

### 9.1 Issuance

Per §6.1. A credential exists from the moment the Authority signs the AID.

### 9.2 Rotation

Per §6.5. Rotation supersedes a prior AID without revoking the subject's
identity. WIMSE-style short validity windows are RECOMMENDED for `aid.expires`.

### 9.3 Revocation

The Authority revokes a compromised agent (R6) before `aid.expires` by:

- publishing the revoked `aid.id` to the revocation list carried in the
  published trust chain (§3.4), resolvable by every verifier;
- when the credential gates TDF data access, notifying the KAS so that
  identity-originated attributes for the revoked agent cease to release keys
  (the KAS performs the key-release decision — AIA does not, §3.2 N5);
- emitting a revocation entry to the orchestrator's lineage stream.

A verifier with access to the revocation list MUST check it and MUST reject a
revoked credential.

### 9.4 Expiry

A credential whose `aid.expires` is in the past MUST be rejected by every
verifier, with no grace period. An agent whose credential has expired MUST
obtain a rotated credential (§6.5) before continuing to act.

---

## 10. Security Considerations

### 10.1 Threat Model

- **The orchestrator as a super-agent** — the motivating threat (§1.1, §3.3).
  Mitigated: identity is issued by an upstream Authority under a Root Identity
  Agent anchor; the orchestrator holds an AIA credential like any agent and
  cannot issue or co-sign identity.
- **Forged identity by a compromised orchestrator** — mitigated as above; the
  orchestrator does not hold the Authority key.
- **Forged model claim** — an agent claims a stronger/safer model than it runs.
  Mitigated by Model Attestation (§5): the weight digest is verified against an
  expected value (§5.7 V-M1), and `trust_level` reflects evidence strength.
- **Privilege smuggling via identity** — an attacker tries to ship `tool_access`
  or `resource_scope` inside a credential. Mitigated by §3.5: an AID MUST NOT
  carry attributes outside `agent.identity.*`, and a KAS MUST NOT accept them
  from a credential.
- **Credential replay / theft** — mitigated by proof-of-possession (§4.6): a
  thief lacking the `key.cnf` private key fails the PoP challenge. A
  flight-scoped credential is additionally bound by `flight_binding` (§4.5).
- **Stale attestation** — mitigated by `evidence.nonce` (§10.4) and short
  `aid.expires`.
- **Owner spoofing** — mitigated by `owner.binding`: `owner_signed` carries the
  owner's signature; `delegated` rests on the Authority.
- **Delegation-chain privilege escalation** — an attacker tries to widen
  authority down a chain. AIA itself never delegates authority (§3.2 N4); the
  identity re-binding it performs is bounded by `may_act` and
  `scope_reduction_only` (§4.9, §6.4).
- **Compromised Authority** — the worst case below the root. Mitigations:
  hardware-rooted Authority keys, the two-tier Root/Authority split (§3.4) so
  the root can re-certify, short credential validity, revocation, and the option
  to require `rats-hardware` evidence the Authority alone cannot fabricate.
- **Compromised Root Identity Agent** — catastrophic, which is why the root is
  offline/hardware-protected and issues no day-to-day credentials (§3.4).

### 10.2 The Super-Agent Foot-Gun

In SwarmKit without AIA, the predicate "agent A is a `coding` agent running
`gemma-4`, trust level attested, for owner O" is supported by exactly one
signature: the orchestrator's. The orchestrator both creates that identity and
grants A its access — a super-agent. AIA splits the two: identity is the
Authority's signature over an AID plus Model Attestation evidence; access is the
orchestrator's capability grant. Neither role can perform the other's act. The
orchestrator's compromise remains serious but is no longer identity forgery.

### 10.3 OWASP Top 10 for Agentic Applications Mapping

The OWASP Top 10 for Agentic Applications (2026) [OWASP-AGENTIC] enumerates the
principal agentic risk classes. AIA is a direct control for several:

| OWASP class | AIA contribution |
|---|---|
| **ASI03 — Identity & Privilege Abuse** | Primary target. Identity is an independently attested, PoP-bound, owner-linked fact; the §3.5 attribute boundary stops privilege riding inside a credential. |
| **ASI10 — Rogue Agents** | An agent with no valid credential, or `trust_level: unverified`, is detectable as rogue at every verifier. |
| **ASI04 — Agentic Supply Chain** | The weight digest (§5.2) pins the exact model artifact; substitution fails §5.7 V-M1. |
| **ASI07 — Insecure Inter-Agent Communication** | A2A peers authenticate by credential + PoP, not transport trust alone. |
| **ASI09 — Human-Agent Trust Exploitation** | The `owner` binding records *whose* authority an agent acts under; `binding: self` makes a missing human principal explicit. |

AIA does **not** address goal hijack (ASI01), tool misuse (ASI02), code
execution (ASI05), memory poisoning (ASI06), or cascading failure (ASI08): those
concern what an agent *does*, which AIA explicitly leaves to the Planner, the
orchestrator, TØR-G, and ARP (§1.2, §3.2 N1–N5).

### 10.4 Attestation Freshness

`evidence.nonce` (§5.4) MUST be unique per issuance and SHOULD be supplied by
the Authority or a verifier, not by the subject. A verifier MUST reject evidence
whose nonce it did not expect for this issuance (§5.7 V-M2).

### 10.5 Owner Compromise

A compromised owner key lets an attacker authorize agents in that owner's name.
This is outside AIA's mitigation surface — AIA attests *that* an owner binding
exists, not that the owner is uncompromised.

---

## 11. Conformance

A conforming **Agent Identity Authority** MUST:

- C-A1. Issue agent identities and credentials per §6, content-addressed and
  signed per §4.12 (R1, R2).
- C-A2. Verify Model Attestation evidence before issuance and set
  `subject.trust_level` from its strength (§5.7, §4.3) (R4).
- C-A3. Sign AIDs with a key chaining to a Root Identity Agent independent of
  the orchestrator, and publish the trust chain and revocation list (§3.4)
  (R5).
- C-A4. Support key rotation and revocation (§6.5, §9.3) (R3, R6).
- C-A5. Never issue an AID carrying authorization — no work assignment, no
  resource scope, no tool grant, no execution-mode permission, no ABAC rule, no
  wrapped key — and confine originated attributes to `agent.identity.*`
  (§3.2 N1–N5, §3.5).
- C-A6. Re-attest, never copy, identity across delegation hops (§6.4).

A conforming **agent** (credential holder) MUST:

- C-S1. Prove possession of its `key.cnf` private key when presenting its
  credential (§4.6).
- C-S2. Refuse a SwarmKit delegation envelope that lacks `aid_ref` in a flight
  that uses AIA (§6.3).
- C-S3. Verify both its delegation envelope and the orchestrator's credential
  before acting (§6.3).

A conforming **verifier** MUST:

- C-V1. Recompute `aid.id` and reject on mismatch (§4.12).
- C-V2. Verify the Authority signature up the published trust chain and confirm
  the Root Identity Agent is trusted and orchestrator-independent (§3.4, §4.11).
- C-V3. Require a successful proof-of-possession challenge; never accept a
  credential as a bearer token (§4.6).
- C-V4. Reject expired (§9.4) and, when the revocation list is reachable,
  revoked (§9.3) credentials.
- C-V5. Reject an AID whose `flight_binding`, when present, does not match the
  flight of presentation, and whose `did` / `spiffe_id` are inconsistent
  (§4.5, §7.3).

A conforming **KAS** integrating AIA MUST:

- C-K1. Accept only `agent.identity.*` attributes derived from a credential, and
  obtain all other agent attributes from the orchestrator's capability grant
  (§3.5).

A conforming **Model Attestor** MUST:

- C-M1. Compute the weight digest per §5.2.1.
- C-M2. Emit a §5.4 evidence block, or §5.5 SPIRE selectors, or both.
- C-M3. Mark software-only measurements `rats-software` and hardware-rooted
  measurements `rats-hardware`; never label software evidence as hardware.

---

## 12. Examples

The example values below (digests, nonces, signatures) are realistic byte
lengths but are not derived from content and are not verifiable.

### 12.1 Issued Agent Identity Credential (compact form)

The token an agent carries and presents — an AID projected to a WIT (§7.1):

```json
{
  "iss": "did:web:swarm-id.arkavo.net",
  "sub": "did:web:agent.arkavo.net:worker-17",
  "agent_type": "coding",
  "trust_level": "attested",
  "runtime": "local",
  "model": "gemma-4",
  "capabilities": ["a2a", "mcp"],
  "cnf": {
    "jwk": { "kty": "OKP", "crv": "Ed25519",
             "x": "Lm9pQrStUvWxYz0aBcDeFgHjKlMnPqRsTuVwXy12345", "alg": "EdDSA" }
  },
  "exp": "2026-06-01T00:00:00Z",
  "aid_id": "blake3:7Yk2pQ9wRtL4mNvX1aBc3dEfGhJkLmNpQrStUvWxYz0"
}
```

### 12.2 Full AID (durable agent identity)

```json
{
  "aia_spec": "0.1.0",
  "aid": {
    "id": "blake3:7Yk2pQ9wRtL4mNvX1aBc3dEfGhJkLmNpQrStUvWxYz0",
    "issued": "2026-05-22T14:00:00Z",
    "expires": "2026-06-01T00:00:00Z",
    "nonce": "k3Jd9sLpQ2mWxYz8aBcDeg"
  },
  "subject": {
    "did": "did:web:agent.arkavo.net:worker-17",
    "spiffe_id": "spiffe://arkavo.net/agent/worker-17",
    "agent_type": "coding",
    "role_type": "operator",
    "trust_level": "attested",
    "runtime": "local",
    "organization": "did:web:arkavo.net",
    "capabilities": ["a2a", "mcp"],
    "display_name": "worker-17"
  },
  "owner": {
    "did": "did:web:arkavo.net",
    "owner_type": "organization",
    "binding": "delegated"
  },
  "key": {
    "cnf": {
      "jwk": { "kty": "OKP", "crv": "Ed25519",
               "x": "Lm9pQrStUvWxYz0aBcDeFgHjKlMnPqRsTuVwXy12345", "alg": "EdDSA" }
    }
  },
  "model_attestation": {
    "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
    "weights": {
      "digest_algorithm": "blake3",
      "digest": "blake3:9f8e7d6c5b4a3210FeDcBa9876543210AbCdEfGh",
      "shard_layout": "single",
      "merkle_root": null,
      "artifact_count": 1
    },
    "evidence": {
      "format": "rats-hardware",
      "produced_by": "arkavo-model-attestor/0.1.0",
      "produced_at": "2026-05-22T13:59:40Z",
      "nonce": "k3Jd9sLpQ2mWxYz8aBcDeg",
      "claims_digest": "blake3:1a2b3c4d5e6f7890AbCdEfGhIjKlMnOpQrStUvWx",
      "hardware_evidence": "Q1VFdmlkZW5jZUJsb2JCYXNlNjR1cmxFbmNvZGVk"
    }
  },
  "runtime_attestation": {
    "backend": "llama.cpp",
    "backend_version": "b4567",
    "backend_digest": "blake3:5e6f7a8b9c0d1234EfGhIjKlMnOpQrStUvWxYz01",
    "host": { "isolation": "container", "tee": "sev-snp", "measured": true },
    "evidence_ref": "rats:blake3:c0ffee1234567890AbCdEfGhIjKlMnOpQrStUvWx"
  },
  "assertions": [],
  "signature": {
    "authority_did": "did:web:swarm-id.arkavo.net",
    "trust_anchor": "did:web:root-id.arkavo.net",
    "algorithm": "ed25519",
    "value": "MEUCIQDxAmpLe5ig6n4tUreSi9gN4tUreVa1ueF0rD0cuMent4t10nPurP0seZ87"
  }
}
```

### 12.3 Sharded Model Attestation

```json
"model_attestation": {
  "model": { "family": "qwen3", "size": "235B-MoE", "quantization": "MXFP4" },
  "weights": {
    "digest_algorithm": "blake3",
    "digest": "blake3:aabbccdd00112233MerkleRootOverShardDigests",
    "shard_layout": "sharded",
    "merkle_root": "blake3:aabbccdd00112233MerkleRootOverShardDigests",
    "artifact_count": 12
  },
  "evidence": {
    "format": "rats-software",
    "produced_by": "arkavo-model-attestor/0.1.0",
    "produced_at": "2026-05-22T13:59:40Z",
    "nonce": "p9QmWxYz8aBcDek3Jd9sLp",
    "claims_digest": "blake3:abcdef0123456789ClaimsDigestOverModelWeights",
    "hardware_evidence": null
  }
}
```

### 12.4 SPIRE Registration Entry (illustrative)

```
spire-server entry create \
  -spiffeID  spiffe://arkavo.net/agent/worker-17 \
  -parentID  spiffe://arkavo.net/aia/swarm-identity-agent \
  -selector  arkavo:model-family:gemma-4 \
  -selector  arkavo:weights:blake3:9f8e7d6c5b4a3210FeDcBa9876543210AbCdEfGh \
  -selector  arkavo:runtime:llama.cpp \
  -selector  arkavo:tee:sev-snp \
  -ttl       864000
```

### 12.5 CAWG Agent Identity Assertion

```json
{
  "label": "cawg.agent_identity",
  "data": {
    "aia_spec": "0.1.0",
    "aid_id": "blake3:7Yk2pQ9wRtL4mNvX1aBc3dEfGhJkLmNpQrStUvWxYz0",
    "subject_did": "did:web:agent.arkavo.net:worker-17",
    "agent_type": "coding",
    "trust_level": "attested",
    "owner_did": "did:web:arkavo.net",
    "owner_binding": "delegated",
    "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
    "weights_digest": "blake3:9f8e7d6c5b4a3210FeDcBa9876543210AbCdEfGh",
    "attestation_format": "rats-hardware",
    "authority_did": "did:web:swarm-id.arkavo.net",
    "aid_signature": "MEUCIQDxAmpLe5ig6n4tUreSi9gN4tUreVa1ueF0rD0cuMent4t10nPurP0seZ87"
  }
}
```

---

## 13. Open Questions

1. **Authority redundancy.** Should a deployment support more than one Authority
   (a credential counter-signed by two), and how do verifiers express an
   `n-of-m` policy?
2. **SPIRE node attestation reuse.** Can the Root Identity Agent (§3.4) be the
   SPIRE node attestation chain directly, collapsing two trust hierarchies?
3. **Weight digest of mutated weights.** LoRA adapters and runtime quantization
   change loaded weights versus the published artifact. Should §5.2 define a
   base-plus-adapter digest pair?
4. **Credential transport.** Should AIA define an A2A JSON-RPC method for
   credential presentation and PoP challenge, or leave it to the WIT form (§7.1)?

---

## 14. References

### 14.1 Normative

- **SwarmKit** — `swarmkit/swarmkit-spec-draft-01.md` (this repo).
- **BCP 14** — RFC 2119, RFC 8174 (requirements language).
- **JSON** — RFC 8259.
- **JSON Canonicalization Scheme (JCS)** — RFC 8785.
- **base64url** — RFC 4648 §5.
- **URI** — RFC 3986.
- **Proof-of-Possession Key Semantics for JWTs** — RFC 7800 (the `cnf` claim).
- **OAuth 2.0 Token Exchange** — RFC 8693 (`act` / `may_act` delegation
  semantics; AIA adopts the delegation pattern, not token exchange).
- **RATS Architecture** — RFC 9334.
- **WIMSE Workload Credentials** — [WIMSE-CREDS] `draft-ietf-wimse-workload-creds-00`.
- **WIMSE Architecture** — [WIMSE-ARCH] `draft-ietf-wimse-arch-07`.
- **BLAKE3** — https://github.com/BLAKE3-team/BLAKE3-specs.

### 14.2 Informative

- **Agent Runtime Policy (ARP)** — `agent-runtime-policy/arp-spec-draft-00.md`
  (this repo).
- **TØR-G** — `torg-decision/draft-arkavo-torg-decision-00.md` (this repo).
- **WIMSE Applicability for AI Agents** — [WIMSE-AI]
  `draft-ni-wimse-ai-agent-identity-02` (owner concept, dual-identity
  credentials, operational models).
- **SPIFFE** — Secure Production Identity Framework for Everyone; SPIFFE ID,
  X.509-SVID, JWT-SVID — https://spiffe.io.
- **C2PA** — Coalition for Content Provenance and Authenticity.
- **CAWG** — Creator Assertions Working Group; identity assertion v1.x.
- **OWASP Top 10 for Agentic Applications (2026)** — [OWASP-AGENTIC] OWASP GenAI
  Security Project, published 2025-12-09.
- **Entity Attestation Token (EAT)** — IETF RATS Working Group; an alternative
  evidence encoding for §5.4.
- **DPoP** — RFC 9449 (a proof-of-possession mechanism usable for §4.6).

### 14.3 Forthcoming and Deferred

- **Task Contract Protocol (TCP)** — a future Arkavo specification (referenced
  by SwarmKit §1.2 and §14.2 as forthcoming). Profiling TCP as an RFC 8693
  OAuth 2.0 Token Exchange profile is **deferred**, blocked on the TCP
  specification being published. AIA §7.5 adopts only RFC 8693 delegation
  *semantics*, not token exchange, so the later profile can be layered without
  revising AIA.
- **Sequence Integrity Specification** — a future Arkavo specification defining
  lineage events; referenced by §9.3.

---

## Appendix A: `arkavo` SPIRE Selector Vocabulary

| Selector key | Example | Source AID field |
|---|---|---|
| `model-family` | `arkavo:model-family:gemma-4` | `model_attestation.model.family` |
| `model-size` | `arkavo:model-size:26B-MoE` | `model_attestation.model.size` |
| `model-quant` | `arkavo:model-quant:Q4_K_M` | `model_attestation.model.quantization` |
| `weights` | `arkavo:weights:blake3:9f8e...` | `model_attestation.weights.digest` |
| `runtime` | `arkavo:runtime:llama.cpp` | `runtime_attestation.backend` |
| `runtime-version` | `arkavo:runtime-version:b4567` | `runtime_attestation.backend_version` |
| `tee` | `arkavo:tee:sev-snp` | `runtime_attestation.host.tee` |
| `evidence` | `arkavo:evidence:rats-hardware` | `model_attestation.evidence.format` |

## Appendix B: Identity-Originated OpenTDF Attributes

The complete `agent.identity.*` namespace AIA originates (§3.5). No other
`agent.*` attribute may originate from a credential.

| Attribute | AID source | Example value |
|---|---|---|
| `agent.identity.type` | `subject.agent_type` | `coding` |
| `agent.identity.trust_level` | `subject.trust_level` | `attested` |
| `agent.identity.runtime` | `subject.runtime` | `local` |
| `agent.identity.owner` | `owner.did` | `did:web:arkavo.net` |
| `agent.identity.organization` | `subject.organization` | `did:web:arkavo.net` |

Orchestrator-issued (never from a credential): `agent.tool_access`,
`agent.code_authority`, `agent.resource_scope`, `agent.execution_mode`.

## Appendix C: The Agent Ecosystem

| Tier | Role | `agent_type` | Issues | Answers |
|---|---|---|---|---|
| Trust anchor | Root Identity Agent | — | Authority certifications | the chain's root |
| Identity tier | Agent Identity Authority | `identity_authority` | Agent Identity Credentials | who is this agent |
| Orchestration | Orchestrator Agent | `orchestrator` | delegation envelopes, capability grants | what work, what access |
| Work | Worker Agents (scribe, historian, planner, critic, operator) | per role | — | the objective |
| Enforcement | OpenTDF KAS | — | key release decisions | may this key release |

Every agent below the trust anchor — the orchestrator included — holds an Agent
Identity Credential issued by an Agent Identity Authority.
