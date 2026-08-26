# Independent review — draft-arkavo-authzen-cwt-00

## Reviewer

claude

## Summary

**Needs revision.** The architecture is sound and the document is unusually honest about its own
gaps, but two verified defects would make a faithful v1 implementation fail in production: (1) the
normative MCP attribute-value FQNs (`.../mcp-tool/value/git.commit`, and the entire `mcp_server`
percent-encoding section) cannot be provisioned as OpenTDF attribute values, because
`CreateAttributeValueRequest` restricts values to `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$` and
lowercases them; and (2) the "each device `sub` MUST equal `subject.id`" rule rejects every real
PE-CWT + DeviceCheck-CWT pair, because `mint_assertion_token` emits a bare UUID `sub` while OIDC
access tokens emit `arkavo:{uuid}` — so multi-device, the headline v1 feature, 401s as written.
A third correctness issue (OpenTDF action-name charset) and a PR-plan issue (the MCP method
dispatcher lives in a crate the PR-4 file list never touches, and has no CWT to verify) also need
addressing before PR 3/PR 4 start. AuthZEN 1.0 conformance is otherwise good — notably the search
pagination, the 401-vs-200/deny split, and the empty-`evaluations` rule are all correct.

Provenance: OpenTDF claims below were verified against `github.com/opentdf/platform` `main` as
fetched 2026-08-26. In-tree claims were verified against the working copies under
`/Users/arkavo/Projects/arkavo`.

## Issues

### Issue 1: MCP attribute-value FQNs cannot exist in OpenTDF policy (charset + case)

- **Severity**: critical
- **Section**: "Resource translation" → `mcp_server` path-segment encoding; "MCP tool FQN mapping — facade only"; Key Decision 20
- **Description**: The spec makes two normative FQN constructions that OpenTDF cannot provision.

  `service/policy/attributes/attributes.proto`, `CreateAttributeValueRequest.value`:

  ```
  (buf.validate.field).cel = {
    id: "attribute_value_format"
    message: "Attribute value must be an alphanumeric string, allowing hyphens and underscores
              but not as the first or last character. The stored attribute value will be
              normalized to lower case."
    expression: "this.matches('^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$')"
  }
  ```

  `lib/identifier/policyidentifier.go` uses the same `validObjectNameRegex`, and
  `lib/identifier/attribute.go` lowercases namespace/name/value on both `FQN()` and `Parse`.

  Consequences:

  1. Every entry in the alias table contains a `.`: `filesystem.read`, `filesystem.write`,
     `git.commit`, `git.push`, `device_management.tap`, `device_management.swipe`. None can be
     created as an attribute value. (This is pre-existing — `arkavo-edge/crates/arkavo-authorization/src/types.rs:156`
     `McpToolMapping::tool_to_resource` already emits them — but the spec promotes it to normative
     facade behavior.)
  2. The normative `mcp_server` example
     `https://arkavo.net/attr/mcp-server/value/https%3A%2F%2Fmcp.arkavo.net` contains `%` and `.`
     in the value — also unprovisionable.
  3. The MUST "Use uppercase HEXDIG (`%3A`, `%2F`)" is contradicted by OpenTDF's documented
     lower-case normalization, so even if `%` were permitted the stored and requested forms would
     differ.

  The decision RPC does *not* catch this: `authorization.proto` `Resource.attribute_values` only
  validates `this.fqns.size() > 0 && this.fqns.all(item, item.isUri())`. So requests carrying these
  FQNs pass proto validation, match no policy, and — under this profile's mandatory fail-closed —
  produce a permanent, silent DENY for every MCP tool and every `tools/list`. That is exactly the
  failure mode hardest to diagnose from the PEP side.
