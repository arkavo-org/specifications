# TDF FairPlay Package Profile (`fmp4-cbcs-fps`, version 2)

|                  |                                                                 |
|------------------|-----------------------------------------------------------------|
| **Version**      | 0.2.0-draft (document `draft-01`)                               |
| **Status**       | Community Draft                                                 |
| **Authors**      | Arkavo Project Contributors                                     |
| **License**      | Apache 2.0                                                      |
| **Date**         | 2026-10-10                                                      |
| **Profile**      | `fmp4-cbcs-fps`, version `2`                                    |
| **Obsoletes**    | [draft-arkavo-tdf-fairplay-00](draft-arkavo-tdf-fairplay-00.md) (profile version 1) |
| **Builds on**    | OpenTDF zip TDF (manifest, wrapped key access, HS256 policy binding); HLS (RFC 8216); ISO/IEC 23001-7 `cbcs`; Apple FairPlay Streaming |
| **Derived from** | arkavo-ios ADR-0055 (accepted 2026-10-10), which refines ADR-0046, ADR-0047, ADR-0048 and ADR-0050; ADR-0045, ADR-0049 and ADR-0052 (accepted); ADR-0054 (proposed); Creator CRE-250 to CRE-260; ArkavoMediaKit 0.4.0; arkavo-rs pull request 86 |

---

## Abstract

This document specifies **`fmp4-cbcs-fps` version 2**, the package that Arkavo Creator writes and the Arkavo viewer plays with Apple FairPlay Streaming (FPS). Version 2 gives each media component its own content key.

A package is a stored ZIP of:

- an OpenTDF-style `manifest.json` with one wrapped key access per component;
- an HLS master playlist;
- for each component, a media playlist, a one-track fMP4 init and `cbcs`-encrypted fMP4 segments.

The video and the audio are demuxed renditions, keyed `skd://<policy uuid>/video` and `skd://<policy uuid>/audio`. The TDF policy carries the creator's content classification and a component list. Each entry of the list binds one content key to its component, so the license service can refuse a swapped key before it issues a CKC.

The document also fixes the license exchange with the media service at `platform.arkavo.net`: one license request per component, each with its own media session, and the CKC terms for each kind (`uhd` for video, `audio` for audio, HDCP Type 1 for both).

Version 2 replaces version 1 ([draft-00](draft-arkavo-tdf-fairplay-00.md)). A conforming reader refuses version 1. §14 records where each implementation stands.

---

## 1. Introduction

### 1.1 Motivation

Two Arkavo apps handle the same FairPlay package:

- **Creator** (macOS) protects a recording. It packages the recording through ArkavoMediaKit's `FMP4RecordingProtectionService`, and its own player plays the result back.
- **The Arkavo viewer** (arkavo-ios) imports the package, decides whether it is supported, applies the device's sensitive-content policy, and plays it through `AVContentKeySession`.

Profile version 1 had one content key for a muxed video and audio presentation. On 2026-10-10 an iPad played a Creator version 1 archive made with ArkavoMediaKit 0.3.0:

- **The video decoded.** The CKC was accepted, the key was set in the hardware decoder, and every frame decoded.
- **The audio never decrypted.** `AudioConverterService` logged `ACCPEDecoderWrapper … Error decrypting buffer … err = -42811` for every one of 2,630 buffers.

The cause is the key's content type. The media service issued the one key as `uhd`. With the version 3 session keys in the device's SPC and the `cbcs` scheme, the FairPlay Server SDK seals an `sd`, `hd` or `uhd` key with the device's *video* session key. Only the hardware video path can use such a key. The audio decrypts in software, so it cannot.

Version 2 gives the audio its own key, issued as `audio`, while the video key stays `uhd`. HLS gives a track its own key only through its own media playlist, so the audio becomes a demuxed rendition. The format allows many components. Version 2 admits video and, optionally, audio (arkavo-ios ADR-0055).

### 1.2 Changes from draft-00

- **Components.** One content key per media component (§3), each in its own `keyAccess` (§5.2), listed in the bound policy key `arkavo:components` with a per-key `keyBinding` (§5.3).
- **Container.** `master.m3u8`, and `<id>.m3u8`, `<id>-init.mp4` and `<id>-segment<N>.m4s` for each component, replace `playlist.m3u8`, `init.mp4` and `segment<N>.m4s` (§4).
- **Identifiers.** The key URI is `skd://<policy uuid>/<id>`, and the SPC content identifier is `<policy uuid>/<id>` (§9.3).
- **License.** The key-request body gains `component` (§10.2). Each component's acquisition and renewal is its own license request (§10.6). The media service checks the component list and issues each key under its kind's terms (§10.7).
- **Keys.** Each content key is exactly 16 bytes. The media service no longer truncates a longer key (§5.2).
- **Audio.** The audio pattern is frozen: `tenc` version 0, pattern byte 0, whole-block full-sample encryption (§8.2). This closes draft-00's OPEN-2.
- **Playlists.** `#EXT-X-TARGETDURATION` is required in a media playlist (§7.3).
- **Trust.** With an empty pin set, the license path uses system trust (§10.1). The certificate route may serve Apple's two-certificate bundle (§10.4).
- **Presentation.** The loopback presentation and the entries it serves (§11).
- **Limits.** `fps-v2`, with `maxEntries` 8,192 (§12).
- **Profile v1 is refused** (§9.6).
- **Draft-00's OPEN-5 deployment blocker is closed.** arkavo-rs pull request 75 was merged (`cc591b3`) and deployed on 2026-10-09 (arkavo-ios ADR-0052). On 2026-10-10 a device accepted its CKC and decoded the video (ADR-0055).

### 1.3 Scope

In scope:

- the components (§3);
- the archive, manifest, policy, playlists and media (§4 to §8);
- identity and admission (§9);
- the license exchange (§10);
- the presentation (§11);
- the import limits (§12);
- the shared test vectors (Appendix A).

Out of scope:

- Profile version 1, apart from its refusal (§9.6). Draft-00 is kept for history.
- The CBC "HLS TDF" archive that Creator also writes. It is played by local decryption, which hard constraint 3 of arkavo-ios forbids in the viewer.
- Standard TDF (`0.payload`).
- NanoTDF streams ([ntdf-rtmp](../ntdf-rtmp/)).
- Later component kinds: audio-only packages, and `image` for artwork (§3).
- Phase 2 of the media-service contract (heartbeat, server-side termination).

### 1.4 Conformance language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

The roles are:

- **Writer:** the packager, which is Creator using ArkavoMediaKit 0.4.0 or later.
- **Reader:** any player that admits a package. It MUST apply every check in §4 to §9 before any certificate, session or license request.
- **Media service:** the FPS license service at `https://platform.arkavo.net/media/v1/*` (arkavo-rs). §10.7 calls it the key service where it acts on the keys.

### 1.5 Document status

This is `draft-01`. It obsoletes `draft-00`. Items marked **OPEN** are unresolved cross-repository questions (§13). A reader MUST treat a value governed by an OPEN item as described there. Until then it fails closed.

---

## 2. Overview

```
recording.tdf  (stored ZIP32)
├── manifest.json        OpenTDF manifest: one wrapped key access per component; base64 policy
│                        with arkavo:classification and arkavo:components; one HS256 binding per key
├── master.m3u8          one variant (video.m3u8) and, with sound, one audio rendition (audio.m3u8)
├── video.m3u8           VOD media playlist, #EXT-X-KEY SAMPLE-AES skd://<policy uuid>/video
├── video-init.mp4       moov: one encv track (H.264, cbcs 1:9)
├── video-segment0.m4s … video-segment<N>.m4s
├── audio.m3u8           (with sound) #EXT-X-KEY SAMPLE-AES skd://<policy uuid>/audio
├── audio-init.mp4       (with sound) moov: one enca track (AAC-LC, cbcs full-sample)
└── audio-segment0.m4s … audio-segment<N>.m4s
```

