---
title: "NTDF Tokens: NanoTDF-based Authentication Tokens"
abbrev: "NTDF Tokens"
docname: draft-arkavo-ntdf-token-00
category: exp
ipr: trust200902
area: Security
workgroup: Arkavo Specifications
keyword:
 - authentication
 - NanoTDF
 - OpenTDF
 - DPoP
 - proof-of-possession

author:
 - fullname: Arkavo Contributors
   organization: Arkavo
   email: specifications@arkavo.org

normative:
  RFC2119:
  RFC9449:
  OpenTDF:
    title: "OpenTDF Specification"
    target: https://github.com/opentdf/spec
  Z85:
    title: "ZeroMQ RFC 32: Z85 - ZeroMQ Base-85 Encoding Algorithm"
    target: https://rfc.zeromq.org/spec/32/

informative:
  Ed25519:
    title: "Ed25519: high-speed high-security signatures"
    target: https://ed25519.cr.yp.to/

--- abstract

This specification defines NTDF (NanoTDF) tokens, a cryptographically-bound authentication token format based on OpenTDF's NanoTDF specification. NTDF tokens replace traditional JWT Bearer tokens with a format that provides policy binding, provenance tracking, optional payload confidentiality, and integration with DPoP (Demonstrating Proof-of-Possession) per RFC 9449.

--- middle

# Introduction

Traditional JWT Bearer tokens suffer from several security limitations:

- **No proof-of-possession**: Tokens can be stolen and replayed
- **No policy binding**: Tokens cannot enforce cryptographic policy constraints
- **No confidentiality**: Claims are base64-encoded, not encrypted
- **Limited provenance**: Signatures validate issuer, not binding to client

NTDF tokens address these limitations by wrapping authentication claims in a NanoTDF container, providing:

- **DPoP proof-of-possession** to prevent token theft
- **Policy binding** enforced cryptographically at the Key Access Service (KAS)
- **Payload confidentiality** through AES-256-GCM encryption
- **Provenance tracking** with Ed25519 signatures
- **Consistency** with content encryption (uses same NanoTDF format)

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

- **NTDF Token**: A NanoTDF-encoded authentication token containing user claims
- **NanoTDF**: OpenTDF's compact binary TDF format
- **Z85**: ZeroMQ Base-85 encoding (RFC 32)
- **DPoP**: Demonstrating Proof-of-Possession (RFC 9449)
- **KAS**: Key Access Service - holds private keys for unwrapping
- **Policy Binding**: Cryptographic binding of token to access control policy
- **DEK**: Data Encryption Key
- **ECDH**: Elliptic Curve Diffie-Hellman key agreement
- **HKDF**: HMAC-based Key Derivation Function

# Wire Format

NTDF tokens are transmitted in HTTP Authorization headers using the "NTDF" scheme:

```
Authorization: NTDF <Z85-encoded-nanotdf>
```

Example:

```
Authorization: NTDF rGN.bK+V#7c@{h.wx-F*0NG%kf7...
```

The token string following "NTDF " is a Z85-encoded NanoTDF container.

## Z85 Encoding

NTDF tokens use Z85 (ZeroMQ RFC 32) encoding, which:

- Encodes binary data into printable ASCII characters
- Uses 85-character alphabet (excludes quotes, backslash, whitespace)
- Encodes 4 bytes into 5 characters (~25% overhead)
- Is URL-safe and suitable for HTTP headers

# NanoTDF Container Structure

The Z85-decoded NTDF token is a NanoTDF container with the following structure:

```
+---------------------------------------------------+
| Magic Number (3 bytes): "L1N" (0x4C 0x31 0x4E)   |
| Version (1 byte): 0x4C (v1.2) or 0x4D (v1.3)     |
+---------------------------------------------------+
| KAS Resource Locator                              |
|   - Protocol (1 byte): HTTPS=0x01, WSS=0x03       |
|   - Length (1 byte)                               |
|   - URL (variable)                                |
+---------------------------------------------------+
| ECC Mode (1 byte)                                 |
|   - Use ECDSA Binding: 0 (GMAC)                   |
|   - Curve: SECP256R1 (P-256)                      |
+---------------------------------------------------+
| Payload Config (1 byte)                           |
|   - Cipher: AES-256-GCM (0x03)                    |
|   - Has Signature: 1 (Ed25519)                    |
+---------------------------------------------------+
| Policy                                            |
|   - Type (1 byte): Remote=0x00 or Embedded=0x01   |
|   - Policy ID (16 bytes): UUID or SHA-256 hash    |
|   - Binding (8 bytes): GMAC tag                   |
+---------------------------------------------------+
| Ephemeral Public Key (33 bytes)                   |
|   - Compressed P-256 point (SEC1 format)          |
+---------------------------------------------------+
| Encrypted Payload                                 |
|   - Length (3 bytes, big-endian)                  |
|   - Ciphertext (variable)                         |
|   - Auth Tag (16 bytes, appended)                 |
+---------------------------------------------------+
| Signature (64 bytes)                              |
|   - Ed25519 over (Header||Policy||Payload)        |
+---------------------------------------------------+
```

