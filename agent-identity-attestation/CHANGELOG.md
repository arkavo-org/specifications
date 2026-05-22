# Agent Identity Attestation (AIA) Changelog

## draft-00 (2026-05-22)

Initial published draft.

`draft-arkavo-aia-00.md` — the AIA specification:

- §3 — Defines the Identity Attestor, a sixth SwarmKit mesh role
  (`role_type: identity_attestor`), with explicit non-responsibilities that keep
  it disjoint from the Planner (assigns work), TØR-G (gates execution), the
  Critic (scores output), and the orchestrator (delegates).
- §4 — The Agent Identity Document (AID): a signed, content-addressed identity
  record; subject, owner, flight binding, proof-of-possession key, delegation
  chain. An AID carries no authorization.
- §5 — The Model Attestation Profile: BLAKE3 weight digests, RATS-aligned
  evidence, and a SPIFFE/SPIRE-compatible `arkavo` workload selector vocabulary.
- §6 — The identity issuance flow, inserted before orchestrator delegation;
  per-hop re-binding for delegation chains.
- §7 — WIMSE interoperability: AID ↔ WIT/WIC mappings, `did:web` ⇄ `spiffe://`.
- §8 — The `cawg.agent_identity` C2PA assertion type for content provenance.
- §10 — Security considerations, including the orchestrator-minting foot-gun and
  a mapping to the OWASP Top 10 for Agentic Applications (2026).

`wimse-swarmkit-alignment-draft-00.md` — a non-normative review document mapping
SwarmKit draft-01 against `draft-ietf-wimse-arch-07` and
`draft-ietf-wimse-workload-creds-00`. Records one high-severity divergence (the
MCP grant bearer token) and four lower-severity items, with recommendations for
a SwarmKit draft-02.

### Deferred

- Profiling the forthcoming Task Contract Protocol (TCP) as an RFC 8693 OAuth
  2.0 Token Exchange profile is deferred; it is blocked on the TCP specification
  being published. AIA §7.5 adopts only RFC 8693 delegation semantics so the
  later profile can be layered without revising AIA. See AIA §14.3.