- **Suggestion**: Delete the `mcp_server` percent-encoding subsection (normative example, negative
  example, and the uppercase-HEXDIG MUST) and Key Decision 20's second sentence. Replace both
  constructions with a slug rule that satisfies `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$` in
  lower case — e.g. `git_commit`, and for `mcp_server` a configured server slug
  (`AUTHZEN_MCP_SERVER_SLUG=mcp_arkavo_net`) rather than a derived URL. Rewrite the alias table to
  dotless values (or drop the table and use the tool name verbatim). Add the charset/case
  constraint to the PR 3 entry-criterion spike so it is confirmed against the deployed platform,
  not just `main`. Note in the spec that the current in-tree `tool_to_resource` FQNs are invalid
  and must change in PR 4.

### Issue 2: `device.sub == subject.id` rejects every real DeviceCheck token

- **Severity**: critical
- **Section**: "`context.device` / `context.environment` JSON (v1)"; "PE + device NPE + environment NPE → one SARC" step 4; Key Decision 16
- **Description**: The spec requires, in four places, that "Each `sub` MUST equal `subject.id`",
  with a 401 (`"entity token subject does not match bearer subject"`) at the PEP and a deny-closed
  at the facade. The two token kinds do not agree on subject format.

  `authnz-rs/src/device_check.rs:577`:

  ```rust
  let claims = crate::cwt::ArkavoClaims::devicecheck(
      &app_state.issuer,
      &user_id.to_string(),   // bare UUID, e.g. "550e8400-e29b-41d4-a716-446655440000"
      AUTH_TOKEN_HOURS,
  ).with_cnf(cnf);
  ```

  `authnz-rs/src/oidc.rs:1583` builds the OIDC subject as `format!("arkavo:{}", account_id)`, and
  `mint_access_token` mints that as `sub`. The spec's own `$token` table asserts `sub` is
  "`arkavo:{UUID}`, `apple:{APPLE_SUB}`, or `client:{CLIENT_ID}`" — true for OIDC access tokens,
  false for DeviceCheck assertion tokens and for `aud=arkavo` auth tokens.

  So for the profile's own worked example — `Authorization: Bearer <PE CWT>` (an OIDC access token,
  `sub=arkavo:550e…`) plus `X-Entity-Token: <DeviceCheck CWT>` (`sub=550e…`) — step 4 returns 401
  and the device example in the spec (`"sub": "arkavo:550e8400-…"` on the device object) is
  unproducible. Multi-device, which Goals and Key Decision 25 elevate to a v1 requirement with a
  chain cap, 400 rules, and dedicated contract tests, cannot be exercised at all.

  `tdf-iroh-s3/crates/tdf-core/src/catalog_api.rs:302` already enforces `claims.sub != pe.sub` →
  401; it happens to work today only when the PE credential is a bare-`sub` `aud=arkavo` auth
  token, which passes this profile's step 9 only if the PEP's configured audience set includes
  `arkavo` — the profile permits that but does not require it.
- **Suggestion**: Pick one and state it normatively. Either (a) define a normalization function —
  strip a leading `arkavo:` from both sides before comparing, and say explicitly that a bare-UUID
  DeviceCheck `sub` binds to `arkavo:{uuid}` — or (b) require `mint_assertion_token` to emit
  `arkavo:{uuid}`, which is a breaking change to existing device tokens and needs its own
  `authnz-rs` PR plus a dual-accept window. Option (a) is cheaper and does not touch `authnz-rs`.
  Either way, correct the `$token` `sub` row to say the format is per-token-kind, and add the
  mismatched-prefix case to the PR 3b contract fixtures.

### Issue 3: OpenTDF action names have the same charset constraint — `issue:cwt:{grant}` and the pass-through row are invalid

