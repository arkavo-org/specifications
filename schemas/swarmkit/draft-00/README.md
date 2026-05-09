# SwarmKit JSON Schemas

**Version:** 1.0.0-draft (draft-00)
**JSON Schema Draft:** 2020-12
**Specification:** [swarmkit-spec-draft-00](../../../swarmkit/swarmkit-spec-draft-00.md)

## Overview

JSON Schema definitions for the SwarmKit specification, enabling validation of manifests, agent_provisioning blocks, and delegation envelopes.

## Schema Files

| File | Description |
|------|-------------|
| `swarmkit.schema.json` | Core `$defs` shared across all schemas — primitives (`Did`, `RFC3339`, `Semver`, `Nonce`, `Multihash`, `KitId`, `RoleId`, `FlightId`), enums (`Topology`, `AuthMode`, …), and sub-specs (`KitMetadata`, `Objective`, `InputSpec`, `DeliverableSpec`, `Skill`, `McpToolGrant`, `TdfAttributeReleasePolicy`, `Handoff`, `ContextScope`, `CoordinationSpec`, `ConstraintsSpec`, `EvaluationSpec`, `CompletionSpec`, `ProvenanceSpec`). `RoleType` is a free-form string (spec §4.3, Appendix C). |
| `manifest.schema.json` | Top-level SwarmKit manifest — the cleartext document inside the TDF payload (spec §4) |
| `agent-provisioning.schema.json` | Per-role provisioning block (spec §5) — `model`, `inference`, `budget`, `tool_use`, `context`, `observability`, `isolation`, `failure`. Distinct from the companion Agent Runtime Policy spec (which governs adaptation of a running specialist). |
| `delegation-envelope.schema.json` | Orchestrator → specialist delegation envelope (spec §7.2). JSON wire format; signature over JCS-canonical form. |

## Cross-Block Validations Not Expressible in JSON Schema

The following invariants from the spec cannot be checked by the schemas alone — they MUST be enforced by the orchestrator at provisioning time:

1. `inference.max_tokens` ≤ model context window minus prompt overhead (§5.1)
2. `agent_provisioning.budget.*` ≤ corresponding `constraints.global_budget.*` (§5.1)
3. `agent_provisioning.isolation.network_egress: true` is denied when `constraints.network.egress_allowed: false` (§5.1)
4. `model.family` is in the orchestrator's supported set (§5.1)
5. `agent_provisioning.failure.fallback_role` resolves to an existing role within `manifest.roles`
6. Every `handoffs[].to` and `evaluation.critic_role` resolves to an existing role
7. `kit.id` equals `BLAKE3(canonical_manifest)` (§9.1)
8. `kit.expires - kit.created` does not exceed the orchestrator's accepted horizon (§10.1; MUST ≤ 1 year, SHOULD ≤ 90 days)
9. `evaluation.rubric.dimensions[].weight` sum equals 1.0 within floating-point tolerance (§4.6)
10. When `evaluation.critic_role` resolves to a role evaluating its own output, the orchestrator tags the result `self_evaluated: true` in the DecisionTrace (§10.1)

## Usage

### Validation with ajv (Node.js)

```javascript
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

// Load all schemas — order matters for $ref resolution
const core = await import('./swarmkit.schema.json', { assert: { type: 'json' } });
const provisioning = await import('./agent-provisioning.schema.json', { assert: { type: 'json' } });
const manifest = await import('./manifest.schema.json', { assert: { type: 'json' } });
const envelope = await import('./delegation-envelope.schema.json', { assert: { type: 'json' } });

ajv.addSchema(core.default);
ajv.addSchema(provisioning.default);
ajv.addSchema(manifest.default);
ajv.addSchema(envelope.default);

const validate = ajv.getSchema(manifest.default.$id);
if (validate(myManifest)) {
  console.log('Valid SwarmKit manifest');
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

core = load('swarmkit.schema.json')
provisioning = load('agent-provisioning.schema.json')
manifest = load('manifest.schema.json')
envelope = load('delegation-envelope.schema.json')

registry = Registry().with_resources([
    (core.contents['$id'], core),
    (provisioning.contents['$id'], provisioning),
    (manifest.contents['$id'], manifest),
    (envelope.contents['$id'], envelope),
    # bare-filename refs used by the schemas
    ('swarmkit.schema.json', core),
    ('agent-provisioning.schema.json', provisioning),
    ('manifest.schema.json', manifest),
    ('delegation-envelope.schema.json', envelope),
])

validator = Draft202012Validator(manifest.contents, registry=registry)
errors = sorted(validator.iter_errors(my_manifest), key=lambda e: e.path)
if not errors:
    print("Valid SwarmKit manifest")
else:
    for error in errors:
        print(f"Error at {list(error.path)}: {error.message}")
```

## Schema References

Schemas use `$ref` to share types via `swarmkit.schema.json`:

```json
{
  "model": {
    "family": { "$ref": "swarmkit.schema.json#/$defs/ModelFamily" }
  }
}
```

`manifest.schema.json` and `delegation-envelope.schema.json` also `$ref` `agent-provisioning.schema.json` directly (without `#/...`) to embed the entire policy block.

## Versioning

Schemas are versioned alongside the specification:

- **URL Pattern:** `https://github.com/arkavo-org/specifications/schemas/swarmkit/{VERSION}/`
- **Current Version:** `draft-00`

When the specification is updated:

1. Create new version directory (e.g., `draft-01/` or `1.0.0/` for release).
2. Copy and update schemas.
3. Update `$id` URLs in each schema to the new version.
4. Update the specification's references to point at the new schema directory.

Per repo convention, never edit existing version directories in place.

## Conformance Testing

Implementations MUST pass validation against these schemas at the indicated levels:

| Level | Required Schemas | Use Case |
|-------|------------------|----------|
| Level 1 (Producer) | `manifest`, `agent-provisioning`, `swarmkit` | Tools that build SwarmKits |
| Level 2 (Orchestrator) | All four | Tools that decrypt, validate, and delegate |
| Level 3 (Specialist) | `delegation-envelope`, `agent-provisioning`, `swarmkit` | Tools that receive delegations |

## License

Apache 2.0 — See [LICENSE](../../../LICENSE)
