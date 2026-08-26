```text
---
title: "CWT Subject Profile and AuthZEN Facade for the Arkavo Ecosystem"
abbrev: "authzen-cwt"
docname: draft-arkavo-authzen-cwt-00
category: exp
ipr: trust200902
area: Security
workgroup: Arkavo Specifications
keyword:
 - AuthZEN
 - CWT
 - COSE
 - OpenTDF
 - ABAC
 - MCP
 - COAZ
 - KAS
 - PDP
 - PEP

author:
 - fullname: Arkavo Contributors
   organization: Arkavo
   email: info@arkavo.com

date: 2026-08-26

normative:
  RFC2119:
  RFC8174:
  RFC6750:
  RFC7519:
  RFC7638:
  RFC8259:
  RFC8392:
  RFC8615:
  RFC8747:
  RFC9052:
  AUTHZEN:
    title: "Authorization API 1.0"
    target: https://openid.net/specs/authorization-api-1_0-final.html
    date: 2026-01-11
  COAZFW:
    title: "COAZ: A Framework for Mapping Information Models to AuthZEN Authorization Requests"
    target: https://openid.github.io/authzen/authzen-coaz-framework-1_0.html
  COAZMCP:
    title: "COAZ-MCP: COAZ Binding for the Model Context Protocol"
    target: https://github.com/openid/authzen/blob/main/profiles/authzen-coaz-mcp-binding-1_0.md
  OPENTDF-AUTHZ-V2:
    title: "OpenTDF platform authorization.v2.AuthorizationService"
    target: https://github.com/opentdf/platform/blob/main/service/authorization/v2/authorization.proto

informative:
  RFC6749:
  RFC7515:
  RFC8707:
  RFC9068:
  RFC9200:
  CEL:
    title: "Common Expression Language"
    target: https://cel.dev/
  JWT-TO-CWT:
    title: "JWT → CWT Migration Design"
    target: https://github.com/arkavo-org/authnz-rs/blob/main/docs/superpowers/specs/2026-05-23-jwt-to-cwt-design.md
  ISSUANCE:
    title: "AuthZEN Profile for OAuth 2.0 Token Issuance"
    target: https://datatracker.ietf.org/doc/html/draft-gazitt-oauth-authzen-issuance-00
  CLAIMS:
    title: "AuthZEN Profile for Authorization Claims in JWT Access Tokens"
    target: https://datatracker.ietf.org/doc/html/draft-gazitt-oauth-authzen-claims-00
  NTDF-TOKEN:
    title: "NTDF Tokens: NanoTDF-based Authentication Tokens (SUPERSEDED)"
    target: https://github.com/arkavo-org/specifications/blob/main/ntdf-token/draft-arkavo-ntdf-token-00.md

--- abstract

This specification profiles the OpenID AuthZEN Authorization API 1.0 as the PEP↔PDP decision protocol for the Arkavo ecosystem, while retaining Arkavo-issued CBOR Web Tokens (CWT, RFC 8392, COSE_Sign1 ES256) as the access-token / subject credential. It defines (1) a CWT Subject Profile that maps verified CWT claims onto AuthZEN's Subject-Action-Resource-Context (SARC) model and onto the COAZ-MCP `$token` input variable; (2) an AuthZEN HTTPS JSON facade over the existing OpenTDF Authorization Service v2 PDP; (3) a CWT-aware reading of COAZ-MCP. CWT is not replaced by JWT. AuthZEN request JSON is not embedded in CWT. Token-carried `arkavo_roles` / `arkavo_entitlements` are enrichment only.

--- middle
```

# CWT Subject Profile and AuthZEN Facade for the Arkavo Ecosystem

- **Document:** `draft-arkavo-authzen-cwt-00`
- **Author:** Arkavo Contributors `<info@arkavo.com>`
- **Date:** 2026-08-26
- **Status:** Draft
- **Intended publication:** `specifications/authzen-cwt/draft-arkavo-authzen-cwt-00.md`

The kramdown-rfc YAML above is fenced to match `specifications/ntdf-rtmp` house style. This draft is published in the specifications repo; it is not an IETF Internet-Draft submission and is not expected to compile with xml2rfc.

## Overview

Arkavo already issues CWT (RFC 8392) access tokens from `authnz-rs` and already evaluates ABAC at OpenTDF Authorization Service v2 (`platform.arkavo.net`, optionally reached through `arks` when `AUTHZ_PROXY=on`). What is missing is a standard PEP↔PDP protocol: `tdf-iroh-s3` speaks ConnectRPC-JSON `GetDecisionMultiResource` with a claims-mode entity chain; `arkavo-edge/crates/arkavo-authorization` speaks OpenTDF v2 `GetDecision` / `GetDecisionBulk` after `CreateEntityChainsFromTokens`; NanoTDF rewrap stays on a CBOR WebSocket path inside `arkavo-rs`. None of those PEPs speak AuthZEN.

This document specifies an **AuthZEN 1.0 HTTPS JSON facade** in `arkavo-rs` (`arks`) that sits beside the existing `AUTHZ_PROXY` relay. PEPs (first: MCP `tools/call` and method `tools/list` in `arkavo-edge`; second: the entitled catalog in `tdf-iroh-s3`) call AuthZEN. The facade verifies the PEP's **service CWT**, translates SARC into the OpenTDF v2 entity-chain shape `tdf-iroh-s3` already uses in claims mode, calls `GetDecision` / `GetDecisionMultiResource` / `GetDecisionBulk` / later `GetEntitlements`, and translates PERMIT/DENY plus obligation FQNs back into AuthZEN decisions. The subject credential remains CWT. The policy engine remains OpenTDF Authorization v2 (subject mappings, attribute FQNs, ERS, obligations). JWT is retained only where OIDC Core or Apple require it (`id_token`) and for AuthZEN `signed_metadata` (defined as a JWT; MUST NOT be CWT-ified).

## Background & Motivation

### Current state (verified in tree)

| Component | Path | Role today |
|---|---|---|
| CWT issuer | `authnz-rs/src/cwt.rs` (`mint`, `verify`, `encode_for_header`, `decode_from_header`) | All Arkavo-issued tokens: Registration (~99y), Auth (1h), DeviceCheck assertion (1h, `aud=arkavo:devicecheck`), OIDC access. Inbound JWT rejected on Arkavo authn paths. |
| Catalog PEP | `tdf-iroh-s3/crates/tdf-core/src/auth.rs`, `authz.rs`, `catalog_api.rs` | Verifies PE CWT + `X-Entity-Token` device CWTs; builds PE→NPE→environment chain; `POST …/GetDecisionMultiResource`; fail-closed `entitled:false`. Production NPE claims are `{sub, iss}` only. `VerifiedClaims` is a subset of this profile. |
| KAS / platform proxy | `arkavo-rs/src/modules/platform_proxy.rs`, `docs/platform-proxy.md`, `src/bin/main.rs` | Local NanoTDF KAS. `AUTHZ_PROXY=on` relays `/authorization.v2.AuthorizationService/*`. CBOR rewrap in `src/modules/cbor_protocol.rs` (`CborRequest::ChainRewrap`). `arks` has **no** CWT verifier today. Historical `src/modules/ntdf_token.rs` implements SUPERSEDED NTDF tokens; it is **not** the AuthZEN subject credential. |
| MCP PEP | `arkavo-edge/crates/arkavo-authorization` | `AuthorizationClient::authorize_mcp_tool`; ERS `CreateEntityChainsFromTokens`; resource FQN `https://arkavo.net/attr/mcp-tool/value/{tool}`; action `execute_tool`; diagnostic allowlist of **tool names** `status`/`health`/`version`/`list_tools`; LRU cache. Sends the **user** token as `Authorization: Bearer {token}` to ERS `CreateEntityChainsFromTokens`; `GetDecision*` requests are sent **unauthenticated** (`make_connect_request` sets no `Authorization` header). JWT-splits `extract_token_ttl` on `.` (CWT always falls back to 1 minute). Does **not** implement the MCP **method** `tools/list`. The only in-tree caller of `authorize_mcp_tool` is `arkavo-edge/crates/arkavo-mcp-claude/src/policy_bridge.rs`, which sources the "token" from `CLAUDE_CODE_SESSION_ACCESS_TOKEN` falling back to `ANTHROPIC_API_KEY` (an API key is not a CWT). In-tree `McpToolMapping::tool_to_resource` currently emits dotted values (`git.commit`) that **cannot** be provisioned as OpenTDF attribute values (see OpenTDF identifier charset). Duplicate tree: `arkavo-edge-swarmkit-apply/crates/arkavo-authorization`. |
| On-chain engine | `arkavo-node/contracts/policy_engine` | Ink! `evaluate_access(account, policy_id)` — README calls it a PDP; `lib.rs` checks `active` only. Not AuthZEN-shaped. |
| Swift clients | `OpenTDFKit`, `app/` | Bearer tokens are opaque (JWT or base64url CWT). They present credentials; they do not speak AuthZEN. |

Pain points:

1. **ERS JWT-only token identifier.** OpenTDF `entity.Token.jwt` is parsed with a JWT parser. `tdf-iroh-s3/crates/tdf-core/src/authz.rs` documents this: `entity_mode = "token"` only works for JWT IdPs; the default `entity_mode = "claims"` extracts `arkavo_patreon` / `email` from a *verified* CWT and sends `Entity_Claims`. `arkavo-edge` still calls `CreateEntityChainsFromTokens`, which is the JWT footgun for Arkavo CWTs.
2. **No standard PEP protocol.** Catalog, MCP, and KAS each have a private client for the same PDP. Interop with Cerbos / Topaz / OPA / OpenFGA / Amazon Verified Permissions / Ping / PlainID is impossible at the PEP boundary.
3. **COAZ-MCP types `$token` as JWT RFC 7519.** Arkavo PEPs cannot conform without a CWT-aware reading.
4. **Claims treated as decisions.** `arkavo_roles` / `arkavo_entitlements` / `arkavo_patreon` are minted into CWT (`authnz-rs` `CustomClaims`). The AuthZEN Claims profile rule MUST apply even though the token is CWT: enrichment at mint, evaluate at use.
5. **Air-gap.** `arkavo-edge/SECURITY.md` requires a self-hosted KAS to unwrap offline. A remote AuthZEN hop on NanoTDF rewrap would break that.

### Architectural decision (not reopened)

**AuthZEN facade over the existing OpenTDF Authorization Service v2 PDP, with CWT as the subject credential.** Policy stays at `platform.arkavo.net` (or a co-located platform sidecar). PEPs speak AuthZEN. KAS rewrap stays on OpenTDF internally until obligation round-trip is proven. Access tokens stay CWT.

## Goals & Non-Goals

### Goals