- **Severity**: major
- **Section**: "Action Registry"; "Sequence: token issuance"; Key Decision 18; PR 7
- **Description**: `service/policy/actions/actions.proto`, `CreateActionRequest.name`:

  ```
  expression: "this.matches('^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$')"
  message: "... The stored action name will be normalized to lower case."
  ```

  Two rows of the Action Registry break this:

  - `issue:cwt:{grant}` → OpenTDF `issue:cwt:{grant}`. Colons are not permitted, so no such action
    can be created. The whole "intentionally divergent from `draft-gazitt-oauth-authzen-issuance-00`"
    argument is moot at the OpenTDF layer: the IETF form `issue:access_token:{grant}` is equally
    unprovisionable.
  - "any other | unchanged | Pass through". AuthZEN action names are routinely slash-delimited
    (`resources/read`, `prompts/get` — both in COAZ-MCP's default table and both slated for PR 4b).
    Passing those through unchanged mints an invalid OpenTDF action name.

  `read`, `decrypt`, and `execute_tool` are fine.
- **Suggestion**: Change the OpenTDF column for `issue:cwt:{grant}` to a conforming name
  (`issue_cwt_{grant}`, e.g. `issue_cwt_authorization_code`) while keeping the AuthZEN action name
  as-is, and say so in Key Decision 18. Replace the "pass through unchanged" row with an explicit
  rule: the facade MUST reject (HTTP 400) any AuthZEN action whose mapped OpenTDF name does not
  match `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$`, rather than forwarding it. Add a note that
  the facade lowercases the mapped action name to match platform normalization.

### Issue 4: PR 4 is not implementable in the files it lists — MCP method dispatch lives elsewhere, and there is no subject CWT

- **Severity**: major
- **Section**: "PR Plan" → PR 4; "COAZ-MCP CWT profile (normative overrides)" 5–8
- **Description**: PR 4's file list is confined to
  `arkavo-edge/crates/arkavo-authorization/src/{client,types,config,error,cache,cwt_verify,cwt_subject,tests}.rs`
  plus that crate's `Cargo.toml`. That crate has no MCP method dispatch and no MCP transport. The
  JSON-RPC dispatcher is `arkavo-edge/crates/arkavo-mcp-proxy`, which:

  - does not depend on `arkavo-authorization` at all (`crates/arkavo-mcp-proxy/Cargo.toml`);
  - explicitly passes through everything that is not `tools/call`
    (`proxy.rs:167`, and `lib.rs:7`: "All other methods (`initialize`, `tools/list`,
    notifications, ...) are passed through");
  - states "Calling identity is deliberately out of scope for this slice"; `CallContext` carries
    only `tool_name` and `arguments` — there is no token to verify;
  - uses `POLICY_DENIED: i64 = -32000` (`proxy.rs:17`), not the `-32001` this profile and COAZ-MCP
    mandate.

  The only production caller of `authorize_mcp_tool` is
  `crates/arkavo-mcp-claude/src/policy_bridge.rs:189`, which is also not in PR 4's file list, and
  which sources its "token" from `CLAUDE_CODE_SESSION_ACCESS_TOKEN` falling back to
  `ANTHROPIC_API_KEY` (`policy_bridge.rs:180-182`). An Anthropic API key is not an Arkavo CWT, so
  "union-verify user CWT → `$token`" has no valid input in the only wired-up path. The spec never
  says where the MCP PEP's subject CWT comes from.

  So the three PR-4 deliverables that are method-level — hardcoded `tools/list`, "unknown MCP
  methods MUST be denied", and the `-32001`/`-32602`/`-32603` codes — land in crates the PR does
  not open.
- **Suggestion**: Add `arkavo-edge/crates/arkavo-mcp-proxy/src/{lib,proxy,policy}.rs` and
  `arkavo-edge/crates/arkavo-mcp-claude/src/policy_bridge.rs` to PR 4's file list, with the
  concrete changes: extend `CallContext` with the caller's credential and the JSON-RPC `method`,
  route non-`tools/call` methods through the PEP, change `POLICY_DENIED` to `-32001` (keeping
  `-32602` for mapping errors and `-32603` for PDP failures), and wire the `arkavo-authorization`
  dependency. Add a subsection to the COAZ-MCP profile specifying the subject-credential source for
  the MCP PEP (an MCP `Authorization: Bearer <CWT>` header, or a named env var that is documented
  as CWT-only), and state that `ANTHROPIC_API_KEY` MUST NOT be accepted as a subject credential.

### Issue 5: `resource.id` from `$token.aud` and the `context.agent` fallback are undefined for the production token shape

- **Severity**: major
- **Section**: "COAZ-MCP CWT profile" overrides 3 and 4; "MCP method `tools/list`" example; SARC table `context.agent` row
- **Description**: `authnz-rs/src/oidc.rs:1736-1741`:

  ```rust
  if let Some(platform_aud) = app_state.platform_audience.as_ref()
      && platform_aud != audience
  {
      claims.aud = Audience::Multiple(vec![audience.to_string(), platform_aud.clone()]);
  }
  ```

  `platform_audience` is `OIDC_PLATFORM_AUDIENCE` (`src/main.rs:399`), documented as
  "e.g. https://platform.arkavo.net" and applied to **every** OIDC access token. When it is set —
  which is the production configuration implied by the `arks`/platform topology this spec is built
  around — no access token has a single-string `aud`.

  Two profile rules then never fire as written:

  - `context.agent` fallback: "else if `aud` is a single string other than `arkavo` and
    `arkavo:devicecheck`, use `aud`". With multi-aud, `context.agent` is silently omitted. Both
    normative `tools/call` examples in the spec ("Aud-fallback" and "Omit `agent`") assume a single
    string.
  - `tools/list` `resource.id = $token.aud`: the spec says only "Same COAZ-MCP resolution: never an
    array." COAZ-MCP's actual rule is stronger and requires configuration the spec never defines:
    "the PEP MUST select the single member that identifies this server (its own resource identifier
    per RFC 8707) and use that value ... If resolution fails, it's a mapping error." A v1 PEP with
    hardcoded mappings and no CEL has nothing to select with, so every `tools/list` becomes a
    `-32602` mapping error.
