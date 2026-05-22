# Agent Identity Attestation (AIA) Specification

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

**Agent Identity Attestation (AIA)** specifies how the identity of an AI agent is
issued, attested, and verified independently of the agent that orchestrates its
work. AIA defines a sixth role for the SwarmKit multi-agent mesh — the
**Identity Attestor** — alongside the existing Scribe, Historian, Planner,
Critic, and Operator archetypes.

AIA introduces the **Agent Identity Document (AID)**: a signed, content-addressed
record that binds a specialist agent to (a) an attested model and runtime, (b)
an owner principal, and (c) a single SwarmFlight role, without conferring any
authorization. This document specifies the AID schema, a SPIFFE/SPIRE-compatible
**Model Attestation Profile**, the identity issuance flow, interoperability
mappings to WIMSE workload credentials, and a CAWG **agent identity assertion**
type for C2PA content provenance.

AIA closes a structural gap in SwarmKit AE-2026-004: in the five-role mesh,
identity issuance is folded into orchestration implicitly — the orchestrator
that decrypts a SwarmKit and signs delegation envelopes is also, in effect, the
authority that mints each specialist's identity. AIA separates these concerns so
that a compromised orchestrator can mis-assign work but cannot forge an attested
identity.

---

## 1. Introduction

### 1.1 Motivation

SwarmKit (`swarmkit/swarmkit-spec-draft-01.md`) decrypts a signed work package
and delegates role-scoped configuration to specialist agents. The orchestrator
signs each delegation envelope (SwarmKit §7.2) with a `delegation_signature`.
Specialists, peers, Key Access Services (KAS), and MCP brokers currently treat
the orchestrator's signature as the sole evidence of *who a specialist is*.

This conflates two distinct questions:

1. **Authorization** — *what work is this agent assigned?* Answered by the
   Planner (which decomposes the objective) and the orchestrator (which
   delegates role-scoped configuration).
2. **Identity** — *who is this agent, what model is it running, and on whose
   behalf does it act?* Currently answered by nothing — it is inferred from
   the same orchestrator signature that answers question 1.

Folding identity into orchestration creates a foot-gun: the orchestrator can
**mint identity**. A compromised orchestrator does not merely mis-route work
(SwarmKit §10.1 already classifies this as "catastrophic by design"); it can
fabricate a specialist with an arbitrary model claim, an arbitrary owner, and
arbitrary attributes, because nothing else vouches for those facts. There is no
artifact a downstream verifier can check that the orchestrator did not itself
produce.

AIA removes identity issuance from the orchestrator and assigns it to a
dedicated role. After AIA, the orchestrator still assigns work, but a specialist
authenticates to KAS, MCP brokers, and peers using an **Agent Identity Document**
signed by the Identity Attestor and bound to attestation evidence the
orchestrator does not control.

### 1.2 Relationship to Existing Specifications

AIA is deliberately narrow. It issues and attests identity. It does not assign
work, gate actions, score outputs, or adapt behavior. The following table fixes
the boundaries.

| Specification / Role | Concern | AIA overlap |
|---|---|---|
| **SwarmKit** (`swarmkit/`) | Work packaging; orchestrator decryption and delegation. | AIA adds the Identity Attestor role to the mesh and a pre-delegation issuance step (§6). AIA does **not** change the SwarmKit manifest schema or the orchestrator's decryption flow. |
| **Planner role** | Decomposes the objective into ordered subtasks; assigns work. | None. AIA never assigns, orders, or routes work. The Planner answers *what*; AIA answers *who*. |
| **TØR-G** (`torg-decision/`) | Boolean-circuit decision IR; gates whether a requested action executes. | None. AIA never gates an action. A verifier MAY feed an AID verification result into a TØR-G input (e.g. `IN_identity_attested`), but AIA itself emits no decision. |
| **Critic role** | Scores outputs against the evaluation rubric. | None. AIA attests identity, not quality. |
| **Agent Runtime Policy (ARP)** (`agent-runtime-policy/`) | Runtime adaptation of a *running* specialist. | None. ARP adapts behavior within boundaries; AIA fixes identity at issuance. An AID is an input to ARP's `integrity` binding, not a product of it. |
| **Orchestrator** | Decrypts the SwarmKit; delegates role-scoped configuration. | AIA strips identity-minting from the orchestrator. The orchestrator continues to sign delegation envelopes (authorization); AIA signs AIDs (identity). |

A one-line summary: **Planner assigns, TØR-G gates, Critic scores, ARP adapts,
the orchestrator delegates — AIA only attests.**

### 1.3 The Sixth Role

SwarmKit Appendix C defines a five-archetype mesh: `scribe`, `historian`,
`planner`, `critic`, `operator`. AIA adds a sixth recommended `role_type`:
`identity_attestor`. SwarmKit Appendix C already declares `role_type` an open
vocabulary, so this requires no change to the SwarmKit schema; a SwarmKit
producer adopts AIA simply by including a role with
`role_type: "identity_attestor"` and the provisioning described in §6.

