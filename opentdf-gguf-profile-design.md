# OpenTDF / GGUF profile design (`gguf-tdf/1`)

**Status:** Informative — **superseded on wire fields** by [`gguf-tdf/draft-arkavo-gguf-tdf-00.md`](gguf-tdf/draft-arkavo-gguf-tdf-00.md)  
**Date:** 2026-08-28  
**Companion:** [llama.cpp loader callback handover](llama-cpp-loader-callback-handover.md) (not in this repo; current upstream surface is `gguf_init_from_callback` / `gguf_reader_callback_t` in `ggml/include/gguf.h`)

This document is product/layout rationale only. **On-the-wire fields, packing, and load rules live only in draft-00.** In particular, conforming writers MUST emit `payload.mimeType` = `application/x-gguf` and `tdf_spec_version` / `payload.tdf_spec_version` / `schemaVersion` = `"4.3.0"`. Examples below that still show historical values are non-normative.

This profile defines how a GGUF becomes a `.gguf.tdf`, how KAS binds policy to the weights, and how Arkavo decrypts **one segment at a time** into llama.cpp’s virtual-GGUF reader.

It is an Arkavo profile of OpenTDF. It is **not** required that `otdfctl decrypt` emit a vanilla `.gguf`. Other SDKs that only know a single `0.payload` will not load this archive as a GGUF; they can still parse `0.manifest.json` and perform KAS rewrap.

## Goals

- At rest, the only on-disk model artifact is ciphertext (plus the TDF manifest).
- At load, plaintext exists only as: GGUF header, one TDF segment scratch (default 4 MiB), and the ggml/GPU weight buffers required for inference.
- One payload key, one KAS policy, AES-256-GCM per segment, GMAC root signature — same crypto as zip TDF today.
- Variable segment sizes aligned to GGUF layout (header / packed small tensors / split large tensors).
- llama.cpp sees a virtual linear GGUF via a generic `read_at` callback (`gguf_reader_callback_t`). It never opens the zip. (There is no `llama_model_load_from_callback` in current public `llama.h`; tensor bytes still need a callback-shaped loader path.)

## Non-goals

- Encrypting weights in VRAM after load.
- NanoTDF (size-capped, not for GB files).
- ZTDF-JSON / `TdfJsonRpc` inline base64 (`arkavo-tdf` `OpenTdfService` today). That path is for small SwarmKit and MCP payloads.
- Writing a plaintext temp `.gguf`.
- Split-GGUF (`NNNNN-of-MMMMM`, 5-digit [GGUF] naming) in v1 as one archive. Each shard can be a separate `.gguf.tdf` later.
- Changing KAS wire protocol.

## Why not current opentdf-rs file APIs

Pinned `opentdf-rs` (`62b1fdf`) already has:

```text
Tdf::encrypt_file(input, output.tdf)  // zip TDF, Stored entries
Tdf::decrypt_file(path).to_file(...)
```

Both `std::fs::read` the entire file and keep ciphertext + plaintext in `Vec<u8>`. Segment size is a fixed 2 MiB cut of the whole buffer. `TdfEntry.payload` is the full concatenated blob.

`arkavo-tdf` `OpenTdfService` uses `opentdf::jsonrpc::TdfJsonRpc` (inline base64). `encrypt_stream` is `read_to_end` then encrypt. Decrypt returns “use decrypt_with_kas”.

None of that can wrap Gemma 12B / Ministral 8B. This profile adds a **GGUF-aware, multi-entry, per-segment** API on top of the same AES-GCM segment primitive (`TdfEncryption::encrypt_with_segments` / decrypt of one slice).

## Archive layout

`model.gguf.tdf` is a ZIP with **Stored** compression (no deflate), same as current opentdf-rs archives.

```
model.gguf.tdf
├── 0.manifest.json      # OpenTDF manifest + gguf hybrid index (plaintext JSON)
├── header               # segment 0: encrypted GGUF header (magic..data_offset)
└── s/1
    s/2
    ...
    s/{n}                # encrypted tensor segments, one zip member each
```

No `0.payload`. Random access is a zip central-directory lookup, not a scan of a 12 GB concatenated blob.

Each encrypted member is one OpenTDF segment:

```
[12-byte IV][AES-256-GCM ciphertext || 16-byte tag]
```

