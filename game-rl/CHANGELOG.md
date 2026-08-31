# Game-RL Specification Changelog

## 3.0.0-draft03 (2026-06-12) — `draft-arkavo-game-rl-03.md`

Promotes four conventions now field-validated across two adapters (RimWorld systemic + Project Zomboid embodied).

### Added
- **RenderMap viewport** (§5.9, REQ-VIS-01): read-only ASCII viewport centered on the agent's Reference Point with absolute coordinate rulers; coordinates read off it are valid action anchors. The standard textual-perception channel for LLM agents; SHOULD at Level 2.
- **Structured construction & one-shot actions** (§7.8, REQ-BLD-01): structural primitives over coordinate spam; one-shot actions that cannot strand intermediate state. Normative finding: rule-driven conductors do not reliably read error text or follow prompt conditionals, so sequencing constraints MUST be enforceable adapter-side.

### Strengthened
- **REQ-ERR-03** (§3.4): partial successes MUST report the shortfall and SHOULD carry the engine's rejection reasons; a partial reported as plain success is non-conformant.
- **REQ-SPA-03** (§7.4): the spatial Reference Point is the agent's locus and depends on agency scope — activity centroid for systemic agents, avatar position for embodied agents; never the map center. Validated across both adapters.

### Conformance
- New checks C-RENDERMAP (L2) and C-ERR-PARTIAL (L1).

## 2.0.0-draft02 (2026-06-11) — `draft-arkavo-game-rl-02.md`

The first revision codifying the dialect proven in released implementations (game-rl 0.6.x for RimWorld and Project Zomboid, consumed by Arkavo Edge). draft-00 is preserved unrevised for historical reference; the **draft-01 designation is intentionally unused** (reserved against interim internal revisions of draft-00 — the numbering gap marks the dialect break).

### Breaking (dialect)
- **Tool names are camelCase, payload fields PascalCase** (§1.5). Rationale: small local models (≤8B) drift on snake_case identifiers; PascalCase matches the .NET runtimes most adapters patch; MCP itself is camelCase at the protocol layer. Migration tables in Appendix B.
- `sim_step` → `step`; parameterized actions flatten `params` beside `Type`.
- Agent types canonicalized to the released set: `Observer | Player | Entity | Controller | System | Director` (Appendix B.3 maps draft-00 archetypes).

### Added
- `observe` (read-only state, `Include`/`Limit` section filtering) — core Level 1 tool.
- `episodeSummary` — episode-boundary metrics for trajectory consolidation/lesson synthesis.
- `manifest` as a **tool** (resource `game://manifest` now optional) — tool-only MCP clients can read capabilities.
- `resolveSpatial` dry-run tool.
- **Spatial Intents v2** (§7): anchor grammar (entity ID, zone label, type name, landmark ID, direction/distance offset, coordinate escape hatch), mandatory landmark discovery (compass regions + centroids + clusters), and resolution-integrity rules — no silent re-anchoring (REQ-SPA-02), reference point is the agent's activity centroid, never the map center (REQ-SPA-03), full audit trail in `ResolvedPlacement` (REQ-SPA-06). Fixes the observed "everything gets placed at map center" degenerate policy.
- **Loud-error requirements** (REQ-ERR-01/02): failed actions must surface as errors; silent log-and-continue is non-conformant.
- Observation contract: `ValidActions` (dynamic action space), `Alerts` (severity-ranked attention), `Landmarks`; "observation diet" guidance (compact step responses, full state via `observe`).
- Input robustness layer (§12): casing/semantic alias normalization, fuzzy action matching, mandatory correction reporting; optional draft-00 alias window (REQ-ROB-04).
- `Ticks: 0` action-only steps.
- Error code `-32005` (spatial resolution failed) with `Alternatives` payload.
- Conformance rebuilt around a runnable suite (`game-rl-conformance` + `game-rl-reference`), with check IDs mapped to requirement IDs (§14.2).

### Changed
- Conformance levels reorganized: L1 = "an LLM agent can play", L2 = "an RL pipeline can train", L3 = full.
- `gameRlVersion`: `2.0.0-draft02`.

## 1.0.0-draft (2025-12-28) — `draft-arkavo-game-rl-00.md`
- Initial draft (snake_case dialect).