Playback:

```
reader ── admit (§4–§9, no network) ──► classification gate (§6.4) ──► loopback (§11) serves master.m3u8
   │                                                                     │
   │                                     AVContentKeySession asks for skd://<uuid>/video and skd://<uuid>/audio
   ├─ GET  /media/v1/certificate          (no credential)                │
   │  for each component, and for each renewal of each component:        │
   ├─ POST /media/v1/session/start        (Bearer) {assetId, protocol}   │
   ├─ POST /media/v1/key-request          (Bearer) {assetId, component, sessionId, spcData, tdfManifest}
   │                                                  └─► CKC ───────────┘
   └─ DELETE /media/v1/session/<id>       (Bearer, best effort)
the item plays only once AVFoundation has accepted every component's key
```

---

## 3. Components

1. **A component** is one protected stream with its own content key.
2. **Version 2 allows:**
   - exactly one component of kind `video`, with id `video`;
   - at most one component of kind `audio`, with id `audio`.

   Each id equals its kind.
3. **Ids** are lowercase ASCII letters. A component's entry names (§4), its key URI (§7.3), its SPC content identifier (§9.3) and its license request (§10.2) all use its id.
4. **A recording without sound** has the one `video` component and nothing else.
5. **Later kinds are left to later versions:** audio-only (music) packages, and `image` for artwork.
   - The container rules, the component list and the per-component key access carry over.
   - Version 2's own rules do not: the video comes first, and `video.m3u8` is the only variant.
   - A later kind needs a later profile version, with its own reader rules and media-service mapping. Artwork is not FairPlay media, so its decrypt path needs its own decision record.

## 4. Container

The container is a stored ZIP32.

1. The writer MUST use compression method 0 (stored) for every entry. It MUST NOT use ZIP64, the encryption flag or data descriptors.
2. The central directory and each local header MUST agree on name, method, flags, CRC-32 and sizes. A reader MUST verify the CRC-32 of every entry it reads.
3. **Exact layout.** The first local header starts at byte 0, and the entries follow one another with no gap or overlap. The central directory follows the last entry, and the end-of-central-directory record ends the file.
   - The archive comment MUST be empty.
   - The archive is a single disk: both disk numbers are 0, and both entry counts equal the actual count.
   - Extra fields and per-entry comments MUST have length 0.
4. **The entries are exactly:**
   - `manifest.json` and `master.m3u8`;
   - for each component of the bound list (§5.3): `<id>.m3u8`, `<id>-init.mp4` and `<id>-segment<N>.m4s`.

   Further rules:
   - `N` is a decimal number from 0 to 4092, with no leading zeros and no gaps.
   - Every component has the same number of segments, at least one.
   - There are no directories, no duplicates and no other names. Names are ASCII.
   - The entries follow the component list exactly. An entry for a component the list does not name, or a listed component without its entries, is refused.
   - A reader MUST refuse a traversal (`..`), absolute or backslash name before it reads anything from that entry.
5. **Admission** reads the central directory, `manifest.json`, `master.m3u8`, every `<id>.m3u8` and every `<id>-init.mp4`. A reader MUST NOT read or decrypt a segment to admit a package. Nothing about the fragments is therefore reader-checked (§8).
6. **Writer refusals.** A writer MUST refuse to package the following, rather than write a package that a reader refuses or cannot decode:
   - more than 4,093 segments per component (about 6 h 49 min at 6 s segments);
   - an archive of more than 4,294,967,295 bytes, the most a ZIP32 holds (about 71 minutes at 8 Mbit/s).

   §8.1 adds the writer's refusals for the video samples.

## 5. Manifest (`manifest.json`)

### 5.1 General

1. The manifest is UTF-8 JSON. A reader MUST refuse a duplicate object key in the manifest or in the decoded policy. The media service refuses one too, for every manifest (§10.7).
2. `keyAccess[i].encryptedMetadata`, `meta` and `method.algorithm` are **informational**. No binding covers them. A reader MUST NOT use them to decide that a package is FairPlay, to decide its identity, or to read its classification. `method.iv` is not informational: §5.4 checks it.

### 5.2 Key access

1. `encryptionInformation.keyAccess` holds **exactly one object per component**, in the order of `arkavo:components` (§5.3). `keyAccess[i]` wraps the content key of `arkavo:components[i]`. A count that differs from the component count is refused.
2. Each object has the form `{type:"wrapped", url, wrappedKey, policyBinding}`.
   - **`url`.** After normalization it MUST be `https://platform.arkavo.net`. Normalization lowercases the scheme and host, drops port 443, and allows an empty path or `/` and no query.
   - **One `url` for all.** Every object MUST carry the same `url` text. A reader MUST refuse a package whose objects name different URLs, as naming no known key service. This is stricter than ADR-0055 §3, which applies the URL rule to each object on its own. ArkavoMediaKit wraps every key to one KAS.
   - **`wrappedKey`** is the component's content key, RSA-OAEP wrapped to the platform KAS key, in standard base64.
3. **The policy bindings.** Each `policyBinding` is `{alg:"HS256", hash}` or a bare string.
   - `hash` MUST be the standard base64 of the **32-byte raw digest** `HMAC-SHA256(content key of that component, utf8(encryptionInformation.policy))`. The HMAC input is the policy's base64 string exactly as stored.
   - Every binding covers the same policy string, each computed with its own component's key.
   - A reader accepts only canonical standard base64: the RFC 4648 alphabet, padded, which decodes and re-encodes to the same text. The same applies to `keyBinding` (§5.3) and `method.iv` (§5.4).
   - The OpenTDFKit form `base64(hex(digest))`, which Creator's standard-TDF path writes (Creator CRE-046), is not a binding of this profile. A writer MUST NOT use it, and a reader refuses it, because it decodes to 64 bytes.
4. **The content keys.**
   - Each content key is 16 random bytes, independent of every other key.
   - Two components MUST NOT share a key.
   - The media service requires each key to unwrap to exactly 16 bytes and never truncates one. Profile version 1 let it unwrap 16 or 32 bytes and use the first 16.

   These are writer obligations that a reader cannot observe. The media service enforces them (§10.7).

### 5.3 `arkavo:components`

1. **The list.** The top-level policy key `arkavo:components` holds a non-empty array of objects. Each object has exactly these keys:
   - `id`: the component id (§3);
   - `kind`: `"video"` or `"audio"`;
   - `keyBinding`: the standard base64 of the 32-byte `HMAC-SHA256(content key, UTF-8 of "arkavo:fps:component:v1:" + id)`.
2. **Version 2's list.** The first entry is `{"id":"video","kind":"video",…}`. An optional second entry is `{"id":"audio","kind":"audio",…}`. Nothing else is allowed.
3. **Placement.** The list sits inside the policy so the policy bindings (§5.2) cover it, as they cover the classification (§6.3).
4. **Position.** A reader MUST NOT depend on where `arkavo:components` sits among the policy's members.
   - The writer inserts it deterministically: as the policy object's first member, serialized compactly, with each entry's keys in the order `id`, `kind`, `keyBinding`. No other byte of the policy changes. This makes Appendix A reproducible.
   - A writer MUST refuse a policy that already has `arkavo:components`, as a producer error. The writer makes the list itself, from the keys it generates.