- Adopt AuthZEN Authorization API 1.0 as the PEP↔PDP *decision* protocol for new and migrated PEPs.
- Keep CWT (RFC 8392, COSE_Sign1 ES256, CBOR tag #6.61) as the Arkavo-issued access-token / subject credential.
- Define a complete, implementable mapping from verified CWT + **zero or more** device NPEs + node-asserted environment onto one AuthZEN subject + context, and from that SARC onto the OpenTDF v2 entity chain already used in `tdf-iroh-s3` claims mode. Multi-device (`context.devices`) is a v1 **wire and verification** requirement (phone + watch, etc.). It has **no effect on v1 decisions**: device NPEs are emitted as `CATEGORY_ENVIRONMENT` and the platform sets `skipEnvironmentEntities=true`. The named future is that the platform stops skipping, or a `kind` / `npe_type` claim distinguishes devices from environment.
- Define a CWT-aware COAZ-MCP profile so `$token` is the decoded claims map after PEP verification.
- Ship an AuthZEN HTTPS JSON facade in `arkavo-rs` (`evaluation` + `evaluations` first; search later) with `/.well-known/authzen-configuration`.
- Fail closed at every PEP. Distinguish HTTP 401 (PEP failed to authenticate to the PDP) from HTTP 200 `{ "decision": false }` (deny).
- Preserve air-gapped NanoTDF unwrap: no remote AuthZEN JSON on the CBOR rewrap hot path in v1.

### Non-Goals

- Replacing CWT with JWT, or dual-issuing access tokens as JWT. OIDC `id_token` and Apple `id_token` remain JWT (OIDC Core / Apple).
- Embedding AuthZEN request JSON inside CWT.
- Treating `arkavo_entitlements` / `arkavo_roles` / `arkavo_patreon` as the live permit condition at a PEP.
- Replacing OpenTDF Authorization Service v2 as the policy engine in v1 (Cerbos/Topaz/OPA are alternatives, not the selected design).
- Putting AuthZEN on the NanoTDF CBOR rewrap path (`arkavo-rs/src/modules/cbor_protocol.rs`).
- An AuthZEN JSON 1.0 client on NFC/BLE. jwt-to-cwt cites those transports as a size motivation for CWT; there is no NFC/BLE PEP in tree. Constrained-transport AuthZEN is a future binding, not a current path to keep off JSON.
- Requiring Swift clients (`OpenTDFKit`, `app/`) to speak AuthZEN. They remain credential presenters.
- Making `arkavo-node/contracts/policy_engine` an AuthZEN 1.0 PDP. It MAY later act as a Policy Information Point (PIP).
- Reviving NTDF tokens (`specifications/ntdf-token/`). That draft is SUPERSEDED by the jwt-to-cwt design; CWT replaced NTDF tokens. This document does not mix NTDF tokens into the wire format. `arkavo-rs/src/modules/ntdf_token.rs` is a historical leftover, not this profile's credential.
- TØR-G (`specifications/torg-decision/`) as a PDP. Out of scope; at most a future policy-compilation IR.
- A CBOR/CoAP AuthZEN binding (AuthZEN 1.0 allows other bindings as profiles; ACE-OAuth RFC 9200 is related prior art). Not 1.0 of *this* spec.
- DPoP / verifier-enforced `cnf` (still bound-at-issuance metadata per jwt-to-cwt design).
- CWT-ifying AuthZEN `signed_metadata` (it is a JWT per AuthZEN §9.1.3).
- A built-in SARC evaluator inside `arks`. `AUTHZEN_FACADE` is `off` or `on` only. Air-gapped AuthZEN JSON, if needed, requires a co-located OpenTDF platform process.

## Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Where this profile is stricter than AuthZEN 1.0, the extra requirement is labeled **(profile)**. AuthZEN 1.0 itself does not require well-known discovery of every PDP (AuthZEN §9: RECOMMENDED). This profile REQUIRES discovery for Arkavo PEPs **(profile)**.

| Term | Meaning |
|---|---|
| **PEP** | Policy Enforcement Point — AuthZEN client. Catalog node, MCP gateway/server, (later) KAS, (later) `authnz-rs` token endpoint. |
| **PDP** | Policy Decision Point — AuthZEN server. In this design: the facade in `arks`, which delegates evaluation to OpenTDF Authorization v2. |
| **Facade** | The AuthZEN 1.0 HTTPS JSON implementation in `arkavo-rs` that translates SARC ↔ OpenTDF v2. |
| **PE** | Person Entity — the human or service principal. Identified by PE CWT `sub`. |
| **NPE** | Non-Person Entity — device, environment, or agent. Device is a CWT; environment is node-asserted; agent is OIDC client identity. |
| **SARC** | Subject, Action, Resource, Context — AuthZEN information model. |
| **Subject CWT** | The user/device CWT whose claims populate SARC. Not used as the AuthZEN `Authorization` header. |
| **Service CWT** | Short-lived CWT minted by `authnz-rs` `client_credentials` (`sub` = `client:{id}`, `arkavo_roles` includes `service-account`). Authenticates the PEP to the facade. |
| **`$token`** | COAZ-MCP input variable: JSON object of **text-named** claims from a PEP-verified CWT. |
| **Claims mode** | OpenTDF `EntityIdentifier.entity_chain` of `Entity_Claims` (Any-wrapped `google.protobuf.Struct`). Default. |
| **Token mode** | OpenTDF `EntityIdentifier.token.jwt`. JWT-only. MUST NOT be used for Arkavo CWTs. |
| **MCP method `tools/list`** | JSON-RPC method. COAZ-MCP default mapping uses `action.name = "tools/list"`, `resource.type = "mcp_server"`. |
| **Tool name `list_tools`** | A tool invoked via MCP method `tools/call` with `params.name = "list_tools"`. Distinct from the method `tools/list`. |

Clock skew constant: `DEFAULT_SKEW_SECS = 60` (`authnz-rs/src/cwt.rs`); `SKEW_SECS = 60` (`tdf-iroh-s3/crates/tdf-core/src/auth.rs`). This profile uses **±60 seconds**. Expiry comparison is `exp <= now - 60` (the stricter of the two existing verifiers).

## Proposed Design

### Architecture

The facade lives in **`arkavo-rs` (`arks`)**, next to `AUTHZ_PROXY`.

**Rationale.** Catalog PEPs already call `https://platform.arkavo.net` or the `arks` host for `/authorization.v2.AuthorizationService/GetDecisionMultiResource` (`arkavo-rs/docs/platform-proxy.md`). Putting AuthZEN on the same process gives PEPs a single HTTPS origin for KAS discovery (`/.well-known/opentdf-configuration` when proxying) and PDP discovery (`/.well-known/authzen-configuration`). Air-gapped deployments already run `arks` locally; AuthZEN JSON there requires a **co-located OpenTDF platform sidecar** (`OPENTDF_PLATFORM_URL` may be loopback). A dedicated AuthZEN microservice would add an extra hop, a second TLS identity, and another mix-up surface without moving the actual policy engine.

`AUTHZ_PROXY` remains a *byte relay* of OpenTDF v2 for rollback and for the platform KAS's own `GetDecision` calls. The facade is a *semantic translator* on new paths. They are independently flagged. When both are on, PEPs MUST pin the AuthZEN well-known endpoints and MUST NOT POST to `/authorization.v2.AuthorizationService/*`.

```mermaid
flowchart TB
  subgraph issuers [Issuance - later phase]
    IdP["authnz-rs<br/>CWT mint / verify<br/>/.well-known/cose-keys"]
  end

  subgraph peps [PEPs speak AuthZEN JSON]
    MCP["arkavo-edge<br/>COAZ-MCP PEP"]
    Catalog["tdf-iroh-s3<br/>entitled catalog PEP"]
    Swift["OpenTDFKit / app<br/>CWT presenter only"]
  end

  subgraph arks [arks = arkavo-rs]
    Fac["AuthZEN facade<br/>/access/v1/evaluation<br/>/access/v1/evaluations<br/>/.well-known/authzen-configuration"]
    Proxy["AUTHZ_PROXY relay<br/>/authorization.v2.AuthorizationService/*"]
    LocalKAS["Local NanoTDF KAS<br/>CBOR /ws ChainRewrap<br/>NO AuthZEN in v1"]
  end

  subgraph pdp [Policy engine unchanged]
    OTDF["OpenTDF Authorization v2<br/>platform or loopback sidecar<br/>GetDecision / MultiResource / Bulk / GetEntitlements<br/>ERS + obligations + attribute FQNs"]
  end

  IdP -->|"user CWT / DeviceCheck CWT"| MCP
  IdP -->|"service CWT via client_credentials"| MCP
  IdP -->|"PE CWT + DeviceCheck CWT"| Catalog
  IdP -->|"service CWT via client_credentials"| Catalog
  IdP -->|"opaque Bearer CWT"| Swift
  MCP -->|"AuthZEN + service CWT"| Fac
  Catalog -->|"AuthZEN + service CWT"| Fac
  Swift -->|"rewrap Bearer"| LocalKAS
  Fac -->|"claims-mode entity chain<br/>caller credential per PR 3 gate"| OTDF
  Proxy -->|"byte relay"| OTDF
  LocalKAS -.->|"v1: local KAS policy<br/>NOT remote AuthZEN"| LocalKAS
```

### Sequence: entitled catalog (v1 consumer)

Load: one `evaluations` call per `GET /catalog/{group}` covering tens–hundreds of items. Existing client timeout is 10s (`ConnectAuthzClient` in `authz.rs`). Facade→OpenTDF timeout is also **10s (profile)**. If the group listing is empty, the PEP MUST NOT call the PDP (see Evaluations batching).

```mermaid
sequenceDiagram
  participant Client
  participant Catalog as tdf-iroh-s3 catalog PEP
  participant IdP as identity.arkavo.net cose-keys
  participant Fac as arks AuthZEN facade
  participant OTDF as OpenTDF GetDecisionMultiResource

  Client->>Catalog: GET /catalog/{group}<br/>Authorization: Bearer <PE CWT><br/>X-Entity-Token: <DeviceCheck CWT> (0..N, repeated header)
  Catalog->>IdP: fetch/cache COSE_Key Set
  Catalog->>Catalog: union verify PE + each device CWT<br/>each device.aud MUST be arkavo:devicecheck<br/>subject_id_bind(device.sub) MUST equal subject_id_bind(PE.sub)
  Catalog->>Catalog: append context.environment<br/>(node region, never from client)
  alt zero catalog items
    Catalog-->>Client: items=[] entitled omitted; no PDP call
  else one or more items
    Catalog->>Fac: POST /access/v1/evaluations<br/>Authorization: Bearer <service CWT><br/>X-Request-ID: <uuid>
    Note over Catalog,Fac: subject from PE; context.device or context.devices; context.environment
    Fac->>Fac: verify service CWT (iss + service-account; no single aud)
    Fac->>OTDF: GetDecisionMultiResource claims-mode
    OTDF-->>Fac: resourceDecisions PERMIT/DENY
    Fac-->>Catalog: evaluations[] decision true/false<br/>X-Request-ID echoed
    Catalog-->>Client: items[].entitled true/false
  end
```

### Sequence: MCP `tools/call` (v1 consumer)

Load: one `evaluation` per tool call. Latency budget: tens of milliseconds with a TTL-aware cache. Cache TTL MUST be derived from `$token.exp` (not JWT `.` splitting). PEP→facade timeout: **5s** (existing `AuthorizationConfig.timeout`).

v1 MCP (PR 4) implements **hardcoded** default mappings for methods `tools/call` and `tools/list` (no CEL), plus pass-through `ping` / `notifications/*`. Full CEL + `resources/*` / `prompts/*` / declared `x-authzen-mapping` are PR 4b. Do not ship a window where method `tools/list` is fail-closed-unknown.

```mermaid
sequenceDiagram
  participant Agent as MCP client
  participant PEP as arkavo-edge COAZ-MCP PEP
  participant IdP as identity.arkavo.net token + cose-keys
  participant Fac as arks AuthZEN facade
  participant OTDF as OpenTDF GetDecision

  Agent->>PEP: tools/call name=git_commit<br/>Authorization: Bearer <user CWT>
  PEP->>PEP: union-verify user CWT → $token (no cnf)<br/>ANTHROPIC_API_KEY MUST NOT be the subject credential
  PEP->>IdP: client_credentials (cached) → service CWT
  PEP->>PEP: hardcoded tools/call mapping<br/>subject.id == $token.sub
  PEP->>Fac: POST /access/v1/evaluation<br/>Authorization: Bearer <service CWT>
  Fac->>OTDF: GetDecision claims-mode<br/>action execute_tool<br/>FQN …/mcp-tool/value/git_commit
  OTDF-->>Fac: nested ResourceDecision
  Fac-->>PEP: decision true/false
  alt permit and no required obligations
    PEP-->>Agent: tool result
  else deny / mapping error / PDP error / required obligations
    PEP-->>Agent: JSON-RPC error -32001 / -32602 / -32603
  end
```

### Sequence: token issuance (later phase, not v1)

`authnz-rs` becomes an AuthZEN PEP *before* `cwt::mint`. Resource Search MAY fill `arkavo_roles` / `arkavo_entitlements`; search MUST NOT influence whether the token is issued.

**Action names (profile, later phase):** this profile uses AuthZEN `action.name = issue:cwt:{grant}` because the issued token is CWT. That is **intentionally divergent** from `draft-gazitt-oauth-authzen-issuance-00`, which uses `issue:access_token:{grant}` / `issue:{token-type}:{grant}`. A later issuance PEP MUST NOT claim conformance to that draft's action registry. The OpenTDF action name MUST be the charset-safe lowercase slug `issue_cwt_{grant}` (e.g. `issue_cwt_authorization_code`); the facade MUST NOT pass `issue:cwt:…` through unchanged (see Action Registry and OpenTDF identifier charset).

Phase 6 Resource Search only returns `resource.type=attribute_value`. Filling `arkavo_roles` / `arkavo_entitlements` from Search therefore requires a later classifier or a dedicated issuance-time mapping; it is **not** in phase 6.

```mermaid
sequenceDiagram
  participant RP
  participant AS as authnz-rs (AS = PEP)
  participant Fac as arks AuthZEN facade
  participant OTDF as OpenTDF v2

  RP->>AS: POST /oauth/token (authorization_code or client_credentials)
  AS->>AS: authenticate client / user
  AS->>Fac: evaluation action=issue:cwt:authorization_code<br/>resource type=audience
  Fac->>OTDF: GetDecision action=issue_cwt_authorization_code
  OTDF-->>Fac: permit/deny
  Fac-->>AS: decision
  alt deny
    AS-->>RP: access_denied
  else permit
    AS->>AS: cwt::mint(ArkavoClaims)
    AS-->>RP: access_token = base64url(CWT)
  end
```

### CWT Subject Profile (normative)

#### Transport of CWT

Arkavo CWTs are COSE_Sign1 (ES256) wrapped in CBOR tag #6.61 (`0xD8 0x3D`), as minted by `authnz-rs/src/cwt.rs::mint`. On HTTP they are **unpadded base64url** in:

- `Authorization: Bearer <b64url>` (RFC 6750; the token is opaque to RFC 6750). On the **caller's** request to a PEP this is the **subject CWT**. On the PEP→facade AuthZEN request this MUST be the **service CWT**.
- `X-Auth-Token: <b64url>` (Arkavo native flows; subject CWT)
- `X-Entity-Token: <b64url>` (one DeviceCheck CWT per header). Catalog clients MAY send **zero, one, or many** by **repeating** the header (HTTP `HeaderMap::get_all("x-entity-token")` in `catalog_api.rs`). Do **not** comma-combine tokens in one header. Header order is the device NPE order.

Discovery of verification keys: `GET {iss}/.well-known/cose-keys` (`Content-Type: application/cose-key-set+cbor`), advertised as `cose_keys_uri` on `/.well-known/openid-configuration` together with `access_token_format: "application/cwt"` (`authnz-rs/src/oidc.rs`). The physical key is P-256. `kid` is the RFC 7638 JWK thumbprint: **raw 32-byte SHA-256** in the COSE_Key / CWT protected header (label 4); **base64url of those same bytes** in JWKS (`authnz-rs/src/main.rs` around the `cwt_kid` construction).

#### PEP verification algorithm (union — stricter than either existing verifier)

`authnz-rs/src/cwt.rs::verify` and `tdf-iroh-s3` `CwtVerifier::verify` are **not** the same. This profile's algorithm is the **union** (stricter). Existing catalog/MCP code MUST be extended to this algorithm; implementers MUST NOT copy one function and call the profile a match.

A PEP MUST verify a subject CWT before populating `$token`, `subject`, or `context.device`.

1. Decode unpadded base64url to bytes (`cwt::decode_from_header`).
2. Reject unless bytes begin with `0xD8 0x3D`. Untagged COSE_Sign1 is rejected (`strip_cwt_tag`).
3. Parse COSE_Sign1. `alg` MUST be ES256 (COSE -7) in the **protected** header. Missing protected `alg` is `UnsupportedAlg`.
4. Protected `kid` MUST be non-empty **(profile; stricter than `cwt::verify`, which takes the key from the caller)**. Resolve the P-256 key from the cached COSE key set. On unknown `kid`, refetch at most once per 60s (`KEY_REFRESH_MIN_INTERVAL` in `auth.rs`); still-unknown → reject. Fetch timeout 10s.
5. Verify the ES256 signature over the COSE to-be-signed bytes with empty external AAD (`b""`).
6. Decode the payload as a CBOR map. Duplicate keys (integer or text) MUST be rejected **(profile; catalog `parse_claims` does not do this today)**.
7. Require claims: `iss`(1), `sub`(2), `aud`(3), `exp`(4), `iat`(6), `cti`(7). `cnf`(8) is OPTIONAL on user tokens; REQUIRED on DeviceCheck assertion tokens used as `context.device` **(profile; catalog does not require `aud`/`cti`/`cnf` today)**.
8. `iss` MUST match the configured issuer (production: `https://identity.arkavo.net` or `OIDC_ISSUER`).
9. For a **subject (PE) CWT**, `aud` MUST contain a configured audience for that PEP (`arkavo`, RP `client_id`, and/or `OIDC_PLATFORM_AUDIENCE`). For a **DeviceCheck CWT**, `aud` MUST equal `arkavo:devicecheck` **(new catalog behavior, not current)**. For a **service CWT** presented to the facade, see Two tokens — the facade does **not** require a single `aud`.
10. Reject if `iat > exp`. Reject if `exp <= now - 60`. Reject if `iat > now + 60`.
11. Algorithm confusion: only ES256. No `alg=none` equivalent.

A failed verification is an authentication failure at the PEP (HTTP 401 to the PEP's caller, or JSON-RPC `-32602` for MCP). The PEP MUST NOT call the PDP with an unverified `$token`.

`cnf` is proof-of-possession / credential-binding **metadata**. It is **omitted from `$token`**. The PEP MUST NOT copy `cnf` into `subject.properties`. A future PoP-enforcing PEP MAY set `context.device.pop_verified: true` **after** a successful PoP check; until then PDPs MUST ignore `cnf` for permit/deny. Device `kid` is copied onto `context.device.kid` (not into `$token`) from `cnf.kid` after verification.

#### `$token` map (text names after decode)

Integer CWT labels MUST be exposed under their RFC 8392 / RFC 8747 **text names**. The `$token` object MUST NOT contain integer keys. This is so CEL `$token.sub` works.

**`cnf` is omitted from `$token` (profile).** A declared mapping MUST NOT read `$token.cnf`. Device key identity belongs in `context.device.kid` after a dedicated DeviceCheck verify.

| CWT label | `$token` key | JSON type | Minimum decoder |
|---:|---|---|---|
| 1 | `iss` | string | REQUIRED |
| 2 | `sub` | string | REQUIRED. Format is **per-token-kind** (verified in `authnz-rs`): DeviceCheck assertion tokens (`mint_assertion_token`) and WebAuthn auth/registration tokens mint a **bare UUID**; OIDC access tokens mint `arkavo:{uuid}`; service CWTs mint `client:{CLIENT_ID}`; Apple-linked tokens MAY mint `apple:{APPLE_SUB}`. Device↔PE binding uses `subject_id_bind`, not raw string equality. |
| 3 | `aud` | string or array of strings | REQUIRED. Single tstr → string; tstr[] → JSON array. **Multi-aud is the normal production shape** when `OIDC_PLATFORM_AUDIENCE` is set (`oidc.rs` `mint_access_token` emits `Audience::Multiple`). NEVER set COAZ `resource.id` to an array; MCP `tools/list` uses `AUTHZEN_MCP_RESOURCE_ID` (see COAZ-MCP override 4). |
| 4 | `exp` | number | REQUIRED. NumericDate (seconds). Cache TTL uses this. |
| 6 | `iat` | number | REQUIRED |
| 7 | `cti` | string | REQUIRED. unpadded base64url of the 16-byte bstr |
| 8 | — | — | **Omitted** from `$token` |
| `"idp"` | `idp` | string | OPTIONAL; emit if present |
| `"email"` | `email` | string | OPTIONAL |
| `"email_verified"` | `email_verified` | boolean | OPTIONAL |
| `"arkavo_account_id"` | `arkavo_account_id` | string | OPTIONAL |
| `"arkavo_roles"` | `arkavo_roles` | array of strings | OPTIONAL. Enrichment / PIP hint |
| `"arkavo_entitlements"` | `arkavo_entitlements` | array of strings | OPTIONAL. Enrichment / PIP hint |
| `"client_id"` | `client_id` | string | OPTIONAL. Not minted today. Additive future claim. |
| `"arkavo_patreon"` | `arkavo_patreon` | object | OPTIONAL. Shape below |

A **minimum decoder** MUST emit every REQUIRED key above and every OPTIONAL key that is present in the CWT, and MUST omit absent optionals and MUST omit `cnf`. Catalog `VerifiedClaims` today (`iss, sub, exp, iat, patreon_user_id, email, arkavo_patreon`) is **not** a minimum decoder. PR 4 and PR 5 MUST implement the minimum decoder (or share the PR 2 mapper). CEL optional selection is `$token.?email` (leading `$`).

Consumer `arkavo_patreon` (top-level `campaign_id` MUST be omitted):

```json
{
  "role": "consumer",
  "patreon_user_id": "12345678",
  "memberships": [
    {
      "campaign_id": "87654321",
      "patron_status": "active_patron",
      "tier_ids": ["111"],
      "tier_slugs": ["supporter"]
    }
  ],
  "verified_at": 1779996400,
  "cache_expires_at": 1780000000
}
```

Creator `arkavo_patreon` (top-level `campaign_id` is creator-only):

```json
{
  "role": "creator",
  "patreon_user_id": "99990001",
  "campaign_id": "87654321",
  "memberships": [],
  "verified_at": 1779996400,
  "cache_expires_at": 1780000000
}
```

Empty `memberships` MAY be omitted. `tier_slugs` MAY be omitted when empty.

#### Subject identifier bind (`subject_id_bind`) **(profile)**

DeviceCheck assertion tokens mint a bare-UUID `sub` (`authnz-rs/src/device_check.rs` `ArkavoClaims::devicecheck(..., &user_id.to_string(), ...)`). OIDC access tokens (the catalog PE CWT) mint `arkavo:{uuid}` (`oidc.rs` `format!("arkavo:{}", account_id)`). Raw string equality therefore rejects every real PE-CWT + DeviceCheck-CWT pair. Today's catalog already does `claims.sub != pe.sub` → HTTP 401 (`catalog_api.rs`); that comparison MUST change in PR 5.

Define:

```
subject_id_bind(s):
  if s starts with the exact prefix "arkavo:", return s with that single leading prefix removed
  else return s unchanged
```

This comparison MUST strip **only** the prefix `arkavo:`. It MUST NOT strip `apple:` or `client:`. Device `sub` binds to PE `sub` if and only if `subject_id_bind(device.sub) == subject_id_bind(subject.id)` (equivalently, `subject_id_bind(device.sub) == subject_id_bind(PE.sub)`).

Worked bind (the minted mismatch): PE `sub = "arkavo:550e8400-e29b-41d4-a716-446655440000"`, device `sub = "550e8400-e29b-41d4-a716-446655440000"` → both bind to `550e8400-e29b-41d4-a716-446655440000` → **bind**. The same PE with device `sub = "arkavo:550e8400-e29b-41d4-a716-446655440000"` also binds. `apple:{id}` does **not** bind to a bare `{id}`. `client:{id}` does **not** bind to a bare `{id}`.

The bind is a **comparison**. It MUST NOT rewrite claims: the PE entity keeps the PE CWT `sub` as minted; each device entity keeps the DeviceCheck CWT `sub` as minted.

PEP 401 uses this normalized comparison, not raw string equality. The catalog error string `"entity token subject does not match bearer subject"` is unchanged.

#### Subject, context, resource, action (SARC)

**Subject type** for this profile: `identity` (the value used by COAZ-MCP default mappings; AuthZEN 1.0 does not define a subject-type registry — `subject.type` is free-form per AuthZEN §5.1). PEPs MUST use this type for PE principals. Do not use `user` or `client` at the catalog/MCP PEPs; those types belong to the later Token Issuance profile.

| AuthZEN field | Source | Trust |
|---|---|---|
| `subject.type` | literal `"identity"` | PEP-supplied constant |
| `subject.id` | PE CWT `sub` | **Trust-anchored.** PEP-verified. COAZ verification rule: resolved value MUST equal `$token.sub`. |
| `subject.properties.email` | CWT `email` | Verified claim (PIP) |
| `subject.properties.email_verified` | CWT `email_verified` | Verified claim |
| `subject.properties.idp` | CWT `idp` | Verified claim |
| `subject.properties.arkavo_account_id` | CWT `arkavo_account_id` | Verified claim |
| `subject.properties.arkavo_roles` | CWT `arkavo_roles` | Hint only — MUST NOT be the sole permit condition |
| `subject.properties.arkavo_entitlements` | CWT `arkavo_entitlements` | Hint only |
| `subject.properties.arkavo_patreon` | CWT `arkavo_patreon` | Verified membership snapshot; still not a live decision |
| `subject.properties.iss` | CWT `iss` | For ERS issuer pinning |
| `context.device` / `context.devices` | 0..N DeviceCheck CWTs | v1 **wire + verification** (no v1 policy effect; see Goals and Key Decision 25). Schema below. Each `aud` MUST be `arkavo:devicecheck`. Each `sub` MUST bind to `subject.id` via `subject_id_bind`. |
| `context.environment` | PEP node config | **MUST NOT** come from the client. Catalog uses `catalog.authz.environment_region`. Facade allowlist `{region}` plus optional `kind`; every other key is dropped. |
| `context.agent` | `$token.?client_id`, else the substring after `client:` when `sub` starts with `client:` (e.g. `client:catalog-node` → `catalog-node`), else if `$token.aud` (string or array) contains **exactly one** member that is neither `arkavo` nor `arkavo:devicecheck` nor the configured platform audience (`OIDC_PLATFORM_AUDIENCE`), use that member; else omit | Agent identity for COAZ-MCP. Production OIDC tokens are commonly multi-aud. |
| `context.pep.fulfillable_obligation_fqns` **(profile)** | PEP capability list; free-form `context` per AuthZEN §5.5.1 | v1 PEPs MUST send `[]` or omit. On Access Evaluation this is top-level `context`; on Access Evaluations it is top-level `context` (shared) unless an entry overrides. |
| `action.name` | see Action Registry | |
| `resource` | see Resource translation | |

#### `context.device` / `context.environment` JSON (v1)

Production `build_chain` currently emits NPE claims `{ "sub", "iss" }` only. `kind` appears only in a unit-test fixture and is **not** a CWT claim — v1 DROPS `kind`. `kid` and `aud` are not in `VerifiedClaims` today; **PR 5 MUST extend `CwtVerifier`** so the catalog can populate this schema. Do not invent `kind`.

Required/optional object for one device:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "iss": "https://identity.arkavo.net",
  "aud": "arkavo:devicecheck",
  "kid": "YWxwaGEtZGV2aWNlLWtpZA"
}
```

This device object **binds** to PE `subject.id = "arkavo:550e8400-e29b-41d4-a716-446655440000"` via `subject_id_bind`. The device `sub` is the minted DeviceCheck value (bare UUID), not a copy of the PE `sub`.

| Field | Required | Source |
|---|---|---|
| `sub` | REQUIRED | Device CWT `sub` as minted. MUST bind to `subject.id` via `subject_id_bind`. |
| `iss` | REQUIRED | Device CWT `iss`. |
| `aud` | REQUIRED | Device CWT `aud`. PEP MUST reject unless `arkavo:devicecheck`. |
| `kid` | REQUIRED | unpadded base64url of `cnf.kid`. DeviceCheck tokens without `cnf` are rejected. |

Facade device-object allowlist **(profile):** `{sub, iss, aud, kid}` only. Unknown keys are dropped silently; they MUST NOT be forwarded as OpenTDF claims.

Missing any REQUIRED device field (`sub` / `iss` / `aud` / `kid`) → HTTP **400** (malformed SARC). Present-but-wrong `aud` (not `arkavo:devicecheck`) or present-but-unbound `sub` (`subject_id_bind` mismatch) → that evaluation is **deny-closed** (`decision: false`, `context.error`). Do not treat a missing REQUIRED field as deny-closed.

`context.environment` (node-asserted only):

```json
{
  "region": "us-east-1"
}
```

`kind: "environment"` MAY be included by the PEP for readability; the facade MUST NOT require it. Facade environment-entity allowlist **(profile):** `{region}` plus optional `kind` only. The facade MUST drop every other key silently (including `sub`, `email`, `arkavo_patreon`, `aud`, `kid`, `patreon_access_token`, `patreon_user_id`, `preferred_username`). The facade MUST NOT use a denylist: those extra keys are exactly the Patreon ERS lookup order (`authz.rs`) and would become an identity-injection vector if the platform ever stopped skipping environment entities.

**`device` vs `devices` (v1 wire + verification; no v1 policy effect):** a request MAY attach **0, 1, or N** DeviceCheck CWTs (e.g. phone + watch). `context.devices` is a specified v1 shape, not forward-compat. Device NPEs are `CATEGORY_ENVIRONMENT` and the platform sets `skipEnvironmentEntities=true`, so v1 permit/deny does not change when devices are present (Goals, Key Decision 25).

| DeviceCheck CWTs | AuthZEN context | Facade chain |
|---|---|---|
| 0 | Omit both `context.device` and `context.devices` (MCP `tools/call` / `tools/list`, PE-only catalog) | PE ± environment; no device NPEs |
| 1 | `context.device` (object) **or** `context.devices` with one element | One device NPE |
| N ≥ 2 | `context.devices` (array of N objects), **not** `context.device` | One device NPE per element, **array order** |

If both `context.device` and `context.devices` are present, HTTP 400. If `context.devices` is present and empty, HTTP 400. Sending both a singleton object and an array is 400 even when they would describe the same device.

Example — two devices (phone + watch). Both device `sub` values are the minted bare UUID; they bind to PE `subject.id = "arkavo:550e8400-e29b-41d4-a716-446655440000"` via `subject_id_bind`:

```json
{
  "devices": [
    {
      "sub": "550e8400-e29b-41d4-a716-446655440000",
      "iss": "https://identity.arkavo.net",
      "aud": "arkavo:devicecheck",
      "kid": "cGhvbmUta2lk"
    },
    {
      "sub": "550e8400-e29b-41d4-a716-446655440000",
      "iss": "https://identity.arkavo.net",
      "aud": "arkavo:devicecheck",
      "kid": "d2F0Y2gta2lk"
    }
  ]
}
```

#### Two tokens (MUST NOT be confused)

| Token | Where it appears | Who it authenticates | Typical `sub` / `aud` |
|---|---|---|---|
| **Service CWT** | AuthZEN `Authorization: Bearer` PEP→facade | The PEP as an OAuth client | `sub=client:catalog-node` (or `client:{mcp-client-id}`), `aud={that client_id}` plus the configured platform audience when `OIDC_PLATFORM_AUDIENCE` is set (RFC 8707-style; production is commonly `Audience::Multiple`), `arkavo_roles` contains `service-account`, no `cnf` |
| **Subject (PE) CWT** | PEP's caller. **Not** the AuthZEN Authorization header | The human or user-agent | OIDC access: `sub=arkavo:{uuid}`; WebAuthn auth token: bare UUID `sub`, `aud=arkavo`; Apple-linked: `sub=apple:…`. Production OIDC access is commonly multi-aud (`client_id` + `OIDC_PLATFORM_AUDIENCE`). |
| **Device CWT** | Repeated `X-Entity-Token`; `context.device` or `context.devices` after verification | Each attested device bound to the same PE via `subject_id_bind` | `sub` = bare UUID (as minted), `aud=arkavo:devicecheck`, `cnf.kid` = App Attest key ID |

**MCP PEP (normative, same as catalog).** `arkavo-edge` today sends the **user** token as `Authorization: Bearer {token}` to ERS `CreateEntityChainsFromTokens`; `GetDecision*` requests are sent unauthenticated. That MUST stop. The MCP PEP MUST obtain a service CWT via `client_credentials`, refresh ≥60s before expiry, and present **only** that token to the facade:

| Config | Example | Notes |
|---|---|---|
| `AUTHZEN_TOKEN_URL` | `https://identity.arkavo.net/oauth/token` | Same as catalog `catalog.authz.token_url` |
| `AUTHZEN_CLIENT_ID` | `mcp-edge` | Confidential OIDC client |
| `AUTHZEN_CLIENT_SECRET` | env only | Do not commit. Catalog uses `CATALOG_AUTHZ_CLIENT_SECRET`. |
| `AUTHZEN_PDP_URL` | `https://kas.arkavo.net` | Discovery base; PEP then GET `/.well-known/authzen-configuration` |
| `AUTHZEN_MCP_RESOURCE_ID` | `https://mcp.arkavo.net` | RFC 8707 resource identifier of **this** MCP server. `tools/list` requires this value to appear in `$token.aud` (string or array member) or the PEP raises a mapping error (`-32602`). |
| `AUTHZEN_MCP_SERVER_SLUG` | `mcp_arkavo_net` | Charset-safe OpenTDF attribute value for `mcp_server`. If unset, derive a lowercase slug from `AUTHZEN_MCP_RESOURCE_ID` that matches `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$` (e.g. `https://mcp.arkavo.net` → `mcp_arkavo_net`). AuthZEN `resource.id` for `type=mcp_server` is this slug, **not** a percent-encoded URL. |
| `OIDC_PLATFORM_AUDIENCE` | `https://platform.arkavo.net` | Configured platform audience to exclude from the `context.agent` aud-fallback (and the extra member on production service/OIDC CWTs). |

**Facade PEP authentication (normative):**

1. Missing/malformed Bearer, failed signature, bad `iss`, expired → HTTP **401**.
2. `iss` MUST equal the configured issuer.
3. `sub` MUST start with `client:` **and** `arkavo_roles` MUST contain `service-account`. Otherwise HTTP **401** (treated as not a PEP, not a subject deny).
4. The facade MUST **not** require a single `aud`. Service CWTs are minted by `handle_client_credentials_grant` calling `mint_access_token(..., &client.client_id, ...)`, which appends `OIDC_PLATFORM_AUDIENCE` when set — so the production shape is `aud = [client_id, platform_audience]`. If `aud` is a single string, it MUST equal the client id (the `sub` without the `client:` prefix). If `aud` is an array, that client id MUST be a member.
5. Optional allowlist `AUTHZEN_PEP_CLIENT_IDS` (comma-separated). If **set**, the client id is `sub` without the `client:` prefix (and `aud` MUST contain that id as above). If the client id is not in the list → HTTP **403**. If the allowlist is **unset**, any valid service-account CWT is accepted and HTTP 403 is not used.
6. User CWTs used as AuthZEN Bearer → HTTP **401**.

The facade MUST treat SARC `subject` as PIP data asserted by an authenticated PEP. It MUST NOT take the AuthZEN Bearer as the subject. It MUST NOT send the subject CWT to OpenTDF `EntityIdentifier.token`.

**PR 3 entry criterion (not an open question):** before merging the facade, spike whether `platform.arkavo.net` accepts a service CWT as the Connect caller credential. Catalog already presents a service CWT to platform today; confirm the deployed build verifies CWT (not JWT-only). If platform is JWT-only, PR 3 MUST implement mint/exchange of a platform-acceptable caller credential inside `arks` and document it. PEPs still send service CWT to the facade either way.

Service CWTs are minted by `handle_client_credentials_grant` in `authnz-rs/src/oidc.rs` (TTL currently 1h). Static long-lived tokens MUST NOT be used in production.

#### PE + device NPE + environment NPE → one SARC

OpenTDF models a chain of PE + NPEs. `authorization.proto`'s `EntityIdentifier.entity_chain` requires `entities.size() > 0` only (`has(this.entities) && this.entities.size() > 0`); `entity.proto`'s `EntityChain` message has no such validation. A 1–10 cap is **not** verified in this workspace and is **not** claimed as an OpenTDF limit. **(profile)** This facade caps the reconstructed chain at **8** entities: 1 PE + D devices + E environment, where D ≥ 0, E is 0 or 1, and `1 + D + E ≤ 8`. With a typical environment entity, **at most 6 devices**. Excess DeviceCheck CWTs → HTTP 400 (catalog PEP SHOULD fail the caller request with 400 before calling the facade).

AuthZEN 1.0 has **one** subject plus free-form context. This profile flattens the chain as follows — this is the only mapping the facade implements in v1.

**PEP construction (catalog; current `build_chain` is insufficient — PR 5 extends it):**

1. If no `Authorization: Bearer` and no `X-Entity-Token`: anonymous. Catalog lists with `entitled:false` and MUST NOT call the PDP. MCP has no anonymous path: missing token is 401 / `-32602`.
2. If `X-Entity-Token` without Bearer: HTTP 401 `"entity tokens require a bearer subject"`.
3. Union-verify PE CWT → `subject` + `$token`-class claims for properties.
4. Collect **every** `X-Entity-Token` value via repeated headers (`get_all`), in header order. Union-verify each DeviceCheck CWT (`aud=arkavo:devicecheck`, `cnf` present); if any `subject_id_bind(device.sub) != subject_id_bind(PE.sub)`, HTTP 401 `"entity token subject does not match bearer subject"`. PR 5 MUST replace today's `claims.sub != pe.sub` exact-string check (`catalog_api.rs`) with this bind. Zero tokens: omit both `context.device` and `context.devices`. One token: MAY set `context.device` **or** `context.devices` with one element. Two or more: MUST set `context.devices` (array, header order) and MUST NOT set `context.device`. If `1 + D + (1 if environment else 0) > 8`, HTTP 400.
5. Append `context.environment` from **node configuration** (`environment_region`). Clients MUST NOT supply this.
6. If the catalog listing has **zero items**, MUST NOT call the PDP; return `items: []` with `decision` `"evaluated"` or `"anonymous"` as appropriate and no `entitled: true`.

**Facade: SARC → OpenTDF `entityChain` (claims mode only).** Order is **PE, then each device NPE (header / `devices` array order), then environment**. One NPE per device object. Every device `sub` MUST bind to `subject.id` via `subject_id_bind`; else that evaluation is deny-closed (`decision: false`, `context.error`). DeviceCheck `aud` MUST be `arkavo:devicecheck` (PEP already verified; facade MUST still deny-closed if `aud` is **present** and not that value). If any REQUIRED device field is **missing**, the request is HTTP **400** (malformed SARC), not deny-closed.

Device entity `value` is `{sub, iss, aud, kid}` only — no `kind`; unknown keys dropped. Environment `value` is `{region}` (and optional `kind`) only; unknown keys dropped.

Rules:

- `e0.value.sub` MUST equal `subject.id` (the PE CWT `sub` as minted; not rewritten by `subject_id_bind`).
- `e0.value.iss` from `subject.properties.iss` if present, else omitted.
- `e0.value.email` from `subject.properties.email`.
- `e0.value.arkavo_patreon` copied from `subject.properties.arkavo_patreon` when present.
- `e0.value.patreon_user_id` from `subject.properties.arkavo_patreon.patreon_user_id` if present. Patreon ERS lookup order is `patreon_access_token` → `patreon_user_id` → `email` → `preferred_username` (`authz.rs` module docs). This facade never has a Patreon access token.
- Device entity `value.sub` is the DeviceCheck CWT `sub` as minted (bare UUID today). It MUST bind to `subject.id` via `subject_id_bind`; else that evaluation is deny-closed (`decision: false`, `context.error`).
- Environment entity: allowlist `{region}` plus optional `kind`; drop every other key.
- OpenTDF currently sets `skipEnvironmentEntities=true` (`authz.rs` comments). Device NPEs are `CATEGORY_ENVIRONMENT` (OpenTDF `Entity.Category` has no device category). Device and environment entities are still forwarded. **v1 decisions are unaffected.** PEPs MUST NOT assume devices or environment affect v1 permit/deny. The named future: the platform stops skipping, or a `kind` / `npe_type` claim distinguishes devices from environment.
- **Token mode is forbidden** for Arkavo CWTs. The facade MUST NOT populate `entityIdentifier.token`.

#### Claims profile rule (CWT)

Even though {{CLAIMS}} is written for JWT access tokens, this profile applies the same rule to CWT:

- `arkavo_roles`, `arkavo_entitlements`, and `arkavo_patreon` MAY be copied into `subject.properties` as PIP attributes / hints.
- A PEP MUST NOT permit an operation solely because those claims are present. The PEP MUST call the PDP. Air-gapped **KAS unwrap** uses local KAS policy, not this JSON facade.
- Issuance of a CWT is independent of Resource Search results (later phase). Search MUST NOT gate `cwt::mint`.

`arkavo_patreon` is a materialized membership snapshot (`verified_at` / `cache_expires_at`). Downstream policy SHOULD treat `cache_expires_at < now` as stale enrichment and rely on ERS/PDP, not the snapshot, for the live decision.

### AuthZEN facade over OpenTDF v2

#### Endpoint mapping

| AuthZEN 1.0 | Default path | OpenTDF v2 | v1 |
|---|---|---|---|
| Access Evaluation | `POST /access/v1/evaluation` | `GetDecision` | **Required (profile)** |
| Access Evaluations | `POST /access/v1/evaluations` | `GetDecisionMultiResource` if shared subject+action; else `GetDecisionBulk` | **Required (profile)** (OPTIONAL in AuthZEN metadata; this profile REQUIRES the endpoint) |
| Resource Search | `POST /access/v1/search/resource` | `GetEntitlements` (approximate; `attribute_value` only) | Phase 6 |
| Subject Search | `POST /access/v1/search/subject` | none | Not offered |
| Action Search | `POST /access/v1/search/action` | none | Not offered |
| Discovery | `GET /.well-known/authzen-configuration` | n/a | **Required (profile)** (RECOMMENDED in AuthZEN §9) |

OpenTDF ConnectRPC-JSON URLs the facade calls:

- `{OPENTDF_PLATFORM_URL}/authorization.v2.AuthorizationService/GetDecision`
- `…/GetDecisionMultiResource`
- `…/GetDecisionBulk`
- `…/GetEntitlements` (later)

The facade SHOULD send `Content-Type: application/json` and `Connect-Protocol-Version: 1`. Forward `X-Request-ID` when present. Upstream timeout: **10 seconds (profile)**.

#### Decision translation

`GetDecisionResponse` is a **nested** `ResourceDecision` (`authorization.proto`). The facade MUST read `response.decision.decision` and `response.decision.requiredObligations`, **not** a top-level enum string (the edge crate's `GetDecisionResponse` is wrong for proto-JSON). `GetDecisionMultiResourceResponse` uses `resourceDecisions[]` (what `tdf-iroh-s3` already parses).

| OpenTDF proto-JSON | AuthZEN |
|---|---|
| `decision.decision` / `resourceDecisions[i].decision` == `"DECISION_PERMIT"` | `{ "decision": true }` |
| `"DECISION_DENY"`, `"DECISION_UNSPECIFIED"`, missing | `{ "decision": false }` |
| RPC / HTTP failure | HTTP **500** to the PEP for the whole request (AuthZEN error table; this profile does not add 502). For `execute_all` evaluations, the facade MAY instead return 200 with each remaining entry `{ "decision": false, "context": { "error": { "status": 500, "message": "upstream" } } }` if some entries already completed; PEPs fail closed either way. |
| `requiredObligations: ["https://…/obl/…"]` | `context.obligations.required` **(profile)** — free-form `context` per AuthZEN §5.5.1. On Access Evaluations this is the **per-evaluation** `context`; on Access Evaluation it is the **top-level** `context`. |
| (none) | `context.evaluation_id` **(profile)** — opaque string, algorithm below; free-form `context` per AuthZEN §5.5.1. Per-evaluation for `/evaluations`; top-level for `/evaluation`. |

`fulfillableObligationFqns` is a **top-level** field on `GetDecisionRequest` / `GetDecisionMultiResourceRequest` / each Bulk member. Map from `context.pep.fulfillable_obligation_fqns` **(profile)**. v1 PEPs send `[]`.

**`evaluation_id` algorithm (profile):** let `rid` be the inbound `X-Request-ID` if non-empty, else a newly generated UUID. For Access Evaluation, `evaluation_id = rid` on the top-level `context`. For Access Evaluations index `i` (0-based), `evaluation_id = rid + ":" + i` on that entry's `context`. This string is **not** necessarily a UUID. AuthZEN 1.0 does not define `evaluation_id`.

**v1 obligations:** PEPs MUST send empty fulfillable lists. Sending an empty `fulfillableObligationFqns` causes the platform to **DENY** any resource with a triggered obligation (`authorization.proto`: "If entitled, checks obligation policy: fulfillable obligations must satisfy all triggered."). Operators MUST confirm no production attribute value carries an obligation trigger before flipping catalog `protocol = "authzen"`. The shadow phase in Rollout step 5 SHOULD diff exactly this (v2 PERMIT-with-obligations vs AuthZEN DENY). If `context.obligations.required` is **non-empty**, the PEP MUST fail closed **even when `decision` is true** (treat as denial / `-32001`). Do not ignore required FQNs. A PERMIT with required obligations and empty fulfillable should not happen if platform enforces fulfillable; if it does, fail closed.

HTTP semantics (AuthZEN §10.1.2), plus this profile's 403:

| HTTP | Meaning |
|---|---|
| 200 + `decision: true/false` | Evaluation succeeded |
| 400 | Malformed SARC (including a device object missing REQUIRED `sub`/`iss`/`aud`/`kid`), Arkavo `evaluations.length > 500`, both `device` and `devices`, empty `devices` array, reconstructed chain `1+D+E > 8`, empty `evaluations` without top-level `resource`, PEP-supplied `attribute_value_fqns` on `resource.type` in `{tool, mcp_server}`, AuthZEN action whose mapped OpenTDF name violates the identifier charset, `mcp_server`/`tool` `resource.id` that is not charset-safe. Omitting **both** `device` and `devices` is valid. Present-but-wrong device `aud`/`sub` is **not** 400 — it is deny-closed. |
| 401 | PEP service CWT missing/invalid — **not** a subject deny |
| 403 | Service CWT valid **and** `AUTHZEN_PEP_CLIENT_IDS` is set **and** this client is not on the list |
| 500 | Facade or upstream failure |

#### Evaluations batching

- Default `options.evaluations_semantic` is `execute_all`.
- If every entry shares the same effective subject and action, the facade MUST use a single `GetDecisionMultiResource`. OpenTDF `Resource.ephemeralId` MUST equal AuthZEN `resource.id`. Response order MUST match the request `evaluations` array. OpenTDF `resources` has `min_items: 1`.
- Otherwise group into `GetDecisionBulk` of `GetDecisionMultiResourceRequest`s.
- `deny_on_first_deny` / `permit_on_first_permit`: the facade SHOULD honor them across groups; within one MultiResource call the platform returns all results and the facade filters. Catalog uses `execute_all`.
- Maximum `evaluations.length` is **500 (Arkavo profile limit, not AuthZEN)**. Larger → 400.
- **Empty evaluations (profile):** a PEP MUST NOT call the PDP with zero resources. Catalog: empty group listing → no AuthZEN request. Facade: `evaluations` absent or empty **and** no top-level `resource` → HTTP **400**. AuthZEN §7.1 says such a request behaves as a single Access Evaluation; AuthZEN §7.1.1 requires `subject`, `action`, and `resource` for a valid evaluation. A request with no evaluations and no top-level `resource` therefore has no valid evaluation and is rejected as malformed (400). Do **not** invent a 200 `{ "evaluations": [] }` exception.

#### OpenTDF identifier charset **(profile)**

OpenTDF `CreateAttributeValueRequest.value` and `CreateActionRequest.name` are constrained (fetched from `opentdf/platform` `main` on 2026-08-26) to:

```
^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$
```

with lowercase normalization of the stored name/value. Dots, colons, slashes, and `%` are not permitted. PR 3's entry spike MUST confirm this regex against the **deployed** `platform.arkavo.net` (not only `main`). This profile adopts the constraint as a MUST either way: `git.commit`, percent-encoded URLs, `issue:cwt:{grant}`, and slash methods are the failure mode if the deployed platform matches, and slugs are cheaper to spec now than to unwind later.

Consequences **(profile):**

- Attribute **values** and OpenTDF **action names** the facade emits MUST match that regex after lowercasing.
- AuthZEN `action.name` MAY still contain `/` or `:` (`tools/list`, `issue:cwt:authorization_code`). The OpenTDF column MUST be a charset-safe lowercase slug (`tools_list`, `issue_cwt_authorization_code`).
- The facade MUST lowercase the mapped OpenTDF action name.
- The facade MUST NOT pass through an unmapped AuthZEN action whose OpenTDF name would violate the charset → HTTP **400**.
- `mcp_server` `resource.id` is a configured slug (`AUTHZEN_MCP_SERVER_SLUG`, or a slug form of `AUTHZEN_MCP_RESOURCE_ID`), **not** a percent-encoded URL. Percent-encoding (uppercase HEXDIG `%3A`/`%2F`, or encode-only-slash) is a **rejected alternative** and MUST NOT be used.
- Tool names that contain `.` map to underscore slugs (`git.commit` → `git_commit`). In-tree `McpToolMapping::tool_to_resource` currently emits `git.commit` and MUST change in PR 4.

#### Resource translation

OpenTDF `Resource` is either `attributeValues.fqns` or `registeredResourceValueFqn`.

| AuthZEN `resource.type` | OpenTDF Resource |
|---|---|
| `catalog_item` | `ephemeralId = resource.id` (BLAKE3 hex). `attributeValues.fqns = resource.properties.attribute_value_fqns`. If FQNs missing/empty → that entry deny-closed. PEP-supplied FQNs are authoritative. |
| `tool` | Facade **derives** the FQN set itself from `resource.id` via the alias table. `ephemeralId = resource.id`. If `properties.attribute_value_fqns` is present → HTTP **400** (do not evaluate). |
| `mcp_server` | Facade **derives** FQN `https://arkavo.net/attr/mcp-server/value/{slug}` where `{slug}` is `resource.id` (already a charset-safe slug). If `properties.attribute_value_fqns` is present → HTTP **400**. If `resource.id` does not match the identifier charset → HTTP **400**. |
| `tdf` | `attributeValues.fqns` from `properties.attribute_value_fqns`; `ephemeralId = resource.id`. PEP-supplied FQNs are authoritative. |
| `kas` | Same as `tdf` for end-state examples; action mapped `rewrap`→`decrypt`. PEP-supplied FQNs are authoritative. |
| `attribute_value` | Search only (phase 6). |
| other, with `properties.attribute_value_fqns` | Use those FQNs. |
| other, no FQNs | 400 or per-entry deny with `context.error` |

For `resource.type` in `{tool, mcp_server}` the facade MUST derive the FQN set itself and MUST reject (HTTP **400**) PEP-supplied `attribute_value_fqns`. PEP-supplied FQNs remain authoritative only for `catalog_item` / `tdf` / `kas`, where the PEP legitimately owns the attribute set. When `AUTHZEN_PEP_CLIENT_IDS` is unset, any valid service-account CWT is an authenticated PEP; without this rule that PEP could substitute a low-sensitivity FQN while the log recorded a high-sensitivity `resource.id`.

**`mcp_server` slug (profile).** `resource.id` is `AUTHZEN_MCP_SERVER_SLUG` if set, else a documented slug form of `AUTHZEN_MCP_RESOURCE_ID` (e.g. `https://mcp.arkavo.net` → `mcp_arkavo_net`). FQN: `https://arkavo.net/attr/mcp-server/value/mcp_arkavo_net`. Percent-encoded URL values (`https%3A%2F%2Fmcp.arkavo.net`) MUST NOT be used.

**MCP tool FQN mapping — facade only.** PEPs send `resource.type = "tool"` and `resource.id = {tool name}` (underscores, not dots). **Only the facade** expands aliases. PEPs MUST NOT call `Resource::mcp_tool` (raw name, no rewrite) or keep a local copy of `McpToolMapping::tool_to_resource` (drift). Catalog sends `attribute_value_fqns` directly and never uses this table.

If `resource.id` contains `.`, the facade MUST map each `.` to `_` before building the FQN, then MUST reject (HTTP **400**) if the result does not match the identifier charset.

| `resource.id` (tool name) | Attribute value FQN |
|---|---|
| `filesystem_read` | `https://arkavo.net/attr/mcp-tool/value/filesystem_read` |
| `filesystem_write` | `https://arkavo.net/attr/mcp-tool/value/filesystem_write` |
| `git_commit` | `https://arkavo.net/attr/mcp-tool/value/git_commit` |
| `git_push` | `https://arkavo.net/attr/mcp-tool/value/git_push` |
| `device_tap` | `https://arkavo.net/attr/mcp-tool/value/device_management_tap` |
| `device_swipe` | `https://arkavo.net/attr/mcp-tool/value/device_management_swipe` |
| any other `{name}` including `list_tools` | `https://arkavo.net/attr/mcp-tool/value/{slug}` where `{slug}` is `{name}` with `.` → `_`, then charset-checked |

Rejected (in-tree today, MUST change in PR 4): `https://arkavo.net/attr/mcp-tool/value/git.commit`.

#### Action Registry

One OpenTDF action per AuthZEN `action.name`. AuthZEN names MAY contain `/` or `:`; OpenTDF names MUST be charset-safe and lowercase (see OpenTDF identifier charset). No unmapped pass-through of an illegal OpenTDF name.

| AuthZEN `action.name` | OpenTDF `action.name` | Used by |
|---|---|---|
| `read` | `read` | Catalog |
| `tools/call` | `execute_tool` | COAZ-MCP **method** `tools/call` (includes tool name `list_tools`) |
| `execute_tool` | `execute_tool` | Legacy; facade accepts as alias of `tools/call`'s OpenTDF action |
| `tools/list` | `tools_list` | COAZ-MCP **method** `tools/list` only. Resource MUST be `mcp_server`. Operators MUST provision the `tools_list` action; the facade MUST NOT send `tools/list` as the OpenTDF name. |
| `rewrap` | `decrypt` | KAS end-state |
| `decrypt` | `decrypt` | Direct |
| `issue:cwt:{grant}` | `issue_cwt_{grant}` (e.g. `issue_cwt_authorization_code`) | Later issuance PEP; AuthZEN name is **not** IETF `issue:access_token:{grant}`. OpenTDF name MUST NOT contain `:`. |
| any other | map `/` and `:` to `_`, lowercase, then charset-check | If the mapped name does not match `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$` → HTTP **400**. The facade MUST NOT forward an unmapped illegal name. |

#### Semantic gaps (honest)

1. **Search ≠ GetEntitlements.** Resource Search returns resources of a type that SHOULD later evaluate to permit. `GetEntitlements` returns `entitlements[]` of `{ ephemeralId, actionsPerAttributeValueFqn }` — entitled **attribute FQNs**, not catalog items. Phase 6 maps **only** `resource.type=attribute_value` (id = FQN). It will **not** replace `GET /catalog/{group}`. It will **not** implement `resource.type=role` or `entitlement` (no deterministic FQN classifier).
2. **Entity chain vs one subject.** NPEs go in **context**; the facade reconstructs the chain. `context.device` is not a second AuthZEN subject.
3. **Environment currently skipped** by OpenTDF decision flow (`skipEnvironmentEntities=true` in `authz.rs`). Device NPEs are also `CATEGORY_ENVIRONMENT`. Both are still forwarded. **v1 multi-device has no policy effect** — it is wire + verification only (Goals, Key Decision 25). Named future: platform stops skipping, or a `kind` / `npe_type` claim distinguishes devices.
4. **ERS token identifier is JWT.** Facade never uses it for Arkavo CWTs.
5. **arkavo-edge types.rs is a simplified subset.** Facade speaks proto-JSON as `tdf-iroh-s3` does. `GetDecisionResponse.decision` is nested `ResourceDecision`.
6. **Obligations.** `context.obligations.required`, `context.pep.fulfillable_obligation_fqns`, and `context.evaluation_id` are Arkavo **(profile)** keys in AuthZEN's free-form `context` (AuthZEN §5.5.1). They are not AuthZEN 1.0 fields. `context.obligations.required` is per-evaluation for `/evaluations` and top-level for `/evaluation`.

#### Resource Search approximation (phase 6 only)

Phase 6 implements Resource Search **only** when `resource.type == "attribute_value"`. `resource.id` in the request is omitted/ignored (AuthZEN). `GetEntitlementsRequest` has **no** action and **no** resource type; the facade calls `GetEntitlements` then **filters** `actionsPerAttributeValueFqn` to entries whose `actions[].name` equals the request `action.name`. **Filter by `action.name` before slicing pages.** Other `resource.type` values → HTTP 400 until a later classifier exists.

**Pagination (profile, AuthZEN §8.2):**

- Default `page.limit` is **100 (Arkavo profile)**. If the request omits `page.limit`, use 100.
- If the request includes `page.limit`, honor it **capped at 100**. `limit = 0` → HTTP 400.
- `page.next_token` is an **opaque** string. PEPs MUST NOT parse it. Empty string `""` means the end of the result set.
- After the first page, a continuation request MUST send a `page` object whose `token` equals the previous `next_token`, and MUST send **identical** `subject`, `action`, `resource`, and `context` (and the same `page.limit` if present). Any change → HTTP 400.
- First request has no `page.token` (or omits `page`).

Paginated AuthZEN response (`next_token` **inside** `page`; empty string = end):

```json
{
  "page": {
    "next_token": "a3M9NDU2O3N6PTI=",
    "count": 2
  },
  "results": [
    {
      "type": "attribute_value",
      "id": "https://patreon.arkavo.com/attr/tier/value/supporter",
      "properties": { "actions": ["read"] }
    },
    {
      "type": "attribute_value",
      "id": "https://patreon.arkavo.com/attr/campaign/value/87654321",
      "properties": { "actions": ["read"] }
    }
  ]
}
```

Last page: `"next_token": ""`. v1 discovery MUST omit `search_resource_endpoint`. Subject Search and Action Search are not offered.

### Wire examples

Examples compile as JSON. Consumer Patreon objects omit top-level `campaign_id`.

#### Catalog item evaluations (person + device + environment)

Minted-format mismatch: PE `subject.id` is `arkavo:550e8400-e29b-41d4-a716-446655440000`; device `sub` is the DeviceCheck bare UUID `550e8400-e29b-41d4-a716-446655440000`. They bind via `subject_id_bind`. `evaluation_id` and `obligations.required` on each evaluations entry are **(profile)** free-form `context` keys (AuthZEN §5.5.1).

```http
POST /access/v1/evaluations HTTP/1.1
Host: kas.arkavo.net
Content-Type: application/json
Authorization: Bearer 2QH9q...service-cwt...
X-Request-ID: 7c9e6679-7425-40de-944b-e07fc1f90ae7
```

```json
{
  "subject": {
    "type": "identity",
    "id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
    "properties": {
      "iss": "https://identity.arkavo.net",
      "idp": "arkavo",
      "email": "alice@example.com",
      "email_verified": true,
      "arkavo_account_id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
      "arkavo_roles": ["user"],
      "arkavo_entitlements": ["tdf:decrypt"],
      "arkavo_patreon": {
        "role": "consumer",
        "patreon_user_id": "12345678",
        "memberships": [
          {
            "campaign_id": "87654321",
            "patron_status": "active_patron",
            "tier_ids": ["111"],
            "tier_slugs": ["supporter"]
          }
        ],
        "verified_at": 1779996400,
        "cache_expires_at": 1780000000
      }
    }
  },
  "action": { "name": "read" },
  "context": {
    "device": {
      "sub": "550e8400-e29b-41d4-a716-446655440000",
      "iss": "https://identity.arkavo.net",
      "aud": "arkavo:devicecheck",
      "kid": "YWxwaGEtZGV2aWNlLWtpZA"
    },
    "environment": {
      "region": "us-east-1"
    },
    "pep": {
      "fulfillable_obligation_fqns": []
    }
  },
  "evaluations": [
    {
      "resource": {
        "type": "catalog_item",
        "id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
        "properties": {
          "attribute_value_fqns": [
            "https://patreon.arkavo.com/attr/campaign/value/87654321",
            "https://patreon.arkavo.com/attr/tier/value/supporter"
          ]
        }
      }
    },
    {
      "resource": {
        "type": "catalog_item",
        "id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
        "properties": {
          "attribute_value_fqns": [
            "https://patreon.arkavo.com/attr/campaign/value/87654321",
            "https://patreon.arkavo.com/attr/tier/value/gold"
          ]
        }
      }
    }
  ],
  "options": { "evaluations_semantic": "execute_all" }
}
```

Response:

```json
{
  "evaluations": [
    {
      "decision": true,
      "context": {
        "evaluation_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7:0",
        "obligations": { "required": [] }
      }
    },
    {
      "decision": false,
      "context": {
        "evaluation_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7:1",
        "obligations": { "required": [] }
      }
    }
  ]
}
```

#### Corresponding OpenTDF `GetDecisionMultiResource` proto-JSON

CamelCase as `tdf-iroh-s3/crates/tdf-core/src/authz.rs`.

Request:

```json
{
  "entityIdentifier": {
    "entityChain": {
      "ephemeralId": "chain",
      "entities": [
        {
          "ephemeralId": "e0",
          "category": "CATEGORY_SUBJECT",
          "claims": {
            "@type": "type.googleapis.com/google.protobuf.Struct",
            "value": {
              "sub": "arkavo:550e8400-e29b-41d4-a716-446655440000",
              "iss": "https://identity.arkavo.net",
              "email": "alice@example.com",
              "patreon_user_id": "12345678",
              "arkavo_patreon": {
                "role": "consumer",
                "patreon_user_id": "12345678",
                "memberships": [
                  {
                    "campaign_id": "87654321",
                    "patron_status": "active_patron",
                    "tier_ids": ["111"],
                    "tier_slugs": ["supporter"]
                  }
                ],
                "verified_at": 1779996400,
                "cache_expires_at": 1780000000
              }
            }
          }
        },
        {
          "ephemeralId": "e1",
          "category": "CATEGORY_ENVIRONMENT",
          "claims": {
            "@type": "type.googleapis.com/google.protobuf.Struct",
            "value": {
              "sub": "550e8400-e29b-41d4-a716-446655440000",
              "iss": "https://identity.arkavo.net",
              "aud": "arkavo:devicecheck",
              "kid": "YWxwaGEtZGV2aWNlLWtpZA"
            }
          }
        },
        {
          "ephemeralId": "e2",
          "category": "CATEGORY_ENVIRONMENT",
          "claims": {
            "@type": "type.googleapis.com/google.protobuf.Struct",
            "value": { "region": "us-east-1" }
          }
        }
      ]
    }
  },
  "action": { "name": "read" },
  "resources": [
    {
      "ephemeralId": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "attributeValues": {
        "fqns": [
          "https://patreon.arkavo.com/attr/campaign/value/87654321",
          "https://patreon.arkavo.com/attr/tier/value/supporter"
        ]
      }
    },
    {
      "ephemeralId": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "attributeValues": {
        "fqns": [
          "https://patreon.arkavo.com/attr/campaign/value/87654321",
          "https://patreon.arkavo.com/attr/tier/value/gold"
        ]
      }
    }
  ],
  "fulfillableObligationFqns": []
}
```

Response (`tdf-iroh-s3` parses `resourceDecisions[].ephemeralResourceId` / `.decision`):

```json
{
  "allPermitted": false,
  "resourceDecisions": [
    {
      "ephemeralResourceId": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "decision": "DECISION_PERMIT",
      "requiredObligations": []
    },
    {
      "ephemeralResourceId": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "decision": "DECISION_DENY",
      "requiredObligations": []
    }
  ]
}
```

#### MCP method `tools/call` (default mapping)

COAZ-MCP default:

```json
{
  "evaluation": {
    "subject": { "type": "identity", "id": "$token.sub" },
    "context": { "agent": "$token.?client_id" },
    "action": { "name": "tools/call" },
    "resource": { "type": "tool", "id": "$params.name" }
  }
}
```

v1 PEP **hardcodes** that SARC (no CEL crate) for method `tools/call`, applying this profile’s `$token` provenance and **`context.agent` fallbacks (override 3)** — not a literal evaluation of `"agent": "$token.?client_id"` alone. Arkavo user CWTs do not mint `client_id` today.

**Aud-fallback (single aud):** `$token.sub = "arkavo:550e8400-e29b-41d4-a716-446655440000"`, `$token.aud = "https://mcp.arkavo.net"` (single string, not `arkavo` / `arkavo:devicecheck` / configured platform audience), no `client_id`, `$params.name = "git_commit"`:

```json
{
  "subject": {
    "type": "identity",
    "id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
    "properties": {
      "iss": "https://identity.arkavo.net",
      "idp": "arkavo",
      "arkavo_roles": ["user"]
    }
  },
  "action": { "name": "tools/call" },
  "resource": { "type": "tool", "id": "git_commit" },
  "context": {
    "agent": "https://mcp.arkavo.net",
    "pep": { "fulfillable_obligation_fqns": [] }
  }
}
```

**Multi-aud (production OIDC):** `$token.aud = ["https://mcp.arkavo.net", "https://platform.arkavo.net"]`, `AUTHZEN_MCP_RESOURCE_ID = "https://mcp.arkavo.net"`, `OIDC_PLATFORM_AUDIENCE = "https://platform.arkavo.net"`, no `client_id`. Exactly one aud member is neither `arkavo` nor `arkavo:devicecheck` nor the platform audience → `context.agent = "https://mcp.arkavo.net"`. `tools/list` would set `resource.id` to `AUTHZEN_MCP_SERVER_SLUG` (e.g. `mcp_arkavo_net`) after confirming `AUTHZEN_MCP_RESOURCE_ID` is an aud member.

```json
{
  "subject": {
    "type": "identity",
    "id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
    "properties": {
      "iss": "https://identity.arkavo.net",
      "idp": "arkavo",
      "arkavo_roles": ["user"]
    }
  },
  "action": { "name": "tools/call" },
  "resource": { "type": "tool", "id": "git_commit" },
  "context": {
    "agent": "https://mcp.arkavo.net",
    "pep": { "fulfillable_obligation_fqns": [] }
  }
}
```

**Omit `agent`:** same call, but `$token.aud = "arkavo"` (auth token; not an RP `client_id`) and no `client_id` claim and `sub` does not start with `client:`. `context.agent` is absent (CEL `$token.?client_id` would also omit; the extra fallbacks do not apply). Multi-aud with **two** remaining members after excluding `arkavo` / `arkavo:devicecheck` / platform audience also omits `agent` (not "exactly one"):

```json
{
  "subject": {
    "type": "identity",
    "id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
    "properties": {
      "iss": "https://identity.arkavo.net",
      "idp": "arkavo",
      "arkavo_roles": ["user"]
    }
  },
  "action": { "name": "tools/call" },
  "resource": { "type": "tool", "id": "git_commit" },
  "context": {
    "pep": { "fulfillable_obligation_fqns": [] }
  }
}
```

#### Corresponding OpenTDF `GetDecision` proto-JSON

Request (singular `resource`, top-level `fulfillableObligationFqns`):

```json
{
  "entityIdentifier": {
    "entityChain": {
      "ephemeralId": "chain",
      "entities": [
        {
          "ephemeralId": "e0",
          "category": "CATEGORY_SUBJECT",
          "claims": {
            "@type": "type.googleapis.com/google.protobuf.Struct",
            "value": {
              "sub": "arkavo:550e8400-e29b-41d4-a716-446655440000",
              "iss": "https://identity.arkavo.net"
            }
          }
        }
      ]
    }
  },
  "action": { "name": "execute_tool" },
  "resource": {
    "ephemeralId": "git_commit",
    "attributeValues": {
      "fqns": ["https://arkavo.net/attr/mcp-tool/value/git_commit"]
    }
  },
  "fulfillableObligationFqns": []
}
```

Response — unwrap `decision.decision` and `decision.requiredObligations`:

```json
{
  "decision": {
    "ephemeralResourceId": "git_commit",
    "decision": "DECISION_PERMIT",
    "requiredObligations": []
  }
}
```

AuthZEN translation (obligations on **top-level** `context` for `/evaluation`):

```json
{
  "decision": true,
  "context": {
    "evaluation_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "obligations": { "required": [] }
  }
}
```

#### MCP method `tools/list` (not tool name `list_tools`)

COAZ-MCP default for the **method**:

```json
{
  "evaluation": {
    "subject": { "type": "identity", "id": "$token.sub" },
    "context": { "agent": "$token.?client_id" },
    "action": { "name": "tools/list" },
    "resource": { "type": "mcp_server", "id": "$token.aud" }
  }
}
```

Resolved (`AUTHZEN_MCP_RESOURCE_ID = "https://mcp.arkavo.net"` present in `$token.aud`; `AUTHZEN_MCP_SERVER_SLUG = "mcp_arkavo_net"`):

```json
{
  "subject": { "type": "identity", "id": "arkavo:550e8400-e29b-41d4-a716-446655440000" },
  "action": { "name": "tools/list" },
  "resource": { "type": "mcp_server", "id": "mcp_arkavo_net" },
  "context": { "agent": "https://mcp.arkavo.net" }
}
```

If `AUTHZEN_MCP_RESOURCE_ID` is not a member of `$token.aud` (string or array), the PEP MUST raise a mapping error (`-32602`) and MUST NOT call the PDP. COAZ-MCP's `"id": "$token.aud"` is overridden: never an array, never a URL that would violate the OpenTDF identifier charset.

Facade → OpenTDF action `tools_list`, FQN `https://arkavo.net/attr/mcp-server/value/mcp_arkavo_net`. **Rejected alternative:** percent-encoded URL values such as `https://arkavo.net/attr/mcp-server/value/https%3A%2F%2Fmcp.arkavo.net`.

Tool name `list_tools` is **not** this mapping. It is `tools/call` with `resource.type=tool`, `resource.id=list_tools`, FQN `https://arkavo.net/attr/mcp-tool/value/list_tools`.

#### OpenTDF `GetEntitlements` proto-JSON (phase 6)

Request (`GetEntitlementsRequest` has no action):

```json
{
  "entityIdentifier": {
    "entityChain": {
      "ephemeralId": "chain",
      "entities": [
        {
          "ephemeralId": "e0",
          "category": "CATEGORY_SUBJECT",
          "claims": {
            "@type": "type.googleapis.com/google.protobuf.Struct",
            "value": {
              "sub": "arkavo:550e8400-e29b-41d4-a716-446655440000",
              "iss": "https://identity.arkavo.net",
              "email": "alice@example.com"
            }
          }
        }
      ]
    }
  },
  "withComprehensiveHierarchy": false
}
```

Response:

```json
{
  "entitlements": [
    {
      "ephemeralId": "e0",
      "actionsPerAttributeValueFqn": {
        "https://patreon.arkavo.com/attr/tier/value/supporter": {
          "actions": [{ "name": "read" }]
        },
        "https://patreon.arkavo.com/attr/campaign/value/87654321": {
          "actions": [{ "name": "read" }, { "name": "decrypt" }]
        }
      }
    }
  ]
}
```

For Resource Search `action.name=read`, emit `attribute_value` results for FQNs whose actions list includes `read`.

#### KAS rewrap (end-state example; not v1)

v1 MUST NOT send this from `cbor_protocol.rs`.

```json
{
  "subject": {
    "type": "identity",
    "id": "arkavo:550e8400-e29b-41d4-a716-446655440000",
    "properties": {
      "iss": "https://identity.arkavo.net",
      "email": "alice@example.com",
      "arkavo_patreon": {
        "role": "consumer",
        "patreon_user_id": "12345678",
        "memberships": [
          {
            "campaign_id": "87654321",
            "patron_status": "active_patron",
            "tier_ids": ["111"],
            "tier_slugs": ["supporter"]
          }
        ],
        "verified_at": 1779996400,
        "cache_expires_at": 1780000000
      }
    }
  },
  "action": { "name": "rewrap" },
  "resource": {
    "type": "tdf",
    "id": "https://kas.arkavo.net/kas#policy-binding",
    "properties": {
      "attribute_value_fqns": [
        "https://patreon.arkavo.com/attr/tier/value/supporter"
      ]
    }
  },
  "context": {
    "pep": { "fulfillable_obligation_fqns": [] }
  }
}
```

### Discovery document

`GET https://kas.arkavo.net/.well-known/authzen-configuration`

`policy_decision_point` MUST equal the identifier into which `/.well-known/authzen-configuration` was inserted (AuthZEN §9.2.3). v1 example:

```json
{
  "policy_decision_point": "https://kas.arkavo.net",
  "access_evaluation_endpoint": "https://kas.arkavo.net/access/v1/evaluation",
  "access_evaluations_endpoint": "https://kas.arkavo.net/access/v1/evaluations"
}
```

After phase 6, add `search_resource_endpoint`. Do not advertise subject/action search.

`signed_metadata` is OPTIONAL. v1 MUST omit it unless a dedicated PDP JWT signing key is configured. If present it MUST be a JWT. It MUST NOT be a CWT.

Operator config (`arks`):

| Variable | Default | Purpose |
|---|---|---|
| `AUTHZEN_FACADE` | `off` | `off` or `on` only. `on` serves AuthZEN routes + well-known. Independent of `AUTHZ_PROXY`. |
| `OPENTDF_PLATFORM_URL` | — | **Required** when `AUTHZEN_FACADE=on`. May be loopback (`http://127.0.0.1:8443`) for a co-located platform sidecar. |
| `AUTHZEN_PEP_CLIENT_IDS` | unset | Optional allowlist → HTTP 403 for other service accounts |
| `OIDC_ISSUER` / cose-keys URL | identity.arkavo.net | Service CWT verification |
| `AUTHZ_PROXY` | `off` | Existing v2 relay; keep for rollback and platform KAS |

There is no `AUTHZEN_FACADE=local` and no built-in evaluator.

### COAZ-MCP CWT profile (normative overrides)

Unmentioned COAZ-MCP text still applies. v1 (PR 4) implements **hardcoded** mappings for methods `tools/call` and `tools/list` (no CEL) + pass-through `ping` / `notifications/*`. PR 4b adds CEL, `resources/*`, `prompts/*`, remaining default mappings, and declared `x-authzen-mapping`. Until 4b, MCP methods other than `tools/call`, `tools/list`, and the pass-through set MUST be denied (fail closed), not silently allowed. Method `tools/list` MUST NOT be in that deny set in PR 4.

1. **`$token` is not necessarily a JWT.** It is the decoded, verified **CWT** claims map using text names. `cnf` is omitted. A PEP MUST union-verify the CWT before populating `$token`.
2. **`subject.id` trust-anchoring still applies.** Default mappings use `$token.sub`. Mismatch → mapping error, JSON-RPC `-32602`, no PDP call.
3. **`context.agent`.** Prefer `$token.?client_id`. If absent: if `sub` starts with `client:`, use the remainder after the `client:` prefix (`client:catalog-node` → `catalog-node`); else if `$token.aud` (string or array) contains **exactly one** member that is neither `arkavo` nor `arkavo:devicecheck` nor the configured platform audience (`OIDC_PLATFORM_AUDIENCE`), use that member; else omit. Production OIDC tokens commonly have `Audience::Multiple` when `OIDC_PLATFORM_AUDIENCE` is set.
4. **`resource.id` for method `tools/list`.** MUST be the configured charset-safe slug (`AUTHZEN_MCP_SERVER_SLUG`, or a slug form of `AUTHZEN_MCP_RESOURCE_ID`). The PEP MUST confirm `AUTHZEN_MCP_RESOURCE_ID` (RFC 8707 resource identifier of this MCP server) appears in `$token.aud` as a string or array member; otherwise mapping error (`-32602`), no PDP call. Never an array. Never a percent-encoded URL. This overrides COAZ-MCP `"id": "$token.aud"`.
5. **Fail closed.** Mapping error / deny / PDP error / non-empty `obligations.required` → do not execute. Codes: `-32602` mapping, `-32001` deny, `-32603` PDP.
6. **Pass-through unchanged:** `ping` and `notifications/*`.
7. **Unknown MCP methods** denied.
8. **Do not conflate method `tools/list` with tool name `list_tools`.**
   - Method `tools/list` → COAZ-MCP default (`mcp_server`, action `tools/list`) **hardcoded in PR 4** (no CEL). Facade maps to OpenTDF action `tools_list` + `mcp-server` FQN using the configured slug.
   - Tool name `list_tools` is invoked via method `tools/call`. Remove `"list_tools"` from `McpToolMapping::is_safe_diagnostic` on PR 4 cutover so it is a normal `tool` evaluation.
   - `status`, `health`, `version` MAY remain a documented PEP exception for **90 days** after PR 4 ships, then MUST be removed or expressed as PDP policy.
9. **Declared mappings** (`x-authzen-mapping`) are PR 4b. Attributes other than verified `subject.id` are untrusted.
10. **Expression language** is CEL (PR 4b). v1 hardcoded `tools/call` / `tools/list` mappings are the COAZ-MCP defaults **plus this profile’s `$token` provenance, `context.agent` fallbacks (override 3), and `AUTHZEN_MCP_RESOURCE_ID` aud check (override 4)**. They are **not** a literal CEL evaluation of `"agent": "$token.?client_id"` / `"id": "$token.aud"` alone (Arkavo user CWTs usually lack `client_id` and often have multi-aud).
11. **Service CWT** authenticates the PEP to the facade. The user CWT MUST NOT be the AuthZEN Bearer.
12. **Cache TTL** from `$token.exp`. `extract_token_ttl` JWT-splitting on `.` MUST be removed.
13. **Subject credential source (profile).** The MCP PEP MUST obtain the subject CWT from inbound `Authorization: Bearer` or from a named environment variable that is documented as **CWT-only** (`CLAUDE_CODE_SESSION_ACCESS_TOKEN` is acceptable only when that env is documented to hold an Arkavo CWT). `ANTHROPIC_API_KEY` MUST NOT be accepted as a subject credential. Today's `arkavo-mcp-claude/src/policy_bridge.rs` falls back to `ANTHROPIC_API_KEY`; PR 4 MUST remove that fallback. There is **no** crate `arkavo-mcp-proxy` in this workspace; do not assume one.

### Air-gapped / local KAS

- NanoTDF unwrap on `/ws` CBOR (`CborRequest::ChainRewrap`) MUST remain local in v1. No AuthZEN HTTP client on that path.
- AuthZEN JSON 1.0 on NFC/BLE is a **non-goal**, not an existing PEP.
- A self-hosted `arks` MUST continue to unwrap with local KAS policy when `AUTHZEN_FACADE=off`.
- If an air-gapped lab needs AuthZEN JSON (catalog/MCP), run a **co-located OpenTDF platform container** and set `OPENTDF_PLATFORM_URL` to it. Do not implement a second evaluator in `arks`.

### Later phases (not first implementation)

1. **AuthZEN Token Issuance profile.** `authnz-rs` PEP before `cwt::mint`. AuthZEN action `issue:cwt:{grant}` (divergent from IETF `issue:access_token:{grant}`); OpenTDF action `issue_cwt_{grant}`. Deny → do not mint.
2. **Claims filling.** Not phase 6 (`attribute_value` only). A later classifier or issuance-specific search would be required.
3. **Optional CBOR/CoAP binding.** Not this draft.
4. **KAS rewrap via facade** only after obligation round-trip is proven.
5. **arkavo-node as PIP.**

### arkavo-node

`evaluate_access` currently checks policy existence and `active`. This spec does **not** claim AuthZEN conformance. PEPs MUST NOT send AuthZEN to the chain RPC.

### OpenTDFKit / app

Bearer tokens stay opaque. Swift code MUST NOT be required to build SARC or call `/access/v1/*`.

## API / Interface Changes

### AuthZEN endpoints on `arks`

New Axum routes in `arkavo-rs/src/bin/main.rs` (beside `authz_router`):

- `POST /access/v1/evaluation`
- `POST /access/v1/evaluations`
- `GET /.well-known/authzen-configuration`
- Later: `POST /access/v1/search/resource`

Module: `arkavo-rs/src/modules/authzen/` (`mod.rs`, `facade.rs`, `cwt_subject.rs`, `cwt_verify.rs`, `discovery.rs`, `translate.rs`).

Existing OpenTDF v2 proxy routes MUST remain when `AUTHZ_PROXY=on`.

### CWT claim → SARC tables

See CWT Subject Profile. No new CWT labels in v1.

### COAZ `$token`

`arkavo-edge` replaces OpenTDF-shaped types at the PEP boundary:

| Remove from PEP boundary | Replace with |
|---|---|
| `GetDecision*` as the **wire** types to the PDP | AuthZEN evaluation(s) JSON |
| `CreateEntityChainsFromTokens` | Local union-verify + minimum `$token` decoder |
| User token as PDP Bearer | Service CWT (`AUTHZEN_CLIENT_*`) |
| `is_safe_diagnostic("list_tools")` | Normal `tools/call` of tool `list_tools` |
| `extract_token_ttl` JWT split | `$token.exp` |
| `McpToolMapping::tool_to_resource` / `Resource::mcp_tool` | Facade-only alias table |

`arkavo-edge` AGENTS.md caps implementation files at ~400 lines and ≥85% coverage — split the crate rather than grow `client.rs`. Apply the same patch to `arkavo-edge-swarmkit-apply/crates/arkavo-authorization`.

### Catalog PEP interface

`[catalog.authz] protocol = "authzen" | "opentdf-v2"`. AuthZEN mode uses discovery `access_evaluations_endpoint`. Public HTTP remains `GET /catalog/{group}` with `Authorization: Bearer` (PE CWT) and **zero or more repeated** `X-Entity-Token` headers (one DeviceCheck CWT each; `get_all`, not comma-separated). PR 5 MUST: extend `CwtVerifier` / `VerifiedClaims` / `parse_claims` (aud, cti, cnf.kid, duplicate-key reject, `exp <= now - 60`, DeviceCheck `aud`); treat every `X-Entity-Token` as a first-class device NPE; bind each device `sub` to PE `sub` via `subject_id_bind` (replace `claims.sub != pe.sub`); emit `context.devices` when N ≥ 2; skip the PDP on empty listings; copy the union verify + `$token`/SARC mapping algorithm (no shared crate). Operators MUST confirm no production attribute value carries an obligation trigger before flipping `protocol = "authzen"`.

## Data Model Changes

**No CWT schema change in v1.** Additive OPTIONAL future claim: `client_id` (tstr). Do not add AuthZEN JSON to CWT.

## Alternatives Considered

### 1. Dual-stack (AuthZEN only for MCP, OpenTDF v2 elsewhere)

Two PEP protocols forever. Rejected.

### 2. Replace OpenTDF Authorization Service with Cerbos / Topaz / OPA

Rewrites the production engine that already knows Arkavo FQNs, Patreon ERS, and KAS obligations. Rejected for v1. The facade enables a later PDP swap without changing PEPs.

### 3. Put AuthZEN JSON inside CWT, or revert access tokens to JWT

Rejected. jwt-to-cwt was an approved hard cutover. Embedding SARC freezes a decision at mint time.

### 4. AuthZEN-shaped types, OpenTDF wire (no facade)

Each PEP maps SARC locally and still POSTs ConnectRPC to platform. Smaller than a facade; avoids putting CWT verify in `arks`. Rejected: PEPs remain OpenTDF-specific (no third-party PDP, no COAZ-MCP interop at the wire); obligation translation and proto-JSON footguns stay copied in every PEP; catalog and MCP would still disagree with `types.rs`.

### 5. Dedicated AuthZEN process vs in `arks`

A sidecar translator adds a hop and a second TLS identity without moving the engine. Rejected for v1; `arks` already terminates PEP TLS and can loopback to a platform sidecar.

## Security & Privacy Considerations

| Threat | Sev. | Mitigation |
|---|---|---|
| **Stolen subject CWT replay** | High | Short TTL. TLS. Device `sub` bound to PE via `subject_id_bind`. MUST NOT log full tokens. |
| **Stolen service CWT** | High | Confidential client + short TTL. Optional `AUTHZEN_PEP_CLIENT_IDS`. |
| **Identity smuggling via `x-authzen-mapping`** | High | Trust-anchor `subject.id`. Reject per-entry `subject` in `evaluations`. PR 4b. |
| **Environment spoofing** | High | Node-asserted region. Facade **allowlist** `{region}` plus optional `kind`; drop every other key (not a denylist). |
| **Resource substitution by an authenticated PEP** | High | For `resource.type` in `{tool, mcp_server}` the facade MUST derive FQNs itself and MUST reject (400) PEP-supplied `attribute_value_fqns`. Keep PEP-supplied FQNs only for `catalog_item` / `tdf` / `kas`. Log evaluated FQNs or their count/hash. |
| **PEP/PDP mix-up** | High | `policy_decision_point` match. PEPs pin well-known AuthZEN URLs, never `AUTHZ_PROXY` paths. No redirects on AuthZEN POST. |
| **Subject vs service token mix-up** | High | MCP MUST use service CWT like catalog. Facade rejects user CWT as Bearer. |
| **ERS JWT-only footgun** | High | Claims mode only. |
| **Obligation downgrade** | High | v1: empty fulfillable; non-empty `required` → fail closed even on permit. |
| **Diagnostic allowlist** | Med | Remove `list_tools` immediately; sunset others. Do not map tool `list_tools` to method `tools/list`. |
| **Token-carried entitlements as live ABAC** | High | Claims profile rule. |
| **Air-gap forced online** | High | No AuthZEN on CBOR rewrap. No built-in fake PDP. |
| **Fail open** | High | Forbidden. Empty catalog listing: no PDP, no entitle. |
| **PII in logs** | Med | MUST NOT log `email` or `arkavo_patreon`. Log `subject_id`, not email (hashed or otherwise). |
| **Key-set fetch amplification** | Med | 60s min refresh; 10s fetch timeout. |
| **CEL on untrusted `params`** | Med | PR 4b; mapping errors fail closed. |

Transport: PEP↔facade and facade↔OpenTDF MUST use TLS except loopback sidecar.

## Observability

**Log fields:** `evaluation_id`, `x_request_id`, `pep_sub` (service CWT `sub` only), `subject_id`, `subject_iss`, `action_name`, `resource_type`, `resource_id` (or batch count), `attribute_fqns` or `attribute_fqns_count` / `attribute_fqns_hash` (the FQNs actually evaluated — not only `resource_id`), `decision`, `upstream_ms`, `obligations_count`. Catalog: `group`, `chain_len`.

**MUST NOT log:** raw CWT/Bearer, `cnf` COSE_Key, `email`, `arkavo_patreon`, `client_secret`.

**Metrics:** `authzen_evaluation_total{action,decision}`, `authzen_evaluations_batch_size`, `authzen_upstream_latency_ms`, `authzen_pep_auth_fail_total`, `authzen_fail_closed_total`. Alert on batch size hitting the **Arkavo** 500 cap.

**Timeouts:** facade→OpenTDF 10s; MCP PEP→facade 5s; catalog PEP→facade 10s.

**Tracing:** echo `X-Request-ID`. Generate `rid` if missing.

When `AUTHZ_PROXY` and `AUTHZEN_FACADE` are both on, metrics MUST label the path (`authzen` vs `opentdf_proxy`) so mix-ups are visible.

## Rollout Plan

1. Spec only.
2. Facade code behind `AUTHZEN_FACADE=off`. PR 3 does not merge until the platform caller-credential spike passes.
3. Lab: contract tests (catalog JSON ↔ MultiResource proto-JSON; MCP evaluation ↔ GetDecision nested response; 401 vs 200/deny; empty evaluations 400).
4. MCP PR 4 (hardcoded `tools/call` + `tools/list` + service CWT + union CWT verify + mock AuthZEN). Then PR 4b CEL / remaining methods / declared mappings.
5. Catalog `protocol=authzen` on staging; shadow; flip. Shadow MUST diff obligation-bearing resources (empty fulfillable ⇒ platform DENY). Operators MUST confirm no production attribute value carries an obligation trigger before the flip. Rollback: `protocol=opentdf-v2`.
6. Search `attribute_value` only.
7. Do not flip KAS rewrap onto AuthZEN.

Cache key MUST include PDP origin so rollback cannot reuse entries.

## Open Questions

**None open.** User decisions (final):

1. **Shared mapping crate vs copy — copy for v1.** Independent repos stay independent. Copy the specified union-verify and CWT→SARC algorithms into `arks`, `arkavo-edge`, and the catalog. A published `arkavo-cwt` crate MAY come later if drift hurts. Do **not** block PRs on a shared crate.
2. **Multiple devices — v1 wire + verification.** A request MAY attach several DeviceCheck CWTs (phone + watch). `context.devices` is a specified v1 shape, not “-01 MAY drop.” Repeated `X-Entity-Token` headers; chain order PE then devices then environment; cap `1+D+E ≤ 8`; bind via `subject_id_bind`. v1 decisions are unaffected (`CATEGORY_ENVIRONMENT` + `skipEnvironmentEntities=true`).

## References

### Normative

- RFC 2119, RFC 8174, RFC 6750, RFC 7519, RFC 7638, RFC 8259, RFC 8392, RFC 8615, RFC 8747, RFC 9052
- OpenID AuthZEN Authorization API 1.0 (Final, 11 January 2026)
- COAZ Framework; COAZ-MCP
- OpenTDF `authorization/v2/authorization.proto`, `entity/entity.proto`, `service/policy/attributes/attributes.proto`, `service/policy/actions/actions.proto`

### Informative

- RFC 6749, RFC 7515, RFC 8707, RFC 9068, RFC 9200
- draft-gazitt-oauth-authzen-issuance-00, draft-gazitt-oauth-authzen-claims-00
- jwt-to-cwt design: `authnz-rs/docs/superpowers/specs/2026-05-23-jwt-to-cwt-design.md`
- `authnz-rs/src/cwt.rs`, `authnz-rs/src/oidc.rs`, `authnz-rs/src/device_check.rs`, `authnz-rs/src/authn.rs`
- `tdf-iroh-s3/README.md`, `tdf-iroh-s3/crates/tdf-core/src/{auth,authz,catalog_api}.rs`
- `arkavo-rs/docs/platform-proxy.md`, `arkavo-rs/src/modules/{platform_proxy,cbor_protocol,ntdf_token}.rs`
- `arkavo-edge/crates/arkavo-authorization/src/{client,types,config,cache}.rs` and duplicate `arkavo-edge-swarmkit-apply/crates/arkavo-authorization`
- `arkavo-edge/crates/arkavo-mcp-claude/src/policy_bridge.rs`
- `arkavo-node/contracts/policy_engine/README.md`
- OpenTDFKit CLAUDE.md (opaque Bearer)
- `specifications/ntdf-token/draft-arkavo-ntdf-token-00.md` (**SUPERSEDED**)

## Key Decisions

1. **AuthZEN is the PEP↔PDP protocol; CWT remains the access token.** Do not encode SARC in CWT. Do not revert to JWT access tokens. `signed_metadata` stays JWT.
2. **Facade over OpenTDF Authorization v2, hosted in `arks` next to `AUTHZ_PROXY`.** Independent flag `AUTHZEN_FACADE=off|on`. `on` requires `OPENTDF_PLATFORM_URL` (loopback sidecar allowed). **No built-in evaluator.**
3. **Claims-mode entity chains only for Arkavo CWTs.** Never `EntityIdentifier.token`.
4. **One AuthZEN subject (`type=identity`, the COAZ-MCP value — not an AuthZEN-registered type; `id=PE sub`) + context for devices, environment, and agent.** Reconstruct chain **PE, then each device NPE, then environment**. Environment is node-asserted. **Multi-device is a v1 wire + verification requirement**; it has no v1 policy effect (Key Decision 25).
5. **`$token` uses text claim names; `cnf` is omitted.** Minimum decoder is normative. CEL is `$token.?email`.
6. **Service CWT authenticates the PEP; subject CWT is not the AuthZEN Bearer.** MCP obtains service CWT the same way as catalog (`AUTHZEN_TOKEN_URL` / `CLIENT_ID` / `CLIENT_SECRET`). Facade: `iss` pin + `sub` `client:` + role `service-account`; **no single `aud`** (production service CWTs are `aud = [client_id, platform_audience]` when `OIDC_PLATFORM_AUDIENCE` is set). Optional `AUTHZEN_PEP_CLIENT_IDS` → 403. Platform caller-credential CWT compatibility is a **PR 3 entry criterion**, not an open question. MCP subject credential MUST be an Arkavo CWT; `ANTHROPIC_API_KEY` MUST NOT be accepted.
7. **`arkavo_roles` / `arkavo_entitlements` / `arkavo_patreon` are PIP hints, not the decision.**
8. **`cnf` is authentication metadata, not an authorization input, and is not in `$token`.** Each device `kid` lives on that device object in `context.device` / `context.devices` after DeviceCheck verify.
9. **No remote AuthZEN on NanoTDF CBOR rewrap in v1.** AuthZEN JSON on NFC/BLE is a non-goal. Air-gapped AuthZEN JSON = co-located platform sidecar.
10. **Fail closed everywhere.** HTTP 401 ≠ deny. Empty catalog listing: no PDP call. Empty `evaluations` without top-level `resource`: facade 400.
11. **First PEPs: MCP hardcoded tools/call and method tools/list (PR 4), then catalog (PR 5).** PR 4b is CEL, `resources/*`, `prompts/*`, and declared mappings — not the first `tools/list` mapping. Search, issuance, CBOR binding, on-chain AuthZEN: later. Swift clients stay presenters.
12. **Method `tools/list` ≠ tool name `list_tools`.** Method maps to OpenTDF action `tools_list` + `mcp-server` slug FQN (`AUTHZEN_MCP_SERVER_SLUG` / slug form of `AUTHZEN_MCP_RESOURCE_ID`, which MUST appear in `$token.aud`). Tool `list_tools` is `tools/call` of a `tool`. Remove `list_tools` from the diagnostic allowlist on PR 4; sunset `status`/`health`/`version`.
13. **`arkavo-node` policy_engine is not AuthZEN 1.0.**
14. **NTDF tokens and TØR-G are out of scope** (superseded / different layer).
15. **Resource Search ≈ GetEntitlements is `attribute_value` only** and MUST NOT be advertised as catalog search or as `role`/`entitlement` search.
16. **Verification algorithm is the union** of `cwt::verify` and `CwtVerifier` (stricter). Device schema is `{sub,iss,aud,kid}` (allowlist; unknown keys dropped); no `kind`. Missing REQUIRED device field → HTTP 400; present-but-wrong `aud`/`sub` → deny-closed. PR 5 extends `CwtVerifier`. DeviceCheck `aud=arkavo:devicecheck` is new catalog behavior. **0, 1, or N devices in v1:** omit both fields; or `context.device`; or `context.devices` (required when N≥2). Sending both, or an empty `devices` array, → 400. Each device `sub` MUST bind to PE `sub` via `subject_id_bind` (strip a single leading `arkavo:` only — not `apple:`/`client:`). Catalog presents devices as **repeated** `X-Entity-Token` headers. DeviceCheck/WebAuthn tokens mint bare-UUID `sub`; OIDC access mints `arkavo:{uuid}`.
17. **v1 MCP is hardcoded `tools/call` + method `tools/list`** (service CWT + `$token` + mock AuthZEN; `context.agent` fallbacks per override 3; `AUTHZEN_MCP_RESOURCE_ID` aud check per override 4). Not full COAZ-MCP (CEL / remaining methods / declared mappings) until PR 4b.
18. **Issuance AuthZEN action names are `issue:cwt:{grant}`**, divergent from IETF `issue:access_token:{grant}`. OpenTDF action names are charset-safe lowercase slugs `issue_cwt_{grant}` (e.g. `issue_cwt_authorization_code`). The facade MUST NOT pass an unmapped action whose OpenTDF name would violate `^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$` (HTTP 400). Mapped OpenTDF action names are lowercased.
19. **v1 PEPs send empty fulfillable obligation FQNs.** Empty fulfillable ⇒ platform DENYs resources with triggered obligations. Operators MUST confirm no production attribute value carries an obligation trigger before flipping catalog `protocol=authzen`. Shadow phase diffs this. Non-empty `context.obligations.required` **(profile, per-evaluation for `/evaluations`, top-level for `/evaluation`)** → fail closed even on permit.
20. **Tool FQN aliases are facade-only.** PEPs send `type=tool` + tool name. Dots map to underscores (`git_commit` not `git.commit`). For `resource.type` in `{tool, mcp_server}` the facade MUST derive FQNs itself and MUST reject (400) PEP-supplied `attribute_value_fqns`. `mcp_server` `resource.id` is a configured slug (`AUTHZEN_MCP_SERVER_SLUG` or slug form of `AUTHZEN_MCP_RESOURCE_ID`), not a percent-encoded URL. Confirm the OpenTDF identifier charset against the deployed platform in the PR 3 entry spike.
21. **Arkavo evaluations cap is 500; reconstructed chain cap is 8** (`1 + D + E ≤ 8`; typically ≤6 devices when environment is present). Neither is an AuthZEN or verified OpenTDF limit.
22. **Discovery is a profile MUST**, not an AuthZEN MUST. Upstream failures map to HTTP 500. `evaluation_id`, `context.pep.fulfillable_obligation_fqns`, and `context.obligations.required` are **(profile)** free-form `context` keys per AuthZEN §5.5.1. `evaluation_id` is opaque (`rid` or `rid:index`), not necessarily a UUID.
23. **Duplicate `arkavo-edge-swarmkit-apply/crates/arkavo-authorization` must track PEP changes.**
24. **Copy algorithms for v1; no shared crate gate.** Spec is the contract. Copy union verify + CWT→SARC into `arks`, `arkavo-edge`, and `tdf-iroh-s3`. A later `arkavo-cwt` crate is optional if drift hurts.
25. **Multi-device is a v1 wire + verification requirement.** `context.devices` is first-class. PEPs MAY send 0, 1, or N DeviceCheck CWTs. Facade emits one NPE per device, PE-then-devices-then-environment. **v1 decisions are unaffected** because device NPEs are `CATEGORY_ENVIRONMENT` and the platform sets `skipEnvironmentEntities=true`. Named future: the platform stops skipping, or a `kind` / `npe_type` claim distinguishes devices from environment. This is not a buried caveat — it is the v1 policy effect.

## PR Plan

Parent workspace has no shared build. Each PR is one repo (except the spec). `arkavo-edge` files stay under ~400 lines (AGENTS.md); split modules rather than grow `client.rs`.

### PR 1 — Spec + examples

- **Title:** `docs(authzen-cwt): draft-arkavo-authzen-cwt-00`
- **Files:** `specifications/authzen-cwt/draft-arkavo-authzen-cwt-00.md`, `specifications/README.md`, ntdf-token SUPERSEDED notes
- **Depends on:** none

### PR 2 — CWT→SARC mapping module in `arks`

- **Title:** `feat(authzen): CWT subject profile mapping`
- **Files:** `arkavo-rs/src/modules/authzen/cwt_subject.rs`, `mod.rs`
- **Depends on:** PR 1
- **Description:** Verification-agnostic projection to `$token` JSON (no `cnf`) and SARC subject / `context.device` / `context.devices` schema (0..N devices). Implement `subject_id_bind`. Tests for text names, consumer vs creator Patreon, environment **allowlist** (not denylist), two-device array, mismatched-prefix bind (PE `arkavo:{uuid}` ↔ device bare UUID). **Copy** this algorithm; do not wait on a published crate.

### PR 2b — CWT verify in `arks`

- **Title:** `feat(authzen): union CWT verifier`
- **Files:** `arkavo-rs/src/modules/authzen/cwt_verify.rs`; `arkavo-rs/Cargo.toml`; tests (tag 61, protected ES256, kid, duplicate keys, `exp <= now-60`, DeviceCheck aud)
- **New deps (Cargo.toml):** `coset` (COSE_Sign1 parse, protected-header `alg`/`kid`, COSE_Key set decode). `ciborium`, `p256`, `base64`, and `reqwest` are already present.
- **Depends on:** PR 2
- **Description:** Port catalog `CwtVerifier` behavior into `arks` (cose-keys fetch, 60s refresh, 10s timeout) **plus** union strictness. Needed before PR 3 can verify PEP service CWTs. **Copy** the algorithm (Key Decision 24); do not block on a shared crate.

### PR 3 — AuthZEN facade (`evaluation` + `evaluations` + discovery)

- **Title:** `feat(authzen): facade over OpenTDF authorization v2`
- **Files:** `arkavo-rs/src/modules/authzen/{facade,translate,discovery}.rs`; `arkavo-rs/src/bin/main.rs`; `arkavo-rs/docs/platform-proxy.md`
- **Depends on:** PR 2b
- **Entry criterion:** (1) spike platform Connect caller CWT vs JWT; mint/exchange if needed; document the result in the PR. (2) confirm OpenTDF attribute-value and action-name charset (`^[a-zA-Z0-9](?:[a-zA-Z0-9_-]*[a-zA-Z0-9])?$`, lowercase) against the **deployed** `platform.arkavo.net`, not only `opentdf/platform` `main`.
- **Description:** `AUTHZEN_FACADE=on` requires `OPENTDF_PLATFORM_URL`. Service CWT authn (no single aud; optional allowlist; multi-aud when `OIDC_PLATFORM_AUDIENCE` is set). Nested `GetDecision` parse. Reconstruct **one NPE per `context.devices` element** (PE then devices then environment; `1+D+E ≤ 8`; `subject_id_bind`; device allowlist `{sub,iss,aud,kid}`; environment allowlist `{region}` plus optional `kind`). Missing REQUIRED device fields → 400; present-but-wrong `aud`/`sub` → deny-closed. For `resource.type` in `{tool, mcp_server}` derive FQNs; reject PEP-supplied `attribute_value_fqns` (400). Map AuthZEN actions to charset-safe lowercase OpenTDF names; HTTP 400 if the mapped name is illegal. 10s upstream timeout. HTTP 500 not 502. Empty evaluations → 400. Do not touch `cbor_protocol.rs`. Keep `AUTHZ_PROXY`.

### PR 3b — Contract tests (mock OpenTDF)

- **Title:** `test(authzen): proto-JSON contract fixtures`
- **Files:** `arkavo-rs` tests (or `specifications/authzen-cwt/examples/` JSON fixtures consumed by tests)
- **Depends on:** PR 3
- **Description:** Catalog evaluations ↔ MultiResource request/response (including **two-device** `context.devices` and the **mismatched-prefix** bind: PE `arkavo:{uuid}` + device bare UUID); MCP evaluation ↔ GetDecision nested `decision.decision`; 401 vs 200/deny; 400 empty evaluations; 400 when `1+D+E > 8`; 400 missing REQUIRED device field; deny-closed present-but-wrong `aud`/`sub`; 400 PEP-supplied FQNs on `tool`/`mcp_server`; 400 charset-illegal mapped action; obligation fail-closed (per-evaluation `context.obligations.required` on `/evaluations`).

### PR 4 — `arkavo-edge` tools/call + tools/list PEP (narrow)

- **Title:** `feat(authorization): AuthZEN tools/call and tools/list PEP with CWT $token`
- **Files:** `arkavo-edge/crates/arkavo-authorization/src/{client,types,config,error,cache,cwt_verify,cwt_subject,tests}.rs` (split to stay <400 lines); `arkavo-edge/crates/arkavo-authorization/Cargo.toml`; `arkavo-edge/crates/arkavo-mcp-claude/src/policy_bridge.rs` (and the swarmkit-apply duplicate of that file); **same authorization-crate change** in `arkavo-edge-swarmkit-apply/crates/arkavo-authorization`; config `AUTHZEN_PDP_URL`, `AUTHZEN_TOKEN_URL`, `AUTHZEN_CLIENT_ID`, `AUTHZEN_CLIENT_SECRET`, `AUTHZEN_MCP_RESOURCE_ID`, `AUTHZEN_MCP_SERVER_SLUG`. **Locate and update the MCP JSON-RPC dispatcher; do not assume `arkavo-mcp-proxy`** (that crate does not exist in this workspace). Candidates inspected: `arkavo-mcp-runtime/src/server.rs` dispatches `execute_tool` / `list_tools` (not MCP `tools/call` / `tools/list`); `arkavo-mcp` is types only; `arkavo-cli/src/mcp_client.rs` is a client. The swarmkit-apply tree has `arkavo-cli/src/commands/mcp.rs` which **does** dispatch `tools/list` and `tools/call`; that file is absent from `arkavo-edge`.
- **New deps (Cargo.toml):** `coset`, `ciborium`, `p256` (workspace or compatible versions). `base64` is already present. **Copy** the PR 2b union verify algorithm into `cwt_verify.rs` and the SARC mapper into `cwt_subject.rs` (Key Decision 24 — copy for v1, no shared crate). `$token` omits `cnf`.
- **Depends on:** PR 1; tests against mock AuthZEN (PR 3 not strictly required if mock is local). Does **not** depend on merging PR 2b first; the algorithm is specified here and copied.
- **Description:** Union-verify user CWT; minimum `$token` decoder; **hardcoded** SARC for MCP methods `tools/call` and `tools/list` (no CEL) using this profile’s `context.agent` fallbacks and `AUTHZEN_MCP_RESOURCE_ID` aud check; service CWT Bearer; trust-anchor `subject.id`; `$token.exp` cache TTL; fail-closed (`-32001` deny / `-32602` mapping / `-32603` PDP); remove `list_tools` from `is_safe_diagnostic`; stop `CreateEntityChainsFromTokens`; change `McpToolMapping::tool_to_resource` so it no longer emits `git.commit` (use `git_commit`). Subject credential MUST be an Arkavo CWT (`Authorization: Bearer` or a named env documented as CWT-only). `ANTHROPIC_API_KEY` MUST NOT be accepted as a subject credential (`policy_bridge.rs` MUST drop that fallback). **No CEL crate. No declared mappings. No `resources/*` / `prompts/*`.** Unknown methods other than `tools/call`, `tools/list`, and pass-through deny. Pass-through `ping` / `notifications/*`.

### PR 4b — CEL + remaining COAZ-MCP defaults + declared mappings

- **Title:** `feat(authorization): COAZ-MCP CEL and default method table`
- **Files:** new modules under `arkavo-edge/crates/arkavo-authorization/` (and swarmkit-apply duplicate)
- **Depends on:** PR 4
- **Description:** CEL evaluator; default mappings for `resources/*`, `prompts/*`, and other COAZ-MCP methods **not** already hardcoded in PR 4; `x-authzen-mapping`; reject per-entry `subject`. Method `tools/list` is already in PR 4 — 4b MUST NOT be the first time listing tools is authorized.

### PR 5 — `tdf-iroh-s3` catalog PEP → AuthZEN evaluations

- **Title:** `feat(catalog): AuthZEN evaluations client`
- **Files:** `tdf-iroh-s3/crates/tdf-core/src/{auth,authz,catalog_api}.rs`; `tdf-iroh-s3/src/{config,main}.rs`; README
- **Depends on:** PR 3 in lab; default `protocol = "opentdf-v2"`
- **Description:** Extend `CwtVerifier` to the union algorithm and device schema (`aud`, `cti`, `cnf.kid`; DeviceCheck `aud=arkavo:devicecheck`; no `kind`) — **copy** the spec algorithm (Key Decision 24). Accept **multiple repeated `X-Entity-Token` headers** as first-class v1 (phone + watch); verify each; bind each device `sub` to PE `sub` via `subject_id_bind` (**MUST** replace `claims.sub != pe.sub` in `catalog_api.rs`); emit `context.devices` when N≥2 (or `context.device` when N=1). Skip PDP when listing is empty. HTTP 400 if `1+D+E > 8`. Build AuthZEN JSON from the extended chain, not from today's `{sub,iss}` NPE claims alone. Before flipping `protocol=authzen`, confirm no production attribute value carries an obligation trigger.

### PR 6 — Resource Search `attribute_value` only

- **Title:** `feat(authzen): resource search via GetEntitlements`
- **Files:** facade + discovery `search_resource_endpoint`
- **Depends on:** PR 3
- **Description:** Filter `actionsPerAttributeValueFqn` by request `action.name` **before** paging. Default `page.limit=100` (Arkavo); honor request `limit` capped at 100. Opaque `next_token`; empty string ends. Continuation MUST repeat SARC unchanged or HTTP 400. HTTP 400 for other `resource.type`.

### PR 7 — `authnz-rs` issuance PEP (optional)

- **Title:** `feat(oidc): AuthZEN evaluation before cwt::mint`
- **Files:** `authnz-rs/src/oidc.rs`
- **Depends on:** PR 3
- **Description:** AuthZEN action `issue:cwt:{grant}`; OpenTDF action `issue_cwt_{grant}` (e.g. `issue_cwt_authorization_code`). Fail closed. Do not put AuthZEN JSON in the CWT. Do not use phase 6 Search to fill roles (no classifier yet).

### Explicitly deferred

- CBOR/CoAP AuthZEN binding
- KAS NanoTDF CBOR rewrap calling AuthZEN
- `arkavo-node` AuthZEN mapping
- Built-in facade evaluator
- Swift AuthZEN clients
- Replacing OpenTDF v2 with Cerbos/OPA