Plaintext length is `segmentSize`. On-disk length is `encryptedSegmentSize` = `segmentSize + 28`.

### Virtual GGUF

The **virtual file** the llama.cpp callback reads is byte-identical to the source `.gguf` before wrapping:

| Virtual range | Source |
|---|---|
| `[0, headerBytes)` | Decrypt `header` |
| `[headerBytes, virtualSize)` | Decrypt `s/{id}` covering that range, copy the overlap |

`virtualSize` = original GGUF file length. Padding between tensors (GGUF 32-byte alignment) is included in segments so offsets match `gguf_get_data_offset + gguf_get_tensor_offset`.

## Manifest

`0.manifest.json` is a standard OpenTDF `TdfManifest` plus a `gguf` object. Unknown-field-tolerant TDF parsers ignore `gguf`. Strict parsers that reject unknown keys need an opentdf-rs allow-list for this profile.

### OpenTDF fields (unchanged meaning)

- `payload.type`: `"reference"`
- `payload.url`: `"header"` (first payload member; not `0.payload`)
- `payload.protocol`: `"zip"`
- `payload.mimeType`: `"application/x-gguf"` (**draft-00**; not `application/x-gguf+tdf`)
- `payload.tdf_spec_version`: `"4.3.0"` (**draft-00**; not `"3.0.0"`)
- `payload.isEncrypted`: `true`
- `encryptionInformation.type`: `"split"`
- `encryptionInformation.method.algorithm`: `"AES-256-GCM"`
- `encryptionInformation.method.isStreamable`: `true`
- `encryptionInformation.method.iv`: empty (IVs live on segments)
- `encryptionInformation.keyAccess[]`: wrapped payload key, KAS URL, policy binding (HS256)
- `encryptionInformation.policy`: base64 policy JSON
- `encryptionInformation.integrityInformation.segmentHashAlg`: `"GMAC"`
- `encryptionInformation.integrityInformation.segments[]`: one entry **per zip member**, in order `header`, `s/1`, … with per-segment `segmentSize` and `encryptedSegmentSize` (variable; do not assume `segmentSizeDefault`)
- `encryptionInformation.integrityInformation.rootSignature`: HMAC-SHA256 over concatenated GMAC tags with the payload key

`segmentSizeDefault` / `encryptedSegmentSizeDefault` may be set to `maxSegment` / `maxSegment+28` for the common tensor-split case. **Readers must use per-segment sizes**, not the default, because the header segment and packed tail differ.

### Hybrid `gguf` object

```json
{
  "gguf": {
    "profile": "gguf-tdf/1",
    "alignment": 32,
    "headerBytes": 123456,
    "virtualSize": 4294967296,
    "maxSegment": 4194304,
    "tensors": [
      {
        "name": "token_embd.weight",
        "offset": 123456,
        "size": 16777216,
        "segments": [1, 5]
      },
      {
        "name": "blk.0.attn_norm.weight",
        "offset": 16900672,
        "size": 4096,
        "segments": [5, 6]
      }
    ],
    "segments": [
      { "id": 0, "kind": "header", "plain": 123456, "entry": "header" },
      { "id": 1, "kind": "tensor", "plain": 4194304, "entry": "s/1" },
      { "id": 5, "kind": "pack",   "plain": 262144,  "entry": "s/5" }
    ]
  }
}
```

`tensors[].segments` is a half-open index range into `gguf.segments` / `integrityInformation.segments` (same order, same length). `offset` / `size` are virtual GGUF coordinates.

`kind`:

- `header` — virtual `[0, headerBytes)`
- `tensor` — a split of one large tensor (or a single tensor that filled the cap)
- `pack` — consecutive small tensors (and their alignment padding) sharing one segment

The index is redundant with “decrypt header and parse GGUF” plus prefix-sum of `segmentSize`. It exists so the reader can map `read_at(offset, len)` → zip members **without** decrypting the header first (header decrypt still happens for llama.cpp metadata). Zip entry names are authoritative.

### Example minimal manifest (shape only)