5. **What `keyBinding` does.**
   - It ties each wrapped key to its component. Swapping two `keyAccess` objects passes every policy binding, because they all cover the same string, but it fails `keyBinding`. The media service therefore never issues the video key under the audio component's terms.
   - `keyBinding` ties a key to its component's `id`. The policy bindings protect the list itself, so an `id` cannot be paired with another `kind` without breaking them. Together they fix each key's kind.
6. **What `keyBinding` reveals.**
   - It does not reveal the key.
   - It lets someone holding a guessed key check the guess. That is infeasible for a random 128-bit key.
   - It shows when the same key and id appear twice.
7. **What a reader can and cannot check.**
   - A reader checks the list's shape: the exact keys, the order, the kinds, and a `keyBinding` that is standard base64 of 32 bytes.
   - A reader cannot verify `keyBinding`, because it never has a key. A swapped or shared key is refused by the media service before any CKC (§10.7).

### 5.4 The content IV

1. `encryptionInformation.method.iv` is REQUIRED. It MUST be the standard base64 of exactly 16 bytes. One IV serves every component.
2. It MUST equal:
   - the constant IV of every component's `tenc` (§8);
   - the `IV` of every media playlist's `#EXT-X-KEY` (§7.3), when that attribute is present.
3. A reader MUST refuse a package whose `method.iv` is missing or malformed, or differs from any of these.
4. One IV under independent keys is sound.
5. The media service takes the CKC's content IV from this field (§10.7).

## 6. Policy (`encryptionInformation.policy`)

The policy is the standard base64 of a JSON object. Besides `uuid` and `body`, version 2 defines two top-level keys: `arkavo:components` (§5.3) and `arkavo:classification` (§6.3).

### 6.1 `uuid`

`uuid` is REQUIRED. It MUST be in the canonical **lowercase** 8-4-4-4-12 form.

- A writer MUST generate it fresh for each package. It is never the recording id.
- A reader MUST compare it case-sensitively.

### 6.2 `body.dataAttributes`

`body.dataAttributes` is REQUIRED and non-empty. Each element is `{"attribute": "<FQN>"}` with a non-empty string `attribute`. `body.dissem` MAY be present.

A writer MUST NOT write a placeholder policy such as `{"uuid":…,"body":{}}` for this profile. The media service refuses a policy with no data attributes (403), and a reader treats it as an incomplete policy.

### 6.3 `arkavo:classification` (PROPOSED, see OPEN-5)

The creator's pre-encryption content classification is a top-level policy key:

```json
"arkavo:classification": {"filter": "passed", "flagged": [], "v": 1}
```

- The object has exactly three members: `v`, `filter` and `flagged`.
- `v` is the integer `1`, written `1`.
- `filter` is the string `"passed"`.
- `flagged` is an array with no duplicates and is a subset of `["nudity", "violence"]`.

The classification lives **inside the policy** because the bindings (§5.2) cover only that string. If someone edits it, every binding breaks and the media service refuses the license. A writer MUST NOT put the classification in `meta` or `encryptedMetadata`.

### 6.4 Reading the classification (normative)

A reader MUST treat any of the following as **unreadable**:

- a missing key;
- a `v` other than `1`;
- a `filter` other than `"passed"`;
- a `flagged` entry outside the vocabulary, or a repeated one;
- a member other than `v`, `filter` and `flagged`;
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

Creator writes the classification only after its filter passes. When the filter does not pass, Creator MUST NOT publish a FairPlay package (OPEN-5).

### 6.5 Other parsers

Every parser of the policy MUST accept the extra top-level keys `arkavo:classification` and `arkavo:components`. That includes the media service, the platform KAS, and tdf-iroh-s3 ingest (OPEN-3).

## 7. Playlists

### 7.1 Line grammar

These rules apply to `master.m3u8` and to every `<id>.m3u8`.

1. The playlist is UTF-8. Lines end with LF; CRLF and a lone CR are refused. A trailing newline is optional.
2. `#EXTM3U` is the first line, byte for byte.
3. Blank lines and comment lines are refused, as is any line that is neither a permitted tag nor a permitted URI.
4. Every URI is relative and names an entry of the archive. No URI has a scheme, authority, query, fragment, `..` or `/`.

### 7.2 `master.m3u8`

The master playlist follows the attribute rules of Apple's HLS Authoring Specification for one variant with one audio rendition. Besides `#EXTM3U`, it holds exactly:

1. `#EXT-X-VERSION:7`.
2. `#EXT-X-INDEPENDENT-SEGMENTS`.
3. **When there is an `audio` component, exactly one audio rendition**: `#EXT-X-MEDIA` with exactly these attributes, in any order:
   - `TYPE=AUDIO`, `GROUP-ID="audio"`, `NAME="Audio"`, `DEFAULT=YES`, `AUTOSELECT=YES` and `URI="audio.m3u8"`;
   - `LANGUAGE="<tag>"`: a BCP 47 tag, `"und"` when the language is unknown. A reader checks only the tag's shape: a primary subtag of 2 or 3 letters, then subtags of 1 to 8 letters or digits, separated by `-`;
   - `CHANNELS="<n>"`: the channel count of the audio init's AudioSpecificConfig. A `channelConfiguration` of 1 to 6 is that many channels, and 7 is 8 channels (7.1). So `<n>` is one of `1` to `6` and `8`. A `channelConfiguration` of 0 or 8 to 15 is refused.

   Without an `audio` component there is no `#EXT-X-MEDIA`.
4. **Exactly one variant**: `#EXT-X-STREAM-INF` with exactly these attributes, in any order. The next line is `video.m3u8`.
   - `BANDWIDTH` and `AVERAGE-BANDWIDTH`: positive decimal integers with no leading zero. `AVERAGE-BANDWIDTH` is at most `BANDWIDTH`.
   - `CODECS`: a quoted list of exactly one `avc1.<6 hex digits>`, plus `mp4a.40.2` exactly when there is an `audio` component.
   - `RESOLUTION=<w>x<h>`, with positive integers.
   - `FRAME-RATE`: a positive decimal number with at most three decimals.
   - `AUDIO="audio"`: present exactly when there is an `audio` component.
5. **Nothing else.** No `EXT-X-KEY`, `EXT-X-SESSION-KEY`, `EXT-X-SESSION-DATA`, `EXT-X-I-FRAME-STREAM-INF` or other tag. No second variant or rendition, and no tag twice.
6. **Order.** The order of the lines after `#EXTM3U` is not constrained, except that `video.m3u8` immediately follows `#EXT-X-STREAM-INF`. Appendix A.3 gives the bytes ArkavoMediaKit writes.

**What a reader checks:**

- the grammar above;
- `CHANNELS` against the audio init's `channelConfiguration` (§8.2);
- `CODECS` against the components present;
- the rendition, and `AUDIO`, present exactly when the `audio` component is.

**What only the writer can make true.** A reader checks only the syntax of `BANDWIDTH`, `AVERAGE-BANDWIDTH`, the `avc1` profile and level, `RESOLUTION` and `FRAME-RATE`. The writer MUST make the values describe the media:

- `BANDWIDTH` is the sum of each rendition's peak segment bit rate, the video's plus the audio's (RFC 8216 §4.3.4.2);
- `AVERAGE-BANDWIDTH` is the sum of their average bit rates;
- the `avc1` profile and level are the `avcC`'s;
- `RESOLUTION` and `FRAME-RATE` are the video track's.

