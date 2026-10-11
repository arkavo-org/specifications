# TDF FairPlay Package Profile (`fmp4-cbcs-fps`, version 1)

> **Obsoleted by [draft-arkavo-tdf-fairplay-01](draft-arkavo-tdf-fairplay-01.md):** profile v1 is refused by viewers as of arkavo-ios ADR-0055. This draft is kept for history.

|                  |                                                                 |
|------------------|-----------------------------------------------------------------|
| **Version**      | 0.1.0-draft (document `draft-00`)                               |
| **Status**       | Community Draft                                                 |
| **Authors**      | Arkavo Project Contributors                                     |
| **License**      | Apache 2.0                                                      |
| **Date**         | 2026-10-06                                                      |
| **Profile**      | `fmp4-cbcs-fps`, version `1`                                    |
| **Builds on**    | OpenTDF zip TDF (manifest, wrapped key access, HS256 policy binding); HLS (RFC 8216); ISO/IEC 23001-7 `cbcs`; Apple FairPlay Streaming |
| **Derived from** | arkavo-ios ADR-0045, ADR-0046, ADR-0047, ADR-0049 and ADR-0050 (accepted); Creator CRE-250 and CRE-251 |

---

## Abstract

This document specifies **`fmp4-cbcs-fps` version 1**, the package that Arkavo Creator writes and the Arkavo viewer plays with Apple FairPlay Streaming (FPS). A package is a stored ZIP of an OpenTDF-style `manifest.json`, one HLS media playlist, an fMP4 `init.mp4` and `cbcs`-encrypted fMP4 segments. The TDF policy names the FairPlay key (`skd://<policy uuid>`), carries the creator's content classification, and is bound to the content key by an HMAC that the license service checks before it issues a CKC.

The document also fixes the viewer's half of the license exchange with the media service at `platform.arkavo.net`.

It writes down a contract that already exists in two places: the arkavo-ios decision records and Creator's behavior specs. **This draft does not change either one.** §12 records where each implementation stands against it.

---

## 1. Introduction

### 1.1 Motivation

Two Arkavo apps handle the same FairPlay package:

- **Creator** (macOS) protects a recording. It packages the recording through ArkavoMediaKit's `FMP4RecordingProtectionService`, and its own player plays the result back.
- **The Arkavo viewer** (arkavo-ios) imports the package, decides whether it is supported, applies the device's sensitive-content policy, and plays it through `AVContentKeySession`.

Until now the contract between them lived only in arkavo-ios ADR-0047 (the package), ADR-0050 (the content IV) and ADR-0046/ADR-0049 (the license client). Creator specified its own half in CRE-250/CRE-251. Each repository cited the other's code by commit. This draft is the shared reference both can cite.

### 1.2 Scope

In scope:

- the archive, manifest, policy, playlist and media profile (§3 to §7);
- identifiers (§8);
- the license exchange (§9);
- the import limits (§10).

Out of scope:

- the CBC "HLS TDF" archive that Creator also writes. It is played by local decryption, which hard constraint 3 of arkavo-ios forbids in the viewer.
- Standard TDF (`0.payload`).
- NanoTDF streams ([ntdf-rtmp](../ntdf-rtmp/)).
- Phase 2 of the media-service contract (heartbeat, server-side termination).

### 1.3 Conformance language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

The roles are:

- **Writer:** the packager, which is Creator using ArkavoMediaKit.
- **Reader:** any player that admits a package. It MUST apply every check in §3 to §8 before any certificate, session or license request.
- **Media service:** the FPS license service at `https://platform.arkavo.net/media/v1/*` (arkavo-rs).

### 1.4 Document status

This is `draft-00`. Items marked **OPEN** are unresolved cross-repository questions (§11). A reader MUST treat a value governed by an OPEN item as described there. Until then it fails closed.

---

## 2. Overview

```
recording.tdf  (stored ZIP32)
├── manifest.json     OpenTDF manifest: one wrapped key access, base64 policy, HS256 binding
├── playlist.m3u8     one VOD media playlist, #EXT-X-KEY SAMPLE-AES skd://<policy uuid>
├── init.mp4          moov: encv (H.264, cbcs 1:9) + optional enca (AAC-LC, cbcs)
├── segment0.m4s
├── …
└── segment<N>.m4s
```