## Size Characteristics

| Component | Size |
|-----------|------|
| Minimum raw | ~380 bytes |
| Typical raw | 450-550 bytes |
| Z85-encoded minimum | ~500 characters |
| Z85-encoded typical | 600-730 characters |

Breakdown:
- Header + Policy: 60-100 bytes
- Ephemeral key: 33 bytes (compressed P-256)
- Encrypted payload: 150-300 bytes (depends on claims)
- Auth tag: 16 bytes
- Signature: 64 bytes
- Z85 overhead: +25%

# Payload Schema

The decrypted NanoTDF payload contains authentication claims in a compact binary format.

## Payload Structure

```rust
struct NtdfTokenPayload {
    sub_id: [u8; 16],              // Subject UUID
    flags: u64,                     // Capability flags
    scopes: Vec<String>,            // OAuth scopes
    attrs: Vec<(u8, u32)>,          // Custom attributes
    dpop_jti: Option<[u8; 16]>,     // DPoP JTI (if bound)
    iat: i64,                       // Issued at (Unix timestamp)
    exp: i64,                       // Expiration (Unix timestamp)
    aud: String,                    // Audience (target service)
    session_id: Option<[u8; 16]>,   // Session tracking ID
    device_id: Option<String>,      // Device identifier
    did: Option<String>,            // Decentralized Identifier
}
```

## Binary Serialization

All multi-byte integers are little-endian.

```
+-----------------------------------------------------+
| sub_id (16 bytes)                                   |
+-----------------------------------------------------+
| flags (8 bytes, u64 little-endian)                  |
+-----------------------------------------------------+
| scopes_count (2 bytes, u16 little-endian)           |
| +─────────────────────────────────────────────────+ |
| | For each scope:                                 | |
| |   - length (1 byte, u8)                        | |
| |   - UTF-8 string (variable)                    | |
| +─────────────────────────────────────────────────+ |
+-----------------------------------------------------+
| attrs_count (2 bytes, u16 little-endian)            |
| +─────────────────────────────────────────────────+ |
| | For each attribute:                            | |
| |   - type (1 byte, u8)                          | |
| |   - value (4 bytes, u32 little-endian)         | |
| +─────────────────────────────────────────────────+ |
+-----------------------------------------------------+
| dpop_jti_present (1 byte, 0=absent, 1=present)      |
| dpop_jti (16 bytes if present)                      |
+-----------------------------------------------------+
| iat (8 bytes, i64 little-endian)                    |
+-----------------------------------------------------+
| exp (8 bytes, i64 little-endian)                    |
+-----------------------------------------------------+
| aud_length (2 bytes, u16 little-endian)             |
| aud (variable, UTF-8 string)                        |
+-----------------------------------------------------+
| session_id_present (1 byte, 0=absent, 1=present)    |
| session_id (16 bytes if present)                    |
+-----------------------------------------------------+
| device_id_present (1 byte, 0=absent, 1=present)     |
| device_id_length (1 byte if present)                |
| device_id (variable UTF-8 string if present)        |
+-----------------------------------------------------+
| did_present (1 byte, 0=absent, 1=present)           |
| did_length (2 bytes, u16 little-endian if present)  |
| did (variable UTF-8 string if present)              |
+-----------------------------------------------------+
```

## Capability Flags

The `flags` field is a 64-bit bitfield representing granted capabilities:

| Flag | Bit | Value | Description |
|------|-----|-------|-------------|
| PROFILE | 0 | 0x01 | User profile access |
| OPENID | 1 | 0x02 | OpenID Connect |
| EMAIL | 2 | 0x04 | Email access |
| OFFLINE_ACCESS | 3 | 0x08 | Refresh token capability |
| DEVICE_ATTESTED | 4 | 0x10 | Device attestation present |
| BIOMETRIC_AUTH | 5 | 0x20 | Biometric authentication used |
| WEBAUTHN | 6 | 0x40 | WebAuthn/Passkey authentication |
| PLATFORM_SECURE | 7 | 0x80 | Platform secure (not jailbroken) |

Additional flags may be defined in bits 8-63.

## Attribute Types

The `attrs` field contains typed key-value pairs:

```rust
enum AttributeType {
    Age = 0,               // User age for content rating
    SubscriptionTier = 1,  // free=0, basic=1, premium=2
    SecurityLevel = 2,     // baseline=0, main=1, high=2
    PlatformCode = 3,      // iOS=0, Android=1, macOS=2, etc.
}
```

Each attribute is encoded as:
- 1 byte: attribute type (from enum)
- 4 bytes: u32 value

# Cryptographic Algorithms

## Elliptic Curve Cryptography

- **Curve**: P-256 (secp256r1, NIST P-256)
- **Key Size**: 256 bits (32 bytes)
- **Public Key Format**: 33 bytes compressed (SEC1), 65 bytes uncompressed
- **ECDH Output**: 32-byte x-coordinate

## Key Derivation (HKDF-SHA256)

HKDF-SHA256 derives the AES key from the ECDH shared secret:

- **Hash Function**: SHA-256
- **Salt**: `SHA256("L1" + version_byte)`
  - v1.2: `SHA256([0x4C, 0x31, 0x4C])`
  - v1.3: `SHA256([0x4C, 0x31, 0x4D])`
- **IKM**: ECDH shared secret (32 bytes)
- **Info**: Empty byte array
- **Output**: 32-byte AES-256 key

## Symmetric Encryption (AES-256-GCM)

- **Cipher**: AES-256-GCM (Galois/Counter Mode)
- **Key Size**: 256 bits (32 bytes from HKDF)
- **Nonce**: 12 bytes of zeros
  - Safe because each NanoTDF has unique ephemeral key
  - ECDH produces unique shared secret per token
  - No key reuse across tokens
- **Authentication Tag**: 16 bytes (appended to ciphertext)

## Digital Signatures (Ed25519)

- **Algorithm**: Ed25519 (EdDSA over Curve25519)
- **Signature Size**: 64 bytes
- **Public Key Size**: 32 bytes
- **Signed Data**: Concatenation of Header, Policy, and Encrypted Payload bytes

**Implementation Status**: Parsing implemented, verification TODO.

# Token Validation

Validating an NTDF token involves the following steps:

## Validation Procedure

1. **Extract Authorization Header**
   ```
   Authorization: NTDF <z85_token>
   ```

2. **Z85 Decoding**
   ```
   nanotdf_bytes = z85_decode(z85_token)
   ```

3. **Version Detection**
   - Check magic number: `[0x4C, 0x31]` ("L1")
   - Read version byte: `0x4C` (v1.2) or `0x4D` (v1.3)
   - Compute salt: `SHA256("L1" + version_byte)`

4. **Parse NanoTDF Header**
   - Magic number and version
   - KAS resource locator
   - ECC mode (curve, binding type)
   - Payload configuration (cipher, has_signature)
   - Policy (type, ID, binding)
   - Ephemeral public key (33 bytes compressed P-256)
   - Encrypted payload length and data

5. **Load KAS Private Key**
   - Load P-256 private key from PEM format
   - Typically stored securely on KAS server

6. **Extract Ephemeral Public Key**
   - Parse 33-byte compressed P-256 public key from header
   - Validate key is on curve

7. **ECDH Key Agreement**
   ```
   shared_point = ephemeral_public * kas_private
   shared_secret = x_coordinate(shared_point)  // 32 bytes
   ```

8. **HKDF Key Derivation**
   ```
   hkdf = HKDF-SHA256(salt, shared_secret)
   aes_key = hkdf.expand(info="", length=32)
   ```

9. **Read Encrypted Payload**
   - Extract 3-byte big-endian payload length
   - Read ciphertext (length - 16 bytes)
   - Read 16-byte authentication tag
   - Combine: `encrypted = ciphertext || tag`