**Hexadecimal case.** The writer writes the playlist `IV` and the `avc1` profile and level in upper case, as RFC 8216 §4.2's hexadecimal-sequence grammar has it. A reader MUST accept either case for both.

**Deliberate departures from Apple's guidance**, for a local, single-recording package: one variant rather than a bit-rate ladder, no I-frame playlist, and delivery from a loopback server (§11).

### 7.3 Media playlists (`<id>.m3u8`)

1. **Required tags:**
   - `#EXT-X-VERSION:7`;
   - `#EXT-X-PLAYLIST-TYPE:VOD`;
   - `#EXT-X-TARGETDURATION:<n>`, a positive integer;
   - `#EXT-X-MAP:URI="<id>-init.mp4"`, with no `BYTERANGE`;
   - `#EXT-X-ENDLIST`, as the last line.

   `#EXT-X-TARGETDURATION` is required, as RFC 8216 §4.3.3.1 requires it. Draft-00 and ADR-0047 listed it as permitted. ArkavoMediaKit writes it, the viewer's reader has required it since 2026-10-06, and the alignment check (§7.4) depends on it.
2. **Permitted tags.** The only other permitted tags are `#EXT-X-MEDIA-SEQUENCE:0`, `#EXT-X-INDEPENDENT-SEGMENTS` and `#EXTINF`.
3. **The key tag.** The playlist has exactly one `#EXT-X-KEY`, before the first `#EXTINF`, in any position relative to the map:

   ```
   #EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://<policy uuid>/<id>",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1"[,IV=0x<32 hex>]
   ```

   - `<policy uuid>` matches §6.1 and `<id>` is this playlist's component. The URI is matched exactly.
   - Any of the following is unsupported: `METHOD=NONE`, a second key tag, or any other attribute.
   - When `IV` is present, it MUST equal `method.iv` (§5.4).
   - A reader MUST NOT strip or rewrite the tag.
4. **Segment durations.** `#EXTINF:<duration>,[title]`.
   - `duration` is a positive decimal number with no sign or exponent. Rounded to the nearest integer, it is at most `TARGETDURATION`.
   - The title is ignored. It MUST NOT contain a control character.
5. **Segment URIs.**
   - Each `#EXTINF` is followed by a URI that names `<id>-segment<N>.m4s`, in order from `<id>-segment0.m4s`.
   - Each entry is used once, and every segment entry of the component is used.
6. **Refused tags.**
   - `#EXT-X-BYTERANGE`.
   - Every URI-bearing tag other than the map: `EXT-X-MEDIA`, `EXT-X-STREAM-INF`, `EXT-X-I-FRAME-STREAM-INF`, `EXT-X-SESSION-KEY`, `EXT-X-SESSION-DATA`, `EXT-X-DEFINE` and `EXT-X-PART`. A media playlist is never a master.
7. **Captions and alternate audio: none.** The single audio rendition is the recording's own sound, not an alternate.

### 7.4 Segment alignment (reader-checked)

Every component's playlist has, compared with the video's:

- the same number of segments;
- the same `#EXT-X-TARGETDURATION`;
- the same `#EXTINF` value, segment by segment.

Segment `N` of each component names the same time range: the range of the video's segment `N`.

### 7.5 Sample timing (writer obligations)

A reader cannot observe these, because it never reads a segment.

1. **Origin.** The package's time 0 is the first video sample's decode time.
2. **Placement.** Each audio sample is placed by `tfdt`, through both tracks' edits. Encoder priming is handled by placement, not by an edit list.
3. **Coverage.** The audio covers the video from its first *presented* frame to its presented end, within one AAC packet. A video with B-frames presents its last frame after its last decode. This refines ADR-0055 §4, which says only that the last audio packet ends within one packet of the video's end. It is what ArkavoMediaKit 0.4.0 writes.
4. **Contiguity.** The audio's decode times, as emitted, are contiguous from segment to segment.
5. **Ranges.** An audio segment holds the packets that start in its range. Every audio segment holds packets.
6. **Late or short sound** is mixed with silence, so that the audio covers the video. A source whose sound cannot be made to cover the video is refused rather than packaged out of alignment.
7. **Every segment** of the video starts with an IDR frame.

## 8. Media (`<id>-init.mp4`)

Each component's init holds exactly one track, of its component's kind. In the tables:

- **R:** the reader checks it, and a failure makes the package unsupported.
- **W:** a writer obligation the reader cannot observe.

### 8.1 `video-init.mp4`

| # | | Requirement |
|---|---|---|
| 1 | R | A well-formed box tree with exactly one `moov`. |
| 2 | R | Exactly one track, of kind video. |
| 3 | R | The sample entry is `encv`. A clear `avc1` entry is unsupported. |
| 4 | R | `sinf/frma`: the original format is `avc1`. |
| 5 | R | `schm`: scheme `cbcs`, version `0x00010000`. |
| 6 | R | `tenc`: `default_isProtected` = 1, `default_Per_Sample_IV_Size` = 0, and a 16-byte constant IV equal to `method.iv` (§5.4). `default_KID` is not checked; an all-zero KID is allowed. |
| 7 | R | `tenc` is version 1, with `crypt_byte_block` 1 and `skip_byte_block` 9. |
| 8 | R | The video is H.264: `avc1` with an `avcC`. |
| 9 | W | Each slice NAL unit is clear through its slice header. The encrypted blocks of its slice data form one AES-128-CBC chain from the constant IV, as ISO/IEC 23001-7 `cbcs` requires. |
| 10 | W | Each video `traf` carries `senc` (with subsamples), `saiz` and `saio`. A video with negative composition offsets (B-frames) uses a version 1 `trun`, whose offsets are signed. |
| 11 | W | The writer refuses a source whose NAL unit length size is not 4, and a sample whose `senc` entry would exceed 255 bytes. ArkavoMediaKit 0.4.0 states the second as a video sample of more than 42 NAL units. |
| 12 | W | The segments are encrypted with the scheme that the init declares. |

### 8.2 `audio-init.mp4`

| # | | Requirement |
|---|---|---|
| 1 | R | A well-formed box tree with exactly one `moov`. |
| 2 | R | Exactly one track, of kind audio. |
| 3 | R | The sample entry is `enca`. A clear `mp4a` entry is unsupported, so a clear audio track is unsupported. |
| 4 | R | `sinf/frma`: the original format is `mp4a`. |
| 5 | R | `schm`: scheme `cbcs`, version `0x00010000`. |
| 6 | R | `tenc`: `default_isProtected` = 1, `default_Per_Sample_IV_Size` = 0, and a 16-byte constant IV equal to `method.iv` (§5.4). `default_KID` is not checked; an all-zero KID is allowed. |
| 7 | R | `tenc` is version 0, with the pattern byte 0. This freezes draft-00's §7 #9 (its OPEN-2). |
| 8 | R | The audio is AAC-LC: the `esds` AudioSpecificConfig has object type 2 and a `channelConfiguration` of 1 to 7, whose channel count `master.m3u8`'s `CHANNELS` names (§7.2). |
| 9 | W | The samples are whole-block full-sample encrypted (ISO/IEC 23001-7, the layout Shaka Packager writes): every whole 16-byte block of each sample is encrypted, with no skip pattern, and a trailing partial block is clear. |
| 10 | W | Each audio `traf` holds a `senc` of version 0 and flags 0. Its `sample_count` equals the `trun`'s, and its per-sample entries are empty: zero bytes, a constant IV and no subsamples. |
| 11 | W | Each audio `traf` has no `saiz` and no `saio`. |
| 12 | W | The segments are encrypted with the scheme that the init declares. |
| 13 | W | **When the source has sound, the package MUST carry the `audio` component.** A writer packages one AAC-LC track that covers the video as it is. It mixes and re-encodes once as AAC-LC anything else: several tracks, another format, sound that starts late or stops early, and a `channelConfiguration` of 0 or 8 to 15. A reader cannot observe that the source had sound, so a structurally valid video-only package is admitted (OPEN-6). |