- **Suggestion**: Add a normative config item — e.g. `AUTHZEN_MCP_RESOURCE_ID` (the server's own
  RFC 8707 resource identifier) — and specify: `resource.id` for `mcp_server` is that configured
  value, and it MUST be present in `$token.aud` (single string or array member) or the PEP raises
  a mapping error. Restate the `context.agent` fallback in terms of "`aud` contains exactly one
  member that is neither `arkavo` nor `arkavo:devicecheck` nor the configured platform audience".
  Add a worked multi-aud example alongside the two single-aud ones, and note in the `$token` table
  that multi-aud is the normal production shape when `OIDC_PLATFORM_AUDIENCE` is set.

### Issue 6: PEP-supplied `attribute_value_fqns` are authoritative even for `tool` — resource substitution

- **Severity**: major
- **Section**: "Resource translation" (the sentence after the type table); Security & Privacy Considerations
- **Description**: The spec states: "If `resource.properties.attribute_value_fqns` is present it is
  authoritative for the OpenTDF attribute set even when `type` is `tool`." Combined with the facade
  authentication rules — where `AUTHZEN_PEP_CLIENT_IDS` is *optional* and, when unset, "any valid
  service-account CWT is accepted" — any holder of any service-account CWT can send
  `{"type":"tool","id":"git_push","properties":{"attribute_value_fqns":["https://arkavo.net/attr/mcp-tool/value/status"]}}`
  and be evaluated against the low-sensitivity attribute while the audit log records
  `resource_id = git_push`. The facade's own alias table, described as the anti-drift control
  ("**Only the facade** expands aliases"), is bypassed by the same request.

  The security table covers identity smuggling (`subject.id` trust-anchoring) and environment
  spoofing but has no row for resource substitution, and the Observability section logs
  `resource_type` / `resource_id` rather than the FQNs actually evaluated — so the substitution is
  invisible in logs.