10. **AES-256-GCM Decryption**
    ```
    cipher = AES-256-GCM(aes_key)
    nonce = [0x00; 12]
    plaintext = cipher.decrypt(nonce, encrypted)
    ```

11. **Deserialize Payload**
    - Parse binary payload per Section 5.2
    - Extract all claims

12. **Validate Claims**
    - Check expiration: `now < exp`
    - Check audience: `aud == expected_audience`
    - Optionally check `iat` (issued-at time)
    - Optionally verify session not revoked (check `session_id` in Redis/DB)

13. **Verify Signature (TODO)**
    - Extract 64-byte Ed25519 signature from NanoTDF
    - Verify signature over `Header || Policy || Encrypted_Payload`
    - Validate against trusted signing public key

## Validation Result

Validation produces one of:

- **Valid**: Token valid, returns `ValidatedNtdfToken`
- **InvalidFormat**: Malformed NanoTDF or binary payload
- **InvalidSignature**: Ed25519 signature verification failed
- **Expired**: Current time ≥ `exp`
- **InvalidAudience**: `aud` doesn't match expected value
- **DecryptionFailed**: AES-GCM authentication tag mismatch
- **PolicyViolation**: Policy binding verification failed

# DPoP Integration

NTDF tokens integrate with DPoP (RFC 9449) to prevent token theft through proof-of-possession.

## DPoP JWT Structure

### Header

```json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "<base64url-encoded-x>",
    "y": "<base64url-encoded-y>"
  }
}
```

### Claims

```json
{
  "jti": "<uuid>",
  "htm": "POST",
  "htu": "https://kas.example.com/kas/v2/rewrap",
  "iat": 1698765432,
  "ath": "<base64url-sha256>"
}
```

## Critical: ath Computation

The `ath` (access token hash) claim **MUST** be computed over the **raw Z85-encoded string**, not the decoded NanoTDF bytes.

**Correct**:
```
z85_token = "rGN.bK+V#7c@{h..."
ath = base64url(SHA256(z85_token.as_bytes()))
```

**Incorrect**:
```
nanotdf_bytes = z85_decode(z85_token)
ath = base64url(SHA256(nanotdf_bytes))  // WRONG
```

This ensures the DPoP proof binds to the exact wire representation.

## JTI Matching

The DPoP JWT's `jti` claim MUST match the `dpop_jti` field in the NTDF token payload:

```
DPoP.jti == NtdfTokenPayload.dpop_jti
```

Both are 16-byte UUIDs. The `jti` in the DPoP JWT is base64url-encoded, while `dpop_jti` in the NTDF payload is raw bytes.

## DPoP Validation Flow

1. Extract `DPoP` header from HTTP request
2. Decode JWT without verification to extract header
3. Verify `typ == "dpop+jwt"`
4. Verify `alg == "ES256"`
5. Extract JWK from header, validate EC P-256
6. Construct public key from JWK coordinates
7. Decode and verify JWT signature with constructed key
8. Verify timestamp freshness: `now - iat ≤ max_age` (typically 60 seconds)
9. Verify HTTP method: `htm == request.method`
10. Verify HTTP URI: `htu == normalize(request.uri)` (strip query/fragment)
11. Compute `ath` over Z85 token string
12. Verify `ath` matches DPoP claim
13. Verify `jti` matches NTDF `dpop_jti`

If all checks pass, the token is bound to the client's ephemeral key pair.

# Usage in Wire Protocols

## WebSocket Upgrade Authentication

NTDF tokens are validated during the WebSocket upgrade handshake:

```
Client -> Server:
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Authorization: NTDF rGN.bK+V#7c@{h...

Server validates NTDF token:
  - Parse Authorization header
  - Validate NTDF token
  - Extract sub_id for user identification
  - If valid: accept upgrade
  - If invalid: return 401 Unauthorized
```

After successful upgrade:
- Server subscribes to user-specific NATS subject: `profile.<hex(sub_id)>`
- WebSocket connection state stores validated claims
- Binary messages use established authentication context

## HTTP Rewrap Endpoint

NTDF tokens authenticate rewrap requests:

```
POST /kas/v2/rewrap HTTP/1.1
Authorization: NTDF rGN.bK+V#7c@{h...
DPoP: eyJhbGciOiJFUzI1NiIsInR5cCI6ImRwb3Aran...
Content-Type: application/json

{
  "signed_request_token": "{...}",
  ...
}
```

