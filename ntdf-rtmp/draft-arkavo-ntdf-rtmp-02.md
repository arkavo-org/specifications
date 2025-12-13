```text
---
title: "NTDF-RTMP: Secure RTMP via NanoTDF Collections"
abbrev: "NTDF-RTMP"
docname: draft-arkavo-ntdf-rtmp-02
category: exp
ipr: trust200902
area: Security
workgroup: Arkavo Specifications
keyword:
 - RTMP
 - streaming
 - NanoTDF
 - OpenTDF
 - Collections
 - encryption

author:
 - fullname: Arkavo Contributors
   organization: Arkavo
   email: info@arkavo.com

normative:
  RFC2119:
  RTMP-SPEC:
    title: "Adobe's Real Time Messaging Protocol"
    target: https://www.adobe.com/devnet/rtmp.html
  OpenTDF:
    title: "OpenTDF Specification"
    target: https://github.com/opentdf/spec

--- abstract

This specification defines **NTDF-RTMP**, a protocol extension for embedding Attribute-Based Access Control (ABAC) and granular data encryption directly into Real-Time Messaging Protocol (RTMP) streams. Unlike RTMPS, which only secures the transport layer (TLS) between hops, NTDF-RTMP secures the data payload itself using **NanoTDF Collections**.

This architecture enables "End-to-End" zero-trust streaming where intermediate relay servers function as transparent couriers without access to the plaintext media. The protocol utilizes RTMP `onMetaData` messages to transport **Collection Manifests** (Headers) and standard FLV Media Tags to transport **Collection Items** (Encrypted Payloads).

--- middle
```
# Introduction

The Real-Time Messaging Protocol (RTMP) remains a standard for low-latency video ingest and delivery. While RTMPS provides transport security (SSL/TLS), it requires the server to terminate encryption, exposing plaintext video data to the media server provider or intermediate CDNs.

To achieve Zero-Trust streaming, the video data must remain encrypted from the source (Producer) to the destination (Consumer).

This specification adapts **NanoTDF Collections** to the RTMP stream structure. A "Collection" allows a single Policy and Wrapped Key (the **Manifest**) to govern an unlimited sequence of subsequent data chunks (**Items**).

## Terminology

- **Collection Manifest (Header)**: The NanoTDF header containing the Policy, KAS URL, and Wrapped DEK. In NTDF-RTMP, this is transmitted via the `onMetaData` AMF message or in-band header frames.
- **Collection Item (Payload)**: The encrypted video or audio data. In NTDF-RTMP, this replaces the raw body of FLV Video/Audio tags.
- **DEK**: Data Encryption Key (AES-256-GCM).
- **NTDF-RTMP**: The resulting stream format containing TDF-protected content.
- **In-Band Header Frame**: A special video frame containing the NanoTDF header, identified by magic bytes.

# Architecture Overview

NTDF-RTMP operates by interleaving control messages (Manifests) with media messages (Items) within a standard RTMP connection.

```text
[ Stream Timeline ] ----------------------------------------------->

| Handshake |  [onMetaData]  |  [Seq Hdr]  |  [NTDF Hdr]  |  [Video Tag]  |  [Video Tag] ...
                   ^               ^              ^               ^               ^
                   |               |              |               |               |
            (Sets Policy/Key) (Cleartext)   (In-band Key)    (Encrypted)    (Encrypted)
```

1.  **Session Initialization**: Standard RTMP Handshake.
2.  **Crypto Context**: The Publisher sends `onMetaData` containing the `ntdf_header` property (base64-encoded NanoTDF header).
3.  **Sequence Headers**: Video/Audio sequence headers (SPS/PPS, AAC config) are sent **unencrypted** for decoder initialization.
4.  **In-Band Header** (Optional): Publisher MAY send an in-band NTDF header frame for redundancy.
5.  **Consumption**: The Subscriber receives the Manifest, authenticates with the Key Access Service (KAS), and unwraps the DEK.
6.  **Playback**: Subsequent Video/Audio tags contain ciphertext. The Subscriber uses the cached DEK to decrypt and render.

# Message Formats

## 1. The Collection Manifest (onMetaData)

The NanoTDF Manifest is embedded within the standard `onMetaData` AMF data message.

