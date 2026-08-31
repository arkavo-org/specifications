# Game-RL Protocol JSON Schemas — draft-02

**Version:** 2.0.0-draft02
**JSON Schema Draft:** 2020-12
**`$id` base:** `https://arkavo.org/schemas/game-rl/draft-02/`
**Specification:** [draft-arkavo-game-rl-02](../../../game-rl/draft-arkavo-game-rl-02.md) (change log in Appendix C)

## Schema Files

| File | Description |
|------|-------------|
| `game-rl.schema.json` | Core `$defs` shared across all schemas (`AgentId`, `AgentType`, `Scope`, `ClockMode`, `SessionType`, `Position`, `StateHash`, `Severity`, `Anchor`, `Event`) |
| `manifest.schema.json` | `manifest` tool result |
| `register-agent.schema.json` | `registerAgent` request/response (AgentManifest) |
| `step.schema.json` | `step` request and StepResult |
| `observe.schema.json` | `observe` request and contract sections (`ValidActions`, `Alerts`, `Landmarks`) |
| `reset.schema.json` | `reset` request and response |
| `spatial.schema.json` | Spatial Intents v2: landmarks, intent catalog, `ResolvedPlacement`, `-32005` error data, `resolveSpatial` request |
| `episode-summary.schema.json` | `episodeSummary` result |
| `events.schema.json` | Event objects and `notifications/event` broadcasts |
| `vision-stream.schema.json` | `configureStreams` request and Stream Descriptors |

## Casing Rules (spec §1.5, normative)

- **Tool names** are camelCase: `registerAgent`, `step`, `observe`, `episodeSummary`, `resolveSpatial`.
- **Payload fields** (tool arguments and results) are **PascalCase**: `AgentId`, `Action`, `Ticks`, `Reward`, `Done`, `StateHash`. Action types and their parameters are PascalCase too.
- **MCP protocol-level fields** keep MCP's conventions (`protocolVersion`, `inputSchema`); JSON Schema keywords inside these files use standard lowercase spelling (`type`, `properties`, `description`).

Cross-file references use relative `$ref`s, e.g. `"$ref": "game-rl.schema.json#/$defs/AgentId"`.

## Validation Example (ajv, Node.js)

```javascript
import Ajv from "ajv/dist/2020.js";
import { readFileSync } from "node:fs";

const load = (f) => JSON.parse(readFileSync(new URL(f, import.meta.url)));

const ajv = new Ajv({ allErrors: true });
for (const f of ["game-rl", "spatial", "step"]) {
  ajv.addSchema(load(`./${f}.schema.json`));
}

const validate = ajv.getSchema(
  "https://arkavo.org/schemas/game-rl/draft-02/step.schema.json#/$defs/request"
);

const request = {
  AgentId: "player1",
  Action: { Type: "PlaceBuildingNear", Building: "Bed", Near: "Stockpile", Count: 3, Stuff: "WoodLog" },
  Ticks: 60,
};

console.log(validate(request) ? "valid" : validate.errors);
```

## License

Apache 2.0 — see [LICENSE](../../../LICENSE)