Validation flow:
1. Validate NTDF token from `Authorization` header
2. If DPoP enabled:
   a. Validate DPoP proof from `DPoP` header
   b. Verify `ath` over Z85 token
   c. Verify `jti` match
3. Parse rewrap request
4. Perform ECDH and key rewrapping
5. Return wrapped keys

## Policy Contract Integration

NTDF tokens provide authenticated claims for policy enforcement:

```rust
// Extract user identifier
user_id = hex_encode(ntdf_claims.sub_id)

// Check capability flags
has_profile = (ntdf_claims.flags & PROFILE) != 0

// Access attributes
subscription_tier = ntdf_claims.attrs.find(SubscriptionTier)
device_attested = (ntdf_claims.flags & DEVICE_ATTESTED) != 0

// Policy decisions
if required_tier > subscription_tier {
    deny_access("Insufficient subscription")
}
```

Available for policy enforcement:
- `sub_id`: User UUID
- `flags`: Capability bitfield
- `scopes`: OAuth scopes
- `attrs`: Typed attributes (age, tier, security level, platform)
- `session_id`: Session tracking
- `device_id`: Device attestation
- `did`: Decentralized identifier

# Security Considerations

## Token Confidentiality

The payload is encrypted with AES-256-GCM. Even if intercepted, the attacker cannot read claims without:
- Possessing the KAS private key
- Performing ECDH with the ephemeral public key
- Having the correct policy binding

This provides confidentiality even over untrusted channels.

## Token Binding (DPoP)

**Without DPoP**: NTDF tokens can be stolen and replayed (similar to JWT Bearer tokens).

**With DPoP**: Tokens are cryptographically bound to the client's ephemeral key pair. An attacker who steals the token cannot use it without the corresponding private key.

DPoP provides:
- Proof-of-possession
- Prevention of token theft
- Protection against man-in-the-middle attacks

## Policy Enforcement

The GMAC policy binding ensures:
- Token cannot be used with a different policy
- Policy changes invalidate existing tokens
- KAS can enforce policy constraints at unwrap time

The policy binding is cryptographically verified during decryption, preventing policy substitution attacks.

## Signature Verification

Ed25519 signatures (when implemented) provide:
- **Integrity**: Token has not been tampered with
- **Non-repudiation**: Issuer cannot deny creating the token
- **Origin authentication**: Verifies token came from trusted issuer

The signature covers `Header || Policy || Encrypted_Payload`, ensuring end-to-end integrity.

## Lifetime Management

- **iat (issued-at)**: Prevents premature token use, detects clock skew
- **exp (expiration)**: Enforces token lifetime, requires validation
- **session_id**: Enables server-side revocation via Redis/database lookup

Tokens with expired `exp` MUST be rejected even if signature is valid.

## Nonce Reuse Safety

AES-256-GCM uses a zero nonce (`[0x00; 12]`), which is safe because:
- Each NanoTDF container has a unique ephemeral public key
- ECDH produces a unique shared secret per token
- The AES key is never reused across different tokens
- GCM authentication tag prevents tampering

This follows the NanoTDF specification's nonce usage.

## Replay Prevention

NTDF tokens combined with DPoP provide replay protection:
- DPoP `jti` is unique per request
- DPoP `iat` timestamp enforces freshness (typically 60 seconds)
- DPoP `htm` and `htu` bind to specific HTTP method and URI
- Server can track used `jti` values to prevent reuse

## Key Management

- **KAS Private Key**: MUST be protected with HSM or secure key storage
- **Ephemeral Keys**: Generated per token, never reused
- **Signing Keys**: Ed25519 private key MUST be secured, ideally rotated periodically

# Implementation Status

## Completed Features

- ✅ NtdfTokenPayload binary serialization/deserialization
- ✅ Z85 encoding/decoding
- ✅ NTDF token validation (parse, decrypt, verify claims)
- ✅ ECDH key agreement with x-coordinate extraction
- ✅ HKDF-SHA256 key derivation with version-based salt
- ✅ AES-256-GCM decryption with authentication
- ✅ WebSocket upgrade authentication
- ✅ HTTP rewrap endpoint authentication
- ✅ DPoP validation per RFC 9449
- ✅ DPoP-NTDF binding (ath over Z85, jti matching)
- ✅ Policy contract integration
- ✅ Claim validation (exp, aud)
- ✅ Session tracking support

## TODO Items

