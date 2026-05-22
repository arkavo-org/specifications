# WIMSE Alignment Review: SwarmKit AE-2026-004

|                  |                                                                  |
|------------------|------------------------------------------------------------------|
| **Version**      | 0.1.0-draft                                                      |
| **Status**       | Community Draft (review document, non-normative)                 |
| **Authors**      | Arkavo Project Contributors                                      |
| **License**      | Apache 2.0                                                       |
| **Reviews**      | SwarmKit `swarmkit/swarmkit-spec-draft-01.md` (spec_version 1.1.0)|
| **Against**      | `draft-ietf-wimse-arch-07`, `draft-ietf-wimse-workload-creds-00`  |
| **Companion to** | Agent Identity Attestation (AIA) `draft-arkavo-aia-00`           |

---

## Abstract

This is a **review document**, not a specification. It maps the SwarmKit
multi-agent work-packaging format against the IETF WIMSE (Workload Identity in
Multi-System Environments) drafts and records where the two diverge. Its purpose
is to catch token-format and delegation-model divergence **before** WIMSE
advances toward Proposed Standard and its milestones close — at which point
divergence becomes expensive to correct.

The review finds SwarmKit structurally compatible with WIMSE, with one
high-severity divergence (the MCP grant bearer token, §4.1) and four
lower-severity items. Recommendations for a SwarmKit draft-02 are collected in
§5. Nothing in this document is normative; it is input to a future SwarmKit
revision and to the AIA specification.

---

## 1. Scope and Method

### 1.1 What is reviewed

SwarmKit draft-01 §6 (TDF encryption), §7 (orchestrator decryption and
delegation, including the §7.2 delegation envelope), §8 (skills and MCP tool
distribution), and §9 (versioning and identity) — i.e. every SwarmKit construct
that carries, asserts, or transports identity or credentials.

The companion **Agent Runtime Policy (ARP)** is touched only where SwarmKit's
`agent_provisioning` hands off to it; ARP's own integrity model is out of scope.

### 1.2 What it is reviewed against

- **[WIMSE-ARCH]** `draft-ietf-wimse-arch-07` — the architecture, in particular
  §3.3.1 (attestation), §3.3.6 (delegation), and §3.3.9 (AI/ML-based
  intermediaries).
- **[WIMSE-CREDS]** `draft-ietf-wimse-workload-creds-00` — the Workload Identity
  Token (WIT) and Workload Identity Certificate (WIC).

### 1.3 Method

Each SwarmKit identity/credential construct is mapped to its nearest WIMSE
equivalent (§3). Where the mapping is exact, the construct is recorded as
aligned. Where it is not, a **divergence** is recorded (§4) with a severity:

- **High** — silent interoperability failure or a security property WIMSE
  guarantees that SwarmKit does not.
- **Medium** — divergence that forces a translation shim but breaks nothing.
- **Low** — cosmetic or vocabulary divergence.

### 1.4 Severity summary

| ID | Divergence | Severity |
|----|------------|----------|
| D1 | MCP grant `bearer_token` is a true bearer token | **High** |
| D2 | Delegation chain not re-bound per hop | Medium |
| D3 | No owner / principal concept | Medium |
| D4 | Provisioned model is asserted, not attested | Medium |
| D5 | Identity is `did:web` only; no SPIFFE ID | Low |

---

## 2. WIMSE Primitives in Brief

For readers not steeped in the WIMSE drafts:

- **WIT — Workload Identity Token** ([WIMSE-CREDS] §3). A JWT, `typ: wit+jwt`,
  with required claims `sub` (workload identity URI), `exp`, and `cnf` (a `jwk`
  binding a public key). A WIT **MUST NOT be used as a bearer token**: the
  presenter MUST prove possession of the `cnf` private key. Targeted at
  application-layer protocols. Short-lived ("on the order of hours").
- **WIC — Workload Identity Certificate** ([WIMSE-CREDS] §4). An X.509
  certificate with the workload identity in a single URI `SubjectAltName`.
  Fully compatible with the SPIFFE X.509-SVID. Targeted at the transport layer.