Playback:

```
reader ── admit (§3–§8, no network) ──► classification gate (§5.4) ──► AVContentKeySession
   │                                                                      │ skd://<policy uuid>
   ├─ GET  /media/v1/certificate            (no credential)               │
   ├─ POST /media/v1/session/start          (Bearer)  {assetId, protocol} │
   └─ POST /media/v1/key-request            (Bearer)  {assetId, sessionId, spcData, tdfManifest}
                                                         └─► CKC ─────────┘
```

---

## 3. Container

The container is a stored ZIP32.

1. The writer MUST use compression method 0 (stored) for every entry. It MUST NOT use ZIP64, the encryption flag or data descriptors.
2. The central directory and each local header MUST agree on name, method, flags, CRC-32 and sizes. A reader MUST verify the CRC-32 of every entry it reads.
3. **Exact layout.** The first local header starts at byte 0, and the entries follow one another with no gap or overlap. The central directory follows the last entry, and the end-of-central-directory record ends the file.
   - The archive comment MUST be empty.
   - The archive is a single disk: both disk numbers are 0, and both entry counts equal the actual count.
   - Extra fields and per-entry comments MUST have length 0.
4. The entries are exactly `manifest.json`, `playlist.m3u8`, `init.mp4` and `segment<N>.m4s`.
   - `N` is a decimal number from 0 to 4092, with no leading zeros and no gaps.
   - There are no directories, no duplicates and no other names. Names are ASCII.
   - A reader MUST refuse a traversal (`..`), absolute or backslash name before it reads anything from that entry.
5. A reader admits a package by reading the central directory, `manifest.json`, `playlist.m3u8` and `init.mp4`. It MUST NOT read or decrypt a segment to admit a package.

## 4. Manifest (`manifest.json`)

1. The manifest is UTF-8 JSON. A reader MUST refuse a duplicate object key in the manifest or in the decoded policy.
2. `encryptionInformation.keyAccess` MUST hold exactly one object of the form `{type:"wrapped", url, wrappedKey, policyBinding}`.
   - `url`, after normalization, MUST be `https://platform.arkavo.net`. Normalization lowercases the scheme and host, drops port 443, and allows an empty path or `/` and no query.
   - `wrappedKey` is the content key, RSA-OAEP wrapped to the platform KAS key.
3. **The policy binding.** `policyBinding` is `{alg:"HS256", hash}` or a bare string. `hash` MUST be the standard base64 of the **32-byte raw digest** `HMAC-SHA256(DEK, utf8(encryptionInformation.policy))`, where the HMAC input is the policy's base64 string exactly as stored.
   - The OpenTDFKit form `base64(hex(digest))`, which Creator's standard-TDF path writes (Creator CRE-046), is **not** a profile-v1 binding. A writer MUST NOT use it for this profile, and a reader refuses it, because it decodes to 64 bytes.
4. **The content key** is 16 bytes. The media service issues the CKC from the first 16 bytes of the unwrapped key. This is a writer obligation. A reader cannot observe it.
5. `keyAccess[0].encryptedMetadata`, `meta` and `method.algorithm` are **informational**. No binding covers them. A reader MUST NOT use them to decide that a package is FairPlay, to decide its identity, or to read its classification. In current writers `method.algorithm` still reads `AES-128-CBC`. `method.iv` is not informational: item 6 checks it.
6. **The content IV** (ADR-0050). `encryptionInformation.method.iv` is REQUIRED. It MUST be the standard base64 of exactly 16 bytes. It MUST equal:
   - the constant IV of every protected track's `tenc` (§7 #6), video and audio;
   - the playlist's `#EXT-X-KEY` `IV` (§6.5), when that attribute is present.

   A reader MUST refuse a package whose `method.iv` is missing or malformed, or differs from any of these. The media service takes the CKC's content IV from this field (§9.6).

## 5. Policy (`encryptionInformation.policy`)

The policy is the standard base64 of a JSON object.

### 5.1 `uuid`

`uuid` is REQUIRED. It MUST be in the canonical **lowercase** 8-4-4-4-12 form.

- A writer MUST generate it fresh for each package. It is never the recording id.
- A reader MUST compare it case-sensitively.

