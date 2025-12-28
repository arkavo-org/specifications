# Game-RL Protocol JSON Schemas

**Version:** 1.0.0-draft (draft-00)
**JSON Schema Draft:** 2020-12
**Specification:** [draft-arkavo-game-rl-00](../../../game-rl/draft-arkavo-game-rl-00.md)

## Overview

This directory contains JSON Schema definitions for the Game-RL Protocol, enabling validation of protocol messages and implementation conformance testing.

## Schema Files

| File | Description |
|------|-------------|
| `game-rl.schema.json` | Core definitions (`$defs`) shared across all schemas |
| `register-agent.schema.json` | Agent registration request/response |
| `sim-step.schema.json` | Simulation step request/observation response |
| `reset.schema.json` | Episode reset request |
| `manifest.schema.json` | `game://manifest` resource schema |
| `events.schema.json` | Event broadcast notifications |
| `vision-stream.schema.json` | Vision stream configuration |

## Usage

### Validation with ajv (Node.js)

```javascript
import Ajv from 'ajv/dist/2020';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

// Load schemas
const gameRl = await import('./game-rl.schema.json');
const registerAgent = await import('./register-agent.schema.json');

ajv.addSchema(gameRl);
const validate = ajv.compile(registerAgent.definitions.request);

// Validate a registration request
const request = {
  agent_id: "claude:game_master",
  agent_type: "GameMaster",
  scope: "systemic",
  config: {
    capabilities: ["admin", "narrative", "spawn"]
  }
};

if (validate(request)) {
  console.log('Valid registration request');
} else {
  console.log('Validation errors:', validate.errors);
}
```

### Validation with jsonschema (Python)

```python
import json
from jsonschema import Draft202012Validator, RefResolver

# Load schemas
with open('game-rl.schema.json') as f:
    game_rl = json.load(f)
with open('register-agent.schema.json') as f:
    register_agent = json.load(f)

# Create resolver for $ref
resolver = RefResolver.from_schema(game_rl)

# Validate request
validator = Draft202012Validator(
    register_agent['definitions']['request'],
    resolver=resolver
)

request = {
    "agent_id": "chatgpt:companion",
    "agent_type": "EntityBehavior",
    "scope": "embodied",
    "config": {
        "avatar_id": "npc_ranger_01"
    }
}

errors = list(validator.iter_errors(request))
if not errors:
    print("Valid registration request")
else:
    for error in errors:
        print(f"Error: {error.message}")
```

## Schema References

All schemas use `$ref` to reference shared definitions in `game-rl.schema.json`:

```json
{
  "agent_id": {
    "$ref": "game-rl.schema.json#/$defs/AgentId"
  }
}
```

## Versioning

Schemas are versioned alongside the specification:

- **URL Pattern:** `https://github.com/arkavo-org/specifications/schemas/game-rl/{VERSION}/`
- **Current Version:** `draft-00`

When the specification is updated:
1. Create new version directory (e.g., `draft-01/` or `1.0.0/` for release)
2. Copy and update schemas
3. Update `$id` URLs in each schema
4. Update specification Appendix A reference

## Conformance Testing

Implementations MUST pass validation against these schemas for:

| Level | Required Schemas |
|-------|------------------|
| Level 1 (Minimal) | `manifest`, `register-agent`, `sim-step`, `reset` |
| Level 2 (Standard) | Level 1 + `events`, `vision-stream` |
| Level 3 (Full) | All schemas |

## License

Apache 2.0 - See [LICENSE](../../../LICENSE)