*   **Message Type ID**: `0x12` (AMF0 Data)
*   **Name**: `onMetaData`
*   **Key Name**: `ntdf_header`
*   **Value Type**: AMF String
*   **Value Content**: Base64-encoded NanoTDF Header.

**Structure**:
```text
+--------------------------------------+
| AMF String: "onMetaData"             |
+--------------------------------------+
| AMF ECMA Array:                      |
|   "width" => 1920                    |
|   "height" => 1080                   |
|   "videocodecid" => "avc1"           |
|   "ntdf_header" => "<base64_string>" |  <-- NTDF Manifest
|   ...                                |
+--------------------------------------+
```

The **NanoTDF Header** MUST have the `Collection` configuration flag set, indicating it is a detached header.

### Advantages of onMetaData Transport

*   **No Client Changes Needed**: Standard tools (FFmpeg, OBS) natively support injecting custom metadata fields.
*   **No "Unknown Message" Errors**: Intermediate relays (nginx-rtmp, SRS) know how to parse and forward `onMetaData`.
*   **Ordering Guarantee**: `onMetaData` is always sent before video frames in a healthy RTMP handshake.

### Example: Publishing with FFmpeg

```bash
# Assuming HEADER_B64 contains a base64-encoded NanoTDF header
HEADER_B64="TDF...<base64_string>..."

ffmpeg -re -i input.mp4 \
  -c:v libx264 -c:a aac \
  -metadata ntdf_header="$HEADER_B64" \
  -f flv rtmp://localhost/live/stream
```

## 2. In-Band NTDF Header Frame

As a redundancy mechanism and for key rotation support, the NanoTDF header MAY also be transmitted as an in-band video frame. This ensures header delivery even if intermediate relays strip custom metadata fields.

**Format**:
```text
+-------------------------------------------------------+
| FLV Video Header (5 bytes)                            |
|   - FrameType (4 bits): 5 (video info/command frame)  |
|   - CodecID (4 bits): 7 (AVC)                         |
|   - AVCPacketType: 0                                  |
|   - CompositionTime: 0                                |
+-------------------------------------------------------+
| Magic Bytes: "NTDF" (4 bytes) = 0x4E 0x54 0x44 0x46   |
+-------------------------------------------------------+
| Header Length (2 bytes, big-endian)                   |
+-------------------------------------------------------+
| NanoTDF Header Bytes (variable)                       |
+-------------------------------------------------------+
```

### Detection

Receivers MUST check bytes 5-8 (after the FLV video header) for the magic sequence `0x4E544446` ("NTDF"). If present, the frame is an in-band header frame, not encrypted video data.

### Use Cases

*   **Redundancy**: Backup delivery if `onMetaData` is stripped by relay servers.
*   **Key Rotation**: Sent immediately before keyframes during mid-stream key changes.
*   **Late Joiners**: Relay servers cache and forward to new subscribers.

## 3. The Collection Item (Encrypted FLV Tag)

Standard RTMP Video (Type `0x09`) and Audio (Type `0x08`) tags are modified to carry encrypted payloads using a compact wire format optimized for streaming.

**Standard FLV Tag Structure**:
```text
| TagType | DataSize | Timestamp | StreamID | [ Data ... ] |
```

**NTDF-RTMP Encrypted Data Structure**:
The `[ Data ]` portion is replaced by the **Collection Item**:

```text
+-------------------------------------------------------+
| IV Counter (3 bytes, big-endian)                      |
+-------------------------------------------------------+
| Payload Length (3 bytes, big-endian)                  |
+-------------------------------------------------------+
| Ciphertext (Variable Length)                          |
|   - AES-256-GCM(DEK, IV_Full, Original_Data)          |
+-------------------------------------------------------+
| Authentication Tag (16 bytes)                         |
+-------------------------------------------------------+
```

### IV Generation

The 3-byte IV counter is expanded to a full 12-byte IV for AES-GCM:

```text
IV_Full (12 bytes) = Salt (4 bytes) || 0x00 0x00 0x00 0x00 0x00 || Counter (3 bytes)
```

