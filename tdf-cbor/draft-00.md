# TDF-CBOR Specification

**Version:** draft-00
**Status:** Draft
**Date:** January 2026

## Abstract

TDF-CBOR defines a CBOR-based container format for the Trusted Data Format (TDF) that provides efficient binary encoding with inline encrypted payloads. This format eliminates base64 overhead while maintaining semantic equivalence with TDF-JSON, making it ideal for bandwidth-constrained environments, edge computing, IoT applications, and high-performance data pipelines.

## 1. Introduction

### 1.1 Background

While TDF-JSON provides a convenient format for JSON-native environments, it incurs ~33% size overhead due to base64 encoding of binary payload data. For bandwidth-constrained environments or high-throughput applications, this overhead can be significant.

CBOR (Concise Binary Object Representation, RFC 8949) is a binary data format designed for:
- Small message size
- Extensibility without version negotiation
- Reasonable processing requirements
- Schema-free encoding (self-describing)

### 1.2 Goals

TDF-CBOR addresses these requirements by:

1. Eliminating base64 encoding overhead (~33% size reduction for payloads)
2. Providing efficient binary representation of all TDF structures
3. Maintaining full semantic equivalence with TDF-JSON
4. Supporting COSE (CBOR Object Signing and Encryption) integration
5. Enabling efficient parsing on constrained devices
6. Supporting deterministic encoding for signature verification

### 1.3 Relationship to Other Formats

| Format | Use Case | Overhead | Parsing |
|--------|----------|----------|---------|
| OpenTDF (ZIP) | File storage | ZIP + base64 in JSON | High |
| TDF-JSON | JSON-native APIs | ~33% (base64) | Medium |
| **TDF-CBOR** | Binary protocols, IoT | Minimal | Low |
| NanoTDF | Extreme constraints | Minimal | Low |

TDF-CBOR sits between TDF-JSON (full-featured, text-based) and NanoTDF (minimal, binary). It provides the full TDF feature set with efficient binary encoding.

### 1.4 CBOR Primer

CBOR uses a type-length-value encoding with the following major types:

| Type | Value | Description |
|------|-------|-------------|
| 0 | 0x00-0x1b | Unsigned integer |
| 1 | 0x20-0x3b | Negative integer |
| 2 | 0x40-0x5b | Byte string |
| 3 | 0x60-0x7b | Text string |
| 4 | 0x80-0x9b | Array |
| 5 | 0xa0-0xbb | Map |
| 6 | 0xc0-0xdb | Tag |
| 7 | 0xe0-0xfb | Simple values (bool, null, float) |

## 2. Specification

### 2.1 Document Structure

A TDF-CBOR document is a CBOR map with integer keys for compactness:

```
TDF-CBOR = {
  1: "cbor",           ; tdf format identifier
  2: [1, 0, 0],        ; version array [major, minor, patch]
  3: uint,             ; created (Unix timestamp, optional)
  4: manifest,         ; manifest map
  5: payload           ; payload map
}
```

**CDDL (Concise Data Definition Language):**

```cddl
tdf-cbor = {
  1 => "cbor",                    ; format
  2 => version,                   ; version
  ? 3 => uint,                    ; created timestamp
  4 => manifest,                  ; manifest
  5 => payload                    ; payload
}

version = [uint, uint, uint]      ; [major, minor, patch]
```

### 2.2 Key Mapping

TDF-CBOR uses integer keys instead of string keys for efficiency:

| Integer Key | JSON Equivalent | Description |
|-------------|-----------------|-------------|
| 1 | `tdf` | Format identifier |
| 2 | `version` | Specification version |
| 3 | `created` | Creation timestamp |
| 4 | `manifest` | Manifest object |
| 5 | `payload` | Payload object |

### 2.3 Manifest Structure

```cddl
manifest = {
  1 => encryption-information,    ; encryptionInformation
  ? 2 => [* assertion]            ; assertions
}

encryption-information = {
  1 => "split",                   ; type
  2 => [+ key-access],            ; keyAccess
  3 => method,                    ; method
  4 => integrity-information,     ; integrityInformation
  5 => bstr                       ; policy (raw bytes, not base64)
}
```

#### 2.3.1 Key Access Structure