```json
{
  "tdf_spec_version": "4.3.0",
  "payload": {
    "type": "reference",
    "url": "header",
    "protocol": "zip",
    "isEncrypted": true,
    "mimeType": "application/x-gguf",
    "tdf_spec_version": "4.3.0"
  },
  "encryptionInformation": {
    "type": "split",
    "keyAccess": [],
    "method": {
      "algorithm": "AES-256-GCM",
      "isStreamable": true,
      "iv": ""
    },
    "integrityInformation": {
      "rootSignature": { "alg": "HS256", "sig": "" },
      "segmentHashAlg": "GMAC",
      "segments": [],
      "segmentSizeDefault": 4194304,
      "encryptedSegmentSizeDefault": 4194332
    },
    "policy": ""
  },
  "schemaVersion": "4.3.0",
  "gguf": {
    "profile": "gguf-tdf/1",
    "alignment": 32,
    "headerBytes": 0,
    "virtualSize": 0,
    "maxSegment": 4194304,
    "tensors": [],
    "segments": []
  }
}
```

## Segment packing

Walk the source GGUF **without loading weights** (`gguf_init_from_file` with `no_alloc = true`, or a tiny Rust GGUF header parser).

Constants:

- `ALIGN = general.alignment` or 32
- `maxSegment = 4 MiB` default, recorded in the index; must be a multiple of `ALIGN`
- `headerBytes = gguf_get_data_offset(ctx)`

Algorithm:

```
emit segment 0: virtual [0, headerBytes) -> zip member "header"

cursor = headerBytes
pack_start = cursor

for each tensor in GGUF order:
    t0 = headerBytes + gguf_get_tensor_offset(t)
    t1 = t0 + gguf_get_tensor_size(t)
    # include alignment padding before this tensor in the current pack
    if t0 < cursor: error (overlapping tensors)
    # padding [cursor, t0) belongs to the upcoming segment

    remaining_off = t0
    remaining_end = t1
    while remaining_end - pack_start > maxSegment and remaining_end > remaining_off:
        take = maxSegment - (remaining_off - pack_start)  # if pack already open
        if pack_start == remaining_off:
            take = min(maxSegment, remaining_end - remaining_off)
            take -= take % ALIGN
            if take == 0: error (tensor smaller than ALIGN but > maxSegment is impossible)
        emit segment virtual [pack_start, pack_start + take) -> "s/{id}"
        pack_start += take
        remaining_off = pack_start
    # tensor tail stays in the open pack
    cursor = t1

if cursor > pack_start:
    emit final pack [pack_start, cursor)  # may include trailing file padding up to virtualSize

assert prefix_sum(segment.plain) == virtualSize
```

Consequences:

- Header is never packed with weights.
- A 16 MiB embedding with `maxSegment = 4 MiB` becomes four `kind=tensor` members.
- LayerNorm tensors (few KiB) share a `kind=pack` member; GCM overhead stays amortized.
- Peak decrypt scratch = `max(headerBytes, maxSegment)`.

Tokenizer KV lives in the header. If a vocab pushes `headerBytes` to tens of MiB, that is accepted: it is still far smaller than the model, and it is one decrypt.

## Wrap (encrypt) flow

Owner: `arkavo-tdf` + CLI (`arkavo model protect` or equivalent). Not the current MCP `tdf_encrypt` (that writes `.tdf.json` via `TdfJsonRpc`).

1. Parse GGUF header; refuse non-GGUF magic.
2. Build segment plan (above).
3. `TdfEncryption::new()` → payload key; wrap with KAS public key (existing opentdf-rs key access).
4. Policy from CLI/env (same FQN style as `PolicyBuilder` / `arkavo_attrs`).
5. Stream each virtual range from the source `.gguf` (file read of that range only) through `encrypt` of that slice; write zip member; record GMAC and sizes.
6. Write `0.manifest.json` last (needs root signature over all tags).
7. Do not delete the source `.gguf` unless the caller passes an explicit flag. Default: leave plaintext where it was; the protected artifact is additive.

Wrap must not load the whole GGUF. Implementation: `std::io::Read` + `Seek` of `[offset, offset+plain)` per segment.

## Unwrap-on-load flow

Owner: Edge `arkavo-llm` / a small `arkavo-tdf` helper. llama.cpp only sees the callback.