*   **Salt**: Derived from the NanoTDF header (first 4 bytes of ephemeral public key).
*   **Counter**: 3-byte big-endian integer, starting at 0, incrementing per item.
*   **Capacity**: 16,777,216 items (16.7M) before key rotation required.
*   **Rotation Warning**: Implementations SHOULD warn at 8M items.

### Wire Format Efficiency

This compact format saves 13 bytes per frame compared to full NanoTDF item format:
- At 30 fps: ~1.4 MB/hour savings
- At 60 fps: ~2.8 MB/hour savings

# Protocol Flow

## Session State Machine

Receivers MUST implement a state machine to handle both encrypted and cleartext streams:

```text
                    ┌─────────────────┐
                    │   INITIALIZING  │
                    └────────┬────────┘
                             │
                    onMetaData received
                             │
              ┌──────────────┴──────────────┐
              │                             │
      ntdf_header present           ntdf_header absent
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │    ENCRYPTED    │           │   PASSTHROUGH   │
    │  (Crypto Active)│           │  (Cleartext)    │
    └─────────────────┘           └─────────────────┘
```

**State Transitions**:

1.  **INITIALIZING**: Connection established, awaiting metadata.
2.  **ENCRYPTED**: `ntdf_header` found in `onMetaData` or in-band header frame received. All subsequent media tags are decrypted using the DEK.
3.  **PASSTHROUGH**: No `ntdf_header` present. Stream is treated as standard cleartext RTMP.

**Error Conditions**:
*   Video/Audio data received while in INITIALIZING state indicates a protocol violation (missing metadata).

## Key Rotation (Mid-Stream Re-keying)

NTDF-RTMP supports mid-stream key rotation to enforce time-limited access or revoke users during a broadcast.

1.  **Trigger**: The Publisher decides to rotate the key (e.g., on keyframe interval).
2.  **Action**: Publisher performs the following sequence:
    a. Generate a new NanoTDF Collection with fresh DEK.
    b. Send updated `onMetaData` with new `ntdf_header` field.
    c. Send an in-band NTDF header frame immediately before the keyframe.
    d. Send the keyframe encrypted with the NEW key.
3.  **Synchronization**: The in-band header frame acts as a barrier. All Video/Audio tags *following* this frame MUST be encrypted with the NEW key.
4.  **Client Behavior**:
    *   Client detects new header (via metadata change or in-band frame).
    *   Client contacts KAS to unwrap the new Manifest.
    *   Client updates the active DEK.
    *   Client resumes decryption of subsequent tags.

**Note**: Clients SHOULD maintain the previous DEK briefly to handle any in-flight frames encrypted with the old key.

## Random Access / Late Joining

RTMP is a push stream. A client joining mid-stream might miss the initial `onMetaData`.

**Relay Server Requirements**:

RTMP Relay Servers (nginx-rtmp, SRS, etc.) **MUST**:

1.  Cache the most recent `onMetaData` (including `ntdf_header` if present).
2.  Cache video/audio sequence headers (SPS/PPS, AAC config).
3.  Cache the most recent in-band NTDF header frame (if supported).
4.  When a new client connects, send cached data in order:
    a. `onMetaData` (with `ntdf_header`)
    b. Video sequence header
    c. Audio sequence header
    d. In-band NTDF header frame (if cached)
    e. Live stream data starting from the next keyframe

**Compatibility Note**: Standard RTMP relays typically cache `onMetaData` and sequence headers (FrameType 1/2), but may NOT cache FrameType 5 (video info/command frames) used for in-band NTDF headers. Therefore:
*   `onMetaData` with `ntdf_header` is the **source of truth** for late-joining clients.
*   In-band header frames provide redundancy during live playback and key rotation, but cannot be relied upon for cold-start scenarios.
*   NTDF-aware relay servers SHOULD cache in-band header frames for enhanced late-joiner support.

# Policy Binding

The Policy embedded in the Manifest dictates access.

**Example Use Case**:
*   **Stream**: Confidential drone feed.
*   **Policy**: `attr:covert-ops/clearance/top-secret` AND `attr:drone-id/reaper-99`.
*   **Flow**:
    1.  Drone (Publisher) generates a Manifest with this policy.
    2.  Drone encrypts video frames as Collection Items.
    3.  RTMP Server relays the stream blindly (it cannot decrypt).
    4.  Command Center (Subscriber) receives the Manifest.
    5.  Subscriber validates credentials with KAS.
    6.  If authorized, Subscriber gets the DEK and views the stream.

