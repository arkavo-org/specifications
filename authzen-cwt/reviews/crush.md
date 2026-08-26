## Reviewer
crush

## Summary
Needs revision. The document is unusually thorough and almost all of its code claims check out against the tree (CWT minting, union-verifier deltas, `build_chain` NPE claims, the nested `GetDecisionResponse` shape, COAZ-MCP default mappings, AuthZEN discovery/evaluations/status-code semantics). One **major** correctness gap: the spec's headline multi-device rule ("each `device.sub` MUST equal `subject.id`") is contradicted by how `authnz-rs` actually mints DeviceCheck tokens (bare UUID `sub`) versus PE/OIDC access tokens (`arkavo:`-prefixed `sub`). Everything else is minor — a misquoted AuthZEN §7.1 citation, "registered" subject-type wording, and a few unlabeled profile extensions.

## Issues

### Issue 1: Device `sub` cannot equal PE `sub` for tokens minted today
- **Severity**: major
- **Section**: CWT Subject Profile → "PE + device NPE + environment NPE → one SARC"; `context.device` schema; `$token` map; wire examples
- **Description**: The spec makes the device↔PE subject equality a hard normative rule with examples that use the same prefixed `sub` on both sides:
  - "Each `sub` MUST equal `subject.id`" (`context.device` table, line 443).
  - "Every device `sub` MUST equal `subject.id`" (facade chain rules, lines 536, 547).
  - Both examples show `"sub": "arkavo:550e8400-e29b-41d4-a716-446655440000"` for the device (lines 434, 474–484, 762–766).
  - The `$token`/subject table (line 352) says `sub` is `arkavo:{UUID}`, `apple:{APPLE_SUB}`, or `client:{CLIENT_ID}`.

  But the issuer mints these token types with *different* `sub` formats:
  - DeviceCheck assertion token: `sub = user_id.to_string()` (bare UUID) — `authnz-rs/src/device_check.rs:586`.
  - WebAuthn auth/registration tokens: `sub = user_id.to_string()` (bare UUID) — `authnz-rs/src/authn.rs:434`, `:449`.
  - OIDC access token (the PE CWT in the catalog flow): `sub = format!("arkavo:{}", account_id)` — `authnz-rs/src/oidc.rs:1583`.

  So for a real phone+watch catalog request, `PE.sub = "arkavo:550e8400-…"` while `device.sub = "550e8400-…"`. The equality the spec mandates never holds, and the existing `build_chain` sub-equality check (`tdf-iroh-s3/crates/tdf-core/src/catalog_api.rs:309`, `claims.sub != pe.sub → 401`) would reject every legitimate device. The spec's `$token.sub` format enumeration also silently omits the bare-UUID format actually used by DeviceCheck/WebAuthn tokens.
- **Suggestion**: Either (a) state that DeviceCheck/WebAuthn tokens use a bare-UUID `sub` and define a normalization rule (e.g. device `sub` MUST equal `subject.id` after stripping the `arkavo:`/`apple:`/`client:` prefix, or vice versa), or (b) fix the issuer to mint DeviceCheck tokens with the same prefixed `sub` as the PE and update the examples accordingly. Whichever is chosen, make the `$token.sub` table list the bare-UUID format explicitly and add a worked example showing the actual minted `sub` strings.

### Issue 2: Empty-evaluations rule misquotes AuthZEN §7.1
- **Severity**: minor
- **Section**: AuthZEN facade → "Evaluations batching"; HTTP semantics table
- **Description**: The spec writes: "Facade: `evaluations` absent or empty **and** no top-level `resource` → HTTP **400** (AuthZEN §7.1 would otherwise require a single evaluation with a resource)." AuthZEN §7.1 actually says the opposite: "If an `evaluations` array is NOT present or is empty, the Access Evaluations Request behaves in a backwards-compatible manner with the (single) Access Evaluation API Request." The 400 is correct, but the reason is §7.1.1 ("`subject`, `action`, and `resource` are required for a valid evaluation"), not §7.1. The parenthetical will mislead a WG reader verifying the citation.
- **Suggestion**: Re-cite §7.1.1 and state plainly that a request with no evaluations and no top-level `resource` has no valid evaluation and is therefore rejected as malformed (400), rather than attributing a "single evaluation" requirement to §7.1.

### Issue 3: "Registered" subject type `identity` is not an AuthZEN registry term
- **Severity**: minor
- **Section**: CWT Subject Profile → "Subject, context, resource, action (SARC)"
- **Description**: "**Registered** subject type for this profile: `identity` (matches COAZ-MCP default mappings)." AuthZEN 1.0 has no subject-type registry (the IANA registrations in §12 cover only PDP metadata parameters, capability URNs, and the well-known URI); `subject.type` is free-form (§5.1), and the spec's own examples use `"user"`. `identity` comes from COAZ-MCP's default mappings, not from an AuthZEN registry.
- **Suggestion**: Rephrase to "Subject type used by this profile is `identity` (matching the COAZ-MCP default mappings)"; drop "Registered", or say it is a profile-selected value, not a registered one.