### 8.3 Key identifiers

Both components keep the all-zero `tenc` KID. FairPlay finds the key by the `skd://` URI, not the KID. The device spike (OPEN-1) is where a problem with that would show.

## 9. Identity and admission

### 9.1 FairPlay by structure only

A package is `fmp4-cbcs-fps` version 2 only if every check in §4 to §8 holds. A flag, `meta`, `encryptedMetadata`, `method.algorithm`, a file name, or a CBC archive relabeled as fMP4 never decides it.

### 9.2 The asset id

The library asset id is the policy uuid. `meta.assetId` and `encryptedMetadata.assetId` (Creator sets these to the recording id) are ignored by a conforming reader.

### 9.3 The content identifier

Each component's FairPlay content identifier is the UTF-8 of `<policy uuid>/<id>`. It is the component's `skd://` URI with the scheme removed. Profile version 1 used the bare policy uuid (ADR-0046 §5).

### 9.4 The duration

The duration is the sum of the `video` playlist's `#EXTINF` values.

### 9.5 Unsupported packages

An unsupported package keeps its encrypted copy. The viewer says "This recording needs a FairPlay version from its creator." It never reaches a certificate, session or license request.

### 9.6 Profile version 1

A reader MUST refuse a profile version 1 package ([draft-00](draft-arkavo-tdf-fairplay-00.md): `playlist.m3u8`, `init.mp4`, one `keyAccess`, `skd://<policy uuid>`) as unsupported (§9.5).

- A version 1 package's audio can never decrypt under a video key (§1.1).
- Only test packages exist. Creator protects the recording again.
- ArkavoMediaKit 0.4.0 no longer writes version 1.

During the transition, the media service keeps its version 1 path (§10.7). That path never serves a version 2 reader, because the reader refuses the package first.

Compatibility (informative):

| Viewer | Package | Media service | Result |
|---|---|---|---|
| version 1 reader | version 2 | either | unsupported at import |
| version 2 reader | version 1 | either | unsupported at import |
| version 2 reader | version 2 | version 1 only | refused at the license |
| version 2 reader | version 2 | versions 1 and 2 | plays |

## 10. License exchange

This section is normative for the viewer. It is frozen against arkavo-rs `main` at `3b8c542`, the merge of pull request 86. A writer's own player SHOULD follow it.

### 10.1 Routes and trust

There are exactly four routes, all on `https://platform.arkavo.net` (port absent or 443; no user-info, query or fragment):

| Method and path | Credential |
|---|---|
| `GET /media/v1/certificate` | none |
| `POST /media/v1/session/start` | `Authorization: Bearer <CWT>` |
| `POST /media/v1/key-request` | `Authorization: Bearer <CWT>` |
| `DELETE /media/v1/session/<id>` | `Authorization: Bearer <CWT>` |

- Redirects are refused before any header is forwarded.
- The heartbeat route is not called.
- The token gate runs immediately before each credentialed request, renewals included.
- **TLS trust** (ADR-0052):
  - With a non-empty leaf public-key pin set in the release configuration, the viewer uses system trust plus the pin. A pin failure is a hard failure.
  - With an empty set, which is the production configuration today, it uses system trust only, for `platform.arkavo.net`. A trust failure is the same hard failure.

### 10.2 Bodies

Bodies are JSON with sorted keys and standard base64. The two POSTs carry `Content-Type: application/json`.

- **Session start:** `{"assetId":"<policy uuid>","protocol":"fairplay"}`. The response is valid only with `status == "started"` and a `sessionId` that is a single safe path segment.
- **Key request:**

  ```json
  {"assetId":"<policy uuid>","component":"<id>","sessionId":"<id>","spcData":"<b64 SPC>","tdfManifest":"<b64 of manifest.json>"}
  ```

  - `component` is REQUIRED for version 2. It is the id of the component whose key the request asks for.
  - Key order carries no meaning. The viewer sorts keys, so its order on the wire is the one shown.
  - `tdfManifest` is the package's `manifest.json` **bytes exactly as packaged**. It is never reconstructed, reduced or re-serialized.
  - The response is valid only with `status == "success"` and a `wrappedKey` (the CKC) of 1 to 65,536 bytes.
  - `metadata.lease_seconds` is a lease only as an integer from 60 to 86,400. Below 60 the response is malformed. Above 86,400, or a non-integer, means no lease.
- **Never sent:** `userId`, `tdfWrappedKey`, `segmentIndex`, a client public key or a TDF3 header. Neither the viewer nor ArkavoMediaKit 0.4.0's key client sends `userId`.

### 10.3 Identifiers

- **The key URIs** are `skd://<policy uuid>/<id>`, one per component, matched exactly. Any other key identifier fails locally with no request. That includes version 1's bare `skd://<policy uuid>`.
- **The SPC content identifier** is the UTF-8 of `<policy uuid>/<id>` (§9.3), without `skd://`.
- **`assetId`** in both bodies is the policy uuid, never `<policy uuid>/<id>`.
- **`component`** names the component of the key URI that the request answers.
- **Pairing.** The reader pairs the identifier, `assetId`, `component` and `tdfManifest` locally: all of them come from the one admitted package. The media service also checks `assetId` and `component` against the manifest, and the SPC's content identifier against both (§10.7).

### 10.4 The certificate (PROPOSED, arkavo-ios ADR-0054)

- It is fetched without a credential, and no redirect is followed.
- Its body is 1 to 16,384 bytes. It is exactly one, or exactly two, DER X.509 certificates back to back:
  - each certificate is accepted on exactly its own bytes;
  - together they consume the whole body, with no trailing bytes, no third element and no truncation.
- The deployed route serves Apple's SDK 26 certificate bundle: an RSA-1024 and an RSA-2048 certificate, in that order.
- The body is passed to AVFoundation unchanged. It is never split, reordered, trimmed or re-encoded.
- It is kept in memory for the process. It is never persisted and never logged. A failed fetch is not cached.
- A 401 to the certificate request is malformed. It is never treated as a rejected token.

Draft-00 accepted one DER certificate only (ADR-0046 §6). ADR-0054 is proposed, and the viewer's `main` implements it (arkavo-ios pull request 25).

### 10.5 Statuses

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
- At `3b8c542`, every refusal under §10.7's version 2 checks is 403 `"manifest refused"`, a missing `component` or a malformed `method.iv` included. A platform deny stays 403 `"platform refused"`. Whether some refusals should be 422 is OPEN-2.

### 10.6 Sessions, leases and components

- Licenses are online streaming licenses only.
- **One license request per component.** Each component's acquisition, and each component's renewal, is a separate license request with its own media session: start, key request, end. The previous session is ended best effort after the new CKC is delivered.
- **Every key before playback.** The reader requests every component's key before the item plays, each through the token gate. The answers may arrive in any order.
  - The item plays only once AVFoundation has **accepted** every component's key. A CKC handed to AVFoundation is not yet a key accepted.
