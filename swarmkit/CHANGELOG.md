# SwarmKit Specification Changelog

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
