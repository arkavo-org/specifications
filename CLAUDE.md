# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Nature of this repository

This is a **specifications-only** repository — Markdown protocol drafts plus JSON Schema definitions. There is no source code, build system, package manager, or test runner. "Tasks" are almost always edits to spec text or schemas, not code changes.

Do not invent build/lint/test commands. There are none. Validation, when it happens, is JSON Schema validation done in downstream consumer projects (see `schemas/game-rl/draft-00/README.md` for `ajv` / `jsonschema` examples).

## Repository layout

Each top-level directory is one specification. Specs are independent — there is no shared toolchain, but they reference each other conceptually:

- `tdf-json/`, `tdf-cbor/` — Trusted Data Format serializations (JSON, CBOR). CBOR is the size-optimized counterpart of JSON; both must remain semantically equivalent. Cross-format edits (e.g., changing a field type or enum) must land in both.
- `ntdf-token/` — **SUPERSEDED.** Historical NanoTDF-based auth tokens (would have replaced JWT Bearer). Access tokens are CWT; see `authzen-cwt/`. Includes `NTDF_E2E_TEST_PLAN.md`. Keep the file; do not treat it as current.
- `authzen-cwt/` — CWT Subject Profile and AuthZEN 1.0 facade over OpenTDF Authorization v2. IETF-style filename `draft-arkavo-authzen-cwt-00.md`.
- `ntdf-rtmp/` — NanoTDF policy manifests over RTMP. Multiple drafts coexist (`draft-arkavo-ntdf-rtmp-00..02.md`); only the highest-numbered draft is current.
- `game-rl/` — Multi-agent AI interface for game environments, JSON-RPC 2.0 over MCP.
- `agent-runtime-policy/` — ARP, the runtime-adaptation companion to ADL. Largest spec (~77KB). Bound to ADL via `adl_ref` and signed with JCS canonicalization.
- `torg-decision/` — TØR-G token-stream IR for LLM policy synthesis.
- `schemas/<spec>/draft-NN/` — JSON Schema (Draft 2020-12) definitions for specs that have machine-readable schemas (currently `game-rl` and `agent-runtime-policy`).

## Spec authoring conventions

- **Drafts are append-only.** Bumping a draft means creating a new file (`draft-01.md` / `draft-arkavo-<name>-NN.md`), not overwriting the prior one. Existing drafts (e.g., `ntdf-rtmp/draft-arkavo-ntdf-rtmp-00.md`) are kept for history.
- **Filename patterns differ by spec** — keep the existing pattern when adding a new draft to an existing spec:
  - `tdf-json/`, `tdf-cbor/`, `agent-runtime-policy/` use short forms (`draft-00.md`, `arp-spec-draft-00.md`)
  - `ntdf-*`, `game-rl/`, `torg-decision/`, `authzen-cwt/` use full IETF Internet-Draft naming (`draft-arkavo-<name>-NN.md`)
- **Two header styles coexist.** Some drafts use kramdown-rfc YAML frontmatter inside a fenced ` ```text ` block (see `ntdf-rtmp/draft-arkavo-ntdf-rtmp-02.md`); others use plain Markdown with a metadata table (`agent-runtime-policy/arp-spec-draft-00.md`) or simple labeled lines (`tdf-cbor/draft-00.md`). Match the existing style of the file being edited — do not convert between styles.
- **RFC 2119 / BCP 14 keywords** (MUST, MUST NOT, SHOULD, MAY, …) are normative and case-sensitive. Only use them in their RFC sense; use lowercase ("must", "should") for non-normative prose.
- **Cross-spec edits**: TDF-JSON ↔ TDF-CBOR must stay semantically equivalent; ARP references ADL via `adl_ref`; Game-RL spec text and `schemas/game-rl/draft-00/*.json` must stay in sync — schema `$id` URLs follow `https://github.com/arkavo-org/specifications/schemas/<spec>/<version>/`.
- When bumping a schema version, create a new `schemas/<spec>/draft-NN/` directory (don't edit existing version dirs in place) and update `$id` URLs accordingly.

## Validating JSON Schema changes

If a change touches `schemas/**/*.json`, validate the JSON is well-formed before committing. There is no in-repo runner; use whatever is available:

```bash
# syntactic check only
python3 -m json.tool schemas/game-rl/draft-00/game-rl.schema.json > /dev/null

# full Draft 2020-12 validation if ajv-cli is installed globally
ajv compile -s schemas/game-rl/draft-00/game-rl.schema.json --spec=draft2020
```

The README in `schemas/game-rl/draft-00/` documents the canonical validation flow for downstream implementers — keep it accurate when schemas change.

## Commits

Conventional Commits with `docs:` (or `docs(<spec>):`) prefix — every commit in this repo is a docs change. Examples from history: `docs: Add TDF-JSON and TDF-CBOR format specifications draft-00`, `docs(tdf-cbor): Use integer enums for ~52 byte size savings`. Keep the README's "Latest Draft" links updated when a new draft supersedes a previous one.