- **Suggestion**: Scope the override: for `resource.type` in `{tool, mcp_server}` the facade MUST
  derive the FQN set itself and MUST reject (400) or ignore any PEP-supplied
  `attribute_value_fqns`. Keep the override only for `catalog_item` / `tdf` / `kas`, where the PEP
  legitimately owns the attribute set. Add a "Resource substitution by an authenticated PEP" row to
  the security table, and add the evaluated FQNs (or their count/hash) to the log fields.

### Issue 7: `context.environment` sanitization is a denylist that misses the ERS lookup keys

- **Severity**: major
- **Section**: "`context.device` / `context.environment` JSON (v1)"; "PE + device NPE + environment NPE → one SARC" (Rules)
- **Description**: "Facade MUST strip `sub` / `email` / `arkavo_patreon` / `aud` / `kid` from
  environment if a PEP sent them." That denylist omits precisely the identifiers the Patreon ERS
  resolves on. `tdf-iroh-s3/crates/tdf-core/src/authz.rs:11-14` documents the lookup order:

  > the Patreon ERS resolves `Entity_Claims` entities via `resolveFromClaims` (lookup order:
  > `patreon_access_token` → `patreon_user_id` → `email` → `preferred_username`)

  `patreon_access_token`, `patreon_user_id`, and `preferred_username` all pass the strip filter. The
  environment entity is forwarded to the PDP as a full `Entity_Claims` struct, and the spec itself
  says environment entities are forwarded and that "PEPs MUST NOT assume they affect v1 decisions"
  — i.e. they may affect decisions in a later platform build. A denylist that is one platform
  change away from becoming an identity-injection vector is the wrong shape here.
- **Suggestion**: Replace the denylist with an allowlist: the facade MUST construct the environment
  entity's claims from `{region}` (plus optional `kind`) only, and MUST drop every other key
  silently rather than enumerating forbidden ones. Apply the same allowlist discipline to the
  device object (`{sub, iss, aud, kid}` only) — the spec already fixes that schema, but does not
  say the facade drops unknown keys.

### Issue 8: multi-device is declared a v1 requirement but has no policy effect in v1

- **Severity**: major
- **Section**: Goals; "device vs devices"; Key Decision 25; "Semantic gaps" item 3
- **Description**: Goals says "Multi-device (`context.devices`) is a v1 requirement (phone +
  watch, etc.)" and Key Decision 25 calls it "first-class". The wire consequences are substantial:
  a `1+D+E ≤ 8` cap, three distinct HTTP 400 conditions, header-order semantics, an extended
  `CwtVerifier` in PR 5, and dedicated two-device contract fixtures in PR 3b.

  But the facade emits every device NPE as `CATEGORY_ENVIRONMENT` (the spec's own worked
  `GetDecisionMultiResource` example shows the device at `e1` with
  `"category": "CATEGORY_ENVIRONMENT"`, matching `authz.rs:243-246` — `entity.proto` `Entity.Category`
  only has `CATEGORY_UNSPECIFIED | CATEGORY_SUBJECT | CATEGORY_ENVIRONMENT`, so there is no device
  category to use). And `authz.rs:22-24` records that the platform sets
  `skipEnvironmentEntities=true`, dropping them before evaluation.

  So in v1: devices are indistinguishable from the environment entity, and both are discarded by
  the PDP. No phone-plus-watch policy can be written or enforced. This is stated once, in a
  sub-bullet ("PEPs MUST NOT assume they affect v1 decisions") and once in Semantic gaps, both of
  which read as caveats rather than as "this feature is wire-format only." A working-group reader,
  or an implementer sizing PR 5, will reasonably conclude the opposite.