### 1.4 Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14
[RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown
here.

---

## 2. Terminology

- **AIA** — Agent Identity Attestation; this specification, and the discipline
  it defines.
- **Identity Attestor** — The mesh role responsible for AIA. `role_type`
  value: `identity_attestor`. Referred to as "the Attestor" below.
- **Agent Identity Document (AID)** — A signed, content-addressed JSON document
  that binds a specialist agent to an attested model/runtime, an owner, and one
  SwarmFlight role. Defined in §4.
- **Subject** — The agent an AID describes; identified by a DID and/or a SPIFFE
  ID.
- **Owner** — The principal (person, device, or policy engine) on whose behalf
  the subject acts. Borrowed from WIMSE's AI-agent identity model.
- **Model Attestation** — Evidence and verified claims about the model weights
  and inference runtime a subject is executing. Defined in §5.
- **Model Attestor** — The component that gathers Model Attestation evidence. In
  a SPIRE deployment, a SPIRE workload attestor plugin (§5.6).
- **Weight Digest** — A BLAKE3 digest of a model's canonical weight artifacts;
  the content address of the model.
- **SPIRE Selector** — A `type:value` string a SPIRE workload attestor returns
  to identify a workload. AIA defines the `arkavo` selector type (§5.5).
- **WIT / WIC** — WIMSE Workload Identity Token / Workload Identity Certificate
  [WIMSE-CREDS].
- **Proof-of-Possession (PoP)** — Demonstration that a presenter holds the
  private key bound to a credential. An AID is not a bearer token (§4.6).
- **Issuance** — The act of an Attestor producing and signing an AID.
- **Re-binding** — Producing a fresh AID for a downstream hop of a delegation
  chain, with a re-scoped security context (§6.4).
- **CAWG** — Creator Assertions Working Group; defines identity assertions for
  C2PA content provenance.
- **SwarmFlight** — A single execution instance of a SwarmKit (SwarmKit §2).

---

## 3. The AIA Role

### 3.1 Position in the SwarmKit Mesh

```
                  ┌──────────────────────────────────────────┐
                  │              SwarmKit.tdf                 │
                  └────────────────────┬─────────────────────┘
                                       │ decrypt
                              ┌────────▼─────────┐
                              │   Orchestrator   │  assigns work
                              └───┬─────────┬────┘
              issue AID request   │         │   delegation envelope
                        ┌─────────▼──┐      │   (authorization)
                        │  Identity  │      │
                        │  Attestor  │ AID  │
                        │   (AIA)    ├──────┤
                        └────────────┘      │
                          attests identity  │
        ┌────────────────┬──────────────────┼───────────────┬─────────────┐
        ▼                ▼                  ▼               ▼             ▼
   ┌─────────┐     ┌──────────┐       ┌──────────┐    ┌──────────┐  ┌──────────┐
   │ Scribe  │     │Historian │       │ Planner  │    │  Critic  │  │ Operator │
   └─────────┘     └──────────┘       └──────────┘    └──────────┘  └──────────┘
   each specialist holds: { AID (from AIA), delegation envelope (from orchestrator) }
```

The Identity Attestor is a peer role in the mesh, not a layer above it. It is
provisioned by the orchestrator like any other specialist (§6.3), but it is the
only role permitted to sign AIDs, and its signing key is rooted in a trust
anchor distinct from the orchestrator's (§3.4).

### 3.2 Responsibilities

A conforming Identity Attestor:

- **R1.** Issues exactly one AID per (SwarmFlight, role) pair (§6).
- **R2.** Gathers or verifies Model Attestation evidence for each subject before
  issuance (§5).
- **R3.** Binds each AID to an owner principal (§4.4).
- **R4.** Re-binds identity on each hop of a delegation chain, re-scoping the
  security context (§6.4).
- **R5.** Maintains AID lifecycle: rotation and revocation (§9).
- **R6.** OPTIONALLY emits a CAWG agent identity assertion for content a subject
  produces (§8).

**Non-responsibilities (normative).** An Identity Attestor **MUST NOT**:

- **N1.** Assign, order, or route work. (That is the Planner and orchestrator.)
- **N2.** Gate, allow, or deny a requested action. (That is TØR-G and policy
  enforcement.)
- **N3.** Score or evaluate a subject's outputs. (That is the Critic.)
- **N4.** Decrypt the SwarmKit payload or hold the SwarmKit-level wrapped key.
  (That is the orchestrator.)
- **N5.** Issue an AID that itself carries authorization grants, scopes, or
  capabilities. An AID states facts; authorization is a separate artifact
  (the delegation envelope, the TDF Attribute Release Policy, the MCP grant).

Constraint N5 is the heart of AIA: **identity and authorization are distinct
artifacts, signed by distinct authorities.** A verifier that needs both checks
both.

### 3.3 Separation from Orchestration

Before AIA, the chain of trust for a specialist's identity is:

```
specialist identity  ⇐  orchestrator's delegation_signature  ⇐  orchestrator key
```

The orchestrator is the sole authority. There is no second signature to check.

After AIA:

```
specialist identity  ⇐  AID  ⇐  Attestor key  ⇐  AIA trust anchor (§3.4)
specialist work      ⇐  delegation envelope  ⇐  orchestrator key
```

A downstream verifier (KAS, MCP broker, peer) authenticates the specialist via
its AID and the Attestor key, and authorizes the specialist's request via the
delegation envelope and the orchestrator key. The two checks are independent.

Consequently, a compromised orchestrator:

- **can** still mis-assign work, forge delegation envelopes, and route data
  incorrectly within its own clearance (unchanged from SwarmKit §10.1);
- **cannot** forge a specialist's attested identity, because it does not hold
  the Attestor key and cannot produce Model Attestation evidence (§5.4) that a
  verifier will accept.

This does not make a compromised orchestrator safe. It bounds the blast radius:
identity-dependent trust signals (KAS attribute release, MCP broker decisions,
peer-to-peer A2A authentication, content provenance) remain sound even when the
orchestrator is hostile.

### 3.4 Trust Anchor

The Attestor's signing key MUST be rooted in a trust anchor that is **not** the
orchestrator's key and **not** derived from the SwarmKit-level wrapped key.
Acceptable anchors, in decreasing order of assurance:

1. A hardware-backed key whose attestation evidence chains to a manufacturer or
   platform root (TPM, TEE, secure element).
2. An organizational root DID (`did:web`) published out-of-band, with the
   Attestor key cross-signed by that root.
3. The SwarmKit author's DID (`kit.authors[].did`, SwarmKit §4.1), when the
   author and the deploying organization are the same trust domain.

A verifier MUST be configured with, or able to resolve, the AIA trust anchor
independently of any artifact the orchestrator produces. An Attestor whose key
chains only to the orchestrator provides no separation and MUST be treated by
verifiers as equivalent to no AIA at all.

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

File extension: `.aid.json`.

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
| `flight_binding` | yes | §4.5 |
| `key` | yes | §4.6 |
| `model_attestation` | yes | §5 |
| `runtime_attestation` | yes | §4.8 |
| `delegation` | no | §4.9 — present iff the subject acts in a delegation chain |
| `assertions` | no | §4.10 — default `[]` |
| `signature` | yes | §4.11 |

### 4.3 subject

The agent the AID describes.

```json
"subject": {
  "did": "did:web:specialist.arkavo.net:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",
  "spiffe_id": "spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479/planner-1",
  "role_type": "planner",
  "display_name": "Planner specialist (gemma-4 26B-MoE)"
}
```

- `did` — REQUIRED. A resolvable DIF DID identifying the subject.
- `spiffe_id` — OPTIONAL but RECOMMENDED. A SPIFFE ID for the subject. When
  present, it MUST be consistent with the WIMSE mapping in §7.3.
- `role_type` — REQUIRED. The SwarmKit `role_type` of the subject.
- `display_name` — OPTIONAL human-readable label.

An AID describes exactly one subject. A subject MAY hold multiple AIDs across
flights, but at most one AID per (flight, role).

### 4.4 owner

The principal on whose behalf the subject acts. This is the WIMSE dual-identity
binding (§7.4): an agent's credential cryptographically references its owner so
that downstream services can evaluate provenance of authority, not merely
identity of process.

```json
"owner": {
  "did": "did:web:arkavo.net",
  "owner_type": "organization",
  "binding": "delegated",
  "owner_signature": "<base64url>"
}
```

- `did` — REQUIRED. Resolvable DID of the owner.
- `owner_type` — REQUIRED. One of `person`, `device`, `organization`,
  `policy_engine`.
- `binding` — REQUIRED. One of:
  - `delegated` — the owner authorized this agent's class of work in advance;
    the Attestor vouches for the link.
  - `owner_signed` — the owner directly signed this AID's owner block
    (`owner_signature` REQUIRED); strongest binding.
  - `self` — the subject is its own owner (autonomous agent with no external
    principal); permitted only when SwarmKit `constraints` admit autonomous
    operation.
- `owner_signature` — REQUIRED iff `binding` is `owner_signed`; an Ed25519
  signature by the owner key over the canonical `subject` + `owner` blocks
  (excluding `owner_signature` itself).

An autonomous action — one the subject takes that is not traceable to an owner
delegation — MUST be issued under `binding: self` or under a `delegation` block
(§4.9) whose `act` chain terminates in `self`. This realizes WIMSE arch §3.3.9.3:
autonomous actions MUST be distinguishable from delegated ones.

### 4.5 flight_binding

Binds the AID to one role of one SwarmFlight.

```json
"flight_binding": {
  "flight_id": "<uuid>",
  "kit_id": "blake3:<base64url>",
  "role_id": "planner-1"
}
```

All three members are REQUIRED. `kit_id` MUST equal the `kit.id` of the SwarmKit
being executed (SwarmKit §9.1). An AID whose `flight_binding` does not match the
flight in which it is presented MUST be rejected by the verifier. This scoping
makes an AID useless if exfiltrated to a different flight.

### 4.6 key

The subject's public key, for proof-of-possession. An AID is **not** a bearer
token: a presenter MUST prove possession of the corresponding private key
(§7.1, [WIMSE-CREDS] §3.1).

```json
"key": {
  "cnf": {
    "jwk": {
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "<base64url>",
      "alg": "EdDSA"
    }
  }
}
```

The `cnf` member follows [RFC7800]. The `jwk` MUST include an `alg`; the value
`none` MUST NOT be used and symmetric algorithms MUST NOT be used. A verifier
that accepts an AID without a successful PoP challenge is non-conforming (§11
C-V3).

### 4.7 model_attestation

The Model Attestation block. Its schema and semantics are defined in §5. It is
REQUIRED in every AID; an AID with no verifiable model attestation states a
weaker fact ("an agent calling itself X") than AIA intends and MUST NOT be
issued.

### 4.8 runtime_attestation

Claims about the execution environment, distinct from the model itself.

```json
"runtime_attestation": {
  "backend": "llama.cpp",
  "backend_version": "b4567",
  "backend_digest": "blake3:<base64url>",
  "host": {
    "isolation": "container",
    "tee": "none",
    "measured": true
  },
  "evidence_ref": "rats:<uri-or-content-hash>"
}
```

- `backend` — REQUIRED. The inference runtime: `llama.cpp`, `mlx`, `vllm`, or
  `custom`. Mirrors SwarmKit `agent_provisioning.model.backend`.
- `backend_version` — REQUIRED. Version string of the runtime build.
- `backend_digest` — RECOMMENDED. BLAKE3 of the runtime binary.
- `host.isolation` — REQUIRED. One of `process`, `container`, `vm`, `none`;
  mirrors SwarmKit `agent_provisioning.isolation.sandbox`.
- `host.tee` — REQUIRED. The trusted execution environment, or `none`. Values:
  `none`, `sev-snp`, `tdx`, `sgx`, `cca`, `nitro`.
- `host.measured` — REQUIRED boolean. Whether `evidence_ref` carries a hardware
  measurement (versus a software-only self-report).
- `evidence_ref` — OPTIONAL. A reference to RATS evidence for the runtime,
  resolvable by the verifier.

### 4.9 delegation

Present iff the subject operates inside a delegation chain. Models RFC 8693
delegation semantics: a chain of actors, each acting on behalf of the one
before, with explicit downstream constraints.

```json
"delegation": {
  "chain": [
    { "actor": "did:web:arkavo.net",                      "kind": "owner"  },
    { "actor": "did:web:...:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",      "kind": "agent"  }
  ],
  "act": "did:web:...:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:operator-1",
  "may_act": {
    "audience": ["spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479"],
    "max_depth": 4,
    "scope_reduction_only": true
  }
}
```

- `chain` — REQUIRED. The ordered delegation chain, root principal first, the
  immediate delegator last. Each entry has an `actor` DID and a `kind`
  (`owner` or `agent`).
- `act` — REQUIRED. The DID of the subject, restating §4.3 `subject.did`; the
  current actor, per RFC 8693 `act`.
- `may_act` — REQUIRED. Constraints the subject MUST observe when it delegates
  further:
  - `audience` — the trust domains within which the subject may re-bind
    identity. Re-binding outside this set MUST be refused (WIMSE arch §3.3.9.4).
  - `max_depth` — the maximum total chain length; an AID whose `chain` length
    plus one would exceed `max_depth` MUST NOT be issued for a further hop.
  - `scope_reduction_only` — when `true`, a re-bound downstream AID MUST carry a
    `model_attestation` and `flight_binding` no broader than this AID's, and
    MUST NOT introduce a new owner. Per WIMSE arch §3.3.9.2, the default is that
    a downstream hop propagates, and only narrows, the upstream security
    context.

### 4.10 assertions

References to CAWG agent identity assertions (§8) the Attestor has emitted or
will emit for content this subject produces. Each entry:

```json
{ "assertion_label": "cawg.agent_identity", "assertion_digest": "blake3:<base64url>" }
```

### 4.11 signature

The Attestor's signature over the AID.

```json
"signature": {
  "attestor_did": "did:web:aia.arkavo.net",
  "trust_anchor": "did:web:arkavo.net",
  "algorithm": "ed25519",
  "value": "<base64url>"
}
```

- `attestor_did` — REQUIRED. DID of the issuing Attestor.
- `trust_anchor` — REQUIRED. The anchor the Attestor key chains to (§3.4). A
  verifier MUST confirm this anchor is one it is configured to trust and is
  independent of the orchestrator.
- `algorithm` — REQUIRED. `ed25519`.
- `value` — REQUIRED. The signature, computed per §4.12.

### 4.12 Content Addressing, Canonicalization, and Signing

`aid.id` MUST be `blake3:` followed by the base64url (no padding) BLAKE3-256
digest of the JCS-canonical [RFC8785] encoding of the AID with the `aid.id` and
`signature` members **excluded**.

```
canonical_bytes = JCS_canonical_json( AID without aid.id and without signature )
aid.id          = "blake3:" || base64url( BLAKE3_256( canonical_bytes ) )
signature.value = base64url( Ed25519_sign( attestor_private_key,
                                           BLAKE3_256( canonical_bytes ) ) )
```

The Attestor signs the BLAKE3 digest of the same canonical bytes that produce
`aid.id`. A verifier MUST recompute `aid.id`, MUST reject the AID if the
recomputed value differs from the declared value, and MUST verify
`signature.value` against the recomputed digest using the Attestor key resolved
from `signature.attestor_did`.

This is the same canonicalization-and-digest discipline SwarmKit uses for
`kit.id` (§9.1) and skill signing (§8.1.2): BLAKE3 over JCS-canonical JSON.

---

## 5. Model Attestation Profile

### 5.1 Overview

Model Attestation answers a question no current agent-identity scheme answers
directly: *is this process actually running model X (weight digest Y) on
runtime Z?* It is the missing primitive — without it, a `model.family` claim in
a SwarmKit `agent_provisioning` block is an unverified assertion.

The profile is designed for **SPIFFE/SPIRE compatibility**. A SPIRE workload
attestor plugin produces selectors (§5.5) that a SPIRE server matches to a
SPIFFE ID; AIA reuses the same evidence to populate the `model_attestation`
block of an AID. A deployment MAY run AIA with SPIRE (selectors drive
SVID issuance) or standalone (the Attestor consumes evidence directly) — the
evidence and claims are identical.

### 5.2 Weight Digest

```json
"model_attestation": {
  "model": {
    "family": "gemma-4",
    "size": "26B-MoE",
    "quantization": "Q4_K_M"
  },
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
- `weights.merkle_root` — REQUIRED when `shard_layout` is `sharded`; the
  BLAKE3 Merkle root over per-shard digests, sorted by shard index. `null`
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
inference process**, not over a packaging container (a `.gguf`/`.safetensors`
file's raw bytes are acceptable as "as loaded" only if the runtime memory-maps
them unmodified).

### 5.3 Runtime Measurement

`model_attestation` records *what model*; `runtime_attestation` (§4.8) records
*what runtime*. The two are separated so that a verifier can pin a model digest
across runtime upgrades, or pin a runtime across model swaps, independently.

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
    the weight bytes and runtime binary and hashed them. Trust rests on the
    integrity of the attestor and its host.
  - `rats-hardware` — the measurement chains to a hardware root of trust
    (TPM quote, TEE attestation report); `hardware_evidence` REQUIRED.
- `produced_by` — REQUIRED. Identifier and version of the Model Attestor.
- `produced_at` — REQUIRED. When evidence was gathered.
- `nonce` — REQUIRED. The verifier-supplied or Attestor-supplied freshness
  nonce; binds the evidence to a specific issuance and defeats replay (§10.4).
- `claims_digest` — REQUIRED. BLAKE3 of the JCS-canonical `model` + `weights`
  blocks; lets evidence be transported separately from the AID.
- `hardware_evidence` — REQUIRED when `format` is `rats-hardware`. An opaque,
  base64url-encoded RATS evidence blob (e.g. a TPM quote, an SEV-SNP report).
  `null` otherwise.

This block aligns with the RATS architecture [RFC9334]: the Model Attestor is
the *Attester*, the Identity Attestor (or SPIRE server) is the *Verifier*, and
the AID is a fragment of the resulting *Attestation Result*. Hardware-backed
evidence (`rats-hardware`) is RECOMMENDED for any flight whose SwarmKit declares
a data classification above `public`.

### 5.5 SPIRE Workload Selectors

A Model Attestor running as a SPIRE workload attestor plugin returns selectors
of type `arkavo`. The defined selector keys are:

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

A SPIRE registration entry that issues a SVID for, say, a Gemma-4 planner pins
the selector set `{arkavo:model-family:gemma-4, arkavo:weights:blake3:9f8e...,
arkavo:runtime:llama.cpp, arkavo:tee:none}`. SPIRE issues the SVID only to a
workload the plugin attests with all pinned selectors. The same selector set,
expanded to full values, populates the AID `model_attestation` block.

Selector values MUST be lowercase except for digest hex and version strings,
which are case-preserved. A value containing a colon MUST be percent-encoded per
[RFC3986] so the `type:key:value` split is unambiguous.

### 5.6 The Model Attestor

The Model Attestor is the component that produces §5.4 evidence. As a SPIRE
workload attestor plugin (`arkavo-model-attestor`) it:

1. Identifies the inference process (by PID handed to the plugin by the SPIRE
   agent).
2. Enumerates the weight artifacts that process has mapped or opened, and
   computes the weight digest (§5.2.1).
3. Identifies the runtime backend and version, and OPTIONALLY hashes the
   runtime binary.
4. Reads TEE attestation evidence if a TEE is present.
5. Returns the §5.5 selectors to the SPIRE agent.

A standalone (non-SPIRE) Model Attestor performs steps 1–4 and hands the
resulting evidence directly to the Identity Attestor. The wire format of the
evidence is the §5.4 `evidence` block in both cases.

This specification defines the *profile* — the evidence format, the digest
discipline, the selector vocabulary. A reference `arkavo-model-attestor`
implementation for `llama.cpp` and `mlx` is a separate engineering deliverable
and is out of scope here.

### 5.7 Verification

A verifier of `model_attestation` MUST:

- **V-M1.** Recompute `weights.digest` against an expected value when one is
  known (e.g. a SwarmKit-pinned digest, a SPIRE registration selector, or a
  published model manifest), and reject on mismatch.
- **V-M2.** Confirm `evidence.nonce` matches the nonce it expects for this
  issuance (§10.4).
- **V-M3.** When `evidence.format` is `rats-hardware`, verify
  `hardware_evidence` against the appropriate hardware root of trust.
- **V-M4.** Treat `rats-software` evidence as asserting only that *the attestor
  host* observed these bytes — no stronger than the attestor host's own
  integrity.

---

## 6. Identity Issuance Flow

### 6.1 Sequence

Issuance happens **before** the orchestrator sends a delegation envelope, so
that the envelope can reference an already-attested identity.

```
1.  Orchestrator decrypts the SwarmKit and validates the manifest
    (SwarmKit §7.1 steps 1–7) — unchanged.
2.  Orchestrator provisions the Identity Attestor role (§6.3).
3.  For each non-Attestor role R in the manifest:
    a. Orchestrator provisions the specialist process for R
       (SwarmKit §7.1.1) and obtains its public key.
    b. Orchestrator sends the Attestor an issuance request:
       { flight_id, kit_id, role_id, subject public key,
         expected model from roles[R].agent_provisioning.model }.
    c. Attestor obtains Model Attestation evidence for the specialist
       process (§5) — directly, or via a SPIRE SVID.
    d. Attestor verifies the evidence against the expected model (§5.7
       V-M1) and resolves the owner binding (§4.4).
    e. Attestor constructs, content-addresses, and signs the AID (§4.12).
    f. Attestor returns the AID to the orchestrator and, if configured,
       to the specialist directly.
4.  Orchestrator sends the SwarmKit delegation envelope to specialist R,
    adding an `aid_ref` member: the `aid.id` of R's AID.
5.  Specialist R, before acting, verifies BOTH:
       - its AID (Attestor signature, content address, model attestation);
       - its delegation envelope (orchestrator signature, SwarmKit §7.3).
6.  SwarmFlight begins.
```

Step 3c MAY run concurrently across roles. The Attestor MUST issue AIDs for all
non-Attestor roles; the Attestor does not issue an AID for itself (its identity
is its trust anchor, §3.4).

### 6.2 Binding to the SwarmFlight

`flight_binding` (§4.5) ties each AID to one (flight, role). The Attestor MUST
NOT issue an AID whose `flight_binding.kit_id` differs from the `kit.id` of the
SwarmKit it was provisioned under, and MUST NOT issue more than one AID per
(flight, role) except as a rotation (§9.2) that explicitly supersedes the prior
AID.

### 6.3 Interaction with the Orchestrator

The Attestor is provisioned by the orchestrator like any specialist (it receives
a SwarmKit delegation envelope). This is not circular: the delegation envelope
authorizes the Attestor to *do the work of attesting*; it does not confer the
Attestor's *identity*, which derives from the trust anchor (§3.4). The
orchestrator selects which Attestor to provision, but cannot forge what a
correctly-anchored Attestor signs.

An orchestrator MUST add the `aid_ref` member to every delegation envelope it
sends to a non-Attestor role once AIA is in use. A specialist that receives a
delegation envelope with no `aid_ref`, in a flight that provisioned an Attestor,
MUST refuse the delegation (§11 C-S2).

`aid_ref` is the only addition AIA makes to the SwarmKit delegation envelope. It
is OPTIONAL in the SwarmKit schema (a non-AIA flight omits it) and REQUIRED in
an AIA flight.

### 6.4 Re-binding on Delegation Chains

When a specialist delegates work to a sub-agent (a hierarchical SwarmKit, or an
A2A hand-off that crosses a trust boundary), the sub-agent MUST receive a fresh
AID, not a copy of the delegator's AID. The Attestor issues the downstream AID
with:

- `delegation.chain` extended by the delegator;
- `delegation.act` set to the sub-agent;
- `delegation.may_act` no broader than the delegator's (§4.9);
- `flight_binding` and `model_attestation` no broader than the delegator's when
  `scope_reduction_only` is `true`.

This realizes WIMSE arch §3.3.9.4: each hop explicitly re-scopes and re-binds
the security context so a downstream verifier can evaluate provenance. An AID
MUST NOT be re-bound to a trust domain outside `delegation.may_act.audience`.

---

## 7. WIMSE Interoperability

AIA is intentionally close to the IETF WIMSE work so that an Arkavo SwarmFlight
can interoperate with WIMSE-native infrastructure. This section maps AIA
constructs to [WIMSE-ARCH] and [WIMSE-CREDS]. The mapping is informative for
AIA conformance but normative for any implementation claiming WIMSE interop.

### 7.1 AID and the Workload Identity Token (WIT)

An AID is a superset of a WIMSE WIT. A conforming implementation MUST be able to
project an AID onto a WIT:

| WIT claim ([WIMSE-CREDS] §3.1) | AID source |
|---|---|
| `sub` (required, URI) | `subject.spiffe_id`, falling back to `subject.did` |
| `exp` (required) | `aid.expires` |
| `cnf` (required, `jwk`) | `key.cnf` verbatim |
| `iss` (optional, URI) | `signature.attestor_did` |
| `jti` (optional) | `aid.id` |
| header `typ` | `wit+jwt` |
| header `alg` | `EdDSA` |

The AID members with no WIT equivalent — `owner`, `flight_binding`,
`model_attestation`, `runtime_attestation`, `delegation`, `assertions` — are
carried as private claims under an `arkavo_aid` claim when an AID is transported
as a WIT, or referenced out-of-band by `aid.id`. Like a WIT, an AID MUST NOT be
used as a bearer token: a presenter proves possession of the `key.cnf` private
key ([WIMSE-CREDS] §3.1).

### 7.2 AID and the Workload Identity Certificate (WIC)

When a SwarmFlight authenticates at the transport layer, the subject MAY present
a WIC ([WIMSE-CREDS] §4.1) — an X.509 certificate with the subject identity in a
single URI `SubjectAltName`. AIA places `subject.spiffe_id` in that SAN, making
the WIC a valid X.509-SVID. The AID then carries the attestation context the
bare certificate cannot, and the two are linked by the SPIFFE ID.

### 7.3 did:web and spiffe://

A subject has both a `did:web` (DIF-resolvable, used for AID signature
verification and SwarmKit author chains) and a `spiffe://` ID (used by SPIRE and
WIMSE infrastructure). They MUST be deterministically related:

```
subject.did        = did:web:<authority>:flight:<flight_id>:<role_id>
subject.spiffe_id  = spiffe://<authority>/flight/<flight_id>/<role_id>
```

A verifier MUST reject an AID whose `did` and `spiffe_id` do not share the same
`<authority>`, `<flight_id>`, and `<role_id>`.

### 7.4 Owner Binding

WIMSE's AI-agent identity model [WIMSE-AI] introduces the *owner* — the person,
device, or policy engine an agent acts for — and *dual-identity credentials*
that bind agent to owner. AIA's `owner` block (§4.4) is exactly this binding.
The three AIA `owner.binding` values correspond to the three WIMSE operational
models: `delegated` ≈ agent-mediated, `owner_signed` ≈ owner-mediated,
and a server-mediated challenge maps to an Attestor that contacts the owner
before setting `binding: delegated`.

### 7.5 Delegation Chains

AIA `delegation` (§4.9) implements the RFC 8693 delegation pattern that
[WIMSE-ARCH] §3.3.6 builds on, and the per-hop re-binding that §3.3.9.4
requires. `delegation.act` is RFC 8693 `act`; `delegation.may_act` constrains
onward delegation as RFC 8693 `may_act` does. AIA does not adopt RFC 8693 token
*exchange* — the OAuth 2.0 Token Exchange profiling of the Arkavo Task Contract
Protocol is tracked separately and deferred (§14.3).