### 5.2 `body.dataAttributes`

`body.dataAttributes` is REQUIRED and non-empty. Each element is `{"attribute": "<FQN>"}` with a non-empty string `attribute`. `body.dissem` MAY be present.

A writer MUST NOT write a placeholder policy such as `{"uuid":…,"body":{}}` for this profile. The media service refuses a policy with no data attributes (403), and a reader treats it as an incomplete policy.

### 5.3 `arkavo:classification` (PROPOSED, see OPEN-1)

The creator's pre-encryption content classification is a top-level policy key:

```json
"arkavo:classification": {"v": 1, "filter": "passed", "flagged": []}
```

- `v` is the integer `1`.
- `filter` is the string `"passed"`.
- `flagged` is an array with no duplicates and is a subset of `["nudity", "violence"]`.

The classification lives **inside the policy** because the binding (§4.3) covers only that string. If someone edits it, the binding breaks and the media service refuses the license. A writer MUST NOT put the classification in `meta` or `encryptedMetadata`.

### 5.4 Reading the classification (normative)

A reader MUST treat any of the following as **unreadable**:

- a missing key;
- a `v` other than `1`;
- a `filter` other than `"passed"`;
- a `flagged` entry outside the vocabulary;
- a member of the wrong type.

An unreadable classification makes the package **unsupported under every device policy**, before any license request.

Over a readable classification, the viewer applies its device policy:

| Device policy | Classification | Result |
|---|---|---|
| disabled | any readable | play |
| simple | clean | play |
| simple | flagged | warning naming the flagged types, with Show and Done; Show covers this playback only |
| descriptive | clean | play |
| descriptive | flagged | block, with Done only |

The viewer analyzes no frames or audio. The classification is the only safety input.

Creator writes the classification only after its filter passes. When the filter does not pass, Creator MUST NOT publish a FairPlay package (OPEN-1).

### 5.5 Other parsers

Every parser of the policy MUST accept the extra top-level key: the media service, the platform KAS, and tdf-iroh-s3 ingest (OPEN-1).

## 6. Playlist (`playlist.m3u8`)

1. **Encoding.** The playlist is UTF-8. Lines end with LF; CRLF and a lone CR are refused. A trailing newline is optional. `#EXTM3U` is the first line.
2. **Lines.** Blank lines and comment lines are refused, as is any line that is neither a permitted tag nor a segment URI.
3. **Required tags:** `#EXTM3U`, `#EXT-X-VERSION:7`, `#EXT-X-PLAYLIST-TYPE:VOD`, `#EXT-X-MAP:URI="init.mp4"` (no `BYTERANGE`) and `#EXT-X-ENDLIST`.
4. **Permitted tags.** The only other permitted tags are `#EXT-X-TARGETDURATION:<n>` (a positive integer), `#EXT-X-MEDIA-SEQUENCE:0`, `#EXT-X-INDEPENDENT-SEGMENTS` and `#EXTINF`.
5. **The key tag.** The playlist has exactly one `#EXT-X-KEY`, before the first `#EXTINF`, in any position relative to the map:

   ```
   #EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://<policy uuid>",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1"[,IV=0x<32 hex>]
   ```

   - `<policy uuid>` matches §5.1 exactly.
   - Any of the following is unsupported: `METHOD=NONE`, a second key tag, or any other attribute.
   - When `IV` is present, it MUST equal `method.iv` (§4.6).
   - A reader MUST NOT strip or rewrite the tag.
6. **Segment durations.** `#EXTINF:<duration>,[title]`: `duration` is a positive decimal number with no sign or exponent. Rounded to the nearest integer, it is at most `TARGETDURATION`.
7. **Segment URIs.**
   - Each `#EXTINF` is followed by a relative URI that names `segment<N>.m4s`, in order from `segment0.m4s`.
   - Each entry is used once, and every segment entry is used.
   - No URI has a scheme, authority, query, fragment, `..` or `/`.
8. **Refused tags.**
   - `#EXT-X-BYTERANGE`.
   - Every URI-bearing tag other than the map: `EXT-X-MEDIA`, `EXT-X-STREAM-INF`, `EXT-X-I-FRAME-STREAM-INF`, `EXT-X-SESSION-KEY`, `EXT-X-SESSION-DATA`, `EXT-X-DEFINE` and `EXT-X-PART`.
   - A master playlist.
   - Captions and alternate audio.