# Implementation Details

## FLV/RTMP Compatibility

NTDF-RTMP preserves the outer FLV Tag structure (`Type`, `Timestamp`, `Size`). This ensures that standard RTMP servers (NGINX, SRS, Wowza) can relay the stream without modification. They view the encrypted data simply as "opaque video data."

However, standard RTMP *Players* (VLC, FFplay) will fail to decode the encrypted bytes. A custom player or a local NTDF-RTMP Proxy is required to decrypt the content before passing it to the decoder.

## Multi-Track Key Management

NTDF-RTMP streams typically contain multiple tracks (video, audio). This specification supports two key management approaches:

**Single Header (Default)**:
*   One `ntdf_header` in `onMetaData` governs all tracks.
*   Single DEK encrypts both video and audio frames.
*   **The Publisher MUST use a single, global atomic counter shared across all tracks (Audio, Video, Data) governed by the same NanoTDF Header.** This ensures IV uniqueness regardless of track interleaving.
*   Simplest implementation; one KAS rewrap per stream.

**Per-Track Headers (Future Consideration)**:
*   Separate policies per track (e.g., `ntdf_header_video`, `ntdf_header_audio`).
*   Enables scenarios where video requires higher clearance than audio, or vice versa.
*   Requires receivers to maintain multiple decryption contexts.
*   Key management implementations SHOULD be efficient enough to support multiple concurrent DEKs without significant latency impact.

**Recommendation**: Implementations SHOULD start with single-header mode. Per-track headers MAY be supported for advanced use cases where differentiated access control is required.

**Note**: When using per-track headers, each track maintains its own IV counter space, preventing counter collision.

## Sequence Headers (Cleartext)

H.264/AVC streams require an `AVCDecoderConfigurationRecord` (SPS/PPS) for decoder initialization. AAC audio requires an `AudioSpecificConfig`.

**NTDF-RTMP Requirement**: Sequence headers MUST be sent **unencrypted**:
*   Required for decoder initialization before first frame.
*   Allows relay servers to cache for late joiners.
*   Reveals codec profile/level but not content.

For maximum security contexts requiring encrypted headers, a custom decoder initialization mechanism would be needed (out of scope for this specification).

## Receiver Implementation Pseudocode

```text
state = INITIALIZING
crypto_context = null

match message {
    "onMetaData" => {
        metadata = parse_amf_map(message.body)
        if metadata.contains("ntdf_header") {
            header_bytes = base64_decode(metadata["ntdf_header"])
            crypto_context = create_decryptor(header_bytes)
            state = ENCRYPTED
        } else {
            state = PASSTHROUGH
        }
    },
    "VideoData" => {
        if is_ntdf_header_frame(message.data) {
            // In-band header (redundancy/key rotation)
            header_bytes = extract_ntdf_header(message.data)
            crypto_context = create_decryptor(header_bytes)
            state = ENCRYPTED
        } else if is_sequence_header(message.data) {
            // SPS/PPS - always cleartext
            init_decoder(message.data)
        } else if state == PASSTHROUGH {
            decode_cleartext_video(message.data)
        } else if state == ENCRYPTED {
            plaintext = crypto_context.decrypt(message.data)
            decode_video(plaintext)
        } else {
            error("Video data before metadata - protocol violation")
        }
    },
    "AudioData" => {
        if is_sequence_header(message.data) {
            // AAC config - always cleartext
            init_audio_decoder(message.data)
        } else if state == PASSTHROUGH {
            decode_cleartext_audio(message.data)
        } else if state == ENCRYPTED {
            plaintext = crypto_context.decrypt(message.data)
            decode_audio(plaintext)
        } else {
            error("Audio data before metadata - protocol violation")
        }
    }
}

fn is_ntdf_header_frame(data: bytes) -> bool {
    // 'data' is the Video Tag Payload (the [Data] portion of the FLV tag)
    // Byte layout within Video Tag Payload:
    //   [0]     FrameType (4 bits) + CodecID (4 bits)
    //   [1]     AVCPacketType (if AVC)
    //   [2-4]   CompositionTime (3 bytes, if AVC)
    //   [5-8]   Magic bytes location for NTDF header frame
    return data.length > 9 && data[5..9] == [0x4E, 0x54, 0x44, 0x46]
}

fn extract_ntdf_header(data: bytes) -> bytes {
    // Header length at bytes 9-10, header data starts at byte 11
    length = (data[9] << 8) | data[10]
    return data[11..11+length]
}
```