- **Renewals per component.** Each component's lease is tracked and renewed on its own.
  - A renewal is due at `max(lease − 120 s, lease / 2)` after the delivery of the CKC that carried the lease. A component's first renewal is timed from its own first CKC.
  - It runs only after the token gate passes.
- **A failure fails the whole item.** If any component's request or renewal fails, or AVFoundation reports a key failure, the item fails. Nothing plays with some of its keys. Every component's request, media session and renewal timer is released.
- Stopping or failing a playback ends its media sessions.

### 10.7 Media-service obligations

**For every manifest**, version 1 included, the media service refuses a duplicate JSON key in the manifest or in the decoded policy, before it chooses the profile.

**For a manifest whose policy has `arkavo:components`**, the media service:

1. Applies the closed schema itself, independently of the reader:
   - the version 2 list shape of §5.3, with unique ids and known kinds;
   - a `keyAccess` count equal to the component count;
   - a policy `uuid` and a non-empty `body.dataAttributes`;
   - a `method.iv` that is the standard base64 of 16 bytes.
2. For every `keyAccess`:
   - its `url` names an accepted KAS;
   - it unwraps, to exactly 16 bytes;
   - its policy binding verifies;
   - its `keyBinding` matches its component.
3. Refuses a package where two components share a key.
4. Requires the request's `component` to be present and to name a component, and `assetId` to equal the policy uuid.
5. Runs the platform's authorization decision once.
6. Issues the CKC with the FairPlay Server SDK. The SDK request's `asset-info[0]` carries:
   - `content-key`: that component's key;
   - `content-iv`: `method.iv`, hex-encoded;
   - `content-type`: `uhd` for kind `video`, `audio` for kind `audio`;
   - `hdcp-type`: 1 for both. A `uhd` key also requires security level Main;
   - `encryption-scheme`: `cbcs`;
   - `lease-duration`: the service's configured lease (`MEDIA_FPS_LEASE_SECONDS`). The license is streaming only.

   The request's `asset-id` is `<policy uuid>/<id>`, but SDK 26 does not read it.
7. Requires the SPC's asset id, which the SDK returns in its response, to equal `<policy uuid>/<id>`.

**Refusals.**

- Every failure in steps 1 to 4 and 7 is the generic 403 `"manifest refused"`. The specific reason is only logged by the service.
- **No CKC is ever returned on a refusal.**
  - Steps 1 to 4 run before the platform decision and before the SDK is called.
  - The SDK reads the SPC's content identifier in the same call that computes the CKC. So on a step 7 mismatch the computed CKC is dropped and never leaves the service.
- Only the media service can detect a swapped `keyAccess` (step 2) or a shared key (step 3). A reader never has a key.

**Transition.** A manifest without `arkavo:components` keeps the version 1 single-key behavior:

- `component` is ignored;
- the key is issued as `uhd`;
- an SPC asset id mismatch is only logged;
- a malformed `method.iv` is 400, not 403.

The media service keeps that path while any version 1 viewer build is in use. A version 2 reader never reaches it (§9.6).

### 10.8 Logging

Tokens, SPCs, CKCs, certificates, manifests, wrapped keys, content keys, IVs, media session ids, policy uuids and server bodies MUST NOT be logged by a reader, a writer or a writer's player.

## 11. Presentation

The viewer presents the package to AVFoundation over a per-playback loopback HTTP server (arkavo-ios ADR-0048, refined by ADR-0055 §10).

1. **The listener** is bound to `127.0.0.1` on an ephemeral port. It exists only while a presentation does. Nothing listens at launch or on session restore.
2. **Paths** are `/<token>/<name>`, with a 128-bit random token per playback. Only `GET` and `HEAD` are answered.
3. **What it serves:** `master.m3u8`, each `<id>.m3u8`, each `<id>-init.mp4` and each `<id>-segment<N>.m4s`. That is all.
   - Each response is the stored entry's bytes, unchanged. The `#EXT-X-KEY` tags are not rewritten.
   - `Range` is honored with `206` and `Content-Range`. A range that cannot be satisfied gets `416`.
   - Everything else gets `404`, or `405` for another method.
4. **`manifest.json` MUST NOT be served.** Neither is any token, SPC or CKC. Any player that serves the package over HTTP MUST NOT serve `manifest.json`, and a writer's player MUST keep it out of any folder it serves.
5. **Media types.** Every `.m3u8` is served as `application/vnd.apple.mpegurl`. Every `.mp4` and `.m4s` is served as `video/mp4`.
6. **The asset** is the URL of `master.m3u8`. Key requests go to `AVContentKeySession`, never to the server.

## 12. Import limits (`fps-v2`)

| Limit | Value |
|---|---|
| `maxInputBytes` | 4,294,967,295 |
| `maxExpandedBytes` | 4,294,967,295 |
| `maxEntries` | 8,192 (six fixed entries, and up to 4,093 segments for each of two components; about 6 h 49 min at 6 s segments) |
| `maxManifestBytes` | 1,048,576 (bounds `manifest.json`, `master.m3u8`, every `<id>.m3u8` and every `<id>-init.mp4`) |

- The six fixed entries are `manifest.json`, `master.m3u8`, and `<id>.m3u8` and `<id>-init.mp4` for each of the two components.
- The limits are enforced incrementally. The central directory is checked against `maxEntries` and the size sums before any entry is read.
- They are set from the ZIP32 format, not measured on a device. A measured change is a new revision of this table, and in arkavo-ios a new ADR.

## 13. Open items

