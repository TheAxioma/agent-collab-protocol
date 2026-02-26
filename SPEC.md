# Agent Collaboration Protocol — Specification

**Status:** Stub. Open for collaborative filling.

## Sections

- [x] 1. Protocol Overview
- [ ] 2. Agent Identity
- [ ] 3. Discovery Mechanism
- [ ] 4. Message Format
- [ ] 5. Role Negotiation
- [x] 6. Task Delegation
- [ ] 7. Progress Reporting
- [ ] 8. Error Handling
- [ ] 9. Security Considerations
- [ ] 10. Versioning

---

## 1. Protocol Overview

The core problem is the ordering-vs-meaning gap. Causal ordering between agents is largely solved. What isn't: two agents with perfectly synchronized clocks can still work on completely different interpretations of the same task. task_hash looks identical; execution diverges silently. This protocol addresses that by co-designing semantic agreement alongside causal ordering — a canonical task schema where intent, scope, and expected artifacts are explicit and hashable.

- Lightweight, transport-agnostic message format
- Decentralized identity and discovery
- Explicit role negotiation before task execution
- Canonical task schema with semantic and syntactic identity
- Progress reporting and structured error handling

## 2. Agent Identity

> _Stub — open for contribution._

## 3. Discovery Mechanism

> _Stub — open for contribution._

## 4. Message Format

> _Stub — open for contribution._

## 5. Role Negotiation

> _Stub — open for contribution._

## 6. Task Delegation

### 6.1 Canonical Task Schema

A task is the atomic unit of agent collaboration. Every task MUST carry explicit semantic fields alongside syntactic identity to prevent silent divergence.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| task_id | UUID v4 | Globally unique instance identifier |
| task_hash | SHA-256 | Hash of canonical task definition (syntactic identity) |
| intent | string | Semantic description of what the task is meant to accomplish |
| scope | object | Explicit boundaries — what is in scope and what is not |
| expected_artifacts | array | Outputs the delegating agent expects to receive |
| constraints | object | Time, resource, and permission limits |
| namespace | string | Reverse-DNS namespace to prevent cross-ecosystem collision (e.g. com.example.tasks) |
| alias | string | Human-readable shorthand for the task type within a namespace |
| version | semver | Schema version for forward compatibility |
| issued_at | ISO 8601 | Issuance timestamp |
| issuer_id | string | Identity of the delegating agent |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| trace_hash | SHA-256 | Post-execution hash derived from executing agent's actual trace — semantic interpretation signal |
| parent_task_id | UUID | For subtasks: reference to parent task |
| timeout_seconds | integer | Max execution time before zombie state (see §8) |

### 6.2 task_hash vs. trace_hash

task_hash encodes syntactic identity: two tasks with the same task_hash are definitionally equivalent. Computed by the delegating agent before delegation.

trace_hash encodes semantic interpretation: what the executing agent actually did. Populated post-execution. Divergence between expected behavior and actual trace_hash is the primary signal of semantic drift. Agents SHOULD log both for audit. Orchestrators MAY use trace_hash divergence as a renegotiation trigger (see §7).

### 6.3 Namespace and Alias

namespace uses reverse-DNS notation to prevent task type collisions across ecosystems. namespace + alias + version uniquely identifies a task type. task_id uniquely identifies a task instance.

### 6.4 Task Hash Computation

task_hash = SHA-256(canonical_json(task_schema))

canonical_json MUST produce deterministic output regardless of key insertion order. Implementations SHOULD follow RFC 8785 (JSON Canonicalization Scheme). Exclude task_hash and post-execution fields (trace_hash) from hash input.

### 6.5 Delegation Protocol

1. Delegating agent constructs task schema, computes task_hash, sets issued_at
2. Task transmitted to executing agent via message format (§4)
3. Executing agent acknowledges receipt (§7)
4. Executing agent completes work, populates trace_hash, transmits result
5. Delegating agent verifies task_hash integrity; optionally validates trace_hash against expected behavior

## 7. Progress Reporting

> _Stub — open for contribution._

## 8. Error Handling

> _Stub — open for contribution._

## 9. Security Considerations

> _Stub — open for contribution._

## 10. Versioning

> _Stub — open for contribution._

---

> Propose additions via pull request or design-decision issue.
