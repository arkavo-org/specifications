# Agent Identity Authority (AIA) Changelog

## draft-00 (2026-05-22)

Initial published draft.

`draft-arkavo-aia-00.md` — the AIA specification:

- §3 — Defines the **Agent Identity Authority**: a dedicated role whose only
  purpose is to issue and attest agent identity. It sits in an identity tier
  **upstream** of the orchestrator (Root Identity Agent → Swarm Identity Agent →
  Orchestrator → Workers); the orchestrator itself holds an AIA-issued identity.
  Responsibilities are fixed normatively — it creates identities, issues
  credentials, rotates keys, attests runtime, publishes the trust chain, and
  revokes; it does not assign work, grant resource access, make ABAC decisions,
  delegate authority, or release TDF keys.
- §3.5 — The OpenTDF attribute boundary: identity-originated `agent.identity.*`
  attributes (type, trust_level, runtime, owner, organization) versus
  orchestrator-issued capability attributes (tool_access, code_authority,
  resource_scope, execution_mode).
- §4 — The Agent Identity Document (AID): a signed, content-addressed identity
  record. Identifies a durable agent by default; `flight_binding` is optional,
  for ephemeral flight-scoped credentials only. An AID carries no authorization.
- §5 — The Model Attestation Profile: BLAKE3 weight digests, RATS-aligned
  evidence, and a SPIFFE/SPIRE-compatible `arkavo` workload selector vocabulary.
- §6 — Issuance and lifecycle: identity is issued when an agent is created,
  before and independently of any work assignment; per-hop re-attestation for
  delegation chains; key rotation.
- §7 — The compact Agent Identity Credential (AID projected to a WIMSE WIT);
  WIC interoperability; `did:web` ⇄ `spiffe://`.
- §8 — The `cawg.agent_identity` C2PA assertion type for content provenance.
- §10 — Security considerations, including the orchestrator super-agent
  foot-gun and a mapping to the OWASP Top 10 for Agentic Applications (2026).

`wimse-swarmkit-alignment-draft-00.md` — a non-normative review document mapping
SwarmKit draft-01 against `draft-ietf-wimse-arch-07` and
`draft-ietf-wimse-workload-creds-00`. Records one high-severity divergence (the
MCP grant bearer token) and four lower-severity items, with recommendations for
a SwarmKit draft-02.

JSON Schemas (`schemas/agent-identity-authority/draft-00/`) for the AID, model
attestation, and CAWG assertion. Spec examples validate; negative cases reject.

### Deferred

- Profiling the forthcoming Task Contract Protocol (TCP) as an RFC 8693 OAuth
  2.0 Token Exchange profile is deferred; it is blocked on the TCP specification
  being published. AIA §7.5 adopts only RFC 8693 delegation semantics so the
  later profile can be layered without revising AIA. See AIA §14.3.