```cddl
key-access = {
  1 => key-access-type,           ; type
  2 => tstr,                      ; url
  3 => tstr,                      ; protocol
  4 => bstr,                      ; wrappedKey (raw bytes)
  5 => policy-binding,            ; policyBinding
  ? 6 => bstr,                    ; encryptedMetadata (raw bytes)
  ? 7 => tstr,                    ; kid
  ? 8 => bstr,                    ; ephemeralPublicKey (compressed SEC1, raw bytes)
  ? 9 => tstr                     ; schemaVersion
}

key-access-type = "wrapped" / "remote"

policy-binding = {
  1 => tstr,                      ; alg
  2 => bstr                       ; hash (raw bytes)
}
```

| Key | JSON Name | Type | Description |
|-----|-----------|------|-------------|
| 1 | type | text | `"wrapped"` or `"remote"` |
| 2 | url | text | KAS endpoint URL |
| 3 | protocol | text | Protocol identifier |
| 4 | wrappedKey | bstr | Wrapped symmetric key (raw bytes) |
| 5 | policyBinding | map | Cryptographic policy binding |
| 6 | encryptedMetadata | bstr | Encrypted metadata (optional) |
| 7 | kid | text | Key identifier (optional) |
| 8 | ephemeralPublicKey | bstr | Compressed SEC1 point. REQUIRED for EC key wrapping |
| 9 | schemaVersion | text | Schema version (optional) |

#### 2.3.2 Method Structure

```cddl
method = {
  1 => algorithm,                 ; algorithm
  2 => bstr,                      ; iv (raw bytes, 12 bytes for GCM)
  ? 3 => bool                     ; isStreamable
}

algorithm = "AES-256-GCM"
```

#### 2.3.3 Integrity Information Structure

```cddl
integrity-information = {
  1 => root-signature,            ; rootSignature
  2 => segment-hash-alg,          ; segmentHashAlg
  ? 3 => [* segment],             ; segments
  ? 4 => uint,                    ; segmentSizeDefault
  ? 5 => uint                     ; encryptedSegmentSizeDefault
}

root-signature = {
  1 => tstr,                      ; alg
  2 => bstr                       ; sig (raw bytes)
}

segment-hash-alg = "GMAC" / "SHA256"

segment = {
  1 => bstr,                      ; hash
  ? 2 => uint,                    ; segmentSize (if different from default)
  ? 3 => uint                     ; encryptedSegmentSize
}
```

### 2.4 Payload Structure

The key advantage of TDF-CBOR is direct binary payload embedding:

```cddl
payload = {
  1 => "inline",                  ; type
  2 => "binary",                  ; protocol (not base64!)
  ? 3 => tstr,                    ; mimeType
  4 => true,                      ; isEncrypted
  ? 5 => uint,                    ; length
  6 => bstr                       ; value (raw ciphertext bytes)
}
```

**Key Difference from TDF-JSON:** The `protocol` is `"binary"` and `value` is a CBOR byte string (`bstr`) containing raw ciphertext, not base64-encoded text.

#### 2.4.1 Chunked Payload

For streaming or large payloads:

```cddl
payload-chunked = {
  1 => "inline",                  ; type
  2 => "binary-chunked",          ; protocol
  ? 3 => tstr,                    ; mimeType
  4 => true,                      ; isEncrypted
  5 => [+ chunk]                  ; chunks
}

chunk = {
  1 => uint,                      ; index
  2 => bstr,                      ; value (raw bytes)
  ? 3 => bstr                     ; hash (GMAC tag)
}
```

### 2.5 Assertions Structure

```cddl
assertion = {
  1 => tstr,                      ; id
  2 => tstr,                      ; type
  3 => tstr,                      ; scope
  ? 4 => tstr,                    ; appliesToState
  5 => any,                       ; statement
  6 => assertion-binding          ; binding
}

assertion-binding = {
  1 => tstr,                      ; method
  2 => bstr                       ; signature (raw bytes)
}
```

## 3. Binary Encoding

### 3.1 Complete Key Map