# Security Considerations

## Header Reuse Vulnerability (CRITICAL)

**Publishers MUST generate a FRESH NanoTDF header (with new ephemeral key pair) for every streaming session.**

Reusing a static header file across sessions is a **fatal cryptographic error**:

1.  The Salt (derived from the ephemeral public key) remains identical.
2.  The IV counter resets to 0 at session start.
3.  Result: Multiple sessions use the exact same (Key + IV) sequence.
4.  **Attack**: An adversary capturing two sessions can XOR the ciphertexts to recover plaintext (known as a "two-time pad" attack).

**Forbidden Pattern**:
```bash
# DANGEROUS: Static header reused across sessions
HEADER_B64="<static_base64_header>"
ffmpeg ... -metadata ntdf_header="$HEADER_B64" ...  # Session 1
# (stream stops)
ffmpeg ... -metadata ntdf_header="$HEADER_B64" ...  # Session 2 - VULNERABLE!
```

**Required Pattern**:
```bash
# SAFE: Fresh header generated per session
HEADER_B64=$(generate_fresh_nanotdf_header)  # New ephemeral key
ffmpeg ... -metadata ntdf_header="$HEADER_B64" ...
```

Implementations MUST:
*   Generate a new ephemeral key pair for each streaming session.
*   Never persist or reuse header files across sessions.
*   Consider embedding session identifiers or timestamps in tooling to prevent accidental reuse.

## Relay Server Trust

The RTMP Relay Server does **not** possess the DEK. It cannot view, record, or modify the content without breaking the GCM Authentication Tag. This achieves "Zero Trust" regarding the infrastructure provider.

## Replay Attacks

The Manifest contains a Policy. If the Policy includes an `expiration` attribute, captured streams cannot be replayed after the window closes, even if the attacker has the encrypted files.

## Metadata Leakage

Standard metadata fields (width, height, codec) in `onMetaData` remain visible to relay servers. Sequence headers also reveal codec profile information. For maximum confidentiality:
*   Publishers SHOULD minimize standard metadata fields.
*   Consider the trade-off between compatibility and metadata privacy.

## IV Counter Exhaustion

The 3-byte IV counter supports 16,777,216 items per key. Implementations MUST:
*   Track the counter and rotate keys before exhaustion.
*   Warn operators at 8M items (50% capacity).
*   Never reuse an IV with the same key.

**Practical Capacity**: At 60fps video + 48kbps AAC (~46 audio frames/sec) = ~106 items/second:
*   Full capacity: ~44 hours of continuous streaming.
*   Warning threshold (8M): ~22 hours.
*   For typical RTMP sessions, key rotation due to IV exhaustion is unlikely.

## Latency

KAS interaction adds latency only at the **Start** of the stream and during **Key Rotation**.
*   Initial Handshake: +1 RTT (Round Trip Time) to KAS.
*   Frame Decryption: Negligible (<1ms AES-GCM overhead).

# IANA Considerations

This document proposes the registration of:
*   Metadata property name: `ntdf_header`
*   In-band header magic bytes: `0x4E544446` ("NTDF")

---
**Document History:**
- **draft-arkavo-ntdf-rtmp-02** (2025-12-13): Removed `onTDFManifest` in favor of `onMetaData`-only transport. Added in-band NTDF header frame for redundancy and key rotation. Updated Collection Item to compact wire format (3-byte IV counter). Clarified sequence headers sent unencrypted. Based on working implementation.
- **draft-arkavo-ntdf-rtmp-01** (2025-11-29): Added `onMetaData` as alternative manifest transport for compatibility with standard RTMP clients. Added session state machine. Added receiver implementation pseudocode.
- **draft-arkavo-ntdf-rtmp-00** (2025-11-22): Initial draft adapting NanoTDF Collections to RTMP.
