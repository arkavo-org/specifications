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

### torg-decision/

**TØR-G: Token-Only Reasoner (Graph) Intermediate Representation**

TØR-G is a zero-parser, boolean-circuit Intermediate Representation designed for AI agent policy synthesis. It treats LLM output as a direct stream of graph construction instructions, providing:

- **No Text**: The language consists solely of atomic tokens
- **No Parsing**: The "compiler" is a deterministic state machine
- **Strict Booleanism**: All logic reduced to boolean decision combinatorics
- **Verifiability**: Every generated program is a finite DAG amenable to SAT/BDD verification

Wire format: Direct token stream with logit masking enforcement

**Latest Draft**: [draft-arkavo-torg-decision-00](torg-decision/draft-arkavo-torg-decision-00.md)