| Context | Key | JSON Name | Type |
|---------|-----|-----------|------|
| Root | 1 | tdf | text |
| Root | 2 | version | array |
| Root | 3 | created | uint |
| Root | 4 | manifest | map |
| Root | 5 | payload | map |
| Manifest | 1 | encryptionInformation | map |
| Manifest | 2 | assertions | array |
| EncInfo | 1 | type | text |
| EncInfo | 2 | keyAccess | array |
| EncInfo | 3 | method | map |
| EncInfo | 4 | integrityInformation | map |
| EncInfo | 5 | policy | bstr |
| KeyAccess | 1 | type | text |
| KeyAccess | 2 | url | text |
| KeyAccess | 3 | protocol | text |
| KeyAccess | 4 | wrappedKey | bstr |
| KeyAccess | 5 | policyBinding | map |
| KeyAccess | 6 | encryptedMetadata | bstr |
| KeyAccess | 7 | kid | text |
| KeyAccess | 8 | ephemeralPublicKey | bstr |
| KeyAccess | 9 | schemaVersion | text |
| PolicyBinding | 1 | alg | text |
| PolicyBinding | 2 | hash | bstr |
| Method | 1 | algorithm | text |
| Method | 2 | iv | bstr |
| Method | 3 | isStreamable | bool |
| Integrity | 1 | rootSignature | map |
| Integrity | 2 | segmentHashAlg | text |
| Integrity | 3 | segments | array |
| Integrity | 4 | segmentSizeDefault | uint |
| Integrity | 5 | encryptedSegmentSizeDefault | uint |
| RootSig | 1 | alg | text |
| RootSig | 2 | sig | bstr |
| Payload | 1 | type | text |
| Payload | 2 | protocol | text |
| Payload | 3 | mimeType | text |
| Payload | 4 | isEncrypted | bool |
| Payload | 5 | length | uint |
| Payload | 6 | value | bstr |
| Chunk | 1 | index | uint |
| Chunk | 2 | value | bstr |
| Chunk | 3 | hash | bstr |

### 3.2 CBOR Tags

TDF-CBOR uses the following CBOR tags:

| Tag | Meaning | Usage |
|-----|---------|-------|
| 55799 | Self-describe CBOR | Optional wrapper for file detection |
| 1 | Epoch timestamp | Created timestamp |
| 21 | Expected base64url | URL strings (optional hint) |

### 3.3 Deterministic Encoding

For signature verification, TDF-CBOR SHOULD use deterministic encoding (RFC 8949 Section 4.2):

1. Map keys in ascending integer order
2. Shortest form for integers
3. No indefinite-length encoding
4. No duplicate keys

## 4. COSE Integration (Optional)

TDF-CBOR may optionally use COSE (RFC 8152) for key wrapping and signatures:

### 4.1 COSE Key Wrapping

Instead of custom key wrapping, use COSE_Encrypt0:

```cddl
cose-wrapped-key = #6.16(COSE_Encrypt0)

COSE_Encrypt0 = [
  protected: bstr .cbor protected-header,
  unprotected: unprotected-header,
  ciphertext: bstr
]

protected-header = {
  1 => 3,                         ; alg: A256KW
  4 => bstr                       ; kid
}
```

### 4.2 COSE Signatures

For assertion binding, use COSE_Sign1:

```cddl
cose-signature = #6.18(COSE_Sign1)

COSE_Sign1 = [
  protected: bstr .cbor {
    1 => -7                       ; alg: ES256
  },
  unprotected: {},
  payload: bstr,
  signature: bstr
]
```

## 4.3 EC Key Wrapping (ECIES)

When using elliptic curve key wrapping, implementations MUST use compressed SEC1 point encoding for the ephemeral public key. Unlike TDF-JSON where the ephemeral key is base64-encoded, TDF-CBOR stores it as raw bytes.

**Supported Curves:**

| Curve | Compressed Size | CBOR Overhead |
|-------|-----------------|---------------|
| P-256 (secp256r1) | 33 bytes | 2 bytes (bstr header) |
| P-384 (secp384r1) | 49 bytes | 2 bytes |
| P-521 (secp521r1) | 67 bytes | 2 bytes |

**Encryption (Key Wrap):**

1. Generate an ephemeral EC key pair on the same curve as the KAS public key
2. Compute shared secret using ECDH: `Z = ECDH(ephemeral_private, kas_public)`
3. Derive wrapping key using HKDF-SHA256:
   - Input: `Z` (shared secret)
   - Salt: empty
   - Info: `"OpenTDF"` (7 bytes)
   - Output: 32 bytes (256-bit AES key)
4. Generate random 12-byte nonce
5. Encrypt payload key using AES-256-GCM with derived key and nonce
6. Encode ephemeral public key as **compressed SEC1 point**
7. Store in keyAccess:
   - Key 4 (`wrappedKey`): `bstr(nonce[12] || ciphertext || tag[16])`
   - Key 8 (`ephemeralPublicKey`): `bstr(compressed SEC1 point)`