## 7. Media (`init.mp4`)

`init.mp4` holds one muxed presentation. In the table:

- **R:** the reader checks it, and a failure makes the package unsupported.
- **W:** a writer obligation the reader cannot observe.
- **OPEN:** not yet frozen.

| # | | Requirement |
|---|---|---|
| 1 | R | A well-formed box tree with exactly one `moov`. |
| 2 | R | Exactly one video track and at most one audio track, and no other track. Track order is free. |
| 3 | R | The sample entries are `encv` (video) and `enca` (audio). A clear `avc1` or `mp4a` entry is unsupported, so a clear audio track is unsupported. |
| 4 | R | `sinf/frma`: the original format is `avc1` for video and `mp4a` for audio. |
| 5 | R | `schm`: scheme `cbcs`, version `0x00010000`. |
| 6 | R | `tenc`: `default_isProtected` = 1, `default_Per_Sample_IV_Size` = 0, and a 16-byte constant IV equal to `method.iv` (§4.6). `default_KID` is not checked; an all-zero KID is allowed. |
| 7 | R | Video `tenc` is version 1, with `crypt_byte_block` 1 and `skip_byte_block` 9. |
| 8 | R | Video is H.264: `avc1` with an `avcC`. |
| 9 | OPEN-2 | Audio is encrypted full-sample: every whole 16-byte block, with a trailing partial block clear. The `tenc` version and crypt/skip values that signal this are not frozen. |
| 10 | R | Audio is AAC-LC: the `esds` AudioSpecificConfig has object type 2. Any other codec MUST be transcoded by the writer. |
| 11 | W | The segments are encrypted with the scheme that `init.mp4` declares. |
| 12 | W | **When the source has sound, the package MUST carry the encrypted AAC-LC audio track.** A structurally valid video-only package is admitted, so this is a writer obligation until OPEN-3 is decided. |

## 8. Identity and admission

1. **FairPlay by structure only.** A package is `fmp4-cbcs-fps` version 1 only if every check in §3 to §7 holds. A flag, `meta`, `encryptedMetadata`, `method.algorithm`, a file name, or a CBC archive relabeled as fMP4 never decides it.
2. **The asset id is the policy uuid.** `meta.assetId` and `encryptedMetadata.assetId` (Creator sets these to the recording id) are ignored by a conforming reader.
3. **Unsupported packages.** An unsupported package keeps its encrypted copy. The viewer says "This recording needs a FairPlay version from its creator." It never reaches a certificate, session or license request.

## 9. License exchange

Normative for the viewer. It is frozen against arkavo-rs pull request 75 at head `8fb3e66`, which is unmerged. A writer's own player SHOULD follow it.

### 9.1 Routes

There are exactly four routes, all on `https://platform.arkavo.net` (port absent or 443; no user-info, query or fragment):

| Method and path | Credential |
|---|---|
| `GET /media/v1/certificate` | none |
| `POST /media/v1/session/start` | `Authorization: Bearer <CWT>` |
| `POST /media/v1/key-request` | `Authorization: Bearer <CWT>` |
| `DELETE /media/v1/session/<id>` | `Authorization: Bearer <CWT>` |

- Redirects are refused before any header is forwarded.
- The heartbeat route is not called.
- The viewer pins the leaf public key of `platform.arkavo.net`. Production pins are empty until the operator publishes them (OPEN-5), so until then the license path is unavailable.

### 9.2 Bodies

Bodies are JSON with sorted keys and standard base64.

- **Session start:** `{"assetId":"<policy uuid>","protocol":"fairplay"}`. The response is valid only with `status == "started"` and a `sessionId` that is a single safe path segment.
- **Key request:** `{"assetId":"<policy uuid>","sessionId":"<id>","spcData":"<b64 SPC>","tdfManifest":"<b64 of manifest.json>"}`.
  - `tdfManifest` is the package's `manifest.json` **bytes exactly as packaged**. It is never reconstructed, reduced or re-serialized.
  - The response is valid only with `status == "success"` and a `wrappedKey` (the CKC) of 1 to 65,536 bytes.
  - `metadata.lease_seconds` is a lease only as an integer from 60 to 86,400. Below 60 the response is malformed. Above 86,400, or a non-integer, means no lease.