- **Suggestion**: Say it plainly in Goals and in Key Decision 25: v1 multi-device standardizes the
  *wire shape and verification rules* so devices are available to policy later; it has **no effect
  on v1 decisions** because devices are carried as `CATEGORY_ENVIRONMENT` and the platform sets
  `skipEnvironmentEntities=true`. Add the intended distinguishing mechanism (e.g. a `kind` or
  `npe_type` key inside the claims struct, once the platform stops skipping) as an explicit future
  item, so PR 5's cost is justified against a named end state.

### Issue 9: Empty `fulfillableObligationFqns` means obligation-bearing resources are always denied — say so

- **Severity**: minor
- **Section**: "Decision translation" → v1 obligations; Key Decision 19
- **Description**: The spec says "v1 PEPs MUST send empty fulfillable lists" and treats a
  PERMIT-with-required-obligations as a should-not-happen edge case. `authorization.proto` documents
  the platform behavior directly: "If entitled, checks obligation policy: fulfillable obligations
  must satisfy all triggered." With an empty fulfillable list, any attribute value carrying a
  triggered obligation yields DENY at the platform, not PERMIT-plus-obligations. The operational
  consequence — every obligation-bearing catalog attribute is unentitled for the entire v1 window —
  is not stated, and it is the kind of thing that surfaces as an unexplained entitlement regression
  during the PR 5 staging flip.
- **Suggestion**: Add one sentence under "v1 obligations": sending an empty fulfillable list causes
  the platform to DENY any resource with a triggered obligation; operators MUST confirm no
  production attribute value carries an obligation trigger before flipping `protocol = "authzen"`,
  and the shadow phase in Rollout step 5 should diff exactly this.

### Issue 10: Obligation placement in the `evaluations` response is unspecified

- **Severity**: minor
- **Section**: "Decision translation" table
- **Description**: The table maps `requiredObligations: [...]` → `context.obligations.required`
  without saying which `context`. `ResourceDecision.required_obligations` is per-resource, so in a
  `GetDecisionMultiResource` response there is one list per evaluation, but the spec's only
  `evaluations` response example shows `context` containing solely `evaluation_id`. AuthZEN 1.0
  permits both a top-level and a per-evaluation `context`, so an implementer can pick either and be
  self-consistent while being wire-incompatible with the other.
- **Suggestion**: State that `context.obligations.required` appears on the **per-evaluation**
  `context` object for Access Evaluations and on the top-level `context` for Access Evaluation, and
  add a non-empty-obligations line to the `evaluations` response example (or to PR 3b's fixtures).

### Issue 11: "Registered subject type" overstates AuthZEN

- **Severity**: minor
- **Section**: "Subject, action, resource, context (SARC)", first line
- **Description**: "Registered **subject type** for this profile: `identity`". AuthZEN 1.0 defines
  `subject.type` as "REQUIRED. A string value that specifies the type of the Subject" and
  establishes no registry and no registered values; `identity` comes from COAZ-MCP's default
  mappings, not from a registration. An AuthZEN WG reader will look for the registry and not find
  one.
- **Suggestion**: Reword to "Subject type for this profile: `identity` (the value used by COAZ-MCP
  default mappings; AuthZEN 1.0 does not define a subject-type registry)."

### Issue 12: PR 2b omits the `arkavo-rs` dependency additions

- **Severity**: minor
- **Section**: "PR Plan" → PR 2b
- **Description**: PR 2b's file list is
  `arkavo-rs/src/modules/authzen/cwt_verify.rs; tests`. `arkavo-rs/Cargo.toml` has `p256`,
  `ciborium`, `base64`, and `reqwest`, but **no `coset`** — and the union algorithm (COSE_Sign1
  parse, protected-header `alg`/`kid`, COSE_Key set decode) needs it. PR 4's entry correctly calls
  out its Cargo.toml and new deps; PR 2b does not, so the two sibling PRs are inconsistent about
  the same dependency set.
- **Suggestion**: Add `arkavo-rs/Cargo.toml` to PR 2b's file list with `coset` as a new dependency,
  matching PR 4's "New deps" line.

### Issue 13: Two-token table understates the service CWT `aud`