---

## 8. CAWG Agent Identity Assertion

### 8.1 Assertion Label and Purpose

SwarmKit `provenance.c2pa_assertions` already requires CAWG identity assertion
v1.x conformance — but the CAWG identity assertion describes a *human creator*.
Content produced by an attested agent needs an assertion that says "this output
was produced by attested agent identity X." AIA defines that assertion type.

- **Assertion label:** `cawg.agent_identity`
- **Assertion kind:** a C2PA assertion, embeddable in a C2PA Manifest, composing
  with — not replacing — the CAWG (human) identity assertion and standard C2PA
  provenance assertions.

### 8.2 Assertion Payload

The payload is a compact projection of the AID — the AID is, by construction,
most of the assertion already.

```json
{
  "label": "cawg.agent_identity",
  "data": {
    "aia_spec": "0.1.0",
    "aid_id": "blake3:<base64url>",
    "subject_did": "did:web:...:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",
    "subject_spiffe_id": "spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479/planner-1",
    "role_type": "planner",
    "owner_did": "did:web:arkavo.net",
    "owner_binding": "delegated",
    "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
    "weights_digest": "blake3:<base64url>",
    "attestation_format": "rats-hardware",
    "flight_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "attestor_did": "did:web:aia.arkavo.net",
    "aid_signature": "<base64url>"
  }
}
```