- **Delegation** ([WIMSE-ARCH] §3.3.6). Built on RFC 8693 OAuth 2.0 Token
  Exchange semantics; delegation and impersonation tokens carry a security
  context downstream services evaluate.
- **AI/ML-based intermediaries** ([WIMSE-ARCH] §3.3.9). AI agents are a special
  case of delegated workloads. §3.3.9.2: a downstream call SHOULD propagate the
  upstream security context. §3.3.9.3: autonomous actions MUST be distinguished
  from delegated ones. §3.3.9.4: each hop of an AI-to-AI chain MUST explicitly
  scope and re-bind the security context so downstream services can evaluate
  provenance.
- **Attestation** ([WIMSE-ARCH] §3.3.1). Validates workload identity during
  credential provisioning; aligns with the RFC 9334 RATS architecture.

---

## 3. Construct Mapping

| SwarmKit construct | Nearest WIMSE equivalent | Status |
|---|---|---|
| `kit.authors[].did` (`did:web`) | A workload/issuer identity URI | Aligned (§3.1) |
| TDF envelope + KAS attribute release (§6) | WIC at the transport layer / X.509-SVID | Aligned in spirit (§3.2) |
| Orchestrator decrypt + delegate (§7.1) | A WIMSE AI intermediary, [WIMSE-ARCH] §3.3.9 | Aligned (§3.3) |
| Delegation envelope `delegation_signature` (§7.2) | A delegation token, [WIMSE-ARCH] §3.3.6 | Diverges — D2 |
| MCP grant `bearer_token` (§7.2, §8.2) | A WIT presented to a resource | **Diverges — D1** |
| Skill signature (Ed25519 over BLAKE3, §8.1.2) | — (no WIMSE analogue; software supply chain) | Aligned, out of WIMSE scope |
| `agent_provisioning.model` (§5) | RATS evidence about the workload, §3.3.1 | Diverges — D4 |
| Per-role `tdf_attribute_release_policy` (§4.3) | Authorization policy, not identity | Aligned (correctly separated) |

### 3.1 Identity URIs

SwarmKit identifies authors and the orchestrator with `did:web` URIs. WIMSE
identity URIs are scheme-agnostic but, where SPIFFE is the substrate,
[WIMSE-CREDS] §3.1 mandates `spiffe://` URIs. SwarmKit and WIMSE agree that
identity is a URI; they disagree only on scheme (D5).

### 3.2 TDF Envelope as Transport-Layer Identity

The SwarmKit TDF envelope plus KAS attribute-release gate (§6.3) performs, at
the data layer, what a WIC performs at the transport layer: it ensures only a
holder of the right attributes obtains the key. This is a sound analogue. It is
not a defect that SwarmKit does not literally use X.509-SVIDs — the TDF model is
deliberately data-centric — but a WIMSE-interop deployment will want a WIC
alongside the TDF envelope, and AIA §7.2 supplies the mapping.

### 3.3 Orchestrator as AI Intermediary

The SwarmKit orchestrator is precisely the "AI/ML-based intermediary" of
[WIMSE-ARCH] §3.3.9: it receives work, holds an upstream security context (its
KAS clearance attributes), and fans work out to downstream specialists. WIMSE's
§3.3.9 requirements therefore apply to it directly — which is what makes D2 a
real divergence rather than a theoretical one.

---

## 4. Divergences

### 4.1 D1 — MCP grant `bearer_token` is a true bearer token (High)