1. Path ends with `.gguf.tdf` (case-insensitive) or magic is ZIP + `gguf.profile == "gguf-tdf/1"`.
2. Open zip; parse `0.manifest.json`; fail closed if profile missing or version ≠ `gguf-tdf/1`.
3. Acquire OAuth token; `KasClient` rewrap → 32-byte payload key (`TdfEncryption::with_payload_key`). Fail closed on deny. No fallback to a sibling plaintext `.gguf`.
4. Decrypt `header` into a buffer of `headerBytes`; `gguf_init_from_buffer` optional on the Edge side for diagnostics; llama.cpp will also read header via the callback.
5. `llama_model_load_from_callback(read_at, userdata, virtualSize, maxSegment, params)` with userdata holding: zip handle (ciphertext, mmap of the **zip** is fine), segment table, payload key, one scratch buffer of `maxSegment + 28` for ciphertext and `maxSegment` for plaintext.
6. `read_at(offset, len)`:
   - Locate segments whose virtual ranges overlap `[offset, offset+len)`.
   - For each: zip-extract that member into ciphertext scratch (or mmap the zip entry if Stored and the crate exposes it), decrypt into plaintext scratch, copy the overlap into `output`.
   - Reuse scratch; `zeroize` plaintext scratch after each segment (and on drop).
   - Return bytes copied, or 0 on auth-tag failure.
7. On `LlamaModel` drop: zeroize payload key; drop zip; no temp files to unlink.

GPU offload (`n_gpu_layers = -1`) still happens inside llama.cpp. Destination buffers are backend memory. Extra RAM is header + one segment, not a second copy of the model.

### `read_at` sketch

```text
fn read_at(ud, dst, offset, len) -> usize:
    if offset >= ud.virtual_size: return 0
    len = min(len, ud.virtual_size - offset)
    written = 0
    while written < len:
        seg = segment_covering(offset + written)
        decrypt_segment(seg) -> ud.plain_scratch   # AES-GCM, verify tag
        local = (offset + written) - seg.virtual_start
        n = min(len - written, seg.plain - local)
        copy dst[written..] from ud.plain_scratch[local..local+n]
        zeroize used portion of ud.plain_scratch
        written += n
    return written
```

Segment covering is binary search on prefix sums of `plain` (monotonic virtual ranges).

## opentdf-rs work

New API on the zip/TDF crate (names indicative):

```text
Tdf::encrypt_segments(plan: impl Iterator<Item = (plain_slice, entry_name)>)
    -> writes zip members + manifest

TdfArchive::open(path) -> handle
TdfArchive::decrypt_entry(&self, entry, payload_key) -> plaintext bytes
    // bounded: caller provides dest buffer of segmentSize
```

`encrypt_with_segments(&[u8], fixed_size)` can stay for small payloads. The GGUF packer must pass **pre-cut slices of different lengths**, not `chunks(2MB)`.

`TdfManifest` in `opentdf-protocol` should deserialize unknown fields or an optional `gguf: serde_json::Value` / typed `GgufTdfIndex` behind a feature. Do not break existing SwarmKit JSON manifests.

Keep rustls-only; no OpenSSL. KAS feature remains `kas` / `kas-client`.

## Edge crate split

| Crate | Responsibility |
|---|---|
| `opentdf-rs` | Variable-size segments, multi-entry zip, `decrypt_entry`, optional `gguf` on manifest |
| `arkavo-tdf` | `gguf-tdf/1` packer + `read_at` userdata; KAS via existing `ArkavoKasClient` |
| `arkavo-llama-cpp` | `LlamaModel::from_callback` FFI (after handover lands) |
| `arkavo-llm` | If path is `.gguf.tdf`, KAS + callback load instead of `LlamaModel::from_file` |
| `arkavo-router` `model_discovery` | Recognize `.gguf.tdf` next to `.gguf` |
| CLI | `arkavo model protect` wrap; `--model foo.gguf.tdf` |

Do not add TDF to `arkavo-llama-cpp` (keep the C++ wrapper free of KAS). Glue lives above it.

`arkavo-config-encryption` stays on in-memory zip TDF for config bundles. Do not route GGUF through it.

## Policy and identity

Reuse existing Arkavo attribute FQNs. Recommended default for a protected model:

```text
https://arkavo.net/attr/data/clearance/value/{internal|confidential|restricted}
https://arkavo.net/attr/model/tier/value/{low_cost|standard|advanced|reasoning}
```

Dissemination optional (agent id / email). Offline: same story as `SECURITY.md` — a KAS in-environment (`identity.arkavo.net` / `100.arkavo.net` defaults, override URLs). No load if rewrap fails.

