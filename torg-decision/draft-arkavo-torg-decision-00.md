```text
---
title: "TØR-G: Token-Only Reasoner (Graph) Intermediate Representation"
abbrev: "TØR-G IR"
docname: draft-arkavo-torg-decision-00
category: exp
ipr: trust200902
area: Security
workgroup: Arkavo Specifications
keyword:
 - intermediate representation
 - boolean circuit
 - policy synthesis
 - AI agent
 - logit masking

author:
 - fullname: Arkavo Contributors
   organization: Arkavo
   email: info@arkavo.com

normative:
  RFC2119:

informative:
  SAT:
    title: "Boolean Satisfiability Problem"
    target: https://en.wikipedia.org/wiki/Boolean_satisfiability_problem
  BDD:
    title: "Binary Decision Diagram"
    target: https://en.wikipedia.org/wiki/Binary_decision_diagram

--- abstract

TØR-G (Token-Only Reasoner - Graph) is a zero-parser, boolean-circuit Intermediate Representation designed for AI agent policy synthesis. Unlike traditional code generation (which emits text requiring tokenization, parsing, and compilation), TØR-G treats Large Language Model (LLM) output as a direct stream of graph construction instructions. This approach eliminates syntax errors, enables tight hardware-level verification (SAT/BDD), and creates a "Machine-to-Machine" protocol that requires no human-in-the-loop.

--- middle
```

| Metadata | Value |
| :--- | :--- |
| **Status** | Draft |
| **Context** | Arkavo Edge Internal Logic Kernel |
| **Type** | Intermediate Representation (IR) |
| **Version** | 1.0.0 |

# Introduction

**TØR-G** (Token-Only Reasoner - Graph) is a zero-parser, boolean-circuit Intermediate Representation designed for AI agent policy synthesis.

Unlike traditional code generation (which emits text requiring tokenization, parsing, and compilation), TØR-G treats Large Language Model (LLM) output as a direct stream of graph construction instructions. This approach eliminates syntax errors, enables tight hardware-level verification (SAT/BDD), and creates a "Machine-to-Machine" protocol that requires no human-in-the-loop.

## Design Philosophy

1.  **No Text:** The language consists solely of atomic tokens.
2.  **No Parsing:** The "compiler" is a deterministic state machine.
3.  **Strict Booleanism:** All high-level logic (arithmetic, state) is computed upstream; TØR-G handles only boolean decision combinatorics.
4.  **Verifiability:** Every generated program is a finite Directed Acyclic Graph (DAG) amenable to formal verification.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Vocabulary & Token Set

TØR-G uses a closed, finite vocabulary. In implementation, these abstract symbols MUST be mapped to specific, invariant Token IDs in the underlying model's embedding matrix.

## Logic Operators (Atomic)

The kernel supports a minimal complete set of boolean operators:

| Symbol | Name | Semantics |
| :--- | :--- | :--- |
| `∨` | **OR** | Logical Disjunction ($A \lor B$) |
| `⊽` | **NOR** | Logical NOR ($\neg(A \lor B)$) |
| `⊻` | **XOR** | Logical Exclusive OR ($A \oplus B$) |

*Note: NAND, AND, NOT, and IMPLIES are derived macros composed of these primitives.*

## Structural Tokens

Used to delimit the graph structure.

| Symbol | Name | Semantics |
| :--- | :--- | :--- |
| `●` | **NODE_START** | Begins a node definition. |
| `○` | **NODE_END** | Ends a node definition. |
| `◎IN` | **INPUT_DECL** | Declares an external boolean input. |
| `◎OUT` | **OUTPUT_DECL** | Marks a node as a graph output. |

## Literals

| Symbol | Name | Semantics |
| :--- | :--- | :--- |
| `◎T` | **TRUE** | Constant True. |
| `◎F` | **FALSE** | Constant False. |

## Identifiers

The vocabulary includes a reserved range of Identifier Tokens (`ID_0` through `ID_N`).

*   **Input Anchoring:** Specific IDs (e.g., `ID_0`...`ID_100`) SHOULD be reserved for semantic inputs defined by the Arkavo Edge runtime (e.g., `ID_0` maps to `is_admin`).
*   **Internal Nodes:** Remaining IDs are used for intermediate wires.

# Syntax & Grammar

A TØR-G stream is valid if and only if it adheres to the following sequence rules. The system MUST enforce these via logit masking during generation and strict validation during loading.

## Stream Structure

The stream must be emitted in **Topological Order**:

1.  **Header:** Input Declarations.
2.  **Body:** Node Definitions (Inputs for a node must exist before the node is defined).
3.  **Footer:** Output Declarations.