### Issue 4: `evaluation_id` and obligation context keys are profile extensions, not labeled
- **Severity**: minor
- **Section**: Decision translation table; Key Decision 22; wire examples
- **Description**: `context.evaluation_id` (with its `rid`/`rid:index` algorithm) and `context.obligations.required` / `context.pep.fulfillable_obligation_fqns` are all Arkavo-specific additions into AuthZEN's free-form `context`. AuthZEN 1.0 defines no such fields (`evaluation_id` does not occur anywhere in the spec). They are legal because `context` is free-form (§5.5.1), but only `context.obligations.required` is explicitly called out as Arkavo-specific ("Semantic gaps" item 6); the SARC table presents `context.pep.fulfillable_obligation_fqns` and `evaluation_id` as if they were standard. A WG reader could mistake them for AuthZEN fields.
- **Suggestion**: Add an explicit **(profile)** label on `evaluation_id`, `context.pep.fulfillable_obligation_fqns`, and `context.obligations.required` wherever they are introduced, and note they live in the free-form `context` per AuthZEN §5.5.1.

### Issue 5: `entities.size() > 0` constraint attributed to the wrong proto
- **Severity**: nit
- **Section**: "PE + device NPE + environment NPE → one SARC"
- **Description**: "`entity.proto` requires `entities.size() > 0` only". The CEL constraint `has(this.entities) && this.entities.size() > 0` is on the `entity_chain` field of `EntityIdentifier` in `authorization/v2/authorization.proto`, not in `service/entity/entity.proto` (whose `EntityChain` message has no such validation). The conclusion (no 1–10 cap, not an OpenTDF limit) is correct; only the file attribution is off.
- **Suggestion**: Change "`entity.proto` requires" to "`authorization.proto`'s `EntityIdentifier.entity_chain` requires".

### Issue 6: "user token sent to ERS and GetDecision" overstates GetDecision auth
- **Severity**: minor
- **Section**: Background table (MCP PEP row); "Two tokens"
- **Description**: "`arkavo-edge` today sends the **user** token as `Authorization: Bearer {token}` to ERS and GetDecision." In code, the user token is sent as Bearer only to `CreateEntityChainsFromTokens` (`arkavo-edge/crates/arkavo-authorization/src/client.rs:145`); `get_decision`/`get_decision_bulk` go through `make_connect_request` (line 214), which sets no `Authorization` header. The security conclusion (the user token must not be the PDP credential) still holds, but the claim that GetDecision currently receives the user token is inaccurate.
- **Suggestion**: Correct to "sends the user token as `Authorization: Bearer {token}` to ERS `CreateEntityChainsFromTokens`; `GetDecision*` requests are sent unauthenticated."

### Issue 7: Device object with missing `aud` is unspecified
- **Severity**: minor
- **Section**: `context.device` schema; facade chain rules
- **Description**: The device schema marks `aud` REQUIRED, and the facade "MUST still deny-closed if `aud` is present and not `arkavo:devicecheck`" — but the behavior when `aud` is absent from a device object is never stated (400 vs deny-closed). Since `aud` is REQUIRED, a missing `aud` is malformed, but the deny-closed rule only covers the present-and-wrong case, leaving an implementer to guess.
- **Suggestion**: State explicitly: device object missing `sub`/`iss`/`aud`/`kid` (any REQUIRED field) → HTTP 400 (malformed SARC), while present-but-wrong `aud`/`sub` → per-entry deny-closed.

## Strengths
- Exceptional code-grounding: nearly every factual claim (mint/verify functions, `DEFAULT_SKEW_SECS`/`SKEW_SECS = 60`, `exp <= now - 60` vs `<`, `kid` raw-32-byte vs base64url JWKS, nested `GetDecisionResponse.decision.decision`, `resourceDecisions[]`, `skipEnvironmentEntities=true`, `entity_mode` JWT-only token identifier, `is_safe_diagnostic("list_tools")`, `extract_token_ttl` JWT `.`-split, `arks` has no CWT verifier) was independently confirmed in source.
- The two-token separation, claims-mode-only rule, and "never `EntityIdentifier.token`" stance are correctly grounded in the ERS JWT-only parser footgun and are the right security call.
- AuthZEN 1.0 compliance is largely accurate: 200+`decision:false` vs 401, `execute_all` default, `policy_decision_point` well-known equality, `signed_metadata`-as-JWT, discovery RECOMMENDED→profile MUST, and request-side `page.token` vs response-side `page.next_token` are all handled correctly.
- The method `tools/list` vs tool name `list_tools` distinction is crisp, correct, and consistent across tables, Key Decisions, and the PR plan.
- Fail-closed discipline (empty catalog listing → no PDP call; non-empty `required` obligations → deny even on permit; 401 ≠ deny) is internally consistent and repeated in the right places.
- The PR plan is concrete, per-repo, correctly flags the ~400-line AGENTS.md constraint and the `arkavo-edge-swarmkit-apply` duplicate, and keeps the facade behind `AUTHZEN_FACADE=off` with a realistic platform-caller-credential spike gate.