**Finding.** SwarmKit §7.2 / §8.2 issues MCP tool grants carrying a
`bearer_token` — described as "a bearer token bound to the flight ID with
explicit expiry." The delegation-envelope schema models it as an opaque value.
A specialist presents it to an MCP server (directly, for `auth: passthrough`, or
via the orchestrator's broker, for `auth: delegated`) and the server grants
access on possession alone.

[WIMSE-CREDS] §3.1 states the opposite property for the WIT: it **MUST NOT be
used as a bearer token**; the presenter MUST prove possession of the bound
private key. WIMSE adopts this specifically because bearer tokens are stealable
and replayable — which SwarmKit's own threat model (§10.1) recognizes for
*other* artifacts but not for the MCP grant.

**Impact.** Two problems. (1) A SwarmKit MCP grant cannot be carried as a WIT
without changing its security model — they are not the same kind of token, so a
WIMSE-native MCP server cannot consume a SwarmKit grant, and vice versa. (2)
Independent of WIMSE: an exfiltrated `bearer_token` is usable by any process
until `expires`. SwarmKit §8.2 ("specialists MUST NOT cache MCP tokens beyond
`expires`"; "orchestrators SHOULD rotate tokens") mitigates the *window* but not
the *bearer* nature.

**Why now.** The MCP ecosystem and WIMSE are both converging on
proof-of-possession for workload-to-resource calls. If SwarmKit ships a bearer
MCP grant into a v1.0, every MCP server integration encodes the bearer
assumption and the later correction is a breaking change across integrations.

### 4.2 D2 — Delegation chain is signed but not re-bound per hop (Medium)

**Finding.** The SwarmKit delegation envelope carries a single
`delegation_signature` by the orchestrator (§7.2). For a single
orchestrator→specialist hop this is adequate. But SwarmKit §13 anticipates
hierarchical SwarmKits ("a role launches a sub-flight"), and §7 already permits
A2A traffic between specialists. There is no construct that re-scopes the
security context at each hop.

[WIMSE-ARCH] §3.3.9.4 requires that each hop of an AI-to-AI chain "explicitly
scope and re-bind the security context so that downstream services can reliably
evaluate provenance," and §3.3.9.3 requires autonomous actions be distinguished
from delegated ones. A single orchestrator signature satisfies neither once a
chain exceeds length one.

**Impact.** Medium today (hierarchical kits are deferred, SwarmKit §13.3), but
it becomes High the moment sub-flights ship. AIA §4.9 / §6.4 already define the
re-binding model (`delegation.chain`, `may_act`, `scope_reduction_only`); the
recommendation (§5) is for SwarmKit to defer to AIA rather than grow its own.

### 4.3 D3 — No owner / principal concept (Medium)

**Finding.** A SwarmKit role has a `role_type`, a model, skills, and TDF
attributes. It has no field recording the principal on whose behalf it acts.
[WIMSE-ARCH] §3.3.9 and [WIMSE-AI] make the *owner* a first-class concept
precisely so that delegated authority can be traced to a human, device, or
policy engine.

**Impact.** Without an owner, "this agent acted" cannot be resolved to "this
principal authorized it." This is the §3.3.9.3 autonomous-vs-delegated
distinction. AIA `owner` (§4.4) supplies it; SwarmKit need not — but the
SwarmKit↔AIA seam must be explicit so a flight using SwarmKit alone is
recognizably owner-less.

### 4.4 D4 — Provisioned model is asserted, not attested (Medium)

**Finding.** `agent_provisioning.model` (SwarmKit §5) declares a model family,
size, quantization, and backend. SwarmKit §5.1 has the orchestrator *validate*
that `model.family` is in its supported set — but nothing *attests* that the
running process actually loaded those weights. The claim is self-asserted by the
manifest.

[WIMSE-ARCH] §3.3.1 expects identity to be attested during provisioning, RATS-
style. A model claim with no evidence is exactly the gap AIA §5 (Model
Attestation Profile) fills.

**Impact.** A specialist could be provisioned against a manifest claiming a
safety-tuned model and in fact run a different one, with no artifact a verifier
can check. Medium because the orchestrator is trusted in the SwarmKit model;
High once a verifier outside the orchestrator's trust domain (a WIMSE-native
peer, a KAS in another domain) must rely on the model claim.

### 4.5 D5 — Identity scheme is `did:web` only (Low)

**Finding.** SwarmKit identities are `did:web`. SPIRE/WIMSE infrastructure keys
on `spiffe://` URIs. Neither is wrong; they are different schemes for the same
abstraction.

**Impact.** Low — it is a deterministic translation, defined in AIA §7.3
(`did:web:...:flight:<id>:<role>` ⇄ `spiffe://.../flight/<id>/<role>`). Recorded
only so a SwarmKit draft-02 can cite the mapping rather than leave integrators
to invent their own.

---

## 5. Recommendations for SwarmKit draft-02

These are recommendations to a future SwarmKit revision. They are not adopted
here and do not modify draft-01.

- **R1 (addresses D1).** Re-specify the MCP grant so the `bearer_token` is
  replaced, or wrapped, by a proof-of-possession credential: bind the grant to
  the specialist's public key (the AID `key.cnf`, AIA §4.6) and require the MCP
  server or broker to run a PoP challenge. Retain `bearer_token` only as a
  clearly-labelled legacy mode for MCP servers that cannot do PoP, and mark it
  NOT RECOMMENDED. This is the one change worth making before a v1.0.
- **R2 (addresses D2).** State that hierarchical SwarmKits and cross-trust-domain
  A2A hops MUST re-bind identity per hop, and reference AIA §6.4 for the
  mechanism, rather than extending `delegation_signature` into a chain format.
- **R3 (addresses D3, D4).** Add an OPTIONAL `roles[].identity` member that
  references an AID by `aid.id`, and add the OPTIONAL `aid_ref` member to the
  delegation envelope (AIA §6.3). A SwarmKit that omits both is, by construction,
  owner-less and unattested — which should be a visible property, not a silent
  default.
- **R4 (addresses D5).** Add a non-normative appendix citing the AIA §7.3
  `did:web` ⇄ `spiffe://` mapping.
- **R5 (process).** Track the WIMSE drafts: `draft-ietf-wimse-arch` and
  `draft-ietf-wimse-workload-creds`. Re-run this review when either reaches
  WG Last Call, because token-format details (the WIT `cnf`/PoP rules in
  particular) are still mutable until then and R1's target should match the
  final WIT shape.

None of R1–R5 requires SwarmKit to adopt WIMSE wholesale. They keep SwarmKit's
TDF-centric data model intact and only correct the credential-layer constructs
that would otherwise diverge silently.

---

## 6. OWASP Cross-Reference

Cross-referenced lightly against the OWASP Top 10 for Agentic Applications
(2026) [OWASP-AGENTIC], to show the divergences are not merely standards
hygiene:

- **D1** (bearer MCP grant) is an instance of **ASI03 — Identity & Privilege
  Abuse** and **ASI07 — Insecure Inter-Agent Communication**: a stealable
  credential is the textbook privilege-abuse primitive.
- **D2** (un-rebound chain) maps to **ASI03** and **ASI08 — Cascading Failures**:
  authority that widens silently down a chain is how a local compromise
  cascades.
- **D4** (unattested model) maps to **ASI04 — Agentic Supply Chain
  Vulnerabilities**: a substituted model is a supply-chain substitution.

A fuller cross-map of OWASP classes against deployment scenarios is more useful
once the forthcoming Sequence Integrity Specification publishes its scenario
catalogue; this section is deliberately limited to the three divergences with a
direct OWASP correspondence.

---

## 7. References

- **SwarmKit** — `swarmkit/swarmkit-spec-draft-01.md` (this repo), the
  document under review.
- **AIA** — `agent-identity-attestation/draft-arkavo-aia-00.md` (this repo).
- **Agent Runtime Policy (ARP)** — `agent-runtime-policy/arp-spec-draft-00.md`
  (this repo).
- **[WIMSE-ARCH]** — `draft-ietf-wimse-arch-07`, Workload Identity in a Multi
  System Environment (WIMSE) Architecture.
- **[WIMSE-CREDS]** — `draft-ietf-wimse-workload-creds-00`, WIMSE Workload
  Credentials.
- **[WIMSE-AI]** — `draft-ni-wimse-ai-agent-identity-02`, WIMSE Applicability
  for AI Agents (informative; the owner concept).
- **RFC 8693** — OAuth 2.0 Token Exchange.
- **RFC 9334** — Remote ATtestation procedureS (RATS) Architecture.
- **[OWASP-AGENTIC]** — OWASP Top 10 for Agentic Applications (2026), OWASP
  GenAI Security Project, 2025-12-09.
