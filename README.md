# Arkavo Specifications

Technical specifications for the Arkavo ecosystem.

## Specifications

### ntdf-token/

**NTDF Tokens: NanoTDF-based Authentication Tokens**

NTDF tokens are cryptographically-bound authentication tokens that replace traditional JWT Bearer tokens. Built on OpenTDF's NanoTDF specification, they provide:

- **Proof-of-Possession**: DPoP (RFC 9449) integration prevents token theft
- **Policy Binding**: Cryptographic GMAC binding enforces access control at the KAS
- **Confidentiality**: AES-256-GCM encrypted payload protects claim data
- **Provenance**: Ed25519 signatures ensure token integrity and origin

Wire format: `Authorization: NTDF <Z85-encoded-nanotdf>`

**Latest Draft**: [draft-arkavo-ntdf-token-00](ntdf-token/draft-arkavo-ntdf-token-00.md)

**Reference Implementation**: [arkavo-rs](https://github.com/arkavo-org/arkavo-rs)