Every `data` member is copied verbatim from the AID it projects. `aid_signature`
is the AID's `signature.value`; a consumer that holds the full AID can verify
the assertion exactly as it verifies the AID (§4.12). A consumer that holds only
the assertion can still establish the chain attestor → subject → model, and
resolve the full AID by `aid_id`.

### 8.3 Binding to a C2PA Manifest

The `cawg.agent_identity` assertion is hashed and referenced by the C2PA
Manifest's claim like any other assertion; the C2PA claim signature therefore
covers it. When both a `cawg.agent_identity` and a (human) CAWG identity
assertion are present, they compose: the human assertion attributes the work to
a creator, the agent assertion attributes the generation to an attested agent
acting under that creator's ownership. The `owner_did` of the agent assertion
SHOULD resolve to the same principal as the human identity assertion.

### 8.4 Verification

A verifier of a `cawg.agent_identity` assertion MUST:

- **V-A1.** Verify the enclosing C2PA Manifest's claim signature, establishing
  the assertion was not altered after signing.
- **V-A2.** Resolve `attestor_did` and confirm its key chains to a trusted AIA
  anchor independent of the content's signer (§3.4).
- **V-A3.** When the full AID is available, verify it per §4.12 and confirm its
  `aid.id` equals the assertion's `aid_id`.