**Decryption (Key Unwrap):**

1. Extract `ephemeralPublicKey` (key 8) as raw bytes
2. Parse compressed SEC1 point to recover full public key
3. Compute shared secret: `Z = ECDH(kas_private, ephemeral_public)`
4. Derive wrapping key using HKDF-SHA256 (same parameters)
5. Extract `wrappedKey` (key 4) to get `nonce || ciphertext || tag`
6. Decrypt using AES-256-GCM to recover payload key

**Example keyAccess in CBOR diagnostic notation:**

```
{
  1: "wrapped",                              ; type
  2: "https://kas.example.com",              ; url
  3: "kas",                                  ; protocol
  4: h'91f667...5b',                         ; wrappedKey (60 bytes)
  5: {                                       ; policyBinding
    1: "HS256",
    2: h'5e346a326cdb18649b356cdf72d81987...'
  },
  8: h'036108758...3da',                     ; ephemeralPublicKey (33 bytes for P-256)
  9: "1.0"                                   ; schemaVersion
}
```

Note: The compressed P-256 point is stored directly as 33 bytes, saving 11 bytes compared to base64 encoding in TDF-JSON.

## 5. Size Comparison

### 5.1 Overhead Analysis

For a 1KB plaintext payload:

| Format | Manifest | Payload | Total | Overhead |
|--------|----------|---------|-------|----------|
| TDF-JSON | ~800B | 1,365B | ~2,165B | +116% |
| **TDF-CBOR** | ~500B | 1,024B | ~1,524B | +52% |
| Savings | 38% | 25% | 30% | |

### 5.2 Example Size Breakdown

**TDF-JSON payload (1024 bytes plaintext):**
- Base64 encoding: 1,368 characters
- JSON overhead: ~20 bytes (`"value":"..."`)
- Total: ~1,388 bytes

**TDF-CBOR payload (1024 bytes plaintext):**
- CBOR byte string header: 3 bytes (for 1024 bytes)
- Raw ciphertext: 1,024 bytes
- Map overhead: ~10 bytes
- Total: ~1,037 bytes

**Savings: ~25% for payload alone**

## 6. Wire Format Example

### 6.1 Diagnostic Notation

```
{
  1: "cbor",
  2: [1, 0, 0],
  3: 1737129600,
  4: {
    1: {
      1: "split",
      2: [{
        1: "wrapped",
        2: "https://kas.example.com/api/kas",
        3: "kas",
        4: h'33b186a63a53b1235c966996f4e6bde0df50cbff9023d893597754826153d4d58294866b28497f02ed779d504383e3fd2b29a9cd5660a4742ff090354fc19c924166199570f3f7c55bd380f74689734766450e5ecec20aef77a0ce594b247b1a312b5a3c5da83f7c1a0b45e2b78b2b2f11c3dbc13c2f86f54426e4ba0a9b2b890de15bf3d7986cac7ab6bf624472604c4f7f79b72d8c0277ef4099c4b2cc36d65d1ca430bb08353b711d0a08512f64a5b8501cbfcc2ef7259664ec6a1109b91705aa325fab27ce0db6797cbda46facf91f97ed212451dbaa7f1618ec44241283852bb6aba2a98672c0dac5441a6588b6373048508e3ed91024be97e1c075c4',
        5: {
          1: "HS256",
          2: h'3034...1cf6'
        }
      }],
      3: {
        1: "AES-256-GCM",
        2: h'0fd17c1c7cb4b6ecb400e3bc',
        3: true
      },
      4: {
        1: {
          1: "HS256",
          2: h'66636339...37616638'
        },
        2: "GMAC",
        3: [],
        4: 1048576,
        5: 1048604
      },
      5: h'7b2275756964223a226235386666346434...'
    }
  },
  5: {
    1: "inline",
    2: "binary",
    3: "text/plain",
    4: true,
    5: 53,
    6: h'9c5d3331770c9a45388664ae80bdbf290cfc054528b0c6343c3122a13a74967646333e7a94d9b01'
  }
}
```

### 6.2 Hex Encoding

