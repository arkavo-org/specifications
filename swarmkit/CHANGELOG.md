# SwarmKit Specification Changelog

## draft-02 (2026-10-16)

Additive draft. Bumps `spec_version` from `1.1.0` to `1.2.0` (per §9.4: adding optional fields is MINOR). Schemas move to `schemas/swarmkit/draft-02/`.

Changes:

- §7.2 — The delegation envelope names its `orchestrator` and `specialist` (both `did:key`), and carries `issued`, `expires` and a 128-bit `nonce`. `delegation_signature` is defined: Ed25519 over the BLAKE3 digest of the JCS form, the §8.1.2 skill-signing profile. The envelope is sent as the parameter of `swarmkit/delegate`. It also carries `host_requirements`, `context.objective`, `context.role_description` and an optional `pairing_proof`. Previously the signature had no algorithm or key, and an envelope named no recipient and never expired, so a captured one could be replayed to any specialist.
- §7.3 — Specialist acceptance is an ordered procedure that checks signature, recipient, window and nonce before the binding, skills and host policy.
- §7.3.1 (new) — Orchestrator binding. A specialist accepts delegations from one bound orchestrator key, bound by `operator` (RECOMMENDED), `pairing` (one-time secret, HMAC proof) or `first-use` (zero configuration, trusted networks only). Previously nothing said which orchestrator a specialist should trust.
- §7.3.2–§7.3.4 (new) — The ready acknowledgement (`envelope_blake3`), refusal reasons under JSON-RPC error `-32040`, and `swarmkit/status`.
- §7.1.2 (new) — Enlisting a running specialist: discovery over DNS-SD `_a2a._tcp`, offers matched on host capabilities, refusal moving the role to the next candidate. Previously §7.1 assumed the orchestrator provisions every specialist.
- §7.4 — Delegations persist across specialist restarts until `expires`. `swarmkit/revoke` is a signed revocation that can also release the binding.
- §4.3.1 (new) — `RoleSpec.host_requirements`, with the capability `image_generation`. A specialist serves a listed capability to the swarm only while its role requires it.
- §8.2 — Grants SHOULD carry no `bearer_token` when every MCP server is local or public; such envelopes may travel signed and unencrypted.
- §10.1 — New threats: unauthorized delegation, delegation replay, first-use capture, discovery spoofing.
- §11 — C-S1 strengthened; new C-O7, C-O8, C-S5, C-S6, C-S7.
- §13 — New open questions: one key per host, capability vocabulary, binding to an owner, envelope confidentiality.

## draft-01 (2026-05-08)

Additive draft. Bumps `spec_version` from `1.0.0` to `1.1.0` (per §9.4: adding optional fields is MINOR).

Changes:

- §4.6 — Promoted "weights sum to 1.0" from descriptive prose to MUST clause; validators MUST reject violating manifests.
- §9.1 — Promoted "kit.id is BLAKE3(canonical_manifest)" from descriptive prose to MUST clause; validators MUST reject mismatching declared kit.id.
- §1.2 — New normative paragraph: SwarmFlight runtimes MUST instantiate exactly one ARP document per role; per-role ARP state MUST be isolated.
- §8.1.1 (new) — SkillContent JSON schema (`name`, `description`, `instructions`, `resources` fields) promoted from descriptive prose to normative schema.
- §8.1.2 (new) — Skill signing protocol: Ed25519 over BLAKE3 of JCS-canonical SkillContent bytes, base64url without padding.
- §8.1.3 (new) — Registry cache layout: `<blake3-hex>.skill.json` files, with reserved `.sig.json` sidecar.
- §10.1 — New threat model entry: "Privilege escalation by sibling role" — producers MAY differentiate per-role TDF access via custom attributes (e.g., `audit_authority/true`).
- Appendix C — Expanded with C.1 (Domain-Specific Examples) citing the four shipped reference kits' role_type values.

## draft-00 (2026-04 baseline)

Initial published draft. Defines the manifest schema (§4), agent_provisioning block (§5), TDF encryption envelope (§6), orchestrator decryption + delegation flow (§7), skills + MCP tool distribution model (§8), versioning + identity (§9), security considerations (§10), and conformance criteria (§11).