- **Never sent:** `tdfWrappedKey`, `segmentIndex`, a client public key or a TDF3 header. `userId` is ignored by the media service after pull request 75. A conforming viewer does not send it.

### 9.3 Identifiers

- The key URI is `skd://<policy uuid>`, matched exactly. Any other key identifier fails locally with no request.
- The SPC content identifier is the UTF-8 of the **bare** policy uuid, without `skd://`.
- `assetId` in both bodies is the policy uuid.
- The media service does not compare `assetId` with the manifest. The reader pairs them locally: the identifier, `assetId` and `tdfManifest` all come from the one admitted package.

### 9.4 The certificate

- It is fetched without a credential.
- It MUST be DER, 1 to 16,384 bytes.
- It is kept in memory for the process. It is never persisted and never logged.
- A 401 to the certificate request is malformed. It is never treated as a rejected token.

### 9.5 Statuses

| Status | Result |
|---|---|
| 400, 422 | malformed |
| 401 (bearer routes only) | unauthorized |
| 403, 404 | denied |
| 429 | concurrency limit |
| 500, 502, 503, 504 | transient |
| any other status | malformed |

- A refusal by the viewer's own token gate sends nothing and is never treated as a 401. It never clears a token (ADR-0049 §6).
- Server error strings are never shown, logged or branched on.
- The client does not retry. `AVContentKeySession` retries are declined.

### 9.6 Sessions and leases

- Licenses are online streaming licenses only.
- Every license request, including each renewal, starts a new media session. The previous session is ended best effort after the new CKC is delivered.
- Renewal is due at `max(lease − 120 s, lease / 2)`. It runs only after the token gate passes.
- Stopping or failing a playback ends its media sessions.
- **The CKC's content** (media-service obligation, ADR-0050). The media service issues the CKC from the first 16 bytes of the unwrapped key, the content IV from the received `tdfManifest`'s `method.iv` (§4.6), and the lease. In the FairPlay Streaming SDK request, it places these as `content-key`, `content-iv` (hex-encoded) and `lease-duration` inside `asset-info[0]`. The viewer's wire does not change: no IV field is sent. A manifest without a valid 16-byte `method.iv` gets the same generic refusal as any other manifest check.

### 9.7 Logging

Tokens, SPCs, CKCs, certificates, manifests, wrapped keys, media session ids, policy uuids and server bodies MUST NOT be logged.

## 10. Import limits (`fps-v1`)

| Limit | Value |
|---|---|
| `maxInputBytes` | 4,294,967,295 |
| `maxExpandedBytes` | 4,294,967,295 |
| `maxEntries` | 4,096 (three fixed entries and up to 4,093 segments; about 6 h 49 min at 6 s segments) |
| `maxManifestBytes` | 1,048,576 (bounds `manifest.json`, `playlist.m3u8` and `init.mp4`) |

The limits are enforced incrementally. They are set from the ZIP32 format, not measured on a device. A measured change is a new revision of this table, and in arkavo-ios a new ADR that supersedes ADR-0047's.

## 11. Open items