```
A5                                      # map(5)
   01                                   # key: 1
   64 63626F72                          # text: "cbor"
   02                                   # key: 2
   83 01 00 00                          # array: [1, 0, 0]
   03                                   # key: 3
   1A 67899F80                          # uint: 1737129600
   04                                   # key: 4
   ... (manifest bytes)
   05                                   # key: 5
   A6                                   # map(6)
      01                                # key: 1
      66 696E6C696E65                   # text: "inline"
      02                                # key: 2
      66 62696E617279                   # text: "binary"
      03                                # key: 3
      6A 746578742F706C61696E           # text: "text/plain"
      04                                # key: 4
      F5                                # true
      05                                # key: 5
      18 35                             # uint: 53
      06                                # key: 6
      58 35                             # bytes(53)
         9C5D3331770C9A453886...        # ciphertext
```

## 7. MIME Type and File Extension

| Property | Value |
|----------|-------|
| MIME Type | `application/tdf+cbor` |
| File Extension | `.tdf.cbor` or `.tdfcbor` |
| Magic Bytes | `D9 D9F7 A5 01` (self-described CBOR + map + key 1) |

## 8. Security Considerations

### 8.1 Inherited Security Properties

TDF-CBOR inherits all security properties from OpenTDF:

- **Confidentiality**: AES-256-GCM authenticated encryption
- **Integrity**: GMAC tags and HMAC signatures
- **Access Control**: ABAC via embedded policy
- **Key Protection**: KAS-managed key wrapping
- **Policy Binding**: Cryptographic binding prevents tampering

### 8.2 CBOR-Specific Considerations

1. **Parsing Safety**: Use CBOR decoders with strict mode to prevent:
   - Duplicate map keys
   - Non-canonical encodings
   - Excessive nesting depth

2. **Memory Limits**: Implement maximum size limits for:
   - Total document size
   - Individual byte strings
   - Array/map element counts

3. **Integer Overflow**: Validate integer values before use

4. **Deterministic Encoding**: For signature verification, ensure canonical encoding

### 8.3 Comparison with TDF-JSON Security

| Property | TDF-JSON | TDF-CBOR |
|----------|----------|----------|
| Encryption | Identical | Identical |
| Key Management | Identical | Identical + COSE option |
| Policy Binding | Identical | Identical |
| Payload Integrity | Base64 adds no security | Raw bytes |
| Signature Support | JWS | COSE_Sign1 |

## 9. Interoperability

### 9.1 Conversion from TDF-JSON

```
1. Parse TDF-JSON document
2. For each base64-encoded field:
   - Decode base64 to bytes
   - Store as CBOR bstr
3. Convert string keys to integer keys
4. Encode as CBOR
```

### 9.2 Conversion to TDF-JSON

```
1. Parse TDF-CBOR document
2. For each bstr field:
   - Encode bytes as base64
   - Store as JSON string
3. Convert integer keys to string keys
4. Serialize as JSON
```

### 9.3 Conversion to/from OpenTDF (ZIP)

Same as TDF-JSON, with CBOR<->JSON conversion step.

### 9.4 Protocol Integration

**gRPC:**
```protobuf
message SecurePayload {
  bytes tdf_cbor = 1;
}
```

**MQTT:**
```
Topic: secure/data/{device_id}
Payload: <TDF-CBOR bytes>
```

**CoAP:**
```
POST /secure-data
Content-Format: application/tdf+cbor
Payload: <TDF-CBOR bytes>
```

## 10. Implementation Guidelines

### 10.1 Recommended Libraries

| Language | CBOR Library | Notes |
|----------|--------------|-------|
| Rust | `ciborium`, `minicbor` | Use deterministic mode |
| Python | `cbor2` | Enable canonical encoding |
| JavaScript | `cbor-x`, `cborg` | Browser compatible |
| Go | `fxamacker/cbor` | Struct tag support |
| C | `tinycbor`, `cn-cbor` | Embedded friendly |

### 10.2 Validation Checklist

- [ ] Format identifier is `"cbor"`
- [ ] Version is valid semver array
- [ ] All required keys present
- [ ] All byte strings are valid lengths
- [ ] Policy binding verifies
- [ ] Ciphertext length matches header

## 11. References