Manifest `payload` and `gguf` index are **plaintext**. They reveal architecture, tensor names, sizes, and KAS URL. Weights and tokenizer bytes in the header segment are encrypted. If tokenizer secrecy matters, it is already covered (header is encrypted). Tensor **names** are not secret in this profile.

## Security properties

| State | Plaintext GGUF |
|---|---|
| Disk at rest | No |
| Named temp file | No |
| Anonymous memfd/shm of full file | No |
| Zip mmap | Ciphertext only (OK) |
| `read_at` scratch | ≤ `maxSegment` (+ header once) |
| After load | ggml/GPU tensors (required) |
| Process dump | Weight buffers + key until drop (same as any local model) |

AES-GCM tag failure aborts the load (fail closed). Do not skip integrity “to keep loading.”

Root signature is verified after all segments if doing a full decrypt (tooling). On load, verify **per-segment GMAC/tag** (AES-GCM tag is the GMAC). Optional: verify root signature after the last tensor as a completeness check; not required for a partial `vocab_only` load.

## Error handling

| Condition | Behavior |
|---|---|
| Not zip / missing `0.manifest.json` | Error: not a TDF |
| `gguf.profile` missing or unknown | Error: unsupported profile |
| KAS unreachable / policy deny | Error; no plaintext fallback |
| Segment tag mismatch | Error; zeroize scratch |
| `virtualSize` ≠ sum of `plain` | Error at open |
| llama.cpp callback returns 0 mid-tensor | Existing loader error path |
| mmproj `.gguf.tdf` without callback `mtmd` API | Error until mtmd handover follow-up |

## Testing

Regression tests (Edge, no GPU required for pack/unpack):

- Pack a tiny fixture GGUF (or a few-KiB synthetic GGUF) to `.gguf.tdf`; assert zip members `0.manifest.json`, `header`, `s/1`; assert `gguf.virtualSize` equals source length.
- `read_at` of `[0, 4)` returns `GGUF` magic after KAS mock.
- `read_at` of a tensor range equals the source GGUF bytes for that range.
- Segment cap: force `maxSegment = 4096` on a tensor larger than 4096; multiple `s/*` members; `read_at` still matches.
- Wrong payload key / flipped ciphertext bit → decrypt error, no panic.
- Discovery: `find_gguf_in_dir` finds `.gguf.tdf`.

Integration (needs llama.cpp callback + a real small GGUF, e.g. Gemma 270M):

- Load `.gguf.tdf` via callback; one prompt; logits match a control load of the source `.gguf` (same sampling seed / temp 0).
- Peak extra RSS during load (excluding destination weights) stays near `headerBytes + maxSegment`, not near file size. Measure with a 270M model first; document the method for 8B/12B.

Do not call production KAS in unit tests. Use the existing mock KAS pattern in `opentdf-rs` / `arkavo-config-encryption` tests.

## Compatibility

| Consumer | v1 behavior |
|---|---|
| Arkavo with this profile + callback loader | Load `.gguf.tdf` |
| Arkavo without TDF/KAS features | Refuse `.gguf.tdf` |
| `otdfctl decrypt` | May fail or produce a non-GGUF concatenation; **not a v1 requirement** |
| HuggingFace cache | Store `.gguf.tdf` as the artifact; do not also write `.gguf` after download-and-protect |
| Windows llama-cpp-less builds | Wrap CLI can still produce `.gguf.tdf`; load is macOS/Linux |

## Phasing

**P0 — opentdf-rs:** variable-length segment encrypt; `decrypt_entry`; multi-member zip writer; `gguf` JSON field.

**P0 — llama.cpp:** `llama_model_load_from_callback` (handover doc). Can proceed in parallel.

**P1 — packer + `read_at` + CLI protect** on a tiny GGUF, mock KAS.

**P1 — `LlamaModel::from_callback` + `arkavo-llm` path detect.**

**P2 — discovery, first-run wrap option, mmproj, docs for `ARKAVO_MODEL_PATH`.

**P3 — split GGUF, mtmd callback, optional root-signature-on-load.

## Decisions locked

- No plaintext temp GGUF (named or memfd of the whole file).
- Custom TDF layout (multi-entry zip); not “decrypt to vanilla GGUF.”
- llama.cpp grows a generic reader, not TDF/AES.
- Default `maxSegment` 4 MiB, multiple of 32.
- Header is its own encrypted segment.
- Fail closed on KAS/policy/tag errors.