| ID | Owner | Item | Until resolved |
|---|---|---|---|
| OPEN-1 | Creator ([Creator#70](https://github.com/arkavo-org/Creator/issues/70), refs [#67](https://github.com/arkavo-org/Creator/issues/67)); arkavo-ios B1 | Fix the classification vocabulary and encoding (§5.3), refuse to publish when the filter fails, record operator overrides, confirm that every policy parser accepts the key, and ship fixtures: clean, nudity, violence, missing, unknown `v`. | Every package is unsupported in the viewer. |
| OPEN-2 | ArkavoMediaKit ([ArkavoMediaKit#6](https://github.com/arkavo-org/ArkavoMediaKit/issues/6); pull request 5 merged at `c915c27`, no tag after 0.1.5); arkavo-ios B2 | Add encrypted AAC-LC audio and freeze its `tenc` values (§7 #9). Publish a tagged release with fixtures (positive with audio, CBC relabeled, missing policy, wrong key URI, clear audio, traversal, mismatched `method.iv`). | The reader asserts nothing about the audio pattern. |
| OPEN-3 | Owner; arkavo-ios ADR-0047 open question 7 | A bound `"audio": true\|false` declaration inside `arkavo:classification`, checked against the track set. | §7 #12 is a writer obligation. |
| OPEN-4 | tdf-iroh-s3 ([tdf-iroh-s3#19](https://github.com/arkavo-org/tdf-iroh-s3/issues/19)); arkavo-ios B7 | Ingest this archive, or agree on a wrapper that opentdf-rs accepts. Agree one shape for `dataAttributes[].attribute` (FQN string or structured object). Accept the classification key. Publish the maximum blob size. | An FPS recording cannot be distributed over Iroh. |
| OPEN-5 | arkavo-rs | **Deployment blocker (pull request 75 review, finding 1):** at `8fb3e66` the wrapper sends `ck` and `lease-duration` at the item level, where the SDK does not read them, so every CKC would carry an all-zero key and IV and no lease. The CKC request MUST carry `content-key`, `content-iv` (§4.6) and `lease-duration` in `asset-info[0]` (§9.6), with a test that runs a sample SPC. Then merge and deploy pull request 75. Publish the TLS pin set and its rotation. Publish phase 2 (lease ceiling, heartbeat, termination). Use opaque session ids. Fix the SPC-identifier wording in `docs/fairplay.md` to the bare uuid. | The viewer's license path is unavailable. |

## 12. Implementation status (informative, 2026-10-06)

This section is a snapshot. It is not normative.

The heads compared:

- **Creator:** `main` at `4ee79a6`, and [Creator#69](https://github.com/arkavo-org/Creator/pull/69) (`feat/fairplay-protect`, `89505b2`, open), which pins ArkavoMediaKit to revision `a1f257e` (pull request 5).
- **arkavo-ios:** `main` at `864cec3`.

### 12.1 In agreement

- **Key URI:** both use `skd://<policy uuid>` (CRE-251; ARK-115, ARK-120).
- **Identifiers:** the SPC content identifier and `assetId` are the policy uuid (CRE-251; ADR-0046 §5).
- **`tdfManifest`:** both send the archive's `manifest.json` bytes as stored (`TDFArchiveReader.extractManifestData`; ARK-121).
- **Bearer:** Creator sends the session CWT on `/media/v1` requests, as the viewer does.
- **Policy shape:** Creator's `TDFPolicyBuilder` writes a lowercase `uuid` and `body.dataAttributes = [{attribute: FQN}]` (§5.1, §5.2).
- **Binding:** the fMP4 binding is the 32-byte raw-digest form (CRE-250), not CRE-046's hex form (§4.3).
- **Key access url:** Creator passes `https://platform.arkavo.net` to the packager (§4.2).
- **Content IV:** ArkavoMediaKit at `c915c27` writes one constant 16-byte IV per recording to `tenc`, the encryptor, the playlist `IV` and `method.iv` (ADR-0050), so it agrees with §4.6. The IV that Creator's player injects into the key tag (D7) is the same value.

### 12.2 Divergences

| # | Topic | Creator / ArkavoMediaKit | arkavo-ios | Effect |
|---|---|---|---|---|
| D1 | Classification (§5.3) | Not written anywhere. | Required. Missing is unsupported under every policy. | **No Creator package plays in the viewer.** Blocks release (OPEN-1). |
| D2 | Placeholder policy (§5.2) | "Protect (FairPlay)" with no tier passes a nil policy, so `{"uuid":…,"body":{}}` (CRE-250). | Incomplete policy, so unsupported. | Public FairPlay recordings never play in the viewer, and stop playing in Creator when pull request 75 deploys. |
| D3 | Audio (§7 #12) | `protectVideo` writes the video track only. | Admits video-only. Profile v1 requires audio. | Packages admit but break the writer obligation (OPEN-2). |
| D4 | FairPlay detection (§8.1) | The player picks the FairPlay path by `encryptedMetadata.type == "fmp4-fairplay"` or `meta.fmp4` (CRE-054). | By structure only (ARK-123). | Creator plays packages the viewer refuses (D1, D2), so a creator's own playback check does not predict viewer playback. |
| D5 | Asset id fallback (§8.2) | Falls back to the fMP4 metadata's `assetId` when the policy has no `uuid` (CRE-251). | Policy uuid only. | Legacy (pre-policy) archives play in Creator only. |
| D6 | `userId` (§9.2) | Still sent, so the pre-#75 service accepts it (CRE-251). | Never sent. | Harmless after pull request 75. Drop after the cutover. |
| D7 | Key tag (§6.5) | The player injects the manifest IV into `#EXT-X-KEY` (CRE-054). | Never rewrites the tag. | Local to Creator's player. The writer MUST still write a conforming tag. |
| D8 | Packager revision | Pinned to the untagged revision `a1f257e`, an ancestor of `c915c27`. Its `CBCSEncryptor` still `print`s the **content key and IV in hex**, which `c915c27` removes. | Needs a tagged release with fixtures. | **Security:** contradicts CRE-250's "never logged on the protect path". Pin `c915c27` now, then the OPEN-2 tag. |
| D9 | License transport (§9.1) | ArkavoMediaKit's key client: no pinning and no redirect refusal. | Pinned, refuses redirects, four routes. | A security difference, not a format one. |

### 12.3 To converge

1. **Creator (spec first, CRE-250/CRE-251):**
   - write `arkavo:classification` into `TDFPolicyBuilder.policyJSON` (OPEN-1);
   - refuse "Protect (FairPlay)" without a data attribute, instead of the placeholder (D2);
   - choose the FairPlay player path by the §8 structure, or at least by a policy that passes §5 (D4);
   - pin ArkavoMediaKit `c915c27` now, which stops the key and IV being logged, then the OPEN-2 tag that carries audio (D3, D8).
2. **ArkavoMediaKit:** audio, a tagged release, fixtures (OPEN-2).
3. **arkavo-ios:** no format change. ADR-0047 is accepted. B1 and B2 no longer block it; they block the reader's real fixtures and the release. `FairPlayPackageReader` (Task 10) starts now against synthetic fixtures. It parses the §5.3 v1 encoding fail-closed and makes the §4.6 IV check. If OPEN-1 settles a different encoding, that is a new revision of this draft and a reader change.

## 13. Security considerations

- **Tampering.** The policy binding is the only integrity protection over identity and classification. A reader MUST NOT take either from an unbound field. The media service MUST verify the binding before it issues a CKC. Otherwise a relabeled classification could obtain a license.
- **The content IV is unbound.** `method.iv` sits outside the policy binding. That is accepted because the IV is not secret: a changed IV cannot reveal a key, it can only make decryption produce garbage, and the reader's consistency check (§4.6) refuses such a package before any request.
- **Cross-asset pairing.** The media service does not pair `assetId` with the manifest. A substituted manifest yields a CKC for a different key, which cannot decrypt the presentation. The reader still pairs them locally.
- **Parser surface.** The archive is restricted to exact stored ZIP32 with no extras, the playlist to a closed tag set, and `init.mp4` to a bounded size. This keeps admission small and deterministic.
- **Clear content.** A clear audio or video track, `METHOD=NONE`, or any raw content key path is outside the profile. The viewer has no software decrypt or clear-key fallback.

## 14. References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119.
- [RFC8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174.
- [RFC8216] Pantos, R., May, W., "HTTP Live Streaming", RFC 8216.
- [CENC] ISO/IEC 23001-7, Common encryption in ISO base media file format files (`cbcs`).
- arkavo-ios: `adr/0045-fairplay-safety-rests-on-creator-filtering.md`, `adr/0046-fairplay-license-client-and-media-service-contract.md`, `adr/0047-fmp4-fairplay-package-profile-v1.md`, `adr/0049-fairplay-license-client-rules-settled-in-implementation.md`, `adr/0050-content-iv-is-the-manifest-method-iv.md` ([arkavo-ios#17](https://github.com/arkavo-org/arkavo-ios/pull/17)), `specs/fairplay-playback.spec.yaml`, `specs/fairplay-safety.spec.yaml`.
- Creator: `specs/content-protection.spec.yaml` (CRE-046, CRE-054, CRE-250, CRE-251).
- arkavo-rs pull request 75 (media-service contract, head `8fb3e66`).