- [RFC 8949 - CBOR](https://datatracker.ietf.org/doc/html/rfc8949)
- [RFC 8152 - COSE](https://datatracker.ietf.org/doc/html/rfc8152)
- [RFC 8610 - CDDL](https://datatracker.ietf.org/doc/html/rfc8610)
- [OpenTDF Specification](https://opentdf.io/spec)
- [TDF-JSON Specification](../tdf-json/draft-00.md)

## Appendix A: Complete CDDL Schema

```cddl
; TDF-CBOR Root Structure
tdf-cbor = {
  1 => "cbor",                          ; format identifier
  2 => version,                         ; version
  ? 3 => #6.1(uint),                    ; created (tagged epoch)
  4 => manifest,                        ; manifest
  5 => payload / payload-chunked        ; payload
}

version = [major: uint, minor: uint, patch: uint]

; Manifest
manifest = {
  1 => encryption-information,
  ? 2 => [* assertion]
}

encryption-information = {
  1 => "split",                         ; type
  2 => [+ key-access],                  ; keyAccess
  3 => method,                          ; method
  4 => integrity-information,           ; integrityInformation
  5 => bstr                             ; policy
}

key-access = {
  1 => "wrapped" / "remote",            ; type
  2 => tstr,                            ; url
  3 => tstr,                            ; protocol
  4 => bstr,                            ; wrappedKey
  5 => policy-binding,                  ; policyBinding
  ? 6 => bstr,                          ; encryptedMetadata
  ? 7 => tstr,                          ; kid
  ? 8 => bstr,                          ; ephemeralPublicKey (compressed SEC1)
  ? 9 => tstr                           ; schemaVersion
}

policy-binding = {
  1 => tstr,                            ; alg
  2 => bstr                             ; hash
}

method = {
  1 => "AES-256-GCM",                   ; algorithm
  2 => bstr .size 12,                   ; iv (12 bytes for GCM)
  ? 3 => bool                           ; isStreamable
}

integrity-information = {
  1 => root-signature,                  ; rootSignature
  2 => "GMAC" / "SHA256",               ; segmentHashAlg
  ? 3 => [* segment],                   ; segments
  ? 4 => uint,                          ; segmentSizeDefault
  ? 5 => uint                           ; encryptedSegmentSizeDefault
}

root-signature = {
  1 => tstr,                            ; alg
  2 => bstr                             ; sig
}

segment = {
  1 => bstr,                            ; hash
  ? 2 => uint,                          ; segmentSize
  ? 3 => uint                           ; encryptedSegmentSize
}

; Payload
payload = {
  1 => "inline",                        ; type
  2 => "binary",                        ; protocol
  ? 3 => tstr,                          ; mimeType
  4 => true,                            ; isEncrypted
  ? 5 => uint,                          ; length
  6 => bstr                             ; value (raw ciphertext)
}

payload-chunked = {
  1 => "inline",                        ; type
  2 => "binary-chunked",                ; protocol
  ? 3 => tstr,                          ; mimeType
  4 => true,                            ; isEncrypted
  5 => [+ chunk]                        ; chunks
}

chunk = {
  1 => uint,                            ; index
  2 => bstr,                            ; value
  ? 3 => bstr                           ; hash
}

; Assertions
assertion = {
  1 => tstr,                            ; id
  2 => tstr,                            ; type
  3 => tstr,                            ; scope
  ? 4 => tstr,                          ; appliesToState
  5 => any,                             ; statement
  6 => assertion-binding                ; binding
}

assertion-binding = {
  1 => tstr,                            ; method
  2 => bstr                             ; signature
}
```

## Appendix B: Comparison Table

| Feature | OpenTDF | TDF-JSON | TDF-CBOR | NanoTDF |
|---------|---------|----------|----------|---------|
| Container | ZIP | JSON | CBOR | Binary |
| Payload Encoding | Raw file | Base64 | Raw bytes | Raw bytes |
| Manifest Format | JSON | JSON | CBOR | Binary |
| Size Overhead | High | Medium | Low | Minimal |
| Streaming | Yes | Yes | Yes | Limited |
| Full Policy | Yes | Yes | Yes | Limited |
| Assertions | Yes | Yes | Yes | No |
| Multiple KAS | Yes | Yes | Yes | Limited |
| Human Readable | Partial | Yes | No | No |
| Parse Complexity | Medium | Low | Low | High |

## Appendix C: Changelog

### draft-00 (January 2026)
- Initial draft specification
- COSE integration defined as optional extension
- CDDL schema defined
- Added EC key wrapping (ECIES) with mandatory compressed SEC1 points
- Added key 8 (`ephemeralPublicKey`) for EC wrapping
- Added key 9 (`schemaVersion`) for versioning