- **Severity**: minor
- **Section**: "Two tokens (MUST NOT be confused)" table; Key Decision 6
- **Description**: The table gives the service CWT as `aud={that client_id}`, and Key Decision 6
  says "Service CWTs are minted with `aud = client.client_id` (`handle_client_credentials_grant`)".
  `handle_client_credentials_grant` calls `mint_access_token(..., &client.client_id, ...)`, which
  appends `platform_audience` when `OIDC_PLATFORM_AUDIENCE` is set — so the production shape is
  `aud = [client_id, platform_audience]`. The facade rule ("if `aud` is a single string, it MUST
  equal that id") degrades gracefully, but the descriptive text is wrong and is the same trap as
  Issue 5.
- **Suggestion**: Change to `aud = {that client_id}` (plus the configured platform audience when
  `OIDC_PLATFORM_AUDIENCE` is set — RFC 8707-style), and cross-reference the multi-aud note added
  for Issue 5.

## Strengths

- **Search pagination is exactly right.** The request field is `page.token` and the response field
  is `page.next_token` with `""` as the terminator — the profile gets this correct, including the
  `count` field and the "opaque, PEPs MUST NOT parse" rule. This is the detail most profiles get
  backwards.
- **Empty `evaluations` handling is compliant and well-reasoned.** AuthZEN 1.0 §7.1 permits omitting
  top-level `subject`/`action`/`resource` when every evaluation entry supplies them, and treats an
  absent/empty `evaluations` array as a single evaluation. The profile's rule — 400 only when
  `evaluations` is empty/absent *and* there is no top-level `resource` — is precisely the right
  narrowing, and the explicit "Do not invent a 200 `{"evaluations": []}` exception" forecloses the
  most common mistake.
- **401 vs 200+deny is drawn correctly and repeated where it matters** (Goals, facade PEP
  authentication, HTTP semantics table, Key Decision 10, PR 3b fixtures). Adding 403 for the
  optional allowlist is the right use of AuthZEN's status table rather than inventing a code.
- **The nested `GetDecisionResponse` correction is a genuine find.** `authorization.proto` has
  `GetDecisionResponse { ResourceDecision decision = 1; }`, and `arkavo-edge`'s `types.rs` models it
  as a flat enum — the spec identifies the bug, names the file, and specifies the fix.
- **The ERS JWT-only footgun is handled at the right layer.** Forbidding `EntityIdentifier.token`
  for CWTs and mandating claims mode, rather than trying to make CWTs survive a JWT parser, is the
  correct call and is stated in five places without contradiction.
- **The "Semantic gaps (honest)" section is unusually good.** Naming Search ≉ GetEntitlements,
  entity-chain-vs-one-subject, and the environment-skipping up front is what makes this document
  reviewable at all; most of the issues above are refinements of gaps the authors already flagged.
- **Two-token separation and the claims-as-decisions rule** are both carried consistently from
  Goals through the security table to the per-PR descriptions, with the CWT-specific reading of
  `draft-gazitt-oauth-authzen-claims-00` made explicit rather than assumed.
- **Verified-accurate against code** on many specifics: `DEFAULT_SKEW_SECS`/`SKEW_SECS` = 60 and the
  choice of `exp <= now - 60` as the stricter comparison; CWT tag `0xD8 0x3D` and untagged-rejection;
  `KEY_REFRESH_MIN_INTERVAL` 60s / 10s fetch timeout; catalog's 10s and edge's 5s client timeouts;
  `get_all("x-entity-token")`; the `arkavo:devicecheck` audience; service-CWT `sub=client:{id}` with
  `arkavo_roles = ["service-account"]`; `extract_token_ttl`'s JWT `.`-split; the `~400 line` /
  `≥85% coverage` constraints in `arkavo-edge/AGENTS.md`; and the absence of any COSE/CWT dependency
  in `arkavo-rs` today.