| ID | Owner | Item | Until resolved |
|---|---|---|---|
| OPEN-1 | arkavo-ios and the owner (ADR-0055 open questions 1 and 2; the coordinating plan's step 3) | **The device spike.** (a) Does AVFoundation make a second key request, for `skd://…/audio`, and play both renditions in sync from the loopback server? Expected: two `consumed key response` lines, video frames decoded, no `-42811`, and an audible tone. (b) Does an `audio` key with HDCP Type 1 play on each audio route: the built-in speaker, Bluetooth (AirPods) and AirPlay audio routing? Apple's own audio sample SPC asks for HDCP −1, and the SDK refuses HDCP Type 1 for a client that does not support it (`-42604`, which the service maps to 503). (c) Both components keep the all-zero KID (§8.3). (d) Record the domain and code of every key failure AVFoundation reports. | No device has played version 2 audio. If only one key request appears, the presentation (§11) changes; the format stands. A blocked route is the owner's decision, made before anything is protected again. |
| OPEN-2 | arkavo-rs; arkavo-ios ADR-0055 open question 3 | **Status codes** for the new refusals: 403 or 422. The question was raised on 2026-10-09 for an empty IV. | Every version 2 refusal is 403 `"manifest refused"`, which the viewer maps to denied (§10.5). |
| OPEN-3 | tdf-iroh-s3 ([tdf-iroh-s3#19](https://github.com/arkavo-org/tdf-iroh-s3/issues/19), filed for profile version 1); arkavo-ios B7 | **Ingest a version 2 archive.** Ingest's payload checks are profile-aware, so they need a version 2 fixture (ArkavoMediaKit's dump test writes one). Still open from draft-00's OPEN-4: agree one shape for `dataAttributes[].attribute`, accept the `arkavo:classification` and `arkavo:components` keys (§6.5), and publish the maximum blob size. | An FPS recording cannot be distributed over Iroh. |
| OPEN-4 | Creator (CRE-258, CRE-260); arkavo-ios (ARK-117) | **Device evidence.** Creator's protect harness writes three archives: a tone (`fps-test-2026-10-09-v2.tdf`), two AAC tracks mixed (`fps-test-two-tracks-v2.tdf`), and video only (`fps-test-silent-v2.tdf`). The viewer's cross-check admits each. On the iPad and the iPhone the log shows both components' `consumed key response`, no `-42811`, `frames decoded` growing with `dropped` 0, and an audible tone. Seek across a segment boundary, lock and unlock, background and foreground, and switch to AirPods. Under a 60-second lease both components renew. Creator's own player plays sound. | CRE-258 stays `wip`. Version 2 playback with sound is unproven on a device. |
| OPEN-5 | Creator ([Creator#70](https://github.com/arkavo-org/Creator/issues/70), refs [#67](https://github.com/arkavo-org/Creator/issues/67)); the owner; arkavo-ios B1 | **The classification encoding** (§6.3) is proposed. Creator writes it (CRE-253, proposed 2026-10-09) and the viewer reads it, but the owner has not accepted it. Also open: refusing to publish when the filter fails, and recording operator overrides. | The encoding of §6.3 stands. A different encoding is a new revision of this draft and a reader change. |
| OPEN-6 | The owner; arkavo-ios ADR-0047 open question 7 | **A bound audio declaration.** The component list is bound, but a writer that drops the sound writes a video-only list, which a reader admits. The viewer's reader refuses any classification member other than `v`, `filter` and `flagged` (§6.4), so a declaration there is a reader change. | §8.2 #13 is a writer obligation. |
| OPEN-7 | arkavo-rs; arkavo-ios ADR-0046 open questions 1 and 2, ADR-0052, ADR-0054 | **The rest of the media-service contract.** Phase 2: the lease ceiling, heartbeat and termination. Opaque session ids: `main` still builds `<subject>:<asset>:<uuid>`. The TLS pin set and its rotation procedure, which ADR-0052 makes optional hardening. The acceptance of ADR-0054, and the operator's certificate renewal procedure. | No heartbeat, and a new session per request (§10.6). The viewer treats the session id as opaque and never logs or shows it. System trust (§10.1). The certificate rule of §10.4. |

## 14. Implementation status (informative, 2026-10-10)

This section is a snapshot. It is not normative.

### 14.1 Where each implementation stands

Read in the repositories on 2026-10-10:

- **arkavo-rs:** pull request 86 (`feat/fps-profile-v2`) is merged to `main` at `3b8c542`. Its `docs/fairplay.md` documents both profiles as §10.7 does.
- **ArkavoMediaKit:** 0.4.0 is released. Pull request 11 is merged, and the tag `0.4.0` maps to commit `9913d5e`.
  - It writes version 2 only.
  - Its `TDFContentKeyDelegate` answers every component's key, each with its own session, the content identifier `<policy uuid>/<id>` and `component` in the body.
  - It reports once: when AVFoundation has accepted every key, or at the first failure. Renewals never report.
  - Its key client refuses redirects and sends no `userId`.
- **arkavo-ios:** ADR-0055 is accepted and merged on `main` (`fc69d22`). The viewer branch `implementation/fairplay-profile-v2` (head `9712d85`, not yet pushed) is in progress:
  - **Done:** the `fps-v2` limits and the component list; the master and media playlists; the reader and layout, with version 1 refused; key requests per component; the engine (every key before playback, renewal per component, AVFoundation key failures failing the item).
  - **Pending:** the loopback (every playlist served, `manifest.json` refused), composition, and the cross-check against a real 0.4.0 archive.
- **Creator:** `main` (`3827501`) pins ArkavoMediaKit 0.3.1 (`213e548`). The branch `feat/fps-profile-v2` (head `4a1ffef`, not yet pushed):
  - pins ArkavoMediaKit 0.4.0 (`9913d5e`);
  - counts only version 2 archives as protected (CRE-252);
  - has a player that opens `master.m3u8`, serves a folder without `manifest.json`, and ends its wait for the keys exactly once (CRE-259, CRE-260);
  - has an opt-in protect harness that writes a version 2 archive.

  Its device hand check is pending.

Reported by the project, not observed in the repositories:

- arkavo-rs pull request 86 was deployed to `platform.arkavo.net` on 2026-10-10.
- After the deploy, replaying a version 1 package on the iPad verified the kept version 1 path.

**No device has yet played version 2 audio** (OPEN-1, OPEN-4).

### 14.2 Draft-00's divergences

| # | Draft-00 divergence | Now |
|---|---|---|
| D1 | Creator wrote no classification. | Creator `main` writes `arkavo:classification` (CRE-253, proposed; OPEN-5). |
| D2 | A placeholder policy with an empty `body`. | Creator's `TDFPolicyBuilder` throws `noTier` for empty attributes, so the placeholder is never written. |
| D3 | Video-only packages. | ArkavoMediaKit 0.4.0 writes the `audio` component whenever the source has sound. |
| D6 | `userId` sent. | ArkavoMediaKit 0.4.0 sends none. |
| D8 | An untagged packager revision that logged the key and IV. | Creator `main` pins the tag 0.3.1, and its branch the tag 0.4.0. |
| D9 | No redirect refusal in ArkavoMediaKit's key client. | 0.4.0 refuses redirects. |

D4, D5 and D7 concern Creator's player. Its version 2 player (CRE-259, CRE-260) is on the unpushed branch and was not compared here.

### 14.3 Shared test vectors

The coordinating plan has every repository pin Appendix A's vectors, so the four implementations agree byte for byte. ArkavoMediaKit 0.4.0's tests pin the policy insertion and each `keyBinding` (Creator CRE-258's notes).

## 15. Security considerations

- **Tampering.** The policy bindings are the only integrity protection over identity, classification and the component list. A reader MUST NOT take any of them from an unbound field. The media service MUST verify every binding before it issues a CKC. Otherwise a relabeled classification could obtain a license.
- **A swapped or shared key.** Every policy binding covers the same string, so a swap of two `keyAccess` objects passes them all. `keyBinding` catches the swap, and only the media service can check it (§10.7). It refuses before any CKC, so the video key is never issued as `audio`. A shared key is refused for the same reason.
- **What `keyBinding` exposes.** It is an HMAC under the content key, so it reveals nothing without the key. It lets someone confirm a guessed key, which is infeasible for a random 128-bit key. It shows when one key is reused with the same id (§5.3).
- **The video key keeps its hardware binding.** It is always issued as `uhd` with HDCP Type 1, which the SDK seals for the device's secure video path. Only the audio key is issued as `audio`. Issuing one key as `audio` for both would have lost that binding.
- **The content IV is unbound.** `method.iv` sits outside the policy bindings. That is accepted because the IV is not secret: a changed IV cannot reveal a key, it can only make decryption produce garbage, and the reader's consistency check (§5.4) refuses such a package before any request. One IV under independent keys is sound.
- **No CKC on a refusal.** The SDK checks the SPC's asset id in the same call that computes the CKC. The media service drops that CKC on a mismatch (§10.7).
- **Cross-asset pairing.** The reader pairs the key URI, `assetId`, `component` and the manifest locally. The media service checks `assetId`, `component` and the SPC's asset id against the manifest.
- **Partial playback.** A component without a key, or a key without a component, is unsupported. A failure of any component's key fails the item. Nothing plays with some of its keys.
- **Parser surface.** The archive is restricted to exact stored ZIP32 with no extras, the playlists to closed grammars, and each init to a bounded size. JSON with a repeated key is refused by the reader and the media service. This keeps admission small and deterministic.
- **The loopback.** It serves only the package's own playlists and ciphertext, behind a per-playback token on an ephemeral loopback port. It never serves the manifest, with its wrapped keys and policy.
- **Clear content.** A clear audio or video track, `METHOD=NONE`, or any raw content key path is outside the profile. The viewer has no software decrypt or clear-key fallback.

## 16. References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119.
- [RFC4648] Josefsson, S., "The Base16, Base32, and Base64 Data Encodings", RFC 4648.
- [RFC8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174.
- [RFC8216] Pantos, R., May, W., "HTTP Live Streaming", RFC 8216.
- [CENC] ISO/IEC 23001-7, Common encryption in ISO base media file format files (`cbcs`).
- Apple, HLS Authoring Specification for Apple Devices.
- arkavo-ios:
  - `adr/0055-fairplay-package-profile-v2-per-component-keys.md` (accepted 2026-10-10, `main` `fc69d22`);
  - `adr/0045-fairplay-safety-rests-on-creator-filtering.md`, `adr/0046-fairplay-license-client-and-media-service-contract.md`, `adr/0047-fmp4-fairplay-package-profile-v1.md`, `adr/0048-fairplay-presentation-transport.md`, `adr/0049-fairplay-license-client-rules-settled-in-implementation.md`, `adr/0050-content-iv-is-the-manifest-method-iv.md`, `adr/0052-media-service-system-trust-without-pins.md`, `adr/0054-fairplay-certificate-bundle.md` (proposed);
  - `docs/superpowers/plans/2026-10-10-fairplay-profile-v2-implementation.md` (the coordinating plan and its shared vectors);
  - `specs/fairplay-playback.spec.yaml`, `specs/fairplay-safety.spec.yaml`.
- Creator: `specs/content-protection.spec.yaml` (CRE-046, CRE-250 to CRE-260).
- ArkavoMediaKit 0.4.0 (pull request 11, tag `0.4.0` → `9913d5e`), `README.md`, "FairPlay package profile v2".
- arkavo-rs pull request 86 (merged at `3b8c542`), `docs/fairplay.md`.
- [draft-arkavo-tdf-fairplay-00](draft-arkavo-tdf-fairplay-00.md), profile version 1 (obsoleted).

---

## Appendix A. Test vectors

These are the coordinating plan's shared vectors, byte for byte. Every repository pins them.

### A.1 Keys, bindings and the policy

| Name | Value |
|---|---|
| video key | `000102030405060708090a0b0c0d0e0f` |
| audio key | `101112131415161718191a1b1c1d1e1f` |
| `keyBinding` (video key, id `video`) | `9n5YWFmN3MC7g9kJH/tZ98P5JMMEGIz1C+zwHbchEB4=` |
| `keyBinding` (audio key, id `audio`) | `rVO1aH+bdtNnKvDGX38OaorFTRMJsPMGbMJNGKKLU3Q=` |
| `keyBinding` (audio key, id `video`), the swap case | `4WqNpABYVXOM11ru1ClCNaEZZ8HOCCY+Ag0tppvil80=` |
| policy uuid | `3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b` |

The policy before and after the writer's insertion (§5.3):

```text
before: {"arkavo:classification":{"filter":"passed","flagged":[],"v":1},"body":{"dataAttributes":[{"attribute":"https://patreon.arkavo.com/attr/campaign-tier/value/13167240_24457368"}],"dissem":[]},"uuid":"3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b"}
after:  {"arkavo:components":[{"id":"video","kind":"video","keyBinding":"9n5YWFmN3MC7g9kJH/tZ98P5JMMEGIz1C+zwHbchEB4="},{"id":"audio","kind":"audio","keyBinding":"rVO1aH+bdtNnKvDGX38OaorFTRMJsPMGbMJNGKKLU3Q="}],"arkavo:classification":{"filter":"passed","flagged":[],"v":1},"body":{"dataAttributes":[{"attribute":"https://patreon.arkavo.com/attr/campaign-tier/value/13167240_24457368"}],"dissem":[]},"uuid":"3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b"}
```

The policy bindings over `base64(after)`:

- video: `YfWHFTZaDRIdmvAJhng0YHwLLHeKeEYNGKXCXd9wWqc=`
- audio: `BYwGfQat7pK/d/NCvjZZqIryPOUA72NPx+1O/oBXF5A=`

### A.2 The video-only package

- **The policy after the insertion:** `{"arkavo:components":[{"id":"video","kind":"video","keyBinding":"9n5YWFmN3MC7g9kJH/tZ98P5JMMEGIz1C+zwHbchEB4="}],` followed by the rest of `before`, unchanged.
- **Its policy binding** under the video key: `gzj4O10aZXXETOaFvB+qGPqBV48CYTEtDdPXk8azn8o=`.

Both components keep the all-zero `tenc` KID (§8.3).

### A.3 Package vectors

ArkavoMediaKit's generator writes these bytes, and the viewer's and Creator's readers admit them. The lines are joined with LF, with no trailing newline.

The shared values:

- uuid `3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b`;
- IV `f0f1f2f3f4f5f6f7f8f9fafbfcfdfeff`, so `method.iv` is `8PHy8/T19vf4+fr7/P3+/w==` (derived here: the standard base64 of those 16 bytes);
- segment durations 6.00000, 6.00000 and 4.50000;
- stereo AAC-LC audio;
- 1920×1080 H.264 High 4.0 video at 30 fps.

`master.m3u8` with audio:

```text
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",NAME="Audio",LANGUAGE="und",DEFAULT=YES,AUTOSELECT=YES,CHANNELS="2",URI="audio.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=2500000,AVERAGE-BANDWIDTH=2000000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=30.000,AUDIO="audio"
video.m3u8
```

`master.m3u8` without audio. It drops the `EXT-X-MEDIA` line, `mp4a.40.2` and `AUDIO`:

```text
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS
#EXT-X-STREAM-INF:BANDWIDTH=2400000,AVERAGE-BANDWIDTH=1900000,CODECS="avc1.640028",RESOLUTION=1920x1080,FRAME-RATE=30.000
video.m3u8
```

`audio.m3u8`:

```text
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-INDEPENDENT-SEGMENTS
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b/audio",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1",IV=0xF0F1F2F3F4F5F6F7F8F9FAFBFCFDFEFF
#EXT-X-MAP:URI="audio-init.mp4"
#EXTINF:6.00000,
audio-segment0.m4s
#EXTINF:6.00000,
audio-segment1.m4s
#EXTINF:4.50000,
audio-segment2.m4s
#EXT-X-ENDLIST
```

`video.m3u8` is the same, with `video` in place of `audio` in the key URI, the map and the segment names:

```text
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-INDEPENDENT-SEGMENTS
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b/video",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1",IV=0xF0F1F2F3F4F5F6F7F8F9FAFBFCFDFEFF
#EXT-X-MAP:URI="video-init.mp4"
#EXTINF:6.00000,
video-segment0.m4s
#EXTINF:6.00000,
video-segment1.m4s
#EXTINF:4.50000,
video-segment2.m4s
#EXT-X-ENDLIST
```

### A.4 The key-request body

For the vectors above, with sorted keys, the video component's key request is:

```text
{"assetId":"3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b","component":"video","sessionId":"<id>","spcData":"<b64 SPC>","tdfManifest":"<b64 of manifest.json>"}
```

The SPC content identifier is `3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b/video`, and the key URI `skd://3f1c9e2a-7b4d-4e8f-9a21-5c6d7e8f9a0b/video`.
