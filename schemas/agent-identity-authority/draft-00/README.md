# Agent Identity Authority (AIA) JSON Schemas

**Version:** 0.1.0-draft (draft-00)
**JSON Schema Draft:** 2020-12
**Specification:** [draft-arkavo-aia-00](../../../agent-identity-authority/draft-arkavo-aia-00.md)

## Overview

JSON Schema definitions for Agent Identity Authority, enabling validation of
Agent Identity Documents (AIDs), Model Attestation blocks, and CAWG agent
identity assertions.

## Schema Files

| File | Description |
|------|-------------|
| `aia.schema.json` | Core `$defs` shared across all schemas — primitives (`Did`, `SpiffeId`, `RFC3339`, `Semver`, `Nonce`, `Base64Url`, `Multihash`, `Blake3Digest`, `FlightId`, `RoleId`, `RoleType`, `Jwk`, `ModelInfo`). |
| `agent-identity-document.schema.json` | Top-level Agent Identity Document — the signed, content-addressed identity record (spec §4). |
| `model-attestation.schema.json` | Model Attestation block (spec §5) — `model`, `weights`, `evidence`. Embedded in an AID; also validatable standalone. |
| `cawg-agent-identity-assertion.schema.json` | The `cawg.agent_identity` C2PA assertion (spec §8) — a compact projection of an AID for content provenance. |

`agent-identity-document.schema.json` `$ref`s `model-attestation.schema.json` by
bare filename (without `#/...`) to embed the entire block, the same pattern the
SwarmKit schemas use for `agent-provisioning.schema.json`.

## Cross-Block Validations Not Expressible in JSON Schema

These invariants from the specification cannot be checked by the schemas alone —
they MUST be enforced by the Agent Identity Authority at issuance and by
verifiers at presentation:

1. `aid.id` equals `BLAKE3` of the JCS-canonical AID excluding `aid.id` and `signature` (§4.12).
2. `signature.value` verifies against the Authority key resolved from `signature.authority_did`, validated up the published trust chain (§4.12, §3.4).
3. `signature.trust_anchor` resolves to a Root Identity Agent trusted by the verifier and independent of the orchestrator (§3.4, §4.11).
4. `subject.did` and `subject.spiffe_id` share the same authority and agent (or flight/role) components (§7.3).
5. When `flight_binding` is present, `flight_binding.kit_id` equals the `kit.id` of the SwarmKit being executed and `flight_binding.flight_id` matches the flight of presentation (§4.5, §6.3).
6. `delegation.act` equals `subject.did` (§4.9).
7. `delegation.chain` length plus one does not exceed `delegation.may_act.max_depth` before a further hop is issued (§4.9, §6.4).
8. `weights.digest` matches the expected/pinned model digest, when one is known (§5.7 V-M1).
9. For `shard_layout: sharded`, `weights.digest` equals `weights.merkle_root` (§5.2.1).
10. `evidence.nonce` matches the nonce expected for this issuance (§5.7 V-M2).
11. When `owner.binding` is `owner_signed`, `owner.owner_signature` verifies over the canonical `subject` + `owner` blocks (§4.4).
12. `aid.expires` is in the future; `aid.id` is absent from any reachable revocation list (§9.3, §9.4).
13. The AID carries no attribute outside the `agent.identity.*` namespace; a KAS accepts only `agent.identity.*` attributes from a credential (§3.5).

## Usage

### Validation with ajv (Node.js)

```javascript
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

const core      = await import('./aia.schema.json', { assert: { type: 'json' } });
const attest    = await import('./model-attestation.schema.json', { assert: { type: 'json' } });
const aid       = await import('./agent-identity-document.schema.json', { assert: { type: 'json' } });
const assertion = await import('./cawg-agent-identity-assertion.schema.json', { assert: { type: 'json' } });

ajv.addSchema(core.default);
ajv.addSchema(attest.default);
ajv.addSchema(aid.default);
ajv.addSchema(assertion.default);

const validate = ajv.getSchema(aid.default.$id);
if (validate(myAid)) {
  console.log('Valid Agent Identity Document');
} else {
  console.log('Validation errors:', validate.errors);
}
```

### Validation with jsonschema (Python)

```python
import json
from referencing import Registry, Resource
from referencing.jsonschema import DRAFT202012
from jsonschema import Draft202012Validator

def load(path):
    with open(path) as f:
        return Resource(contents=json.load(f), specification=DRAFT202012)

core      = load('aia.schema.json')
attest    = load('model-attestation.schema.json')
aid       = load('agent-identity-document.schema.json')
assertion = load('cawg-agent-identity-assertion.schema.json')

registry = Registry().with_resources([
    (core.contents['$id'], core),
    (attest.contents['$id'], attest),
    (aid.contents['$id'], aid),
    (assertion.contents['$id'], assertion),
    # bare-filename refs used by the schemas
    ('aia.schema.json', core),
    ('model-attestation.schema.json', attest),
    ('agent-identity-document.schema.json', aid),
    ('cawg-agent-identity-assertion.schema.json', assertion),
])

validator = Draft202012Validator(aid.contents, registry=registry)
errors = sorted(validator.iter_errors(my_aid), key=lambda e: e.path)
if not errors:
    print("Valid Agent Identity Document")
else:
    for error in errors:
        print(f"Error at {list(error.path)}: {error.message}")
```

## Versioning

- **URL Pattern:** `https://github.com/arkavo-org/specifications/schemas/agent-identity-authority/{VERSION}/`
- **Current Version:** `draft-00`

When the specification is updated, create a new version directory, copy and
update the schemas, and update the `$id` URLs. Per repo convention, never edit
existing version directories in place.

## Conformance Testing

| Level | Required Schemas | Use Case |
|-------|------------------|----------|
| Level 1 (Issuer) | all four | Agent Identity Authorities that issue AIDs and emit assertions |
| Level 2 (Verifier) | `agent-identity-document`, `model-attestation`, `aia` | KAS, MCP brokers, A2A peers verifying AIDs |
| Level 3 (Model Attestor) | `model-attestation`, `aia` | Components producing model attestation evidence |

## License

Apache 2.0 — See [LICENSE](../../../LICENSE)
