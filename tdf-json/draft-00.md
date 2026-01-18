# TDF-JSON Specification

**Version:** draft-00
**Status:** Draft
**Date:** January 2026

## Abstract

TDF-JSON defines a JSON-based container format for the Trusted Data Format (TDF) that embeds encrypted payloads inline rather than using a ZIP archive. This format is optimized for JSON-RPC protocols, REST APIs, Agent-to-Agent (A2A) communication, and Model Context Protocol (MCP) integrations where JSON is the native transport.

## 1. Introduction

### 1.1 Background

The standard OpenTDF format packages a `manifest.json` file and encrypted payload within a ZIP archive. While effective for file-based workflows, this approach introduces overhead in environments where JSON is the native data format:

- ZIP encoding/decoding overhead
- Base64 encoding of entire ZIP for JSON transport
- Complexity in streaming scenarios
- Impedance mismatch with JSON-RPC and REST APIs

### 1.2 Goals

TDF-JSON addresses these challenges by:

1. Eliminating ZIP container overhead
2. Providing a single JSON document containing both manifest and payload
3. Maintaining full compatibility with OpenTDF semantics and security properties
4. Enabling seamless embedding in JSON-RPC messages and API responses
5. Supporting streaming scenarios through chunked payload encoding

### 1.3 Relationship to OpenTDF

TDF-JSON is a **container format** that uses the same cryptographic primitives, key access mechanisms, policy structures, and KAS protocols as OpenTDF. The manifest schema is extended to support inline payloads while remaining backwards-compatible with existing tooling.

## 2. Specification

### 2.1 Document Structure

A TDF-JSON document is a single JSON object with the following top-level structure:

```json
{
  "tdf": "json",
  "version": "1.0.0",
  "created": "2026-01-17T12:00:00Z",
  "manifest": { ... },
  "payload": { ... }
}
```

### 2.2 Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tdf` | string | Yes | MUST be `"json"` to identify this as a TDF-JSON document |
| `version` | string | Yes | Semantic version of this specification (e.g., `"1.0.0"`) |
| `created` | string | No | ISO 8601 timestamp of document creation |
| `manifest` | object | Yes | The TDF manifest containing encryption and policy information |
| `payload` | object | Yes | The inline encrypted payload container |

### 2.3 Manifest Object

The manifest object follows the OpenTDF manifest schema with the following structure:

```json
{
  "manifest": {
    "encryptionInformation": {
      "type": "split",
      "keyAccess": [ ... ],
      "method": { ... },
      "integrityInformation": { ... },
      "policy": "base64_encoded_policy"
    },
    "assertions": [ ... ]
  }
}
```

#### 2.3.1 encryptionInformation

The `encryptionInformation` object is identical to OpenTDF specification:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Key split type. MUST be `"split"` |
| `keyAccess` | array | Yes | Array of Key Access Objects |
| `method` | object | Yes | Encryption method details |
| `integrityInformation` | object | Yes | Payload integrity information |
| `policy` | string | Yes | Base64-encoded policy object |

#### 2.3.2 keyAccess Array

Each Key Access Object follows the OpenTDF schema:

```json
{
  "type": "wrapped",
  "url": "https://kas.example.com/api/kas",
  "protocol": "kas",
  "wrappedKey": "base64_wrapped_key",
  "policyBinding": {
    "alg": "HS256",
    "hash": "base64_policy_binding_hash"
  },
  "encryptedMetadata": "base64_encrypted_metadata",
  "kid": "key_identifier"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | MUST be `"wrapped"` or `"remote"` |
| `url` | string | Yes | KAS endpoint URL |
| `protocol` | string | Yes | Protocol identifier (typically `"kas"`) |
| `wrappedKey` | string | Yes | Base64-encoded wrapped symmetric key |
| `policyBinding` | object | Yes | Cryptographic binding of key to policy |
| `encryptedMetadata` | string | No | Base64-encoded encrypted metadata |
| `kid` | string | No | Key identifier for key lookup |

#### 2.3.3 method Object

```json
{
  "algorithm": "AES-256-GCM",
  "iv": "base64_initialization_vector",
  "isStreamable": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `algorithm` | string | Yes | Encryption algorithm. MUST be `"AES-256-GCM"` |
| `iv` | string | Yes | Base64-encoded initialization vector |
| `isStreamable` | boolean | No | Whether payload supports streaming decryption |

#### 2.3.4 integrityInformation Object

```json
{
  "rootSignature": {
    "alg": "HS256",
    "sig": "base64_root_signature"
  },
  "segmentHashAlg": "GMAC",
  "segments": [ ... ],
  "segmentSizeDefault": 1048576,
  "encryptedSegmentSizeDefault": 1048604
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rootSignature` | object | Yes | Root signature over all segments |
| `segmentHashAlg` | string | Yes | Hash algorithm for segments (`"GMAC"` or `"SHA256"`) |
| `segments` | array | No | Array of segment hash objects (for large payloads) |
| `segmentSizeDefault` | integer | No | Default segment size in bytes |
| `encryptedSegmentSizeDefault` | integer | No | Default encrypted segment size |

#### 2.3.5 assertions Array (Optional)

Assertions provide cryptographically verifiable statements about the TDF:

```json
{
  "id": "assertion_uuid",
  "type": "handling",
  "scope": "tdo",
  "appliesToState": "encrypted",
  "statement": { ... },
  "binding": {
    "method": "jws",
    "signature": "base64_signature"
  }
}
```

### 2.4 Payload Object

The payload object contains the encrypted data inline:

```json
{
  "payload": {
    "type": "inline",
    "protocol": "base64",
    "mimeType": "application/octet-stream",
    "isEncrypted": true,
    "length": 1234,
    "value": "base64_encoded_ciphertext"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | MUST be `"inline"` for TDF-JSON |
| `protocol` | string | Yes | Encoding protocol. MUST be `"base64"` |
| `mimeType` | string | No | MIME type of the plaintext content |
| `isEncrypted` | boolean | Yes | MUST be `true` |
| `length` | integer | No | Length of ciphertext in bytes (before base64) |
| `value` | string | Yes | Base64-encoded ciphertext |

#### 2.4.1 Chunked Payload (Optional)

For large payloads or streaming scenarios, the payload may use chunked encoding:

```json
{
  "payload": {
    "type": "inline",
    "protocol": "base64-chunked",
    "mimeType": "application/octet-stream",
    "isEncrypted": true,
    "chunks": [
      {
        "index": 0,
        "value": "base64_chunk_0",
        "hash": "base64_gmac_tag"
      },
      {
        "index": 1,
        "value": "base64_chunk_1",
        "hash": "base64_gmac_tag"
      }
    ]
  }
}
```

## 3. Cryptographic Operations

### 3.1 Encryption

TDF-JSON uses the same encryption process as OpenTDF:

1. Generate a random 256-bit symmetric key (payload key)
2. Generate a random 96-bit (12-byte) IV
3. Encrypt plaintext using AES-256-GCM with the payload key and IV
4. The ciphertext includes the GCM authentication tag
5. Wrap the payload key using the KAS public key (RSA-OAEP or ECDH+AES-KW)
6. Compute policy binding: `HMAC-SHA256(wrappedKey, SHA256(policy))`

### 3.2 Decryption

1. Parse the TDF-JSON document
2. Request key unwrap from KAS (providing wrapped key and policy binding)
3. KAS validates policy and returns unwrapped payload key
4. Decrypt ciphertext using AES-256-GCM with payload key and IV
5. Verify integrity using rootSignature and segment hashes

### 3.3 Key Wrapping Algorithms

| Algorithm | Key Type | Description |
|-----------|----------|-------------|
| `RSA-OAEP` | RSA-2048+ | RSA with OAEP padding (SHA-256) |
| `ec:secp256r1` | P-256 | ECDH key agreement + AES-256-KW |
| `ec:secp384r1` | P-384 | ECDH key agreement + AES-256-KW |
| `ec:secp521r1` | P-521 | ECDH key agreement + AES-256-KW |

## 4. MIME Type and File Extension

| Property | Value |
|----------|-------|
| MIME Type | `application/tdf+json` |
| File Extension | `.tdf.json` or `.tdfjson` |

## 5. Complete Example

```json
{
  "tdf": "json",
  "version": "1.0.0",
  "created": "2026-01-17T12:00:00Z",
  "manifest": {
    "encryptionInformation": {
      "type": "split",
      "keyAccess": [
        {
          "type": "wrapped",
          "url": "https://kas.example.com/api/kas",
          "protocol": "kas",
          "wrappedKey": "M7GGpjpTsSNclmmW9Oa94N9Qy/+QI9iTWXdUgmFT1NWClIZrKEl/Au13nVBDg+P9KympzVZgpHQv8JA1T8GckkFmGbVwc/fFW9OA90aJc0dsUo5ezewgrvd6DOWUskexoxK1o8Xag/fBoLReK3iysvEcPbwTwvhvVEpuS6CpsriQ3hW/PXmGysera/YkRyYMT395ty2MAnfvQJnEssw21l0cpDC7CDU7cR0KCFEvZKW4UBy/zC73JZZk7GoRCbkXBaoyX6snzg22eXy9pG+s+R+X7SEkUduqfxYY7EQkEoOFK7aroqmGcsDaxUQaZYiLY3cwSFCOPtkQJL6X4cB1xA==",
          "policyBinding": {
            "alg": "HS256",
            "hash": "MzA0NGFjM2ViYzliYWU1ODg4Yjg2MDM1ZWZiZTIzYjJlN2U4Y2YyZTdjOWEwMjZlZDg1YTQwYjg0YTk4NzFjZg=="
          },
          "encryptedMetadata": "eyJjaXBoZXJ0ZXh0IjoiRDlGM0VIeTB4dXkwQU80OFBZTXc3UkI5Y3FTeC9ZVHdwcTZtR1M1cyIsIml2IjoiRDlGM0VIeTB4dXkwQU80OCJ9"
        }
      ],
      "method": {
        "algorithm": "AES-256-GCM",
        "iv": "D9F3EHy0xuy0AO48",
        "isStreamable": true
      },
      "integrityInformation": {
        "rootSignature": {
          "alg": "HS256",
          "sig": "ZmNjOTc4Y2VkZWM3Yjg4NWQ4MTkzMzg0NTU2N2M3YWUxMGUyOWUxYWFjNWFiNDY3NzNiZmNmNjJhOTI3N2FmOA=="
        },
        "segmentHashAlg": "GMAC",
        "segments": [],
        "segmentSizeDefault": 1048576,
        "encryptedSegmentSizeDefault": 1048604
      },
      "policy": "eyJ1dWlkIjoiYjU4ZmY0ZDQtNjc2OC00ODRlLWJjMzEtMjEwYjU0YmYxY2I4IiwiYm9keSI6eyJkYXRhQXR0cmlidXRlcyI6W10sImRpc3NlbSI6W119fQ=="
    }
  },
  "payload": {
    "type": "inline",
    "protocol": "base64",
    "mimeType": "text/plain",
    "isEncrypted": true,
    "value": "nF0zMXcMmkU4hmSugL2/KQz8BUUosMY0PDEioTp0lnZGM+epTZsB"
  }
}
```

## 6. Security Considerations

### 6.1 Inherited Security Properties

TDF-JSON inherits all security properties from OpenTDF:

- **Confidentiality**: AES-256-GCM encryption with authenticated encryption
- **Integrity**: GMAC tags and root signatures
- **Access Control**: Attribute-based access control (ABAC) via policy
- **Key Protection**: Keys wrapped and managed by KAS
- **Policy Binding**: Cryptographic binding prevents policy tampering

### 6.2 JSON-Specific Considerations

1. **Size Overhead**: Base64 encoding adds ~33% size overhead compared to binary payload
2. **Memory**: Large payloads require full document parsing; use chunked encoding for streaming
3. **Character Encoding**: Documents MUST use UTF-8 encoding
4. **Whitespace**: Whitespace in JSON is not significant for parsing but affects size

### 6.3 Transport Security

- TDF-JSON documents SHOULD be transmitted over TLS 1.3+
- Documents MAY be signed at the transport layer for additional authenticity
- Cache-Control headers SHOULD prevent caching of encrypted content

## 7. Interoperability

### 7.1 Conversion to/from OpenTDF

TDF-JSON documents can be converted to standard OpenTDF (ZIP) format:

1. Extract `manifest` object, serialize as `manifest.json`
2. Decode payload `value` from base64, write as `0.payload`
3. Update manifest `payload.type` to `"reference"` and `payload.url` to `"0.payload"`
4. Package into ZIP archive

### 7.2 JSON-RPC Integration

TDF-JSON is designed for embedding in JSON-RPC messages:

```json
{
  "jsonrpc": "2.0",
  "method": "sendSecureMessage",
  "params": {
    "recipient": "user@example.com",
    "content": {
      "tdf": "json",
      "version": "1.0.0",
      "manifest": { ... },
      "payload": { ... }
    }
  },
  "id": 1
}
```

### 7.3 MCP Integration

For Model Context Protocol, TDF-JSON enables secure context sharing:

```json
{
  "context": {
    "type": "encrypted",
    "format": "tdf-json",
    "data": {
      "tdf": "json",
      "version": "1.0.0",
      "manifest": { ... },
      "payload": { ... }
    }
  }
}
```

## 8. References

- [OpenTDF Specification](https://opentdf.io/spec)
- [RFC 8259 - JSON](https://datatracker.ietf.org/doc/html/rfc8259)
- [RFC 5116 - AES-GCM](https://datatracker.ietf.org/doc/html/rfc5116)
- [RFC 4648 - Base64](https://datatracker.ietf.org/doc/html/rfc4648)

## Appendix A: JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://opentdf.io/schemas/tdf-json/1.0.0",
  "title": "TDF-JSON",
  "type": "object",
  "required": ["tdf", "version", "manifest", "payload"],
  "properties": {
    "tdf": {
      "type": "string",
      "const": "json"
    },
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$"
    },
    "created": {
      "type": "string",
      "format": "date-time"
    },
    "manifest": {
      "$ref": "#/$defs/manifest"
    },
    "payload": {
      "$ref": "#/$defs/payload"
    }
  },
  "$defs": {
    "manifest": {
      "type": "object",
      "required": ["encryptionInformation"],
      "properties": {
        "encryptionInformation": {
          "$ref": "#/$defs/encryptionInformation"
        },
        "assertions": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/assertion"
          }
        }
      }
    },
    "encryptionInformation": {
      "type": "object",
      "required": ["type", "keyAccess", "method", "integrityInformation", "policy"],
      "properties": {
        "type": {
          "type": "string",
          "const": "split"
        },
        "keyAccess": {
          "type": "array",
          "minItems": 1,
          "items": {
            "$ref": "#/$defs/keyAccess"
          }
        },
        "method": {
          "$ref": "#/$defs/method"
        },
        "integrityInformation": {
          "$ref": "#/$defs/integrityInformation"
        },
        "policy": {
          "type": "string"
        }
      }
    },
    "keyAccess": {
      "type": "object",
      "required": ["type", "url", "protocol", "wrappedKey", "policyBinding"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["wrapped", "remote"]
        },
        "url": {
          "type": "string",
          "format": "uri"
        },
        "protocol": {
          "type": "string"
        },
        "wrappedKey": {
          "type": "string"
        },
        "policyBinding": {
          "type": "object",
          "required": ["alg", "hash"],
          "properties": {
            "alg": {
              "type": "string"
            },
            "hash": {
              "type": "string"
            }
          }
        },
        "encryptedMetadata": {
          "type": "string"
        },
        "kid": {
          "type": "string"
        }
      }
    },
    "method": {
      "type": "object",
      "required": ["algorithm", "iv"],
      "properties": {
        "algorithm": {
          "type": "string",
          "const": "AES-256-GCM"
        },
        "iv": {
          "type": "string"
        },
        "isStreamable": {
          "type": "boolean"
        }
      }
    },
    "integrityInformation": {
      "type": "object",
      "required": ["rootSignature", "segmentHashAlg"],
      "properties": {
        "rootSignature": {
          "type": "object",
          "required": ["alg", "sig"],
          "properties": {
            "alg": {
              "type": "string"
            },
            "sig": {
              "type": "string"
            }
          }
        },
        "segmentHashAlg": {
          "type": "string",
          "enum": ["GMAC", "SHA256"]
        },
        "segments": {
          "type": "array"
        },
        "segmentSizeDefault": {
          "type": "integer"
        },
        "encryptedSegmentSizeDefault": {
          "type": "integer"
        }
      }
    },
    "payload": {
      "type": "object",
      "required": ["type", "protocol", "isEncrypted", "value"],
      "properties": {
        "type": {
          "type": "string",
          "const": "inline"
        },
        "protocol": {
          "type": "string",
          "enum": ["base64", "base64-chunked"]
        },
        "mimeType": {
          "type": "string"
        },
        "isEncrypted": {
          "type": "boolean",
          "const": true
        },
        "length": {
          "type": "integer"
        },
        "value": {
          "type": "string"
        },
        "chunks": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["index", "value"],
            "properties": {
              "index": {
                "type": "integer"
              },
              "value": {
                "type": "string"
              },
              "hash": {
                "type": "string"
              }
            }
          }
        }
      }
    },
    "assertion": {
      "type": "object",
      "required": ["id", "type", "scope", "statement", "binding"],
      "properties": {
        "id": {
          "type": "string"
        },
        "type": {
          "type": "string"
        },
        "scope": {
          "type": "string"
        },
        "appliesToState": {
          "type": "string"
        },
        "statement": {
          "type": "object"
        },
        "binding": {
          "type": "object"
        }
      }
    }
  }
}
```

## Appendix B: Changelog

### draft-00 (January 2026)
- Initial draft specification