1. **Ed25519 Signature Verification**
   - Parse 64-byte signature from NanoTDF
   - Verify over `Header || Policy || Encrypted_Payload`
   - Validate against trusted signing public key

2. **Remote Policy Fetching**
   - HTTP/HTTPS policy URL resolution
   - Policy caching
   - Metadata parsing

3. **Advanced Policy Features**
   - Geolocation extraction for geofence contracts
   - Device attestation validation
   - Age verification flag definition

# Examples

## WebSocket Authentication Flow

```
Client                                    Server
  |                                         |
  |--- WS Upgrade + Authorization: NTDF --> |
  |    (Z85-encoded NanoTDF token)          |
  |                                         |
  |                          [Validate NTDF token]
  |                          [Extract sub_id, flags]
  |                          [Subscribe to profile.<sub_id>]
  |                                         |
  | <------------- WS 101 Switching --------|
  |                                         |
  |<============ Authenticated Connection ==|
  |                                         |
```

## HTTP Rewrap with DPoP Flow

```
Client                                    Server
  |                                         |
  |--- POST /kas/v2/rewrap ---------------> |
  |    Authorization: NTDF <token>          |
  |    DPoP: <jwt>                          |
  |    Body: { rewrap request }             |
  |                                         |
  |                          [Validate NTDF token]
  |                          [Validate DPoP proof]
  |                          [Verify ath over Z85]
  |                          [Verify jti match]
  |                          [Perform rewrap]
  |                                         |
  | <------------- 200 OK ------------------|
  |    { rewrapped keys }                   |
  |                                         |
```

## Token Generation Example

Pseudocode for generating an NTDF token:

```rust
// 1. Create payload
let payload = NtdfTokenPayload {
    sub_id: user_uuid,
    flags: WEBAUTHN | PLATFORM_SECURE | PROFILE,
    scopes: vec!["openid", "profile"],
    attrs: vec![(Age, 25), (SubscriptionTier, 2)],
    dpop_jti: Some(dpop_jti),
    iat: now(),
    exp: now() + 3600,
    aud: "https://kas.example.com",
    session_id: Some(session_uuid),
    device_id: Some("iPhone14,2"),
    did: Some("did:key:z6Mk..."),
};

// 2. Serialize payload to bytes
let payload_bytes = payload.to_bytes();

// 3. Generate ephemeral key pair
let ephemeral_private = generate_p256_key();
let ephemeral_public = ephemeral_private.public_key();

// 4. Perform ECDH with KAS public key
let shared_secret = ecdh(ephemeral_private, kas_public_key);

// 5. Derive AES key with HKDF
let salt = sha256("L1" + nanotdf_version);
let aes_key = hkdf_sha256(salt, shared_secret, info="", len=32);

// 6. Encrypt payload with AES-256-GCM
let nonce = [0u8; 12];
let (ciphertext, tag) = aes256gcm_encrypt(aes_key, nonce, payload_bytes);

// 7. Build NanoTDF header
let header = build_nanotdf_header(
    kas_url,
    policy_id,
    ephemeral_public,
    ciphertext_length,
);

// 8. Sign with Ed25519
let signature = ed25519_sign(signing_key, header || policy || ciphertext || tag);

// 9. Assemble NanoTDF
let nanotdf = header || policy || ephemeral_public ||
              length || ciphertext || tag || signature;

// 10. Z85 encode
let z85_token = z85_encode(nanotdf);

// 11. Return
return "NTDF " + z85_token;
```

## Complete Request Example

