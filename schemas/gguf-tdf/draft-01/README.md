# gguf-tdf/1 JSON Schemas

**Version:** 0.2.0-draft (draft-01)
**JSON Schema Draft:** 2020-12
**Specification:** [draft-arkavo-gguf-tdf-01](../../../gguf-tdf/draft-arkavo-gguf-tdf-01.md)

Wire profile remains `gguf-tdf/1`. The index schema is unchanged from [draft-00](../draft-00/) except `$id`. Discovery, KAS transport, error-code, and segment-boundary (tensor-aligned packing, §11.3) changes in draft-01 are not expressible here.

## Overview

Machine-readable schema for the hybrid `gguf` object that `gguf-tdf/1` adds to an OpenTDF zip TDF manifest. OpenTDF `payload` and `encryptionInformation` are defined by [OpenTDF 4.3.0](https://github.com/opentdf/spec); this directory does not fork that schema.

## Schema Files

| File | Description |
|------|-------------|
| `gguf-index.schema.json` | The top-level `gguf` object (spec §9). Validate `manifest.gguf`. |
| `example-manifest.json` | Complete example manifest matching Appendix A (shape only; dummy crypto). `gguf` validates against the schema. Packing plan is T15. |
| `appendix-a-packing-plan.json` | Normative expected `gguf.segments[]` / tensors for Appendix A (`>=` packing). |
| `tensor-aligned-packing-plan.json` | Normative expected plan for the tensor-aligned vector (spec §15.5, T20): two 96 B tensors at a 128 B cap are two `kind=tensor` members; a draft-00 fixed-window writer would split the second. |

## Invariants Not Expressible in JSON Schema

The following MUST be enforced by the reader at open (spec §9.4, §11):

1. `sum(gguf.segments[i].plain) == gguf.virtualSize`
2. Segments partition `[0, virtualSize)` with `segments[0]` covering `[0, headerBytes)`
3. `segments[0].kind == "header"`, `entry == "header"`, `plain == headerBytes`, `id == 0`; `kind=header` only at index 0
4. `len(gguf.segments) == len(encryptionInformation.integrityInformation.segments)`
5. For each `i`, `plain == integrityInformation.segments[i].segmentSize` and `encryptedSegmentSize == segmentSize + 28`
6. Zip central directory contains each `segments[i].entry` (exact UTF-8 `s/{k}` with `/`) with uncompressed size `encryptedSegmentSize`
7. `alignment` is a power of two and a multiple of 8 ([GGUF] requires multiple of 8; this profile also requires power of two, spec §7.4); `headerBytes` and `maxSegment` are multiples of `alignment` (schema `headerBytes` minimum 24 is not a multiple of default ALIGN 32)
8. Non-header `plain <= maxSegment`; header `plain` MAY exceed `maxSegment`
9. Tensor `segments` ranges are half-open `[start, end)` with `start >= 1` and `end > start` (`[1,1]` is invalid); cover each tensor's virtual range; do not include the header
10. Tensor names unique; UTF-8 **byte** length ≤ 64 (schema `maxLength` is characters); `offset` strictly increasing; `offset >= headerBytes`
11. After header decrypt: `gguf_get_data_offset == headerBytes` and tensor names/offsets/sizes match the GGUF header (spec §9.5)

## Usage

### Validation with ajv (Node.js)

```javascript
import Ajv from 'ajv/dist/2020.js';
import addFormats from 'ajv-formats';
import { readFileSync } from 'node:fs';

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

const schema = JSON.parse(readFileSync('./gguf-index.schema.json', 'utf8'));
const manifest = JSON.parse(readFileSync('./example-manifest.json', 'utf8'));
const validate = ajv.compile(schema);

if (validate(manifest.gguf)) {
  console.log('Valid gguf-tdf/1 index');
} else {
  console.log('Validation errors:', validate.errors);
}
```

### Validation with jsonschema (Python)

```python
import json
from jsonschema import Draft202012Validator

with open("gguf-index.schema.json") as f:
    schema = json.load(f)
with open("example-manifest.json") as f:
    manifest = json.load(f)

validator = Draft202012Validator(schema)
errors = sorted(validator.iter_errors(manifest["gguf"]), key=lambda e: list(e.path))
if not errors:
    print("Valid gguf-tdf/1 index")
else:
    for error in errors:
        print(f"Error at {list(error.path)}: {error.message}")
```

## Versioning

- **URL pattern:** `https://github.com/arkavo-org/specifications/schemas/gguf-tdf/{VERSION}/`
- **Current version:** `draft-01`
- **Previous:** `draft-00` (index schema identical except `$id`)

When the specification is updated:

1. Create a new version directory (`draft-02/` or `1.0.0/` for release).
2. Copy and update schemas.
3. Update `$id` URLs.
4. Point the specification at the new directory.

Per repo convention, never edit existing version directories in place.

## Conformance Testing

| Level | Required | Use case |
|-------|----------|----------|
| Level 1 (Producer) | `gguf-index.schema.json` on emitted `manifest.gguf`, plus §9.4 invariants | `arkavo model protect` / packer |
| Level 2 (Reader) | Level 1 + zip layout + GMAC/root-sig checks in the spec | Inference load path for an explicit `.gguf.tdf` |
| Level 3 (Full) | Level 2 + callback logit match (spec §15.4) | Edge integration |

Discovery rules (plaintext-first when both artifacts exist) are spec §13.1 / T12 / T14 / T19, not this schema.

## License

Apache 2.0 — See [LICENSE](../../../LICENSE)