- **V-A4.** Treat the assertion as evidence of *generation provenance* only — it
  does not assert correctness, safety, or rubric quality of the content.

### 8.5 Upstream Intent

`cawg.agent_identity` is specified here so the Arkavo ecosystem can use it
immediately. It is structured as a candidate contribution to the Creator
Assertions Working Group: the label is namespaced `cawg.` in anticipation of
adoption, and the payload is a strict projection of an AID so the assertion adds
no vocabulary CAWG would have to re-derive. Until CAWG adopts an agent identity
assertion, implementations MUST treat `cawg.agent_identity` as an Arkavo
extension assertion.

---

## 9. Lifecycle and Revocation

### 9.1 Issuance

Per §6. An AID exists from the moment the Attestor signs it.

### 9.2 Rotation

An AID is rotated by issuing a new AID for the same (flight, role) with a fresh
`aid.id`, a fresh `aid.nonce`, a later `aid.issued`, and a `delegation` or
metadata note superseding the prior `aid.id`. Rotation is REQUIRED when the
subject's `key.cnf` changes, and RECOMMENDED on long flights at an interval no
longer than the SwarmKit's `constraints.global_budget.max_wallclock_seconds`.
WIMSE-style short validity windows ([WIMSE-CREDS] §3.1, "on the order of hours")
are RECOMMENDED for `aid.expires`.

### 9.3 Revocation

The Attestor MAY revoke an AID before `aid.expires` by:

- publishing the revoked `aid.id` to a revocation list resolvable by verifiers;
- when the AID gates TDF data access, issuing a revocation event to the KAS so
  role-scoped wrapped keys are invalidated (consistent with SwarmKit §7.4);
- emitting a revocation entry to the orchestrator's lineage stream.

A verifier with access to the revocation list MUST check it and MUST reject a
revoked AID. Because AIDs are short-lived (§9.2), revocation is a containment
mechanism, not the primary control — expiry is.

### 9.4 Expiry

An AID with `aid.expires` in the past MUST be rejected by every verifier, with
no grace period. A specialist whose AID has expired MUST obtain a rotated AID
(§9.2) before continuing to act.

---

## 10. Security Considerations

### 10.1 Threat Model

- **Forged identity by a compromised orchestrator** — the motivating threat
  (§1.1, §3.3). Mitigated: AIDs are signed by the Attestor under an independent
  trust anchor; the orchestrator cannot produce a valid AID.
- **Forged model claim** — an agent claims to run a stronger/safer model than it
  does. Mitigated by Model Attestation (§5): the weight digest is verified
  against an expected value (§5.7 V-M1).
- **AID replay across flights** — an AID exfiltrated and presented in a
  different flight. Mitigated by `flight_binding` (§4.5) and `aid.nonce`.
- **AID theft (bearer use)** — an AID copied and presented by another process.
  Mitigated by proof-of-possession (§4.6): a thief lacking the `key.cnf` private
  key fails the PoP challenge.
- **Stale attestation** — old, valid evidence re-presented for a process that
  has since changed. Mitigated by `evidence.nonce` (§10.4) and short
  `aid.expires`.
- **Owner spoofing** — an AID claims an owner that did not authorize it.
  Mitigated by `owner.binding`: `owner_signed` carries the owner's signature;
  `delegated` rests on the Attestor, which a verifier already trusts.
- **Delegation-chain privilege escalation** — a downstream hop widens its
  authority. Mitigated by `delegation.may_act` and `scope_reduction_only`
  (§4.9, §6.4), enforcing WIMSE arch §3.3.9.4.