```http
POST /kas/v2/rewrap HTTP/1.1
Host: kas.example.com
Content-Type: application/json
Authorization: NTDF rGN.bK+V#7c@{h.wx-F*0NG%kf7w6jUVUqNT_Xyw5...
DPoP: eyJhbGciOiJFUzI1NiIsInR5cCI6ImRwb3Aran0.eyJqdGkiOiI...

{
  "signed_request_token": "{"clientPublicKey":"-----BEGIN PUBLIC KEY-----\nMFkw...",
  ...
}
```

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "responses": [{
    "policyId": "uuid-or-hash",
    "results": [{
      "keyAccessObjectId": "kao-id",
      "status": "permit",
      "kasWrappedKey": "base64-encoded-wrapped-key"
    }]
  }],
  "sessionPublicKey": "-----BEGIN PUBLIC KEY-----\n...",
  "schemaVersion": "1.0.0"
}
```

# IANA Considerations

This document requests registration of the "NTDF" authentication scheme in the HTTP Authentication Scheme Registry.

**Scheme Name**: NTDF

**Reference**: This document

# References

## Normative References

- **RFC 2119**: Key words for use in RFCs to Indicate Requirement Levels
- **RFC 9449**: OAuth 2.0 Demonstrating Proof-of-Possession at the Application Layer (DPoP)
- **OpenTDF Specification**: https://github.com/opentdf/spec
- **Z85 (ZeroMQ RFC 32)**: https://rfc.zeromq.org/spec/32/

## Informative References

- **Ed25519**: https://ed25519.cr.yp.to/
- **NIST P-256**: FIPS 186-4, Digital Signature Standard
- **AES-GCM**: NIST SP 800-38D, Galois/Counter Mode
- **HKDF**: RFC 5869, HMAC-based Key Derivation Function

# Acknowledgments

This specification is based on the implementation in arkavo-rs (https://github.com/arkavo-org/arkavo-rs).

Contributors:
- arkavo-rs implementation team
- OpenTDF community for NanoTDF specification

# Implementation Reference

Reference implementation: https://github.com/arkavo-org/arkavo-rs

Key modules:
- `src/modules/terminal_link_ntdf.rs` - NTDF token validation
- `src/modules/dpop.rs` - DPoP validation
- `src/modules/crypto.rs` - Cryptographic primitives
- `src/modules/http_rewrap.rs` - HTTP rewrap endpoint
- `src/bin/main.rs` - WebSocket server integration

# Appendix A: Comparison with JWT Bearer Tokens

| Feature | JWT Bearer | NTDF Token |
|---------|-----------|------------|
| **Proof-of-Possession** | ❌ No | ✅ Yes (via DPoP) |
| **Policy Binding** | ❌ No | ✅ Yes (cryptographic GMAC) |
| **Payload Confidentiality** | ❌ No (base64) | ✅ Yes (AES-256-GCM) |
| **Provenance** | Signature validates issuer | Signature + policy binding |
| **Token Theft** | Easy to steal and replay | Prevented by DPoP binding |
| **Size** | ~200-400 chars | ~600-730 chars |
| **Parsing** | JSON (flexible) | Binary (compact) |
| **Server-side Revocation** | Requires token blacklist | Session ID lookup |
| **Policy Enforcement** | Application-level | Cryptographic (KAS) |

JWT Bearer tokens are suitable for scenarios where:
- Proof-of-possession is not required
- Payload confidentiality is not needed
- Policy binding is not necessary
- Minimal size is critical

NTDF tokens are suitable for scenarios requiring:
- High security (token theft prevention)
- Policy-driven access control
- Confidential claims
- Integration with TDF content encryption

# Appendix B: Security Analysis

## Threat Model

Assumptions:
- Network may be untrusted (eavesdropping, MITM possible)
- Client devices may be compromised
- Tokens may be exfiltrated from client storage
- KAS server is trusted and secured

Threats addressed:
1. **Token theft**: DPoP binding prevents stolen tokens from being used
2. **Eavesdropping**: Payload encryption protects claim confidentiality
3. **Token forgery**: Ed25519 signature + AES-GCM auth tag prevent forgery
4. **Policy substitution**: GMAC binding prevents policy changes
5. **Replay attacks**: DPoP timestamp + jti uniqueness prevent replay

Threats NOT addressed:
- Client device compromise (if attacker gets ephemeral private key + token)
- KAS private key compromise (all tokens become decryptable)
- Malicious KAS (can decrypt all payloads)

## Cryptographic Strength

- **P-256 ECDH**: ~128-bit security level
- **AES-256-GCM**: 256-bit security level
- **Ed25519**: ~128-bit security level
- **SHA-256**: 256-bit security level

Overall security level: ~128 bits (limited by P-256 and Ed25519)

## Forward Secrecy

NTDF tokens provide forward secrecy for individual tokens:
- Each token has unique ephemeral key pair
- Compromise of one token's ephemeral key doesn't affect other tokens
- Compromise of KAS private key allows decryption of all tokens (no forward secrecy against KAS compromise)

For forward secrecy against KAS compromise, regular key rotation is recommended.

---

**Document History:**

- **draft-arkavo-ntdf-token-00** (2025-10-31): Initial draft based on arkavo-rs implementation

