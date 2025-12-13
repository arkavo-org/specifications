```text
---
title: "NTDF-RTMP: Secure RTMP via NanoTDF Collections"
abbrev: "NTDF-RTMP"
docname: draft-arkavo-ntdf-rtmp-00
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

This architecture enables "End-to-End" zero-trust streaming where intermediate relay servers function as transparent couriers without access to the plaintext media. The protocol utilizes RTMP AMF Data Messages to transport **Collection Manifests** (Headers) and standard FLV Media Tags to transport **Collection Items** (Encrypted Payloads).

--- middle
```
# Introduction

The Real-Time Messaging Protocol (RTMP) remains a standard for low-latency video ingest and delivery. While RTMPS provides transport security (SSL/TLS), it requires the server to terminate encryption, exposing plaintext video data to the media server provider or intermediate CDNs.

To achieve Zero-Trust streaming, the video data must remain encrypted from the source (Producer) to the destination (Consumer).

This specification adapts **NanoTDF Collections** to the RTMP stream structure. A "Collection" allows a single Policy and Wrapped Key (the **Manifest**) to govern an unlimited sequence of subsequent data chunks (**Items**).

## Terminology

- **Collection Manifest (Header)**: The NanoTDF header containing the Policy, KAS URL, and Wrapped DEK. In NTDF-RTMP, this is transmitted via an AMF Data Message.
- **Collection Item (Payload)**: The encrypted video or audio data. In NTDF-RTMP, this replaces the raw body of FLV Video/Audio tags.
- **DEK**: Data Encryption Key (AES-256-GCM).
- **NTDF-RTMP**: The resulting stream format containing TDF-protected content.

# Architecture Overview

NTDF-RTMP operates by interleaving control messages (Manifests) with media messages (Items) within a standard RTMP connection.

```text
[ Stream Timeline ] --------------------------------------------->

| Handshake |  [onTDFManifest]  |  [Video Tag]  |  [Video Tag] ...
                  ^                   ^               ^
                  |                   |               |
            (Sets Policy/Key)    (Encrypted)     (Encrypted)
```

1.  **Session Initialization**: Standard RTMP Handshake.
2.  **Crypto Context**: The Publisher sends an `onTDFManifest` data message.
3.  **Consumption**: The Subscriber receives the Manifest, authenticates with the Key Access Service (KAS), and unwraps the DEK.
4.  **Playback**: Subsequent Video/Audio tags contain ciphertext. The Subscriber uses the cached DEK to decrypt and render.

# Message Formats

## 1. The Collection Manifest (AMF Data Message)

To establish a cryptographic context, NTDF-RTMP defines a custom AMF0 Data Message.

*   **Message Type ID**: `0x12` (AMF0 Data)
*   **Name**: `onTDFManifest`
*   **Body**: A binary ByteArray containing the standard NanoTDF Header.

**Structure**:
```text
+------------------------------+
| AMF String: "onTDFManifest"  |
+------------------------------+
| AMF ByteArray: [Length]      |
| [NanoTDF Header Bytes...]    |
+------------------------------+
```

The **NanoTDF Header** MUST have the `Collection` configuration flag set, indicating it is a detached header.

## 2. The Collection Item (Encrypted FLV Tag)

Standard RTMP Video (Type `0x09`) and Audio (Type `0x08`) tags are modified to carry encrypted payloads.

**Standard FLV Tag Structure**:
```text
| TagType | DataSize | Timestamp | StreamID | [ Data ... ] |
```

**NTDF-RTMP Encrypted Data Structure**:
The `[ Data ]` portion is replaced by the **Collection Item**:

```text
+-------------------------------------------------------+
| Initialization Vector (IV) (12 bytes)                 |
+-------------------------------------------------------+
| Ciphertext (Variable Length)                          |
|   - AES-256-GCM(DEK, IV, Original_Video_Data)         |
+-------------------------------------------------------+
| Authentication Tag (16 bytes)                         |
+-------------------------------------------------------+
```

### IV Generation
To support stateless seeking and packet loss recovery, the IV **MUST NOT** be dependent on a chained state (like CBC).
*   Recommended: `Salt (4 bytes) || Sequence_Number (8 bytes)`
*   Alternatively: Random 12 bytes (adds overhead but ensures uniqueness).

# Protocol Flow

## Key Rotation (The "Crypto Keyframe")

NTDF-RTMP supports mid-stream key rotation (re-keying) to enforce time-limited access or revoke users during a broadcast.

1.  **Trigger**: The Publisher decides to rotate the key.
2.  **Action**: Publisher injects a **new** `onTDFManifest` message into the RTMP stream.
3.  **Synchronization**: This message acts as a barrier. All Video/Audio tags *following* this message MUST be encrypted with the NEW key defined in that Manifest.
4.  **Client Behavior**:
    *   Client receives `onTDFManifest`.
    *   Client pauses playback (buffering).
    *   Client contacts KAS to unwrap the new Manifest.
    *   Client updates the active DEK.
    *   Client resumes decryption of subsequent tags.

## Random Access / Late Joining

Unlike HLS (where the playlist links to the key), RTMP is a push stream. A client joining mid-stream might miss the initial `onTDFManifest`.

**The Cache-Key mechanism**:
RTMP Servers (like NGINX-RTMP) typically cache the "GOP Header" (Sequence Header / AVCDecoderConfigurationRecord). NTDF-RTMP extends this requirement:

*   The RTMP Relay Server **MUST** cache the most recent `onTDFManifest` message.
*   When a new client connects, the Server **MUST** send the cached `onTDFManifest` immediately after the RTMP `NetStream.Play.Start` event and before the first Video Tag.

# Policy Binding

The Policy embedded in the `onTDFManifest` dictates access.

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

## Audio/Video Configuration Packets
H.264/AVC streams usually start with an `AVCDecoderConfigurationRecord` (SPS/PPS).
*   **Option A (Encrypted Headers)**: Encrypt the SPS/PPS. High security, but prevents the relay server from knowing resolution/framerate.
*   **Option B (Cleartext Headers)**: Leave the AVC Configuration packet unencrypted. Allows the server to parse metadata, but leaks resolution/profile info.
*   **Recommendation**: Option A is preferred for high-security contexts.

# Security Considerations

## Relay Server Trust
The RTMP Relay Server does **not** possess the DEK. It cannot view, record, or modify the content without breaking the GCM Authentication Tag. This achieves "Zero Trust" regarding the infrastructure provider.

## Replay Attacks
The `onTDFManifest` contains a Policy. If the Policy includes a `expiration` attribute, captured streams cannot be replayed after the window closes, even if the attacker has the encrypted files.

## Latency
KAS interaction adds latency only at the **Start** of the stream and during **Key Rotation**.
*   Initial Handshake: +1 RTT (Round Trip Time) to KAS.
*   Frame Decryption: Negligible (<1ms AES-GCM overhead).

# IANA Considerations

This document proposes the registration of the AMF Data Message name: `onTDFManifest`.

---
**Document History:**
- **draft-arkavo-ntdf-rtmp-00** (2025-11-22): Initial draft adapting NanoTDF Collections to RTMP.