- **Compromised Attestor** — the worst case. An attacker with the Attestor key
  forges AIDs at will. Mitigations: hardware-rooted Attestor keys (§3.4 anchor
  1), short AID validity, revocation, and the option to require `rats-hardware`
  model evidence the Attestor alone cannot fabricate. A compromised Attestor is
  to AIA what a compromised orchestrator is to SwarmKit — bounded by the trust
  anchor's assurance, not eliminated.

### 10.2 The Orchestrator-Minting Foot-Gun

Restating the core property for emphasis. In SwarmKit without AIA, the predicate
"agent A is a `planner` running `gemma-4` for owner O" is supported by exactly
one signature: the orchestrator's. The orchestrator is therefore a single point
of identity forgery. SwarmKit §10.1 accepts this ("Compromised orchestrator:
Catastrophic by design") because at the time of that draft there was no separate
identity authority to appeal to.

AIA supplies that authority. After AIA, the same predicate is supported by the
Attestor's signature over an AID, plus model attestation evidence, plus
(optionally) an owner signature — none of which the orchestrator can produce.
The orchestrator's compromise remains serious but is no longer *identity*
forgery; it is *authorization* misuse, bounded by the orchestrator's own
clearance attributes.

### 10.3 OWASP Top 10 for Agentic Applications Mapping

The OWASP Top 10 for Agentic Applications (2026) [OWASP-AGENTIC] enumerates the
principal agentic risk classes. AIA is a direct control for several:

| OWASP class | AIA contribution |
|---|---|
| **ASI03 — Identity & Privilege Abuse** | Primary target. AIA makes agent identity an independently attested, PoP-bound, owner-linked fact rather than an orchestrator assertion. |
| **ASI10 — Rogue Agents** | An agent with no valid AID, or an AID failing model attestation, is detectable as rogue at every verifier (KAS, MCP broker, A2A peer). |
| **ASI04 — Agentic Supply Chain** | The weight digest (§5.2) pins the exact model artifact; a substituted or tampered model fails §5.7 V-M1. |
| **ASI07 — Insecure Inter-Agent Communication** | A2A peers authenticate each other by AID + PoP, not by transport trust alone. |
| **ASI09 — Human-Agent Trust Exploitation** | The `owner` binding records *whose* authority an agent acts under; `binding: self` makes a missing human principal explicit. |

AIA does **not** address goal hijack (ASI01), tool misuse (ASI02), code
execution (ASI05), memory poisoning (ASI06), or cascading failure (ASI08): those
concern what an agent *does*, which AIA explicitly leaves to the Planner, TØR-G,
and ARP (§1.2, §3.2 N1–N3).

### 10.4 Attestation Freshness

`evidence.nonce` (§5.4) MUST be unique per issuance and SHOULD be supplied by
the verifier or the Attestor at issuance time, not by the subject. A verifier
MUST reject evidence whose nonce it did not expect for this issuance (§5.7
V-M2). Combined with short `aid.expires`, this bounds the window in which stale
attestation is accepted.

### 10.5 Owner Compromise

A compromised owner key lets an attacker authorize agents in that owner's name.
This is outside AIA's mitigation surface — AIA attests *that* an owner binding
exists, not that the owner is uncompromised. Deployments SHOULD apply the same
key-hygiene controls to owner keys as to Attestor keys (§3.4).

---

## 11. Conformance

A conforming **Identity Attestor** MUST:

- C-A1. Issue exactly one AID per (flight, role), per §6, content-addressed and
  signed per §4.12.
- C-A2. Verify Model Attestation evidence against the expected model before
  issuance (§5.7 V-M1).
- C-A3. Sign AIDs with a key chaining to a trust anchor independent of the
  orchestrator (§3.4).
- C-A4. Never issue an AID carrying authorization grants (§3.2 N5).
- C-A5. Re-bind, never copy, identity across delegation hops (§6.4).
- C-A6. Support AID rotation and revocation (§9).

A conforming **specialist** (AID holder) MUST:

- C-S1. Prove possession of its `key.cnf` private key when presenting its AID.
- C-S2. Refuse a delegation envelope that lacks `aid_ref` in a flight that
  provisioned an Attestor (§6.3).
- C-S3. Verify both its AID and its delegation envelope before acting (§6.1
  step 5).

A conforming **verifier** MUST:

- C-V1. Recompute `aid.id` and reject on mismatch (§4.12).
- C-V2. Verify the Attestor signature and confirm the trust anchor is trusted
  and orchestrator-independent (§4.11, §3.4).
- C-V3. Require a successful proof-of-possession challenge; never accept an AID
  as a bearer token (§4.6).
- C-V4. Reject expired (§9.4) and, when a revocation list is reachable, revoked
  (§9.3) AIDs.
- C-V5. Reject an AID whose `flight_binding` does not match the flight of
  presentation, and whose `subject.did` / `subject.spiffe_id` are inconsistent
  (§4.5, §7.3).

A conforming **Model Attestor** MUST:

- C-M1. Compute the weight digest per §5.2.1.
- C-M2. Emit a §5.4 evidence block, or §5.5 SPIRE selectors, or both.
- C-M3. Mark software-only measurements `rats-software` and hardware-rooted
  measurements `rats-hardware`; never label software evidence as hardware.

---

## 12. Examples

The example values below (digests, nonces, signatures) are realistic byte
lengths but are not derived from content and are not verifiable.

### 12.1 Full AID

```json
{
  "aia_spec": "0.1.0",
  "aid": {
    "id": "blake3:7Yk2pQ9wRtL4mNvX1aBc3dEfGhJkLmNpQrStUvWxYz0",
    "issued": "2026-05-22T14:00:00Z",
    "expires": "2026-05-22T18:00:00Z",
    "nonce": "k3Jd9sLpQ2mWxYz8aBcDeg"
  },
  "subject": {
    "did": "did:web:specialist.arkavo.net:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",
    "spiffe_id": "spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479/planner-1",
    "role_type": "planner",
    "display_name": "Planner specialist (gemma-4 26B-MoE)"
  },
  "owner": {
    "did": "did:web:arkavo.net",
    "owner_type": "organization",
    "binding": "delegated"
  },
  "flight_binding": {
    "flight_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "kit_id": "blake3:3kW9D3RPLdzL6UYJgCsjxn5gV-AFQxPd4w88aiEmBck",
    "role_id": "planner-1"
  },
  "key": {
    "cnf": {
      "jwk": {
        "kty": "OKP",
        "crv": "Ed25519",
        "x": "Lm9pQrStUvWxYz0aBcDeFgHjKlMnPqRsTuVwXy12345",
        "alg": "EdDSA"
      }
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
  "delegation": {
    "chain": [
      { "actor": "did:web:arkavo.net", "kind": "owner" }
    ],
    "act": "did:web:specialist.arkavo.net:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",
    "may_act": {
      "audience": ["spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479"],
      "max_depth": 4,
      "scope_reduction_only": true
    }
  },
  "assertions": [],
  "signature": {
    "attestor_did": "did:web:aia.arkavo.net",
    "trust_anchor": "did:web:arkavo.net",
    "algorithm": "ed25519",
    "value": "MEUCIQDxAmpLe5ig6n4tUreSi9gN4tUreVa1ueF0rD0cuMent4t10nPurP0seZ87"
  }
}
```

### 12.2 Sharded Model Attestation

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

### 12.3 SPIRE Registration Entry (illustrative)

```
spire-server entry create \
  -spiffeID  spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479/planner-1 \
  -parentID  spiffe://arkavo.net/aia/attestor \
  -selector  arkavo:model-family:gemma-4 \
  -selector  arkavo:weights:blake3:9f8e7d6c5b4a3210FeDcBa9876543210AbCdEfGh \
  -selector  arkavo:runtime:llama.cpp \
  -selector  arkavo:tee:sev-snp \
  -ttl       14400
```

SPIRE issues the SVID for `planner-1` only to a workload the
`arkavo-model-attestor` plugin attests with all four selectors.

### 12.4 CAWG Agent Identity Assertion

```json
{
  "label": "cawg.agent_identity",
  "data": {
    "aia_spec": "0.1.0",
    "aid_id": "blake3:7Yk2pQ9wRtL4mNvX1aBc3dEfGhJkLmNpQrStUvWxYz0",
    "subject_did": "did:web:specialist.arkavo.net:flight:f47ac10b-58cc-4372-a567-0e02b2c3d479:planner-1",
    "subject_spiffe_id": "spiffe://arkavo.net/flight/f47ac10b-58cc-4372-a567-0e02b2c3d479/planner-1",
    "role_type": "planner",
    "owner_did": "did:web:arkavo.net",
    "owner_binding": "delegated",
    "model": { "family": "gemma-4", "size": "26B-MoE", "quantization": "Q4_K_M" },
    "weights_digest": "blake3:9f8e7d6c5b4a3210FeDcBa9876543210AbCdEfGh",
    "attestation_format": "rats-hardware",
    "flight_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "attestor_did": "did:web:aia.arkavo.net",
    "aid_signature": "MEUCIQDxAmpLe5ig6n4tUreSi9gN4tUreVa1ueF0rD0cuMent4t10nPurP0seZ87"
  }
}
```

---

## 13. Open Questions

1. **Attestor redundancy.** Should a flight support more than one Attestor (an
   AID counter-signed by two anchors) for higher assurance, and if so how do
   verifiers express an `n-of-m` policy?
2. **SPIRE node attestation reuse.** Can the AIA trust anchor (§3.4) be the
   SPIRE node attestation chain directly, collapsing two trust hierarchies?
3. **Weight digest of mutated weights.** LoRA adapters and runtime quantization
   change loaded weights versus the published artifact. Should §5.2 define a
   base-plus-adapter digest pair rather than a single digest?
4. **AID transport.** Should AIA define an A2A JSON-RPC method for AID
   presentation and PoP challenge, or leave it to the WIT projection (§7.1)?

---

## 14. References

### 14.1 Normative

- **SwarmKit** — `swarmkit/swarmkit-spec-draft-01.md` (this repo). AIA adds the
  Identity Attestor role to the SwarmKit mesh.
- **BCP 14** — RFC 2119, RFC 8174 (requirements language).
- **JSON** — RFC 8259.
- **JSON Canonicalization Scheme (JCS)** — RFC 8785.
- **base64url** — RFC 4648 §5.
- **URI** — RFC 3986.
- **Proof-of-Possession Key Semantics for JWTs** — RFC 7800 (the `cnf` claim).
- **OAuth 2.0 Token Exchange** — RFC 8693 (`act` / `may_act` delegation
  semantics; AIA adopts the delegation pattern, not token exchange).
- **RATS Architecture** — RFC 9334 (Attester / Verifier / Attestation Result
  roles).
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
- **SPIFFE** — Secure Production Identity Framework for Everyone;
  SPIFFE ID, X.509-SVID, JWT-SVID — https://spiffe.io.
- **C2PA** — Coalition for Content Provenance and Authenticity, technical
  specification.
- **CAWG** — Creator Assertions Working Group; identity assertion v1.x.
- **OWASP Top 10 for Agentic Applications (2026)** — [OWASP-AGENTIC] OWASP GenAI
  Security Project, published 2025-12-09.
- **Entity Attestation Token (EAT)** — IETF RATS Working Group; an alternative
  evidence encoding for §5.4.
- **DPoP** — RFC 9449 (a proof-of-possession mechanism usable for §4.6
  challenges over HTTP).

### 14.3 Forthcoming and Deferred

- **Task Contract Protocol (TCP)** — a future Arkavo specification (referenced
  by SwarmKit §1.2 and §14.2 as forthcoming). Profiling TCP as an RFC 8693
  OAuth 2.0 Token Exchange profile is **deferred**: it is blocked on the TCP
  specification being published. AIA §7.5 deliberately adopts only RFC 8693
  delegation *semantics* (`act` / `may_act`), not token exchange, so that the
  later TCP profile can be layered without revising AIA.
- **Sequence Integrity Specification** — a future Arkavo specification defining
  lineage events; referenced by §9.3 for revocation logging.

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

## Appendix B: AID → WIT Field Mapping

See §7.1. The projection is lossy in one direction only: a WIT minted from an
AID drops the AIA-specific blocks unless they are carried in the `arkavo_aid`
private claim. An AID cannot be reconstructed from a bare WIT; the `aid.id`
serves as the resolution handle.

## Appendix C: The Six-Role Mesh

| Role | `role_type` | Answers | Authority |
|---|---|---|---|
| Scribe | `scribe` | what happened | — |
| Historian | `historian` | what is the context | — |
| Planner | `planner` | what work, in what order | assigns |
| Critic | `critic` | how good is the output | scores |
| Operator | `operator` | execute the action | acts |
| **Identity Attestor** | **`identity_attestor`** | **who is this agent** | **attests** |

The Identity Attestor is the only role added by this specification. The
orchestrator is a delegation responsibility layered onto a role (SwarmKit §2),
not a seventh archetype.