```ebnf
STREAM    ::= INPUTS NODES OUTPUTS
INPUTS    ::= ( '◎IN' ID )*
NODES     ::= ( NODE_DEF )*
NODE_DEF  ::= '●' ID OP SOURCE SOURCE '○'
OUTPUTS   ::= ( '◎OUT' ID )*
OP        ::= '∨' | '⊽' | '⊻'
SOURCE    ::= ID | '◎T' | '◎F'
```

## Constraints

1.  **Single Definition:** An `ID` MUST NOT be defined as a Node or Input more than once.
2.  **No Forward References:** In a definition `● N_3 OP N_1 N_2 ○`, `N_1` and `N_2` MUST be defined (as Inputs, Literals, or previous Nodes) prior to this token.
3.  **Atomic Chunks:** A Node definition (`● ... ○`) is an atomic unit.

# Semantics & Execution

## Evaluation Model

TØR-G represents a **Combinatorial Boolean Circuit**.

*   **State:** The graph itself holds no state. It is a pure function: $f(\vec{I}) \rightarrow \vec{O}$.
*   **Timing:** Execution is instantaneous (1 tick).

## Handling Complex Logic

TØR-G does **not** support arithmetic, string manipulation, or loops.

*   **Arithmetic:** Comparisons (e.g., `requests < 10`) MUST be computed by the Arkavo Edge runtime *before* graph evaluation. The runtime feeds the result as a boolean input (e.g., `IN_req_limit_ok`).
*   **State/History:** Counters and windows MUST be managed externally.

## Node Evaluation

Given a node definition: `● n_res ⊽ n_a n_b ○`:

1.  Resolve value of `n_a` and `n_b`.
2.  Compute `val = NOT (n_a OR n_b)`.
3.  Store `val` in `n_res`.

# Integration Architecture

This section describes how TØR-G resides within the Arkavo Edge Agent logic.

## The Pipeline

1.  **Spec Injection:** The Agent receives a structured prompt containing the current context and goal tokens.
2.  **Logit Masking:** The LLM inference engine engages "Graph Mode," masking all vocabulary except TØR-G tokens.
3.  **Synthesis:** The LLM emits the TØR-G stream.
4.  **Compilation (Builder):**
    *   The stream is consumed token-by-token.
    *   The graph is built in memory.
    *   **Validation:** Structural checks (cycles, undefined refs) are performed immediately.
5.  **Verification (Solver):**
    *   (Optional) SAT/BDD solver checks the graph against invariant constraints (e.g., "Output MUST be False if `IN_is_admin` is False").
6.  **Execution:** The graph gates the requested action.

## Mapping Strategy

To avoid retraining embeddings, TØR-G symbols SHOULD be mapped to existing unused/rare tokens in the base model vocabulary.

*   *Example:* Map `⊽` to token ID `8942` (an obscure Unicode char).
*   This mapping MUST be consistent across Training (LoRA) and Inference (Edge).

# Example Graph

**Goal:** Allow action if User is Admin **OR** (User is Owner **XOR** Resource is Public).

**Input Mapping (Runtime):**

*   `ID_0`: `is_admin`
*   `ID_1`: `is_owner`
*   `ID_2`: `is_public`

**TØR-G Stream:**

```text
◎IN ID_0
◎IN ID_1
◎IN ID_2
● ID_3 ⊻ ID_1 ID_2 ○     <-- ID_3 = Owner XOR Public
● ID_4 ∨ ID_0 ID_3 ○     <-- ID_4 = Admin OR ID_3
◎OUT ID_4
```

# Security & Safety

## Denial of Service (DoS) Prevention

The Builder MUST enforce the following limits:

*   **Max Nodes:** (e.g., 256) To prevent memory exhaustion.
*   **Max Depth:** To prevent latency spikes.

## Syntactic Correctness

By enforcing **Topological Emission** and using **Logit Masking**, the system mathematically guarantees that the LLM cannot produce a syntax error or a cyclic graph.

## Functional Correctness

Since the LLM is probabilistic, the generated graph MUST be validated against a "Golden Vector" suite (known Input/Output pairs) before being applied to a live decision.

# IANA Considerations

This document has no IANA actions.

# References

## Normative References

- **RFC 2119**: Key words for use in RFCs to Indicate Requirement Levels

## Informative References

- **SAT**: Boolean Satisfiability Problem - https://en.wikipedia.org/wiki/Boolean_satisfiability_problem
- **BDD**: Binary Decision Diagram - https://en.wikipedia.org/wiki/Binary_decision_diagram

# Acknowledgments

This specification is developed as part of the Arkavo Edge Internal Logic Kernel.

---

**Document History:**

- **draft-arkavo-torg-decision-00** (2025-12-13): Initial draft
