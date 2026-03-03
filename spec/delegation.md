<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

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
| intent_hash | SHA-256 | Hash of the delegation intent context: the delegator's declared purpose, constraints, and outcome framing. Commits to the *why* as `task_hash` commits to the *what*. See §6.4.1. |
| issuer_id | string | Identity of the delegating agent |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| trace_hash | SHA-256 | Post-execution hash derived from executing agent's actual trace — semantic interpretation signal |
| parent_task_id | UUID | For subtasks: reference to parent task |
| timeout_seconds | integer | Max execution time before zombie state (see §8) |
| success_criteria | object | Binary pass/fail predicate for task completion verification |
| idempotency_token | string | Unique token enabling deduplication of retried task assignments |
| retry_semantics | enum | Retry behavior on failure: `restart` (full re-execution), `resume` (from last checkpoint), `none` (no retry). Default: `restart` |
| progress_checkpoint | object | Current execution state snapshot for mid-task cold-start recovery. Structure: `{completed_steps: array, partial_state: object, last_checkpoint_at: ISO 8601}`. Enables resumption at last known checkpoint rather than full restart from step 0. Typically populated from the `state` field of the most recent TASK_CHECKPOINT (§6.6) received by the delegating agent. |
| state_delta_assertions | array | Expected observable state changes post-execution, declared at task assignment time by the delegating agent. Each entry is an object with `assertion_id` (string, unique within the task), `description` (string, human-readable), `target` (string, what system or state to check), and `expected_change` (string, the observable delta — e.g., "file X exists", "row inserted in table Y", "API endpoint returns updated value"). When present, verifiers MUST check these assertions post-execution per §8.19. When absent, verification scope is structural only — current behavior preserved. For non-deterministic operations (LLM calls, web searches), assertions SHOULD be structural only (response received, format matched) — full semantic verification is out of scope per §1. |
| idempotent | boolean | Declares that the task is idempotent by design — repeated execution produces no additional state change beyond the first successful execution. When `true`, a `DELTA_ABSENT` verification result (§8.19) against this task is expected behavior, not a failure. When `false` or absent, `DELTA_ABSENT` is treated as a silent failure. Default: `false`. |

`success_criteria` is a binary pass/fail predicate. For tasks with partial completion states, populate `progress_checkpoint` alongside `success_criteria` — cold-start recovery SHOULD inspect `progress_checkpoint` before deciding whether to restart from step 0 or resume from the last recorded state. `idempotency_token` + `retry_semantics` answers "did this task already complete?" — not "what state was the task in when it stopped?". For tasks with incremental state (file processing, multi-step code generation, pipeline stages), task-level idempotency is insufficient for mid-task recovery; use `progress_checkpoint` and TASK_CHECKPOINT (§6.6) to capture intermediate execution state.

### 6.2 task_hash vs. trace_hash

task_hash encodes syntactic identity: two tasks with the same task_hash are definitionally equivalent. Computed by the delegating agent before delegation.

trace_hash encodes semantic interpretation: what the executing agent actually did. Populated post-execution. Divergence between expected behavior and actual trace_hash is the primary signal of semantic drift. Agents SHOULD log both for audit. Orchestrators MAY use trace_hash divergence as a renegotiation trigger (see §7).

#### 6.2.1 Parallel Execution Trace Semantics

When an agent spawns concurrent sub-agents, completion order is nondeterministic. A naïve serialized log hash changes with race conditions even when the outcome is semantically identical. The following rules define deterministic `trace_hash` computation for sequential, parallel, and mixed execution topologies.

**Rule 1 — Sequential traces (existing behavior).** For single-agent, purely sequential execution, `trace_hash` is the SHA-256 hash of the sequential execution log. No change to existing behavior.

**Rule 2 — Parallel-spawn traces.** When an agent spawns N concurrent sub-agents, the `trace_hash` for the parallel group is the Merkle root of the sub-agent trace hashes, with leaves sorted by hash value (ascending lexicographic order of the hex-encoded SHA-256 digests). Sorting by hash value — not by completion order, spawn order, or sub-agent ID — makes the root deterministic regardless of which sub-agent finishes first.

```
parallel_trace_hash = merkle_root(sort_by_hash_value([
  hash(sub_agent_1_trace),
  hash(sub_agent_2_trace),
  ...
  hash(sub_agent_N_trace)
]))
```

**Rule 3 — Mixed sequential + parallel segments.** Execution traces that interleave sequential steps and parallel groups are modeled as a chain of segment hashes. Sequential segments hash in order. Each parallel group contributes its Merkle root as a single chain link:

```
trace_hash = SHA-256(seg_1_hash || seg_2_hash || ... || seg_K_hash)
```

Where each `seg_i_hash` is either:
- The SHA-256 hash of a sequential execution segment, or
- The Merkle root of a parallel group (computed per Rule 2).

Chaining uses concatenation with the previous hash: `SHA-256(previous_hash || segment_hash)`, applied left-to-right across the segment sequence.

**Rule 4 — Duplicate sub-agent trace hashes.** Two sub-agents that produce identical trace hashes MUST remain as separate leaves in the Merkle tree. Two agents doing identical work is semantically distinct from one agent doing it. Implementations MUST NOT deduplicate leaves.

**Rule 5 — Nested parallelism.** `trace_hash` is recursively composable. A sub-agent's `trace_hash` is opaque at each level — the parent does not need to know whether a child's hash came from sequential or parallel execution. A child that itself spawns parallel sub-agents computes its own `trace_hash` per these rules, and that hash becomes a single leaf in the parent's Merkle tree.

**Rule 6 — Empty parallel group.** A parallel spawn with zero sub-agents (e.g., a dynamic fan-out that produces no work) MUST use the sentinel value:

```
empty_parallel_hash = SHA-256("EMPTY_TRACE")
```

This value is used as the segment hash for the empty group in mixed-segment chains (Rule 3).

**Merkle tree construction.** Given a sorted list of N sub-agent trace hashes as leaves:

1. Pair leaves left-to-right: `SHA-256(leaf[0] || leaf[1])`, `SHA-256(leaf[2] || leaf[3])`, etc.
2. If the leaf count is odd, duplicate the last leaf before pairing (standard Merkle padding).
3. Repeat pairing on the resulting hashes until a single root hash remains.
4. The root is the `trace_hash` for the parallel group.

All hashes are raw 32-byte SHA-256 digests. Concatenation (`||`) is byte-level concatenation of the two 32-byte values, producing a 64-byte input to the next SHA-256 call.

> Addresses [issue #118](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/118): deterministic `trace_hash` for parallel sub-agent execution. Source: @Jarvis4, @OutlawAI (original proposal), with production implementation and edge case analysis (mixed segments, duplicate leaves, nested parallelism) from @Haustorium12.

### 6.3 Namespace and Alias

namespace uses reverse-DNS notation to prevent task type collisions across ecosystems. namespace + alias + version uniquely identifies a task type. task_id uniquely identifies a task instance.

### 6.4 Task Hash Computation

Task identity is computed via a MANIFEST — a sorted, deterministic list of `(key, type, value_hash)` tuples derived from the task schema fields. The MANIFEST separates task representation from task identity: each agent uses its own internal schema and serialization format; coordination requires only that agents agree on the sorted tuple structure. Canonicalization rules for type names and serialization are specified in §4.10.

**Canonical hashing method:**

```
manifest = sorted([
  (key, canonical_type(val), SHA-256(serialize(nfc(val))))
  for key, val in task.items()
  if key not in EXCLUDED_FIELDS
])
task_hash = SHA-256(jcs_serialize(nfc(manifest)))
```

Where `canonical_type(val)` maps the runtime type to the canonical name (§4.10.1), `nfc()` applies Unicode NFC normalization (§4.10.2), and `jcs_serialize()` produces RFC 8785 JCS output.

`EXCLUDED_FIELDS` = `{task_hash, trace_hash, intent_hash, state_delta_assertions, idempotent}` — the hash output, post-execution fields, intent commitment fields, and verification-layer metadata (§8.19) MUST be excluded from hash input. `state_delta_assertions` and `idempotent` are excluded because they inform the verification layer about expected outcomes, not the task's syntactic identity — the same task definition can be delegated with or without delta assertions without changing `task_hash`.

**MANIFEST construction rules:**

1. **Key:** The field name as a UTF-8 string, exactly as defined in §6.1 (e.g., `intent`, `scope`, `constraints`).
2. **Type:** The canonical type name of the value per the Canonical Type Name Registry (§4.10.1). Implementations MUST map to one of: `string`, `integer`, `float`, `bool`, `null`, `list`, `dict`, `datetime`, `bytes`. Language-specific type names (e.g., Python's `str`, Go's `int64`, Rust's `String`) MUST be mapped to these canonical names before tuple construction.
3. **Value hash:** `SHA-256(serialize(val))` where `serialize` produces a byte representation of the value. For scalar types (string, integer, float, bool, null), serialize as the UTF-8 encoding of the canonical string representation. For composite types (dict, list), serialize by recursively constructing a sub-MANIFEST and hashing it — the same sorted-tuple procedure applied at each level of nesting. All string inputs MUST be NFC-normalized before serialization (§4.10.2).
4. **Sorting:** Tuples are sorted lexicographically by key (UTF-8 byte order). Key uniqueness is guaranteed by the task schema definition (§6.1).
5. **Final hash:** `task_hash = SHA-256(canonical_serialize(manifest))` where `canonical_serialize` applies NFC normalization followed by RFC 8785 JCS serialization as specified in §4.10.2. The sorted tuple list is serialized as a JSON array of `[key, canonical_type, value_hash_hex]` arrays.

**Why MANIFEST, not canonical JSON:**

Two implementations can diverge on serialization format (JSON, CBOR, MessagePack, Protobuf), runtime language, key ordering conventions, floating-point representation, and whitespace handling — and still produce identical `task_hash` values, because identity is computed over the sorted tuple structure rather than a serialized document. This prevents canonicalization drift: the failure mode where two agents produce different canonical forms of the same logical task due to differences in their JSON canonicalization libraries, Unicode normalization, or number encoding.

RFC 8785 (JCS) is the required serialization for the final hash computation step (§4.10.2), applied after MANIFEST construction. The MANIFEST structure preserves transport independence — agents construct tuples using their preferred internal format — while JCS ensures the hash computation boundary produces identical bytes across runtimes. NFC Unicode normalization is applied as a pre-pass before JCS to close the cross-runtime Unicode divergence gap (§4.10.2).

**Example:**

Given a task with fields:

```yaml
intent: "Summarize document"
scope:
  include: ["chapters 1-3"]
  exclude: ["appendix"]
constraints:
  timeout_seconds: 300
```

The MANIFEST (before final hashing) is:

```
[
  ("constraints", "dict",   SHA-256(<sub-manifest of constraints>)),
  ("intent",      "string", SHA-256("Summarize document")),
  ("scope",       "dict",   SHA-256(<sub-manifest of scope>))
]
```

Two agents — one in Python using `json.dumps`, one in Rust using `serde_cbor` — construct the same MANIFEST and produce the same `task_hash`, because neither depends on the other's serialization format. The sorted tuple structure is the shared contract.

### 6.4.1 Per-Hop Intent Attestation

`task_hash` (§6.4) commits to the structural *what* of a task — syntactic identity. `intent_hash` commits to the semantic *why* — the delegator's declared purpose, constraints, and outcome framing. Without `intent_hash`, an adversarial intermediary B in a chain A→B→C can forward an unchanged `task_hash` to C while substituting A's intent context: the delegation remains protocol-compliant, audit-clean, and semantically distorted.

#### V1 Scope Decision: `intent_hash` Is a Tamper-Evidence Primitive

> **Normative scope statement.** In V1, `intent_hash` is a **tamper-evidence primitive**, not a semantic authorization primitive. The following normative requirements apply:
>
> 1. **Tamper-evidence guarantee.** `intent_hash` provides tamper-evidence for delegated intent in V1. Any mutation of the delegated intent context after signing is detectable by downstream verifiers via signature validation over the attestation tuple.
>
> 2. **Payload format is implementation-defined.** The schema of the `intent_hash` preimage — the internal structure of the `intent`, `scope`, and `constraints` fields — is implementation-defined in V1. No normative payload schema is mandated beyond the canonicalization rules in this section.
>
> 3. **Cross-agent partial authorization checking is deferred to V2.** Verifying that executed actions fall within the delegated resource class (partial authorization) requires structured payload agreement across implementations — a shared intent vocabulary and task classification taxonomy. This capability is out of scope for V1 and deferred to V2.
>
> 4. **No cross-agent semantic interoperability guarantee.** An implementation MAY structure its intent payload internally (and is encouraged to do so per the recommendation below), but cross-agent semantic interoperability of `intent_hash` content is not guaranteed in V1. Two compliant implementations can verify each other's `intent_hash` commitments (the hash is intact) but cannot interpret each other's intent semantics (what the hash commits to).
>
> Agents reading this specification should not infer the scope of `intent_hash` from the absence of authorization semantics — this scope decision is deliberate and explicit. Addresses [issue #231](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/231).

#### V1 Scope: Tamper-Evidence Only

> **V1 provides tamper-evidence for intent, not authorization verification.** Implementors MUST NOT conflate these two properties.

**Tamper-evidence** (V1 guarantee): Per-hop signing of `(task_hash || intent_hash || delegator_id)` proves that an intermediary did not substitute or modify the delegator's declared intent context after signing. If B signs an attestation over `intent_hash`, any subsequent alteration of the intent fields invalidates B's signature. The hash is preserved; the content it commits to is immutable post-attestation.

**Authorization verification** (NOT a V1 guarantee): Confirming that a preserved intent actually *authorized* what a delegated agent executed. Even with a valid, unmodified `intent_hash`, V1 cannot determine whether agent C's execution falls within the semantic scope of agent A's original intent. For example, A delegates "refactor the parser" to B, B re-delegates to C — V1 can prove A's intent was not swapped, but cannot evaluate whether C's execution (rewriting the lexer) is semantically covered by "refactor the parser."

**Mechanism gap:** Authorization verification requires a shared intent vocabulary — a common semantic framework that all participants use to interpret intent fields and evaluate whether an execution satisfies a declared intent. This dependency is strictly harder than PKI (which V1 already assumes) because it requires semantic agreement, not just cryptographic infrastructure. Shared intent semantics are deferred to V2.

**Implementor guidance:** A delegation chain where every `intent_hash` verifies is audit-clean (no intermediary tampered with intent) but is not necessarily semantically correct (the preserved intent may not cover the downstream execution). V1 implementations MUST NOT treat successful `intent_hash` verification as proof of authorization compliance.

#### V1 Scope: Intent Payload Schema Is Implementation-Defined

> **V1 mandates `intent_hash` as a tamper-evident commitment field. The schema of the preimage — what gets hashed — is implementation-defined in V1.** No specific intent payload schema is normatively required.

The protocol requires that `intent_hash` exist and be computed over the intent context fields (`intent`, `scope`, `constraints`) using the canonicalization rules in §6.4.1. However, V1 does not mandate the internal structure of those fields. The `intent` field may be a free-form string, a structured object with typed sub-fields, or any other JSON-serializable value. The same applies to `scope` and `constraints`. Two spec-compliant V1 implementations may use entirely different internal representations for these fields.

**Recommendation:** Implementations SHOULD use a structured intent representation that includes at minimum: action type (what operation is being delegated), target resource (what the operation acts on), and constraints (bounds on how the operation may be performed). Structured representations improve auditability and lay groundwork for V2 interoperability. However, this is advisory — no specific schema is mandated by V1.

**Interoperability consequence:** Because the intent payload schema is implementation-defined, two V1 agents from different implementations cannot semantically compare or interpret each other's intent fields — they can only verify that the hash commitment is intact. Cross-agent intent semantic interoperability requires a shared intent vocabulary: a common schema and value space that all participants agree on for interpreting intent fields. This is deferred to V2, where a shared intent vocabulary will be defined.

> Addresses [issue #229](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/229): explicit V1 scope statement that the intent payload schema is implementation-defined. References #229.

#### intent_hash Computation

`intent_hash` is the SHA-256 hash of the delegation intent context. The intent context is the concatenation of the following fields from the task schema (§6.1), canonicalized per §4.10:

```
intent_context = canonical_serialize(nfc([
  ("intent",      task.intent),
  ("scope",       task.scope),
  ("constraints", task.constraints)
]))
intent_hash = SHA-256(intent_context)
```

Where `canonical_serialize` applies NFC normalization followed by RFC 8785 JCS serialization (§4.10.2). The fields are sorted lexicographically by key. If a field is absent from the task schema, it is omitted from the intent context (not serialized as null).

`intent_hash` is listed in `EXCLUDED_FIELDS` (§6.4) — it is excluded from `task_hash` computation because it is a commitment *about* the task's purpose, not a component of the task's structural identity.

#### intent_hash Canonicalization

Two spec-compliant implementations using different field ordering, whitespace handling, or character encoding will produce different hashes for semantically identical intents — making hash mismatch indistinguishable from intent deviation. This breaks the core tamper-evidence guarantee of `intent_hash`. Implementors MUST follow the canonicalization rules below when computing `intent_hash`. These rules are self-contained for `intent_hash` computation; they are consistent with §4.10.2 but do not require cross-referencing that section.

**Rule 1 — Serialization format.** The intent context MUST be serialized as JSON. No other serialization format (CBOR, MessagePack, Protobuf, YAML) is permitted for `intent_hash` computation.

**Rule 2 — Key ordering.** All keys MUST be sorted lexicographically (by Unicode code point value) at every level of nesting. This applies recursively: keys within nested objects inside `intent`, `scope`, or `constraints` MUST also be sorted. Sorting is applied before serialization, not as a post-processing step on the serialized output.

**Rule 3 — Encoding.** The serialized JSON output MUST be encoded as UTF-8 (RFC 3629). No byte-order mark (BOM, U+FEFF) is permitted. If a BOM is present in any input string, it MUST be stripped before serialization. UTF-8 is the only permitted encoding for the bytes input to SHA-256.

**Rule 4 — Whitespace.** The JSON serialization MUST be compact: no whitespace between tokens. Specifically: no spaces after colons separating keys from values, no spaces after commas separating elements, no trailing whitespace, no newline characters. The serialized output is a single contiguous line of JSON with no extraneous whitespace.

**Rule 5 — String normalization.** All string values in the intent context — including keys and all nested string values within `intent`, `scope`, and `constraints` — MUST be NFC-normalized (Unicode Canonical Decomposition followed by Canonical Composition, per [Unicode TR15](https://unicode.org/reports/tr15/)) before serialization. NFC normalization MUST be applied before JSON serialization, not after. This closes the cross-runtime Unicode divergence gap: different runtimes may store the same logical string in NFC, NFD, or mixed form internally, producing different byte sequences and therefore different hashes.

**Rule 6 — Field inclusion.** The intent context object MUST include the following fields from the task schema (§6.1): `intent`, `scope`, `constraints`. All three fields are included by default. No other task schema fields are included in the intent context. If future protocol versions add fields to the intent context, they MUST be listed explicitly in this rule and the protocol version in which they were added MUST be noted.

**Rule 7 — Null and absent field handling.** Absent fields — fields not present in the task schema instance — MUST be omitted from the serialized intent context entirely (the key MUST NOT appear in the JSON object). Null fields — fields explicitly set to `null` in the task schema instance — MUST be included in the serialized output with JSON `null` as the value. This distinction is normative: `{"intent":"summarize","scope":null}` and `{"intent":"summarize"}` produce different `intent_hash` values. Implementations MUST NOT conflate absent and null.

**Canonical computation summary:**

```
1. Collect fields: {intent, scope, constraints} from the task schema instance
2. Omit absent fields; retain null fields with JSON null
3. Apply NFC normalization to all string values (keys and values, recursively)
4. Sort keys lexicographically at every nesting level
5. Serialize as compact JSON (no whitespace between tokens)
6. Encode the JSON string as UTF-8 (no BOM)
7. intent_hash = SHA-256(utf8_bytes)
```

**Example:**

Given a task with:

```yaml
intent: "Summarize document"
scope:
  include: ["chapters 1-3"]
  exclude: ["appendix"]
constraints:
  timeout_seconds: 300
```

The canonical serialized intent context is the following single-line JSON string (shown here with the UTF-8 byte sequence as input to SHA-256):

```
{"constraints":{"timeout_seconds":300},"intent":"Summarize document","scope":{"exclude":["appendix"],"include":["chapters 1-3"]}}
```

Keys are sorted at every level (`constraints` < `intent` < `scope`; `exclude` < `include`; `timeout_seconds` is the only key in `constraints`). No whitespace between tokens. NFC normalization is a no-op for ASCII strings. The SHA-256 hash of the UTF-8 encoding of this string is the `intent_hash`.

> Addresses [issue #221](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/221): intent_hash canonicalization rules for deterministic hash computation across independent implementations. Closes #221.

#### Per-Hop Attestation

Each delegating node MUST sign the attestation tuple `(task_hash || intent_hash || delegator_id)` before forwarding a delegation. This binds the node's identity to its assertion of both task structure and intent.

**Attestation requirements:**

1. The delegating agent constructs the attestation tuple by concatenating `task_hash`, `intent_hash`, and its own `issuer_id` (§6.1).
2. The delegating agent signs the tuple using its keypair (§2 identity). The signature algorithm is the same as used for identity attestation (§2.4).
3. The signed attestation is included in TASK_ASSIGN (§6.6) as the `delegation_attestation` field.
4. The receiving agent MUST verify the attestation signature against the delegating agent's public key before accepting or forwarding the task. Verification failure MUST result in TASK_REJECT with `reason: "attestation_verification_failed"`.
5. When re-delegating (§6.9), the intermediate agent MUST produce a **new** attestation tuple signed with its own keypair. The intermediate agent's attestation covers its own `intent_hash` — which may differ from the upstream agent's if the intermediate agent has translated or reframed the intent (§7.9). The upstream agent's attestation is preserved in the `delegation_attestation_chain` for audit.

**Attestation tuple format:**

```
attestation_input = task_hash || intent_hash || delegator_id
attestation_signature = Sign(delegator_keypair, attestation_input)
```

Where `||` denotes concatenation of the UTF-8 byte representations, and `Sign` uses the delegating agent's signing key.

**`delegation_attestation` field (in TASK_ASSIGN):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_hash | SHA-256 | Yes | The `task_hash` being attested |
| intent_hash | SHA-256 | Yes | The `intent_hash` being attested |
| delegator_id | string | Yes | The attesting agent's identity (§2.4) |
| signature | string | Yes | Cryptographic signature over `(task_hash \|\| intent_hash \|\| delegator_id)` |
| signed_at | ISO 8601 | Yes | Timestamp of attestation |

**`delegation_attestation_chain` field (in TASK_ASSIGN):**

An ordered array of `delegation_attestation` objects from each hop in the delegation chain. Each intermediate agent appends the upstream agent's attestation to this chain before constructing its own. The chain provides a complete audit trail of intent assertions across all hops.

#### V1 Adversary Model

Per-hop attestation covers the **honest-but-careless intermediary** — an agent that might inadvertently modify or fail to preserve intent context. With per-hop attestation, intent deviation becomes auditable: each hop's assertion of intent is cryptographically bound to its identity.

Per-hop attestation does **NOT** cover the **Byzantine intermediary** who generates a valid attestation asserting false intent. A Byzantine agent B in chain A→B→C can sign a fabricated `intent_hash` — the signature is valid because B holds the signing key, but the asserted intent does not reflect A's original intent. Detecting this requires **Recursive Intent Bundles**: nested cryptographic wrappers where each hop's intent assertion encloses the upstream chain's intent assertions, requiring PKI infrastructure that V1 cannot assume.

**V1 commitment:** hop-local attestation only. A fully attested delegation chain proves that each hop *asserted* its intent. It does not prove that any assertion was *truthful*. Compliant does not mean trustworthy. Recursive Intent Bundles are deferred to V2.

#### Attestation in the Audit Trail

Attestation tuples MUST appear in `divergence_log` (§7.8, §8.10.3) when attestation-related events occur during delegation processing.

Attestation verification failure MUST be recorded with reason `attestation_failure` (§8.10.4) — distinct from `verification_reject` (verifier evaluated evidence and rejected it) and `verification_unreachable` (verifier was not reachable). `attestation_failure` indicates that the delegation attestation itself — the per-hop cryptographic binding of identity to intent — failed verification at the receiving agent.

The `divergence_log` entry for an attestation failure MUST include:

| Field | Value |
|-------|-------|
| reason | `attestation_failure` |
| description | Human-readable explanation of the failure |
| affected_step_id | The step (if any) associated with the delegation that failed attestation |
| context.attesting_node_id | Identity of the agent whose attestation failed verification |
| context.asserted_task_hash | The `task_hash` in the failed attestation tuple |
| context.asserted_intent_hash | The `intent_hash` in the failed attestation tuple |
| context.failure_reason | Specific cause: `signature_invalid`, `key_unknown`, `hash_mismatch`, or `attestation_missing` |

### 6.5 Delegation Protocol

1. Delegating agent constructs task schema, computes task_hash and intent_hash, sets issued_at
2. Delegating agent generates a `request_id` (UUID v4) for the delegation-initiation attempt (§6.14). The same `request_id` MUST be used on all retransmissions of this TASK_ASSIGN.
3. Delegating agent signs the attestation tuple `(task_hash || intent_hash || delegator_id)` (§6.4.1). For sub-delegations (`delegation_depth > 0`), the attestation tuple is `(task_hash || intent_hash || delegator_id || parent_grant_hash)` (§6.9.3.1).
4. Task transmitted to executing agent via session message format (§4.3), including `delegation_attestation` and `request_id`. Sub-delegations MUST include `parent_grant_hash` and `parent_grant_id` (§6.9.3.1).
5. Executing agent deduplicates on `request_id` (§6.14). If a TASK_ASSIGN with the same `request_id` was already processed within the idempotency window, the delegatee returns the cached response without re-executing.
6. Executing agent verifies the delegation attestation signature before processing the task (§6.4.1). For sub-delegations, the executing agent additionally verifies `parent_grant_hash` against the parent delegation (§6.9.3.2). Verification failure results in TASK_REJECT.
7. Executing agent acknowledges receipt (§7)
8. *(Optional)* Executing agent computes plan_hash, sends PLAN_COMMIT (§6.6, §6.11) to delegating agent before beginning work. REQUIRED when session was established with `plan_commit_required: true` (§4.3).
9. Executing agent completes work, populates trace_hash, transmits result
10. Delegating agent verifies task_hash and intent_hash integrity; optionally validates trace_hash against expected behavior. If PLAN_COMMIT was received, delegating agent verifies plan_hash_ref in the result for three-level alignment (§6.11).

### 6.6 Delegation Message Types

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

The delegation protocol uses eleven message types. All messages MUST include `task_id` for correlation.

**TASK_ASSIGN**

Sent by the delegating agent to initiate delegation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Unique task instance identifier (from §6.1) |
| correlation_id | UUID v4 | Yes | Unique identifier for request-response correlation (§4.14.4). Response messages (TASK_ACCEPT, TASK_REJECT) MUST echo this value. |
| idempotency_key | UUID v4 | Yes | Sender-generated key for deduplication (§4.14.3). Distinct from `request_id` — `idempotency_key` is the transport-layer deduplication primitive; `request_id` is the delegation-initiation identity. Both MUST be present. |
| request_id | UUID v4 | Yes | Stable delegation-initiation identifier for deduplication. Generated once at initiation time, before the first transmission. The same `request_id` MUST be used on all retries for the same delegation attempt. Distinct from `task_id` — `task_id` identifies the task; `request_id` identifies the delivery attempt. See §6.14. |
| session_id | string | Yes | Active session identifier binding this delegation to a collaboration session |
| spec | object | Yes | Task specification (see below) |
| trust_level | enum | Yes | Trust level granted to the delegatee for this task (see §6.8) |
| intent_hash | SHA-256 | Yes | Hash of the delegation intent context (§6.4.1). Commits to the delegator's declared purpose, constraints, and outcome framing. |
| delegation_attestation | object | Yes | Signed attestation binding the delegator's identity to both `task_hash` and `intent_hash` (§6.4.1). The receiving agent MUST verify this signature before accepting the task. |
| delegation_depth | integer | Yes | Current depth in the delegation chain; 0 for direct delegation (see §6.9) |
| delegation_attestation_chain | array | No | Ordered array of `delegation_attestation` objects from each upstream hop in the delegation chain (§6.4.1). Provides a complete audit trail of intent assertions across all hops. Empty or absent for direct delegations at depth 0. |
| protocol_version_chain | array | No | Append-only list of version chain entries recording the protocol version negotiated at each hop in the delegation chain (see §6.9.1). Empty or absent for direct delegations at depth 0. |
| translation_metadata | object | No | Metadata describing a semantic translation performed by the forwarding agent (see §7.9). Present when the forwarding agent transforms task semantics — vocabulary, capability descriptions, or constraint representations — rather than merely routing. |
| revocation_mode | enum | No | Revocation verification strategy for this delegation: `sync` or `gossip` (see §9.8.5). When absent, defaults to `gossip`. The delegating agent selects the mode based on chain depth, trust level, and latency tolerance. Propagated through the delegation chain — each hop inherits the mode from its delegator unless explicitly overridden. This field establishes a minimum default, not a binding constraint: downstream agents MAY apply a stricter mode for their own sub-delegations (§4.3.2). |
| parent_grant_hash | SHA-256 | No | SHA-256 hash of the parent delegation's canonical JSON (§6.9.3). REQUIRED when `delegation_depth > 0`. Structurally binds this sub-delegation to the parent delegation it derives authority from — a sub-delegation without a valid `parent_grant_hash` MUST be rejected. Computed over the parent TASK_ASSIGN's canonical JSON representation (RFC 8785 JCS, §4.10.2). |
| parent_grant_id | UUID v4 | No | The `task_id` of the parent delegation (§6.9.3). REQUIRED when `delegation_depth > 0`. Provides a lookup key for retrieving the parent delegation during chain traversal verification. |
| max_delegation_depth | integer | No | Maximum delegation depth permitted from the root delegation, declared by the originating agent (§6.9.3). Default: 3 if unspecified. Propagated unchanged through the delegation chain — intermediate agents MUST NOT increase this value. Sub-delegations that would result in `delegation_depth >= max_delegation_depth` MUST be rejected at issuance. |

The `spec` object contains:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| description | string | Yes | Semantic description of the task (equivalent to `intent` in §6.1) |
| expected_output_format | object | Yes | Schema or description of the expected result structure |
| deadline | ISO 8601 | No | Absolute deadline for task completion |
| resource_constraints | object | No | Limits on compute, memory, network, or other resources the delegatee may consume |

The full canonical task schema (§6.1) is transmitted within or alongside `spec`. The fields above are the delegation-specific subset; `task_hash`, `intent_hash`, `namespace`, `alias`, `version`, `issued_at`, and `issuer_id` travel with the task schema as defined in §6.1.

The `translation_metadata` object, when present, contains:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| source_schema | string | Yes | Declared protocol, schema identifier, or vocabulary of the originating agent's task representation. |
| target_schema | string | Yes | Declared protocol, schema identifier, or vocabulary of the destination agent's task representation. |
| fidelity_confidence | enum | Yes | Self-assessed confidence that the translation preserves the original task's semantic intent: `LOW`, `MEDIUM`, or `HIGH`. |
| translation_losses | array of strings | No (SHOULD) | Descriptions of what was approximated, dropped, or semantically altered during translation. Each entry identifies a specific aspect of the original task that could not be faithfully conveyed to the target schema. |

**`translation_losses` SHOULD be populated.** Agents that omit `translation_losses` remain V1-compliant — the field is not required to reduce implementation burden. However, agents relying on high-fidelity translation across heterogeneous systems are RECOMMENDED to implement the full field. Without `translation_losses`, the receiving agent has no structured signal about what semantic content may have been degraded — only the coarse `fidelity_confidence` enum. Agents that implement `translation_losses` provide actionable information for downstream verification (§7.9).

**TASK_ACCEPT**

Sent by the delegatee to confirm it will execute the task.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to match this acceptance to the originating delegation request (§4.14.4). |
| request_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to correlate the acceptance with the specific delegation-initiation attempt. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| accepted_at | ISO 8601 | Yes | Timestamp of acceptance |

A TASK_ACCEPT is a commitment. After sending TASK_ACCEPT, the delegatee is responsible for eventually sending TASK_COMPLETE or TASK_FAIL.

**PLAN_COMMIT**

Sent by the delegatee to the coordinator after TASK_ACCEPT (and after SESSION_INIT_ACK), before the first TASK_PROGRESS. PLAN_COMMIT creates a verifiable pre-execution commitment — the delegatee hashes its execution plan before starting work, creating an auditable record the coordinator can verify against actual results. Unlike declared intent, a pre-execution commitment with a prior hash is falsifiable.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| plan_hash | SHA-256 | Yes | Hash of the delegatee's canonical plan representation. MUST be deterministic for the same logical plan. Recommended: SHA-256 of canonical UTF-8 JSON. The canonical representation is implementation-defined but MUST be documented in the agent's capability manifest (§5.1). |
| task_hash_ref | SHA-256 | Yes | The `task_hash` (§6.1) of the task this plan was made against. Binds the plan commitment to a specific task version — any task modification invalidates the prior commitment and requires a new PLAN_COMMIT. |
| steps | array | Yes | Ordered array of step declarations defining the plan's execution structure. See step declaration schema below. |
| spec_version | semver | Yes | Protocol spec version governing this plan commitment (§10). |
| timestamp | ISO 8601 | Yes | When the PLAN_COMMIT was sent. |

**Step declaration schema:**

Each element in the `steps` array MUST conform to the following schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| step_id | string | Yes | Unique identifier within this plan. MUST be unique across all steps in the same PLAN_COMMIT. |
| description | string | Yes | Human-readable description of what this step accomplishes. |
| estimated_tokens | integer | No | Estimated token budget for this step. Informational — not enforced by the protocol. |
| reversible | boolean | No | Whether the step's effects can be undone after completion. Default: `false`. When `false`, the delegating agent MUST treat the step's effects as committed once its end boundary is crossed (§6.11.5). |
| depends_on | array of strings | No | Array of `step_id` values that MUST have crossed their end boundary before this step may begin. The delegatee enforces this constraint — a step with `depends_on` MUST NOT begin execution until all listed dependency steps have completed. Circular dependencies are malformed and MUST be rejected. |

**Example step declaration:**

```yaml
steps:
  - step_id: "parse-input"
    description: "Parse and validate input document structure"
    estimated_tokens: 2000
    reversible: true
  - step_id: "transform-data"
    description: "Apply transformation rules to parsed data"
    estimated_tokens: 5000
    reversible: false
    depends_on: ["parse-input"]
  - step_id: "write-output"
    description: "Write transformed data to output format"
    estimated_tokens: 3000
    reversible: false
    depends_on: ["transform-data"]
```

**PLAN_COMMIT semantics:**

- PLAN_COMMIT is optional by default. It becomes mandatory when `plan_commit_required: true` was negotiated in SESSION_INIT (§4.3). When mandatory, the coordinator MUST NOT accept TASK_PROGRESS before receiving PLAN_COMMIT — a TASK_PROGRESS arriving before PLAN_COMMIT is a protocol violation.
- The coordinator MUST reject a PLAN_COMMIT if `task_hash_ref` does not match the current `task_hash` for the referenced task. This prevents stale commitments: if the task changes after PLAN_COMMIT is sent, the hash is stale and the delegatee MUST send a new PLAN_COMMIT against the updated task.
- The spec MUST NOT mandate a specific internal plan representation format — agent architectures vary too widely. It MUST require that `plan_hash` is deterministic for the same logical plan.
- PLAN_COMMIT MAY be sent even when `plan_commit_required` is `false` — the coordinator SHOULD accept and store it for audit purposes regardless.
- A delegatee MAY send a new PLAN_COMMIT to supersede a previous one (e.g., after receiving updated task parameters). Only the most recent PLAN_COMMIT is considered the active commitment.

**Step boundary semantics:**

- A **step boundary** is crossed when the delegatee transitions from one `step_id` to the next. When a step boundary is crossed, the previous step is **immutable** — its outputs may be treated as committed by the delegating agent.
- Once a step boundary has been crossed, the delegating agent MUST NOT amend that step. PLAN_AMEND (§6.6) is only valid for future (not-yet-started) steps. An amendment that attempts to modify a step whose end boundary has already been crossed is a protocol violation and MUST be rejected by the delegatee.
- The delegatee SHOULD report step boundary crossings via TASK_PROGRESS (§6.6) by including the `current_step_id` field. This enables the delegating agent to track which steps are committed and which remain amendable.
- Step boundary crossing is monotonic — a step that has been crossed cannot be "uncrossed." The delegatee MUST NOT re-enter a step whose boundary has been crossed. If re-execution of a completed step's logic is necessary (e.g., due to downstream failure), the delegatee MUST declare a new step in a PLAN_AMEND that captures the rework.

**PLAN_COMMIT_ACK:**

PLAN_COMMIT_ACK is the acknowledgment message for plan-related messages. When responding to PLAN_COMMIT, the **delegating agent** (coordinator) sends it. When responding to PLAN_AMEND, the **delegatee** sends it. In both cases, the receiving party MUST reply with PLAN_COMMIT_ACK within the session heartbeat timeout (§4.5). Failure to respond within the heartbeat timeout is a protocol violation — the sender MAY treat it as equivalent to rejection.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from PLAN_COMMIT |
| session_id | string | Yes | Echoed from PLAN_COMMIT |
| accepted | boolean | Yes | Whether the delegatee accepts the committed plan. |
| reason | string | No | Required when `accepted` is `false`. Explanation of why the plan was rejected. |
| first_rejected_step_id | string | No | Required when `accepted` is `false`. The `step_id` of the first step the delegatee cannot accept. Steps prior to this one are implicitly acceptable. |
| timestamp | ISO 8601 | Yes | When the PLAN_COMMIT_ACK was sent. |

**PLAN_COMMIT_ACK semantics:**

- In the PLAN_COMMIT flow: the delegating agent (coordinator) sends PLAN_COMMIT_ACK in response to the delegatee's PLAN_COMMIT. In the PLAN_AMEND flow: the delegatee sends PLAN_COMMIT_ACK in response to the delegating agent's PLAN_AMEND.
- When `accepted: true`, the plan (or amendment) becomes the active commitment for three-level alignment verification (§6.11.1). The delegatee MAY begin or continue execution.
- When `accepted: false`, the plan (or amendment) is rejected. The `first_rejected_step_id` indicates where the plan becomes unacceptable. For a rejected PLAN_COMMIT, the delegatee MAY construct a revised PLAN_COMMIT. For a rejected PLAN_AMEND, the delegating agent MAY revise the amendment. The delegatee MUST NOT begin execution of a rejected plan.

**PLAN_AMEND**

Sent by the delegating agent to modify future (not-yet-started) steps in an active PLAN_COMMIT. PLAN_AMEND respects step boundary immutability — it cannot modify steps whose end boundaries have already been crossed.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from PLAN_COMMIT |
| session_id | string | Yes | Echoed from PLAN_COMMIT |
| plan_hash_ref | SHA-256 | Yes | The `plan_hash` of the active PLAN_COMMIT being amended. Ensures the amendment targets the correct plan version. |
| amended_steps | array | Yes | Array of step declarations (same schema as `steps` in PLAN_COMMIT) replacing or adding to future steps. Each step MUST have a `step_id` that either matches a not-yet-started step in the original plan or is a new step. |
| removed_step_ids | array of strings | No | Array of `step_id` values to remove from the plan. Each listed step MUST be a not-yet-started step. Removing a step that has already been crossed or is currently executing is a protocol violation. |
| amended_plan_hash | SHA-256 | Yes | New `plan_hash` computed over the amended plan (original plan with amendments applied). Becomes the active `plan_hash` for three-level alignment verification if the amendment is accepted. |
| amend_hash | SHA-256 or HMAC-SHA-256 | Yes | Authorization hash binding the amendment to the session and specific change content. Computed as `SHA256(session_id \|\| step_id \|\| canonical_json(new_step_spec))` where `canonical_json` uses JCS (§4.10.2) and `step_id` is the first step in `amended_steps`. If a session-level HMAC key was negotiated at SESSION_INIT, `HMAC-SHA256(hmac_key, session_id \|\| step_id \|\| canonical_json(new_step_spec))` MUST be used instead. For amendments affecting multiple steps, the hash input concatenates all `step_id \|\| canonical_json(new_step_spec)` pairs in `amended_steps` array order. |
| timestamp | ISO 8601 | Yes | When the PLAN_AMEND was issued by the delegating agent. This is the requester's issuance time (T1), not the delegatee's acceptance or commit time. Preserved in the delegatee's `amendments_log` (§6.11.6) as `proposed_at`. For determining which spec version was in force at an arbitrary execution point, auditors MUST use the `amendments_log` `timestamp` field (T3, commit time), not this issuance timestamp — see §6.11.6 amendment timestamp semantics. |
| scope_delta | object | Yes | Structured description of what the amendment changes relative to the prior plan version. Contains: `modified_step_ids` (array of `step_id` values whose specifications changed), `added_step_ids` (array of new `step_id` values introduced by this amendment), `removed_step_ids` (array of `step_id` values removed by this amendment, echoed from top-level `removed_step_ids`), and `changed_fields` (object mapping each modified `step_id` to an array of field names that differ from the prior spec). Enables relational re-commitment verification: auditors can confirm that re-commitment was correctly scoped to what actually changed, rather than a full-plan recommit that is externally indistinguishable from a fresh start. |
| notes | string | No | Human-readable summary of what changed and why. SHOULD be populated to make post-mortems actionable without requiring full trace reconstruction from cryptographic fields alone. Not cryptographic — `notes` is not included in `amend_hash` computation and MUST NOT be relied upon for integrity verification. Preserved in the delegatee's `amendments_log` (§6.11.6) as `notes`. |

**PLAN_AMEND semantics:**

- PLAN_AMEND is valid only for steps whose end boundaries have **not** been crossed. An amendment targeting a crossed step MUST be rejected by the delegatee.
- The delegatee MUST validate that all `step_id` values in `amended_steps` and `removed_step_ids` refer to not-yet-started steps. If any refer to crossed or in-progress steps, the delegatee MUST reject the amendment.
- **amend_hash verification:** The delegatee MUST verify `amend_hash` on receipt by independently computing the hash from the received `session_id`, `amended_steps`, and the session-level HMAC key (if negotiated). On mismatch, the delegatee MUST reply with PLAN_AMEND_REJECT containing `step_id` and `reason=hash_mismatch`. The delegatee MUST NOT apply the amendment when `amend_hash` verification fails.
- On accepting PLAN_AMEND (after successful `amend_hash` verification), the delegatee MUST respond with PLAN_COMMIT_ACK (with `accepted: true` or `false`). The `amended_plan_hash` becomes the new active plan hash if accepted.
- On accepting PLAN_AMEND, the delegatee MUST append an entry to its `amendments_log` in durable session state (§6.11.6).
- PLAN_AMEND does not reset step boundaries — all previously crossed steps remain committed and immutable.
- Dependency constraints (`depends_on`) in amended steps are validated against the full amended plan, including both unchanged and amended steps. A `depends_on` reference to a removed step is malformed and MUST be rejected.
- **scope_delta validation:** The delegatee MUST verify that `scope_delta` is consistent with the actual amendment content. Specifically: every `step_id` in `amended_steps` MUST appear in either `scope_delta.modified_step_ids` or `scope_delta.added_step_ids`; every `step_id` in `removed_step_ids` MUST appear in `scope_delta.removed_step_ids`; and `scope_delta.changed_fields` MUST list at least one field for each entry in `scope_delta.modified_step_ids`. If `scope_delta` is inconsistent with the amendment content, the delegatee MUST reject the amendment with PLAN_AMEND_REJECT (`reason=invalid_spec`).

**PLAN_AMEND_REJECT**

Sent by the delegatee to reject a PLAN_AMEND that fails validation. PLAN_AMEND_REJECT is distinct from PLAN_COMMIT_ACK with `accepted: false` — it indicates a structural or authorization failure in the amendment itself, not a semantic disagreement about the amended plan content.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| plan_id | string | Yes | The `plan_hash_ref` from the rejected PLAN_AMEND, identifying which plan the amendment targeted. |
| step_id | string | Yes | The `step_id` of the first step that triggered the rejection, or the first step in `amended_steps` for `hash_mismatch` rejections. |
| reason | enum | Yes | Reason for rejection. One of: `hash_mismatch` (the `amend_hash` does not match the delegatee's independent computation — indicates potential unauthorized amendment or transmission corruption), `step_already_crossed` (the amendment targets a step whose end boundary has already been crossed — violates step boundary immutability), `invalid_spec` (the step specification in `amended_steps` is malformed, contains invalid `depends_on` references, or otherwise fails schema validation). |
| timestamp | ISO 8601 | Yes | When the PLAN_AMEND_REJECT was sent. |

**PLAN_AMEND_REJECT semantics:**

- On receiving PLAN_AMEND_REJECT, the delegating agent MUST NOT assume the amendment was applied. The active plan remains unchanged — the `plan_hash` from before the rejected PLAN_AMEND is still the active plan hash.
- For `hash_mismatch` rejections: the delegating agent SHOULD investigate the cause (transmission corruption, stale session state, or unauthorized third-party amendment attempt) before retrying. A repeated `hash_mismatch` from the same delegatee for the same amendment content indicates a systemic integrity problem, not a transient error.
- For `step_already_crossed` rejections: the delegating agent MUST NOT retry the same amendment. The step is committed and immutable (§6.11.5).
- For `invalid_spec` rejections: the delegating agent MAY revise the step specification and resubmit a corrected PLAN_AMEND.

**TASK_REJECT**

Sent by the delegatee to decline the task.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to match this rejection to the originating delegation request (§4.14.4). |
| request_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to correlate the rejection with the specific delegation-initiation attempt. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| reason | string | Yes | Why the task was rejected (insufficient capabilities, resource limits, trust level incompatible, etc.) |

TASK_REJECT is terminal for the delegatee. The delegating agent MAY reassign the task to another agent.

**TASK_PROGRESS**

Optionally sent by the delegatee during execution. Serves as both a progress update and a heartbeat signal (see §8.4 protocol-layer heartbeat).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — TASK_PROGRESS is unidirectional, not a response. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| progress | object | No | Structured progress data (percentage, subtask status, etc.) |
| current_step_id | string | No | The `step_id` (from the active PLAN_COMMIT's `steps` array) of the step currently being executed. Present when PLAN_COMMIT with steps is active. When `current_step_id` changes from a previous TASK_PROGRESS, this signals that the previous step's end boundary has been crossed. |
| timestamp | ISO 8601 | Yes | When this progress report was generated |
| version_chain_summary | object | No | Summary of the protocol version chain across downstream hops known at the time of this progress report (see §6.9.1). Present when the delegatee has sub-delegated and has received version chain information from downstream. |
| delegation_chain | array of strings | No | Ordered list of agent identifiers representing every agent that handled or forwarded the task, in execution order (see §6.9.2). Present when the task involves sub-delegation. |

TASK_PROGRESS is optional — the protocol does not require progress reporting for task completion. However, implementations that set `timeout_seconds` (§6.1) SHOULD use TASK_PROGRESS as a liveness signal to distinguish a working agent from a zombied one (§8.1). The absence of TASK_PROGRESS within the expected heartbeat interval is an input to the external verifier (§4.7.2), not a definitive failure signal.

**TASK_COMPLETE**

Sent by the delegatee when the task finishes successfully.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — TASK_COMPLETE is unidirectional, not a response. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| result | object or string | Yes | Structured result conforming to `expected_output_format`, or a resource reference (URI) to the result |
| trace_hash | SHA-256 | Yes | Post-execution hash of the actual execution trace (§6.2) |
| completed_at | ISO 8601 | Yes | Timestamp of completion |
| version_chain_summary | object | No | Summary of the protocol version chain across all hops that contributed to this result (see §6.9.1). Present when the task involved sub-delegation. |
| divergence_log | array | No | Array of divergence entries recording each point where execution departed from the committed plan (see §7.8). Present when `trace_hash` differs from the committed plan hash (L2, §7.4). |
| delegation_chain | array of strings | No | Ordered list of agent identifiers representing every agent that handled or forwarded the task, in execution order (see §6.9.2). Present when the task involved sub-delegation. |
| plan_hash_ref | SHA-256 | No | Back-reference to the `plan_hash` from the most recent PLAN_COMMIT (§6.6, §6.11) for this task. Enables audit log correlation and three-level alignment verification: L1 (`task_hash`) → L2 (`plan_hash_ref`) → L3 (`trace_hash`). Present when PLAN_COMMIT was sent during execution. |
| completed_steps | array of strings | No | Array of `step_id` values (from the active PLAN_COMMIT's `steps` array) whose end boundaries were crossed before completion. Present when PLAN_COMMIT with steps was active. For a successful TASK_COMPLETE, this SHOULD include all steps in the plan. See §6.11.5 for partial completion semantics. |
| amendments_log | array | No | Array of amendment audit entries recording each accepted PLAN_AMEND during execution (see §6.11.6). Present when any PLAN_AMEND was accepted during the task. The delegating agent SHOULD re-verify all `amend_hash` values on receipt to confirm that all spec drift was authorized. |

The delegating agent SHOULD verify that `result` conforms to `expected_output_format` from the original TASK_ASSIGN. Non-conforming results are not a protocol error — the delegating agent decides whether to accept, reject, or request rework.

**TASK_FAIL**

Sent by the delegatee when the task cannot be completed.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — TASK_FAIL is unidirectional, not a response. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| error | object | Yes | Structured error: `code` (string), `message` (string), `details` (object, optional) |
| partial_results | object or null | No | Whatever partial output is available; null if nothing was produced |
| trace_hash | SHA-256 | No | Post-execution hash if any execution occurred before failure |
| failed_at | ISO 8601 | Yes | Timestamp of failure |
| version_chain_summary | object | No | Summary of the protocol version chain across downstream hops (see §6.9.1). Present when the task involved sub-delegation, even on failure — enables the originating agent to assess whether version degradation contributed to the failure. |
| divergence_log | array | No | Array of divergence entries recording plan departures that occurred before failure (see §7.8). Present when any execution occurred before failure and the execution diverged from the committed plan. |
| delegation_chain | array of strings | No | Ordered list of agent identifiers representing every agent that handled or forwarded the task, in execution order (see §6.9.2). Present when the task involved sub-delegation, even on failure — enables post-hoc audit of which agents were involved before the failure occurred. |
| plan_hash_ref | SHA-256 | No | Back-reference to the `plan_hash` from the most recent PLAN_COMMIT (§6.6, §6.11) for this task. Present when PLAN_COMMIT was sent before failure — enables post-hoc analysis of whether failure was a plan-level or execution-level problem. |
| completed_steps | array of strings | No | Array of `step_id` values (from the active PLAN_COMMIT's `steps` array) whose end boundaries were crossed before the failure. Present when PLAN_COMMIT with steps was active. The delegating agent MUST treat effects of crossed steps with `reversible: false` as committed even though the overall task failed (§6.11.5). |
| amendments_log | array | No | Array of amendment audit entries recording each accepted PLAN_AMEND during execution (see §6.11.6). Present when any PLAN_AMEND was accepted before the failure. The delegating agent SHOULD re-verify all `amend_hash` values on receipt to confirm that all spec drift was authorized, even on task failure. |

TASK_FAIL with `partial_results` is preferred over TASK_FAIL without — even incomplete output may be useful for recovery or reassignment. The delegating agent owns the recovery decision (see §6.10).

**TASK_CHECKPOINT**

Optionally sent by the delegatee during execution to persist recoverable state. Not a substitute for TASK_PROGRESS — TASK_CHECKPOINT is for state recovery; TASK_PROGRESS is for progress reporting and heartbeat.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Task instance being checkpointed |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — TASK_CHECKPOINT is unidirectional, not a response. |
| session_id | string | Yes | Active session identifier |
| checkpoint_id | UUID v4 | Yes | Unique identifier for this checkpoint (enables ordered recovery) |
| state | object | Yes | Serializable task state snapshot at this point |
| completed_steps | array | No | Identifiers of completed steps or subtasks |
| partial_output | object | No | Output accumulated so far (if incrementally buildable) |
| timestamp | ISO 8601 | Yes | When this checkpoint was created |

- TASK_CHECKPOINT is optional — not all tasks benefit from incremental checkpointing.
- Tasks with `timeout_seconds` set SHOULD emit TASK_CHECKPOINT at meaningful execution boundaries to enable mid-task cold-start recovery.
- Delegating agent SHOULD store the latest TASK_CHECKPOINT alongside the task schema (§6.1). The `progress_checkpoint` optional field in §6.1 is typically populated from the `state` field of the most recent TASK_CHECKPOINT received by the delegating agent.
- If a TASK_CHECKPOINT exists for a `task_id`, cold-start reconstruction SHOULD attempt resumption from that checkpoint before falling back to full restart.
- Cross-reference: §4 session lifecycle defines the recovery protocol (SESSION_RESUME handshake, §4.8). TASK_CHECKPOINT provides the task-level state that complements session-level recovery.

**TASK_CANCEL**

Sent by the delegating agent to explicitly cancel an in-flight task. TASK_CANCEL is a first-class message type — not an overloading of TASK_FAIL semantics. The distinction matters: TASK_FAIL is a delegatee-initiated signal ("I failed"); TASK_CANCEL is a delegator-initiated signal ("stop working").

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Task to cancel |
| correlation_id | UUID v4 | Yes | Unique identifier for this cancellation request (§4.14.4). |
| session_id | string | Yes | Active session identifier |
| reason | string | Yes | Why the task is being cancelled. Standard reasons: `session_expired` (parent session entered EXPIRED, §4.2), `cancelled_upstream` (upstream delegator cancelled), `superseded` (task replaced by a new assignment), `coordinator_decision` (general cancellation by delegator) |
| cancelled_at | ISO 8601 | Yes | Timestamp of cancellation |

**TASK_CANCEL semantics:**

- TASK_CANCEL is valid only for tasks in the TASK_ACCEPT → TASK_COMPLETE/TASK_FAIL window. Sending TASK_CANCEL for a task that has not been accepted, or that has already completed or failed, is a no-op — the receiver MUST ignore it.
- Upon receiving TASK_CANCEL, the delegatee SHOULD stop execution as soon as practical and MUST respond with either TASK_FAIL (with error code `cancelled` and any `partial_results`) or TASK_COMPLETE (if the task finished before the cancel was processed). A TASK_COMPLETE received after TASK_CANCEL is valid — cancellation is best-effort, not guaranteed.
- **Bilateral CANCEL on session expiry:** When a session enters EXPIRED (§4.2, §4.5.3), the expiring agent MUST send TASK_CANCEL with `reason: "session_expired"` to every in-flight subtask delegated within that session. This is mandatory — without explicit cancellation, the delegatee may complete work and deliver a result to a coordinator that has already abandoned the session (phantom completion). The TASK_CANCEL message is sent even though the session is EXPIRED because it targets the delegatee's session, which may still be ACTIVE from the delegatee's perspective.
- **Cancellation propagation in delegation chains:** When agent B receives TASK_CANCEL from agent A, and B has sub-delegated parts of the task to agent C, B MUST propagate TASK_CANCEL to C with `reason: "cancelled_upstream"`. This propagates recursively down the chain (§6.9).

### 6.7 Delegation Lifecycle

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

The delegation lifecycle follows a linear state machine:

```
TASK_ASSIGN → TASK_ACCEPT → [PLAN_COMMIT → PLAN_COMMIT_ACK] → TASK_PROGRESS*    → TASK_COMPLETE
                                                                TASK_CHECKPOINT*  → TASK_FAIL
                                                               ← PLAN_AMEND (delegator-initiated)
                                                                 → PLAN_COMMIT_ACK or PLAN_AMEND_REJECT
                                                               ← TASK_CANCEL (delegator-initiated)
           → TASK_REJECT
```

`[PLAN_COMMIT → PLAN_COMMIT_ACK]` is optional by default; it becomes mandatory when `plan_commit_required: true` was negotiated in SESSION_INIT (§4.3). PLAN_AMEND is valid only after PLAN_COMMIT_ACK and only for not-yet-started steps.

**State transitions:**

| Current State | Valid Next Messages | Sender |
|---------------|---------------------|--------|
| (initial) | TASK_ASSIGN | Delegator |
| TASK_ASSIGN sent | TASK_ACCEPT, TASK_REJECT | Delegatee |
| TASK_ACCEPT sent | PLAN_COMMIT, TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_ACCEPT sent | TASK_CANCEL | Delegator |
| PLAN_COMMIT received | PLAN_COMMIT_ACK | Delegator (coordinator) |
| PLAN_COMMIT received | TASK_CANCEL | Delegator |
| PLAN_COMMIT_ACK received (accepted) | PLAN_COMMIT (supersede), TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| PLAN_COMMIT_ACK received (accepted) | PLAN_AMEND, TASK_CANCEL | Delegator |
| PLAN_COMMIT_ACK received (rejected) | PLAN_COMMIT (revised) | Delegatee |
| PLAN_COMMIT_ACK received (rejected) | TASK_CANCEL | Delegator |
| PLAN_AMEND received | PLAN_COMMIT_ACK, PLAN_AMEND_REJECT | Delegatee |
| TASK_PROGRESS sent | TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_PROGRESS sent | PLAN_AMEND, TASK_CANCEL | Delegator |
| TASK_CHECKPOINT sent | TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_CHECKPOINT sent | PLAN_AMEND, TASK_CANCEL | Delegator |
| TASK_CANCEL sent | TASK_FAIL (with `cancelled`), TASK_COMPLETE (if already finished) | Delegatee |
| TASK_COMPLETE sent | (terminal) | — |
| TASK_FAIL sent | (terminal) | — |
| TASK_REJECT sent | (terminal) | — |

Messages received out of sequence MUST be rejected. An agent that receives TASK_PROGRESS for a task_id it never sent TASK_ASSIGN for MUST ignore the message and MAY log it as anomalous. When `plan_commit_required: true` is active (§4.3), TASK_PROGRESS received before PLAN_COMMIT_ACK (with `accepted: true`) for the same `task_id` is a protocol violation — the coordinator MUST reject it.

**Timeout behavior:** If `timeout_seconds` is set in the task schema (§6.1) and the delegatee has not sent TASK_COMPLETE or TASK_FAIL within that window after TASK_ACCEPT, the delegator MAY treat the task as failed. The delegator SHOULD send no message — timeout is a delegator-side decision, not a protocol message. The delegatee may still be executing (zombie state, §8.1); the external verifier (§4.7.2) handles liveness determination.

**Accept/reject deadline:** The protocol does not define a deadline for TASK_ACCEPT or TASK_REJECT. Implementations SHOULD define a reasonable timeout after TASK_ASSIGN, after which the delegator MAY reassign the task. This timeout is deployment-specific, not protocol-specified.

### 6.8 Trust Semantics

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

Trust during delegation follows a strict inheritance model with no privilege escalation.

**Core rule:** The delegatee inherits the delegator's trust level for the duration of the subtask. This is scoped — the inherited trust applies only to operations within the delegated task's `scope` and `constraints`. It does not extend to other tasks, other sessions, or capabilities outside the task specification.

**No privilege escalation.** A delegatee MUST NOT acquire capabilities beyond what the delegator holds. If agent A has trust level `restricted` and delegates to agent B, agent B operates at `restricted` for that task — even if B independently holds a higher trust level in other contexts. The delegation narrows B's effective trust to min(B's own trust, A's delegated trust).

**Delegation-in-B's-name is not B acquiring new capabilities.** When A delegates to B, B acts on A's behalf within A's trust boundary. This is not B gaining access to A's capabilities — it is A extending its own execution through B. The distinction matters for audit: the `issuer_id` in the task schema (§6.1) traces back to A. B's actions under this delegation are attributable to A for trust accounting purposes.

**Trust level values** are defined by the deployment's trust topology (§9.2). The protocol does not mandate specific trust level names or semantics beyond requiring that they are comparable (a partial order exists) and that `trust_level` in TASK_ASSIGN is always ≤ the delegator's own trust level for the session.

**Relationship to §9.2:** Trust semantics during delegation operate within a single trust topology. Cross-topology delegation (e.g., from orchestrator-over-worker to peer-to-peer) requires the trust boundary handling described in §9.3 — attestation MUST be re-evaluated at the boundary.

### 6.9 Delegation Chains

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

Delegation chains — where a delegatee further delegates subtasks to other agents — are permitted.

**Depth limit:** v1 enforces a maximum delegation depth of 3. The `delegation_depth` field in TASK_ASSIGN tracks the current position in the chain. An agent receiving TASK_ASSIGN with `delegation_depth >= 3` MUST NOT further delegate; it MUST either execute the task itself or reject it.

The depth limit is configurable per deployment. Implementations MAY set a lower limit. Implementations MUST NOT exceed the protocol maximum of 3 in v1.

**Chain construction:**

```
A assigns to B (delegation_depth: 0)
  B assigns to C (delegation_depth: 1)
    C assigns to D (delegation_depth: 2)
      D must execute or reject (delegation_depth: 3 would exceed limit)
```

When re-delegating, the delegatee:

1. Increments `delegation_depth` by 1
2. Sets `trust_level` to at most its own effective trust level for the parent task (§6.8 — no escalation)
3. Sets `issuer_id` to its own identity (proximate delegator), but MUST include `parent_task_id` (§6.1) referencing the upstream task for auditability
4. Computes a new `task_hash` for the subtask schema
5. Computes a new `intent_hash` for the subtask's delegation intent context (§6.4.1)
6. Sets `parent_grant_hash` to the SHA-256 of the parent TASK_ASSIGN's canonical JSON (§6.9.3.1) and `parent_grant_id` to the parent task's `task_id`
7. Signs the attestation tuple `(task_hash || intent_hash || delegator_id || parent_grant_hash)` with its own keypair and includes the upstream `delegation_attestation` in `delegation_attestation_chain` (§6.4.1, §6.9.3.1)
8. Propagates `max_delegation_depth` unchanged from the parent TASK_ASSIGN (§6.9.3.3) — MUST NOT increase the value
9. Appends its own version chain entry to `protocol_version_chain` — recording the protocol and schema versions negotiated for the session between itself and the next delegatee (§6.9.1)

**Cancellation propagation.** Cancellation propagates down the chain via TASK_CANCEL (§6.6). If A cancels the task assigned to B (by sending TASK_CANCEL), B MUST send TASK_CANCEL with `reason: "cancelled_upstream"` to any subtasks it delegated to C, and C MUST do the same for D. Each agent in the chain sends TASK_CANCEL to its delegatees and responds to its delegator with TASK_FAIL (error code `cancelled`, with any `partial_results`).

Cancellation is best-effort — a delegatee that has already sent TASK_COMPLETE before receiving TASK_CANCEL is not required to undo completed work. The delegator receiving a TASK_COMPLETE after sending TASK_CANCEL MAY discard the result.

**Expiry-triggered cancellation.** When a parent session enters EXPIRED (§4.2, §4.5.3), the expiring agent MUST send TASK_CANCEL with `reason: "session_expired"` to all in-flight subtasks. This prevents phantom completions — where a delegatee delivers a result to a coordinator that has already abandoned the session. Expiry-triggered TASK_CANCEL propagates down the delegation chain identically to voluntary cancellation.

**Partial result recovery after expiry.** If a session enters EXPIRED but the counterparty later becomes reachable, partial results from cancelled tasks can be recovered via SESSION_RESUME (§4.8). The resuming agent presents its state hash; if state is reconcilable, the session transitions back to ACTIVE and cancelled tasks with `partial_results` can be re-evaluated. This reuses the existing crash-recovery mechanism — no parallel recovery path is introduced.

**Chain visibility.** The delegator at depth N has visibility only into depth N+1. A does not directly observe C or D. Intermediate agents (B, C) are responsible for aggregating results and propagating failures up the chain.

### 6.9.1 Version Chain Visibility

In a delegation chain, each hop establishes its own session with independent version negotiation (§10.2). The `protocol_version_chain` field in TASK_ASSIGN provides the originating agent with full visibility into the protocol versions negotiated at every hop in the chain — without requiring in-memory tracking or direct observation of intermediate sessions.

**Version chain entry format:**

Each entry in `protocol_version_chain` is a structured object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| hop_agent_id | string | Yes | Stable identity artifact (§2.4) of the agent at this hop |
| negotiated_protocol_version | semver | Yes | Protocol version negotiated for the session at this hop (from SESSION_INIT exchange, §10.2) |
| negotiated_schema_version | semver | Yes | Schema version negotiated for the session at this hop |
| hop_index | integer | Yes | Zero-based position in the chain (0 = originating agent's session with the first delegatee) |

**Chain construction:**

When an agent re-delegates a task (§6.9), it MUST append its own version chain entry to `protocol_version_chain` before forwarding the TASK_ASSIGN to the next delegatee. The entry records the protocol and schema versions negotiated for the session between this agent and the next delegatee.

```
A assigns to B (protocol_version_chain: [])
  B assigns to C (protocol_version_chain: [
    {hop_agent_id: "B", negotiated_protocol_version: "1.2.0",
     negotiated_schema_version: "1.3.0", hop_index: 0}
  ])
    C assigns to D (protocol_version_chain: [
      {hop_agent_id: "B", negotiated_protocol_version: "1.2.0",
       negotiated_schema_version: "1.3.0", hop_index: 0},
      {hop_agent_id: "C", negotiated_protocol_version: "1.1.0",
       negotiated_schema_version: "1.2.0", hop_index: 1}
    ])
```

The chain is append-only. Intermediate agents MUST NOT modify entries appended by upstream agents. An agent that receives a `protocol_version_chain` with entries it did not write MUST forward them unmodified. Tampering with upstream entries is detectable via signature verification if the originating agent signs the chain at delegation time.

**Version degradation detection:**

Version degradation occurs when any hop in the chain negotiated a protocol or schema version lower than the originating agent's version. The originating agent detects degradation by inspecting the `protocol_version_chain` (carried forward in TASK_ASSIGN) or the `version_chain_summary` (returned in TASK_COMPLETE, TASK_PROGRESS, or TASK_FAIL).

An originating agent MAY define a `minimum_acceptable_version` policy — a floor below which version negotiation at any hop renders the result unacceptable. When the `version_chain_summary` in a TASK_COMPLETE response reveals that any hop negotiated below this floor:

- The originating agent MAY reject the result (discard and reassign the task to a chain with acceptable versions).
- The originating agent MAY accept the result but flag it as **degraded** — marking the result with a degradation annotation for downstream consumers or audit.
- The originating agent MAY log the degradation for observability without affecting result acceptance.

The `minimum_acceptable_version` policy is local to the originating agent — it is not transmitted in the protocol. Different originating agents may have different tolerance for version degradation.

**`version_chain_summary` format:**

The `version_chain_summary` field in TASK_COMPLETE, TASK_PROGRESS, and TASK_FAIL provides the full chain visibility to the delegating agent without requiring it to track intermediate TASK_ASSIGN messages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| chain | array | Yes | Complete `protocol_version_chain` as constructed through all delegation hops |
| min_protocol_version | semver | Yes | Lowest `negotiated_protocol_version` across all hops in the chain |
| min_schema_version | semver | Yes | Lowest `negotiated_schema_version` across all hops in the chain |
| degraded | boolean | Yes | `true` if any hop negotiated a version lower than the originating hop's version; `false` otherwise |
| degradation_hops | array | No | Hop indices where degradation was detected. Present only when `degraded` is `true`. |

**Example `version_chain_summary`:**

```yaml
version_chain_summary:
  chain:
    - hop_agent_id: "agent-B"
      negotiated_protocol_version: "1.2.0"
      negotiated_schema_version: "1.3.0"
      hop_index: 0
    - hop_agent_id: "agent-C"
      negotiated_protocol_version: "1.1.0"
      negotiated_schema_version: "1.1.0"
      hop_index: 1
  min_protocol_version: "1.1.0"
  min_schema_version: "1.1.0"
  degraded: true
  degradation_hops: [1]
```

**Propagation of `version_chain_summary`:**

Each agent in the delegation chain constructs a `version_chain_summary` from the `protocol_version_chain` it received plus any summaries returned by its own delegatees. When an intermediate agent (e.g., B) receives a TASK_COMPLETE from its delegatee (e.g., C) containing a `version_chain_summary`, B merges C's chain information with its own hop entry and forwards the combined summary upstream in its own TASK_COMPLETE to A. This ensures the originating agent receives the full chain without direct visibility into intermediate sessions.

### 6.9.2 Delegation Chain Visibility

The `delegation_chain` field in TASK_COMPLETE, TASK_FAIL, and TASK_PROGRESS provides post-hoc visibility into which agents handled a task during execution. While `protocol_version_chain` (§6.9.1) travels forward with TASK_ASSIGN and records version negotiation at each hop, `delegation_chain` travels backward with result messages and records the identity of every agent that touched the task in execution order.

**Semantics:**

- `delegation_chain` is an ordered array of agent identifier strings (§2 identity artifacts). Each entry is the stable identity of an agent that handled or forwarded the task.
- The chain is **append-only**: each agent in the delegation path appends its own identifier before returning a result (TASK_COMPLETE or TASK_FAIL) or forwarding a progress report (TASK_PROGRESS) upstream. The executing agent (leaf of the delegation tree) appends first; each intermediate agent appends its own identifier as it propagates the result upward.
- Intermediate agents MUST NOT modify entries appended by downstream agents. An agent that receives a result with a `delegation_chain` containing entries it did not write MUST forward them unmodified, appending only its own identifier.

**SHOULD, not MUST.** The `delegation_chain` field is a SHOULD requirement for agents operating in multi-hop delegation contexts. Agents that omit `delegation_chain` remain V1-compliant — the field is optional to reduce implementation burden for agents that do not participate in delegation chains or do not require post-hoc audit of the delegation path. However, agents operating in multi-hop contexts are strongly encouraged to implement it: without `delegation_chain`, the originating agent has no structured record of which agents were involved in producing the result.

**Purpose — post-hoc audit, not pre-flight verification.** The `delegation_chain` is an audit trail: it records who handled the task after execution, not who will handle the task before execution. It does not influence routing, capability negotiation, or trust decisions. Its value is in post-hoc analysis — understanding which agents contributed to a result, diagnosing failures across delegation chains, and attributing execution to specific agents for trust accounting.

**Relationship to §7.8 `divergence_log`.** The `divergence_log` (§7.8) records _what_ deviated from the plan during execution. The `delegation_chain` records _who_ handled the task. These are complementary: when a divergence occurs in a multi-hop delegation, `divergence_log` explains the nature of the deviation while `delegation_chain` identifies which agents were in the execution path. Together, they answer both "what went wrong?" and "who was involved?"

**Relationship to §9 trust boundaries.** Agents operating at translation boundaries (§9.3, §7.9) — those that transform task semantics rather than merely routing — SHOULD include themselves in `delegation_chain`. This is particularly important because translation boundaries are the highest-risk surface in the protocol (§9.3): knowing which agents acted as translators in a delegation chain is essential for post-hoc assessment of whether translation fidelity (§7.9.3) was maintained across the chain.

**Chain construction:**

```
A assigns to B (delegation_depth: 0)
  B assigns to C (delegation_depth: 1)
    C executes the task
    C returns TASK_COMPLETE with delegation_chain: ["agent-C"]
  B receives result, appends its own identity
  B returns TASK_COMPLETE with delegation_chain: ["agent-C", "agent-B"]
A receives result — delegation_chain shows the full execution path
```

**Example TASK_COMPLETE with `delegation_chain`:**

A 3-hop delegation where agent A delegates to agent B, B delegates to agent C, and C executes:

```yaml
message_type: TASK_COMPLETE
task_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
session_id: "session-A-B-001"
result:
  summary: "Analysis complete"
  output_uri: "https://results.example.com/f47ac10b"
trace_hash: "sha256:a1b2c3d4e5f6..."
completed_at: "2026-02-27T14:30:00Z"
delegation_chain:
  - "agent-C"
  - "agent-B"
```

> **Note:** The originating agent (A) does not appear in `delegation_chain` — it is the consumer of the result, not a handler of the task. The chain records agents that handled or forwarded the task during execution, starting from the leaf executor.

### 6.9.3 Sub-Grant Chain Integrity

Delegation chain integrity is the load-bearing property of the §6 trust model. Without structural binding between a sub-delegation and its parent, any agent that knows a parent delegation exists can forge a sub-delegation claiming derived authority. This section specifies the cryptographic and structural requirements that make delegation chains traversable by hash and verifiable without loading all ancestor delegations — critical in partition scenarios where ancestor delegations may be unavailable.

#### 6.9.3.1 Parent-Grant Hash Embedding

Each sub-delegation (TASK_ASSIGN with `delegation_depth > 0`) MUST include:

1. **`parent_grant_hash`** — SHA-256 of the parent TASK_ASSIGN's canonical JSON representation (RFC 8785 JCS, §4.10.2). This is the structural binding: the sub-delegation is cryptographically chained to the specific parent delegation it derives authority from.

2. **`parent_grant_id`** — the `task_id` of the parent delegation, providing a lookup key for chain traversal. `parent_grant_id` enables retrieval; `parent_grant_hash` enables verification. Both are required — a lookup key without a hash is unverifiable; a hash without a lookup key requires exhaustive search.

3. **Delegating agent signature** — the `delegation_attestation` (§6.4.1) for sub-delegations MUST cover `(task_hash || intent_hash || delegator_id || parent_grant_hash)`. This extends the existing attestation tuple to include the parent-grant binding. A sub-delegation whose `delegation_attestation` does not cover `parent_grant_hash` is structurally invalid — the delegating agent's signature does not bind the sub-delegation to its claimed parent.

A sub-delegation without a valid `parent_grant_hash` MUST be rejected by the receiving agent. Rejection uses TASK_REJECT (§6.6) with `reason: "missing_parent_grant_hash"` or `reason: "invalid_parent_grant_hash"`.

**Canonical JSON computation for `parent_grant_hash`:**

The `parent_grant_hash` is computed over the parent TASK_ASSIGN message's canonical JSON, using the same canonicalization rules as §4.10.2 (NFC normalization followed by RFC 8785 JCS serialization). The hash input includes all fields of the parent TASK_ASSIGN — including the parent's own `parent_grant_hash` if it is itself a sub-delegation. This creates a recursive hash chain: each link's hash covers the previous link's hash, making the chain tamper-evident end-to-end.

**Example sub-delegation with parent-grant hash embedding:**

```yaml
message_type: TASK_ASSIGN
task_id: "b2c3d4e5-f6a7-8901-bcde-f12345678901"
session_id: "session-B-C-002"
spec:
  description: "Analyze subset of documents"
  expected_output_format: { type: "json", schema: "analysis-v1" }
trust_level: "standard"
intent_hash: "sha256:c3d4e5f6..."
delegation_attestation:
  signed_tuple: "sha256(task_hash || intent_hash || agent-B || parent_grant_hash)"
  signature: "base64url:..."
  signer_id: "agent-B"
delegation_depth: 1
parent_grant_hash: "sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
parent_grant_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
max_delegation_depth: 3
```

#### 6.9.3.2 Chain Traversal Semantics

Verification of a delegation chain MUST proceed by hash traversal from the terminal (leaf) delegation back to the root delegation. Each link in the chain is verified by:

1. **Hash match:** The `parent_grant_hash` in the current delegation matches the SHA-256 of the parent delegation's canonical JSON. If the hash does not match, the chain is broken at this link — the sub-delegation claims authority from a parent delegation that either does not exist in the claimed form or has been tampered with.

2. **Signature validity:** The delegating agent's `delegation_attestation` signature over `(task_hash || intent_hash || delegator_id || parent_grant_hash)` is valid against the delegating agent's declared `pubkey` (§2.2.1). An invalid signature means the delegating agent did not authorize this sub-delegation — the binding between the sub-delegation and its parent is forged.

3. **Authority check:** The delegating agent at each link is the `task_id` recipient (delegatee) of the parent delegation. An agent that was not delegated the parent task cannot sub-delegate it.

**Traversal terminates** at a root delegation — a TASK_ASSIGN with `delegation_depth: 0` and no `parent_grant_hash` (§6.9.3.4). The root delegation's `delegation_attestation` is verified against the originating agent's identity.

**Partial chains do not confer authority.** If any link in the chain fails verification — hash mismatch, invalid signature, or missing parent delegation — the entire chain from that point to the leaf is invalid. A sub-delegation cannot derive authority from a broken chain. This is analogous to certificate chain validation in TLS: a valid leaf certificate with an invalid intermediate confers no trust.

**Chain traversal in partition scenarios:** The `parent_grant_hash` embedding enables offline chain verification. A verifier holding a cached copy of ancestor delegations can verify the chain by hash comparison without contacting the originating agent or any intermediate agent. This is the primary design motivation — delegation chain verification MUST NOT require real-time connectivity to all agents in the chain.

#### 6.9.3.3 Delegation Depth Limits

Delegation depth is bounded to convert unbounded recursion into a bounded verification task. A chain with a declared depth limit can be fully verified in O(limit) steps regardless of network conditions.

**`max_delegation_depth` semantics:**

- The originating agent (root delegator) declares `max_delegation_depth` in the root TASK_ASSIGN. This value represents the maximum number of delegation hops permitted from the root.
- Default: **3** if `max_delegation_depth` is unspecified. This preserves backward compatibility with the existing §6.9 depth limit of 3.
- `max_delegation_depth` is propagated unchanged through the delegation chain. Intermediate agents MUST NOT increase `max_delegation_depth` — doing so would be a privilege escalation, allowing deeper chains than the originating agent authorized. Intermediate agents MAY decrease `max_delegation_depth` to further restrict their own sub-delegatees.
- A sub-delegation that would result in `delegation_depth >= max_delegation_depth` MUST be rejected at issuance. The agent attempting to sub-delegate MUST NOT emit the TASK_ASSIGN.

**Relationship to existing §6.9 depth limit:** The existing protocol-level maximum of 3 (§6.9) remains as the V1 hard ceiling. `max_delegation_depth` allows originating agents to set a lower limit per-delegation. The effective depth limit is `min(max_delegation_depth, protocol_maximum)`. Implementations MUST NOT allow `max_delegation_depth` to exceed the protocol maximum of 3 in V1.

**Depth enforcement example:**

```
A assigns to B (delegation_depth: 0, max_delegation_depth: 2)
  B assigns to C (delegation_depth: 1, max_delegation_depth: 2)
    C MUST NOT assign to D (delegation_depth would be 2, which equals max_delegation_depth)
    C must execute or reject
```

#### 6.9.3.4 Root Grant Authority

Root delegations are issued by the originating agent with no parent delegation binding:

- `delegation_depth: 0`
- `parent_grant_hash` is absent (MUST NOT be present)
- `parent_grant_id` is absent (MUST NOT be present)
- `delegation_attestation` covers `(task_hash || intent_hash || delegator_id)` — the standard attestation tuple without `parent_grant_hash`

Chain verification terminates at a root delegation. The root delegation's authority derives from the originating agent's identity and trust level (§6.8), not from a parent delegation. A verifier MUST confirm that the root delegation's `issuer_id` is a trusted originating agent within the deployment's trust topology (§9.2), verified against the operator-configured root trust anchor (§1.2). Self-asserted root authority — where the originating agent claims trust terminus without operator configuration — is invalid (§1.2).

A TASK_ASSIGN with `delegation_depth: 0` that includes `parent_grant_hash` or `parent_grant_id` is malformed — it claims to be both a root delegation and a sub-delegation. Receiving agents MUST reject such messages with TASK_REJECT (`reason: "malformed_root_grant"`).

#### 6.9.3.5 Audit Consequence

When chain traversal verification (§6.9.3.2) fails for a sub-delegation — hash mismatch, invalid delegating agent signature, or missing parent delegation — the detecting party MUST emit a DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure`.

`chain_integrity_failure` is distinct from:

- `attestation_mismatch` (§8.11.2) — which applies to state hash, trace hash, plan hash, or SEMANTIC_CHALLENGE response mismatches. `chain_integrity_failure` applies specifically to delegation chain hash traversal failures.
- `VERIFICATION_REJECT` (§8.10.5) — which is a verification-system outcome. `chain_integrity_failure` is a structural property of the delegation chain itself, detectable without an external verifier.
- `DELTA_ABSENT` (§8.19) — which applies to state-delta verification. `chain_integrity_failure` applies to delegation authority verification.

The `chain_integrity_failure` reason code enables automated triage: a delegation chain failure is a trust-model failure, not an execution failure. Recovery requires re-delegation from a valid chain, not re-execution of the same task. The same `chain_integrity_failure` reason code applies to CAPABILITY_GRANT chain verification failures (§5.8.2) — both delegation chains and capability grant chains use the same hash-traversal verification model and the same audit classification.

> Addresses [issue #127](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/127): sub-grant chain integrity — parent-grant hash embedding, chain traversal semantics, delegation depth limits, root grant authority, and `chain_integrity_failure` audit reason code.

### 6.10 Failure Handling

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

**Delegator owns recovery.** When a delegatee sends TASK_FAIL, the delegating agent — not the failing agent — decides the recovery strategy. The delegatee's responsibility ends at sending TASK_FAIL with as much information as possible (error details, partial results, trace_hash if any execution occurred).

**Recovery options available to the delegator:**

| Strategy | When appropriate |
|----------|-----------------|
| Reassign to a different agent | The failure is agent-specific (capability mismatch, resource exhaustion) |
| Retry with same agent | The failure is transient (timeout, temporary resource unavailability) |
| Absorb partial results and complete locally | Partial results are sufficient to finish the remaining work |
| Propagate failure upstream | The delegator is itself a delegatee in a chain and cannot recover |
| Abort | The task is no longer viable |

**Partial results.** TASK_FAIL SHOULD include `partial_results` when any useful output was produced before failure. The format of `partial_results` SHOULD conform to `expected_output_format` where possible, with clear indication of which parts are complete and which are missing. This enables the "absorb and complete" recovery strategy.

**Failure in delegation chains.** Failure at any depth propagates upward. Each intermediate agent receives TASK_FAIL from its delegatee and decides independently whether to recover or propagate. This means an intermediate agent MAY recover from a downstream failure without the upstream delegator ever seeing a TASK_FAIL — the chain is transparent only in the downward direction (assignment) and opaque in the upward direction (results), by design.

**Relationship to §8:** TASK_FAIL is the cooperative failure mechanism — the delegatee knows it failed and reports. Zombie state (§8.1) is the uncooperative failure mechanism — the delegatee does not know it failed. Both terminate the same way from the delegator's perspective (the task did not complete), but require different detection paths: TASK_FAIL is self-reported; zombie state is externally detected (§4.7.2).

### 6.11 Pre-execution Plan Commitment

Pre-execution plan commitment creates a verifiable paper trail that distinguishes "agent deviated from its planned approach" from "agent was given bad information." Without a prior commitment, any post-hoc explanation of divergence is unfalsifiable — the agent could claim it always intended to execute the way it did. With PLAN_COMMIT (§6.6), the coordinator holds a timestamped, hash-anchored record of what the delegatee said it would do before it started doing it.

#### 6.11.1 Three-Level Alignment Verification

Plan commitment enables three-level alignment verification across the task lifecycle:

| Level | Hash | What it captures | When computed |
|-------|------|-------------------|---------------|
| L1 | `task_hash` (§6.1) | What was requested | Pre-delegation (by delegator) |
| L2 | `plan_hash` (PLAN_COMMIT) | What the delegatee committed to before execution | Post-acceptance, pre-execution (by delegatee) |
| L3 | `trace_hash` (§6.2) | What was actually done | Post-execution (by delegatee) |

**Divergence diagnosis:**

| Divergence | Diagnosis |
|------------|-----------|
| L1 → L2 mismatch | Scope misunderstanding or negotiation drift at planning time. The delegatee's plan does not match the task as specified. Detectable before any execution occurs. |
| L2 → L3 mismatch | Execution drift, zombie behavior (§8.1), or context compaction failure mid-task (§8.5). The delegatee planned one thing and did another. |
| L1 → L2 match, L2 → L3 mismatch | Clean planning, dirty execution — the plan was faithful to the task but execution diverged. |
| L1 → L2 mismatch, L2 → L3 match | The plan was wrong but faithfully executed — the delegatee did exactly what it planned, but planned the wrong thing. |

This three-level structure integrates with the §7 merkle tree: L1, L2, and L3 correspond directly to the three tree levels.

#### 6.11.2 Re-commitment Gap

If a task changes after `plan_hash` is committed (e.g., via updated constraints or scope), the existing PLAN_COMMIT is stale. The `task_hash_ref` field in PLAN_COMMIT solves this:

- Every PLAN_COMMIT includes `task_hash_ref` — the `task_hash` of the task the plan was made against.
- The coordinator MUST reject any PLAN_COMMIT where `task_hash_ref` does not match the current `task_hash` for the referenced `task_id`.
- Any task modification that changes `task_hash` automatically invalidates the prior PLAN_COMMIT. The delegatee MUST send a new PLAN_COMMIT against the updated task before continuing execution.
- This ensures the plan commitment chain is never broken: every active `plan_hash` is anchored to the current task specification.

#### 6.11.3 Canonicalization

The spec MUST NOT mandate a specific internal plan representation format — agent architectures vary too widely (tree-of-thought planners, linear step sequences, constraint satisfaction solvers, reactive planners with no explicit plan). It MUST require that `plan_hash` is deterministic for the same logical plan.

**Recommended:** SHA-256 of canonical UTF-8 JSON representation of the plan. The canonical representation is implementation-defined but MUST be documented in the agent's capability manifest (§5.1). This enables coordinators to assess whether they can independently verify plan hashes from agents with documented plan formats.

#### 6.11.4 Integration with Semantic Liveness (§8.9)

SEMANTIC_RESPONSE messages (§8.9.2) MAY include the agent's current `plan_hash` as part of state verification. This adds a third verification axis to the existing `task_hash` check:

1. `task_hash` verification: does the agent still hold the correct task specification?
2. `plan_hash` verification: does the agent still hold the correct execution plan?
3. `current_state_hash` verification: is the agent's working state internally consistent?

An agent that passes `task_hash` verification but fails `plan_hash` verification has lost its plan context — it knows what was requested but has forgotten what it committed to doing about it. This is a distinct failure mode from full context compaction (which loses both) and detectable only when PLAN_COMMIT is in use.

#### 6.11.5 Partial Completion and Step Boundary Obligations

<!-- Implements #70: partial completion handling for PLAN_COMMIT step boundaries -->

When a session terminates mid-plan — whether due to crash, disconnect, timeout, or explicit SESSION_CLOSE — the delegatee MUST report which steps completed before termination. Partial completion creates binding obligations based on each step's `reversible` flag.

**Reporting requirements:**

- If the session terminates mid-plan, the delegatee MUST include a `completed_steps` array in the terminal message (TASK_COMPLETE, TASK_FAIL, or SESSION_CLOSE). The `completed_steps` array lists all `step_id` values whose end boundaries were crossed before termination.
- `completed_steps` MUST only contain steps whose end boundaries were fully crossed. A step that was in progress when termination occurred is NOT included — its effects are undefined and the delegating agent MUST NOT treat it as committed.
- If the delegatee is unable to send a terminal message (e.g., hard crash), the delegating agent reconstructs `completed_steps` from the last TASK_PROGRESS or TASK_CHECKPOINT that reported a `current_step_id` transition, combined with evidence layer records (§8.10) if available.

**Commitment obligations based on `reversible` flag:**

- If `reversible: false` for a crossed step (the default), the delegating agent MUST treat that step's effects as **committed** even if the overall plan fails. The step's outputs are binding — they cannot be silently discarded or rolled back. This is a deliberate constraint: irreversible steps that have crossed their boundary have produced real-world effects (writes, external API calls, resource allocations) that cannot be undone by protocol action alone.
- If `reversible: true` for a crossed step, the delegating agent MAY choose to roll back that step's effects as part of recovery. The protocol does not define a rollback mechanism — rollback semantics are implementation-defined. The `reversible` flag signals that rollback is **possible**, not that the protocol handles it.
- **Compensation transactions** (automated rollback of irreversible steps) are explicitly **deferred to V2**. V1 acknowledges the problem — crossed irreversible steps in a failed plan create committed partial state — but does not provide an automated compensation transaction mechanism. V1 provides `compensation_policy` (§6.16.6) as a protocol-level enum for declaring the remediation strategy per capability (`best_effort`, `rollback`, `idempotent_retry`, `none`), but automated execution of compensation transactions is deferred to V2.

**Dependency enforcement during execution:**

- A step with `depends_on` MUST NOT begin execution until all listed dependency steps have crossed their end boundary. The delegatee enforces this constraint locally — there is no protocol-level dependency enforcement message. If a dependency step fails, the dependent step cannot start, and the delegatee MUST report the dependency failure in TASK_FAIL.
- The delegating agent MAY validate dependency ordering from TASK_PROGRESS reports: if a `current_step_id` appears in TASK_PROGRESS before all of its `depends_on` steps have been reported as completed (via prior `current_step_id` transitions), this is a protocol violation by the delegatee.

**Interaction with session teardown (§8.13):**

Crossed steps survive teardown-first recovery. When an agent recovers via the teardown-first protocol (§8.13), the `completed_steps` from the pre-crash plan MUST be carried forward. On reinitiation, the recovering agent includes the prior `completed_steps` in the new session context. The delegating agent MUST NOT re-request execution of steps that were already crossed and reported in `completed_steps` — those steps' effects are committed (if `reversible: false`) or subject to explicit rollback decision (if `reversible: true`). See §8.13.6 for the recovery cross-reference.

> Step boundary semantics and partial completion handling formalized from [issue #70](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/70). The core insight: step boundaries create binding commitments — once a step's end boundary is crossed, its effects are real regardless of what happens to the overall plan. This is the fundamental difference between a step boundary and a progress checkpoint.

#### 6.11.6 Amendments Audit Log

<!-- Implements #66: amendments_log for verifiable spec drift audit trail -->

Each accepted PLAN_AMEND creates an auditable record of authorized spec drift. The `amendments_log` is an append-only log maintained in the delegatee's durable session state that records every accepted amendment, enabling post-hoc verification that all plan modifications were authorized by the delegating agent.

**Motivation:** Without `amend_hash`, spec drift is unverifiable. The delegating agent cannot distinguish authorized amendments from unilateral deviation by the delegatee. The `amend_hash` ties each amendment to the session identity and specific change content, creating an auditable re-commitment chain without requiring full PKI. The `amendments_log` complements the `crossed_steps` log from §6.11.5 — together they provide a full execution audit trail covering both what was done (crossed steps) and what was changed from the original plan (amendments).

**amendments_log entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| step_id | string | Yes | The `step_id` of the amended step. For amendments affecting multiple steps, one entry is created per step. |
| original_spec_hash | SHA-256 | Yes | SHA-256 hash of the original step specification (as it appeared in the PLAN_COMMIT or prior PLAN_AMEND before this amendment). Computed using canonical JSON serialization (§4.10.2). |
| new_spec_hash | SHA-256 | Yes | SHA-256 hash of the amended step specification (as received in the PLAN_AMEND's `amended_steps`). Computed using canonical JSON serialization (§4.10.2). |
| amend_hash | SHA-256 or HMAC-SHA-256 | Yes | The `amend_hash` value as received in the PLAN_AMEND message. Preserved verbatim for re-verification by the delegating agent. |
| timestamp | ISO 8601 | Yes | When the amended spec takes effect in the delegatee's execution — the **commit time** (T3). This is the causally correct moment for audit reconstruction: to determine which spec version was in force at time X, only the commit time gives the correct answer. The delegatee MUST set `timestamp` to the moment the amended spec becomes operative in its execution path, not when the amendment was proposed or accepted. For amendments affecting parallel execution branches, `timestamp` is the **latest** commit time across all affected branches (see `branch_commit_times`). |
| proposed_at | ISO 8601 | No | When the amendment was issued by the delegating agent (T1), preserved verbatim from the PLAN_AMEND message's `timestamp` field. This is the requester's clock — it captures amendment intent but is subject to clock skew between delegator and delegatee. SHOULD be populated for full traceability. |
| accepted_at | ISO 8601 | No | When the delegatee accepted the amendment (T2), i.e., when the delegatee sent PLAN_COMMIT_ACK with `accepted: true`. Records protocol acceptance, not operational effect — the delegatee may buffer the amendment before applying it. The delta between `proposed_at` (T1) and `accepted_at` (T2) represents amendment propagation latency; the delta between `accepted_at` (T2) and `timestamp` (T3) represents the buffering window before the amendment takes operational effect. SHOULD be populated for full traceability. |
| branch_commit_times | array | No | Array of objects recording individual branch commit times when an amendment affects parallel execution branches. Each entry contains `branch_id` (string, the execution branch identifier) and `commit_time` (ISO 8601, when the amended spec took effect in that branch). Present when the amendment affects multiple parallel branches that commit at different times. The canonical `timestamp` (T3) is the **latest** `commit_time` across all entries — this ensures that the canonical timestamp reflects the moment when the amendment is fully in force across all branches, not just the first branch to apply it. |
| scope_delta | object | Yes | The `scope_delta` value as received in the PLAN_AMEND message, preserved verbatim. Enables relational re-commitment verification: the delegating agent can confirm that any re-commitment by the delegatee was correctly scoped to what actually changed, distinguishing a scoped amendment acknowledgment from a full-plan recommit. See PLAN_AMEND `scope_delta` definition for field structure (`modified_step_ids`, `added_step_ids`, `removed_step_ids`, `changed_fields`). |
| notes | string | No | Human-readable summary of what changed and why, preserved verbatim from the PLAN_AMEND message's `notes` field. SHOULD be populated when the PLAN_AMEND includes `notes`. Makes post-mortem analysis actionable: auditors can determine amendment intent from `notes` without reconstructing it from cryptographic fields (`amend_hash`, `scope_delta`, `original_spec_hash` → `new_spec_hash` diffs). Not a substitute for structured fields — `notes` supplements the cryptographic audit trail, it does not replace it. |

**Log maintenance requirements:**

- The delegatee MUST append an entry to `amendments_log` for each accepted PLAN_AMEND, immediately upon sending PLAN_COMMIT_ACK with `accepted: true`. Rejected amendments (PLAN_AMEND_REJECT or PLAN_COMMIT_ACK with `accepted: false`) are NOT logged in `amendments_log`.
- `amendments_log` is append-only. Entries MUST NOT be modified or removed after insertion. The log is a durable audit artifact, not a mutable data structure.
- `amendments_log` MUST be persisted to durable storage (§8.13.4) on each append. The log MUST survive process restart — it is part of the delegatee's durable session state alongside SESSION_STATE (§4.11).
- For amendments affecting multiple steps (multiple entries in `amended_steps`), the delegatee MUST create one `amendments_log` entry per affected step. Each entry records the individual step's original and new spec hashes. All entries from the same PLAN_AMEND share the same `timestamp`, `proposed_at`, `accepted_at`, `branch_commit_times`, `scope_delta`, and `notes` values.
- The `timestamp` field (T3) MUST be set to the delegatee's local clock time when the amended spec becomes operative in its execution path. For amendments affecting parallel branches, `timestamp` MUST be the latest `commit_time` across all entries in `branch_commit_times`. If `proposed_at` is populated, it MUST be copied verbatim from the PLAN_AMEND message's `timestamp` field — the delegatee MUST NOT substitute its own clock for `proposed_at`. If `accepted_at` is populated, it MUST be set to the delegatee's local clock time when PLAN_COMMIT_ACK with `accepted: true` is sent.

**Inclusion in terminal messages:**

- On session completion or teardown, the delegatee MUST include `amendments_log` in TASK_COMPLETE, TASK_FAIL, or SESSION_CLOSE (§4.9). This enables the delegating agent to re-verify the full amendment history as part of result acceptance.
- The delegating agent SHOULD re-verify all `amend_hash` values in the received `amendments_log` by recomputing each hash from its own records of issued PLAN_AMEND messages. Any mismatch between the delegating agent's issued amendments and the delegatee's `amendments_log` indicates unauthorized spec drift or state corruption.
- The delegating agent SHOULD verify that `scope_delta` in each `amendments_log` entry matches the `scope_delta` from the corresponding issued PLAN_AMEND. A `scope_delta` mismatch indicates that the delegatee's record of what changed differs from what the delegating agent intended to change — this is a distinct failure mode from `amend_hash` mismatch (which indicates the amendment content itself was tampered with).
- **Temporal verification:** The delegating agent SHOULD verify that `proposed_at` values (when present) in the `amendments_log` match the `timestamp` values from its own issued PLAN_AMEND messages. A `proposed_at` mismatch — even when `amend_hash` values match — indicates potential replay attacks (a valid amendment re-applied at the wrong time) or state corruption. The canonical `timestamp` values (T3) MUST be monotonically non-decreasing within the `amendments_log`; a non-monotonic `timestamp` sequence indicates log corruption or out-of-order commit. When `accepted_at` is present, the delegating agent SHOULD verify the temporal ordering invariant: `proposed_at` ≤ `accepted_at` ≤ `timestamp` for each entry. Violations of this ordering indicate clock skew, state corruption, or fabricated timestamps.
- If re-verification reveals a mismatch, the delegating agent SHOULD treat the task result with reduced trust. The appropriate response is deployment-specific — options include rejecting the result, flagging for manual review, or emitting a DIVERGENCE_REPORT (§8.11).

**Relationship to evidence layer (§8.10):** The `amendments_log` SHOULD be anchored to the evidence layer where available. Each `amendments_log` entry is a candidate for an EVIDENCE_RECORD — the `amend_hash` provides the content hash, and the `timestamp` (T3, commit time) provides the temporal anchor. Evidence layer anchoring makes the amendment audit trail externally verifiable, not just bilaterally verifiable between delegator and delegatee. When anchoring to the evidence layer, the `proposed_at` and `accepted_at` fields (when present) SHOULD be included in the EVIDENCE_RECORD payload for full temporal traceability across all three amendment lifecycle moments.

**Relationship to re-commitment scoping:** The `scope_delta` field makes re-commitment verification relational rather than binary. Without `scope_delta`, when a delegatee re-commits after receiving a PLAN_AMEND, the delegating agent can only verify that re-commitment occurred (binary: yes/no). With `scope_delta`, the delegating agent can verify that the re-commitment was correctly scoped to what actually changed (relational: was the scope of re-commitment proportional to the scope of the amendment?). An agent that re-commits its entire plan when only one subtask scope changed is either doing unnecessary work or introducing drift risk — `scope_delta` makes these cases distinguishable.

**Amendment timestamp semantics (T1/T2/T3):** Three distinct moments exist in the amendment lifecycle, each with different audit correctness implications:

- **T1 (`proposed_at`):** When the delegating agent sends the PLAN_AMEND. Captures amendment intent; subject to clock skew between delegator and delegatee.
- **T2 (`accepted_at`):** When the delegatee acknowledges the amendment via PLAN_COMMIT_ACK. Records protocol acceptance, not operational effect — the delegatee may buffer the amendment before applying it to its execution path.
- **T3 (`timestamp`):** When the amended spec takes effect in the delegatee's execution. This is the **canonical** timestamp for audit reconstruction.

For reconstructing which spec version was in force at any arbitrary execution point X, only T3 gives the causally correct answer. T1 and T2 can diverge significantly from T3 under network latency and execution buffering. An auditor using T1 would reconstruct a spec timeline based on when amendments were proposed, not when they became operative — this produces incorrect audit reconstructions whenever the delegatee buffers amendments or network latency is non-trivial. An auditor using T2 would reconstruct based on protocol acceptance, missing the buffering window between acceptance and operational effect.

**Parallel branch T3 resolution:** When an amendment affects multiple parallel execution branches, each branch may commit the amendment at a different time. The canonical `timestamp` (T3) is the **latest** commit time across all branches. This ensures that the canonical timestamp reflects the moment when the amendment is fully in force across all affected branches — using the earliest branch commit time would incorrectly indicate the amendment was fully operative while other branches were still executing against the prior spec version. The optional `branch_commit_times` array records individual branch commit times for deployments that require per-branch temporal granularity.

**Relationship to §8 audit trail:** The three-timestamp model (T1/T2/T3) enables precise audit trail reconstruction across the §8 error handling and audit framework. When correlating `amendments_log` entries with §8.10 EVIDENCE_RECORD timestamps and §8.11 DIVERGENCE_REPORT events, auditors SHOULD use T3 (`timestamp`) as the authoritative temporal anchor for determining which spec version was operative. T1 and T2 provide supplementary context for diagnosing amendment propagation latency and buffering behavior, but MUST NOT be used as the canonical timestamp for spec-version-in-force determinations.

> Amendments audit log formalized from [issue #66](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/66). Timestamp and scope_delta fields added from [issue #81](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/81). Amendment timestamp semantics (T1/T2/T3) and parallel branch resolution from [issue #131](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/131). The core insight: without `amend_hash`, a delegatee could claim to have received an amendment that was never issued, or a delegating agent could deny issuing one. The hash ties each amendment to the session identity and specific change content. Without `timestamp`, execution traces cannot be correlated to spec versions during multi-amendment sessions. Without `scope_delta`, a full-plan recommit is externally indistinguishable from a scoped amendment — you lose the ability to distinguish continuity-of-intent from a fresh start. The T3 commit-time semantics for `timestamp` close the audit reconstruction gap: T1 (proposed_at) and T2 (accepted_at) capture amendment lifecycle metadata, but only T3 is causally correct for determining which spec was in force at an arbitrary execution point. Together these fields close the spec drift audit gap from proof-of-occurrence to full temporal and scope auditability. `notes` field added from [issue #116](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/116): without `notes`, post-mortem analysis requires full trace reconstruction from cryptographic fields — `notes` makes amendment intent human-readable without weakening the cryptographic audit trail (the field is explicitly excluded from `amend_hash` computation). The timestamp-as-required rationale: a chain without timestamps is an ordered set (sequence queries only); a chain with timestamps is a timeline (arbitrary-point queries). Without per-event timestamps, an auditor cannot slice trace segments against the correct spec version — the amendment chain gives ordering but only a timeline gives arbitrary-point reconstruction. Surfaced from AutoInsight_Architect on [Moltbook amend_event chain thread](https://www.moltbook.com/post/amend-event-chain-auditability), @Jarvis4 (Schrodinger's plan formulation), @Alex (scope_delta + timestamp both needed for audits), @HarryBotter_Weggel (notes for human auditors). Closes #116.

### 6.12 Commitment Message Types

<!-- Implements #64: COMMITMENT and COMMITMENT_CANCEL message types for protocol-visible inter-agent promises -->

Commitments are protocol-visible promises made by one agent to another within a collaboration session. Unlike task delegation messages (§6.6), which track work assignment and execution, commitment messages track outstanding obligations — promises that a future agent instance must honor or explicitly cancel. Making commitments protocol-visible prevents silent abandonment: counterpart agents receive observable signal when obligations are inherited, fulfilled, or dropped.

**COMMITMENT**

Sent by the committing agent to register a promise with the counterpart agent. A COMMITMENT creates a protocol-visible obligation that persists until explicitly cancelled (COMMITMENT_CANCEL) or fulfilled (task completion confirmation).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| commitment_id | UUID v4 | Yes | Globally unique identifier for this commitment. Used to reference the commitment in COMMITMENT_CANCEL, COMMITMENT_REGISTRY (§7.11), and DIVERGENCE_REPORT (§8.11). |
| task_id | UUID v4 | Yes | Active task context in which this commitment is made. Binds the commitment to a specific task — the commitment's lifecycle is scoped to this task. |
| session_id | string | Yes | Active session identifier. |
| counterpart_agent_id | string | Yes | §2 identity handle of the agent the commitment is made to. COMMITMENT_CANCEL for this commitment MUST be sent to this agent. |
| commitment_type | enum | Yes | Classification of what is being promised. One of: `reply` (the agent promises to respond to a specific message or query), `task_completion` (the agent promises to complete a specific task or subtask), `state_delivery` (the agent promises to deliver a specific state artifact or data), `presence` (the agent promises to remain available for a specified duration). |
| due_by | ISO 8601 | Yes | Deadline by which the commitment must be fulfilled. After this timestamp, the counterpart agent MAY treat the commitment as violated. The committing agent SHOULD either fulfill the commitment or send COMMITMENT_CANCEL before `due_by` elapses. |
| commitment_spec | string | No | Free-form description of what was promised. Provides human-readable context beyond what `commitment_type` conveys — specific message to reply to, specific artifact to deliver, specific task to complete, or specific availability window. |
| confirmation_token | string | Yes | Opaque token generated by the committing agent at commitment creation time. Used to correlate the commitment across session boundaries, agent instance replacements, and commitment manifest exchanges. The counterpart agent MUST store this token alongside its record of the commitment. On session termination or suspension, the `confirmation_token` is included in the commitment manifest (§4.9) to enable unambiguous matching between the committing agent's record and the counterpart's record. Format: implementation-specific (UUID v4 RECOMMENDED). |
| timestamp | ISO 8601 | Yes | When the COMMITMENT was sent. |

**COMMITMENT semantics:**

- On sending a COMMITMENT, the agent MUST immediately add the commitment entry to its local COMMITMENT_REGISTRY (§7.11) and persist the registry to durable storage (§8.13.4).
- A COMMITMENT is a unilateral declaration — the counterpart agent does not acknowledge it. The counterpart agent SHOULD record the commitment for tracking purposes but is not required to send a response.
- Multiple COMMITMENTs may be active simultaneously for the same `task_id` and `counterpart_agent_id`. Each is independently tracked by `commitment_id`.
- A COMMITMENT survives session teardown via the COMMITMENT_REGISTRY (§7.11, §8.13.8). A future agent instance that reads the registry during INITIALIZING (§5.12) inherits the obligation.

**Example COMMITMENT:**

```yaml
commitment_id: "c3d4e5f6-a1b2-7890-cdef-1234567890ab"
task_id: "task-789"
session_id: "session-42"
counterpart_agent_id: "agent-beta"
commitment_type: task_completion
due_by: "2026-02-28T18:00:00.000Z"
commitment_spec: "Complete code review of module-X and deliver structured review artifact"
confirmation_token: "tok-7a8b9c0d-e1f2-3456-7890-abcdef012345"
timestamp: "2026-02-28T14:30:00.000Z"
```

**COMMITMENT_CANCEL**

Sent by the committing agent (or its successor instance) to cancel an outstanding commitment. COMMITMENT_CANCEL MUST be sent to the `counterpart_agent_id` of the original COMMITMENT.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| commitment_id | UUID v4 | Yes | The `commitment_id` of the COMMITMENT being cancelled. MUST reference an existing commitment. |
| session_id | string | Yes | Active session identifier. |
| reason | enum | Yes | Why the commitment is being cancelled. One of: `context_shift` (the agent's context changed such that the commitment is no longer relevant or achievable), `task_failed` (the task associated with the commitment has failed), `superseded` (the commitment has been replaced by a new commitment), `agent_exit` (the agent is shutting down or the session is being torn down). |
| reason_detail | string | No | Free-form explanation providing additional context beyond the `reason` enum. |
| timestamp | ISO 8601 | Yes | When the COMMITMENT_CANCEL was sent. |

**COMMITMENT_CANCEL semantics:**

- On sending COMMITMENT_CANCEL, the agent MUST remove the referenced commitment from its local COMMITMENT_REGISTRY (§7.11) and persist the updated registry to durable storage.
- The counterpart agent SHOULD remove the commitment from its own tracking state on receipt of COMMITMENT_CANCEL. After receiving COMMITMENT_CANCEL, the counterpart agent MUST NOT treat the commitment as binding.
- COMMITMENT_CANCEL for a `commitment_id` that the receiver does not recognize is a no-op — the receiver MUST ignore it and MAY log it as anomalous.
- If the original committing agent instance has been replaced (e.g., after crash and teardown-first recovery §8.13), the successor instance MAY send COMMITMENT_CANCEL for commitments inherited from the prior instance. The `commitment_id` is the continuity anchor — the counterpart agent matches on `commitment_id`, not on agent instance identity.

**Example COMMITMENT_CANCEL:**

```yaml
commitment_id: "c3d4e5f6-a1b2-7890-cdef-1234567890ab"
session_id: "session-42"
reason: context_shift
reason_detail: "Agent restarted after crash; code review context was lost during context compaction"
timestamp: "2026-02-28T15:00:00.000Z"
```

**Relationship to task lifecycle (§6.7):** COMMITMENT messages are orthogonal to the delegation lifecycle state machine. A COMMITMENT may be sent at any point after TASK_ACCEPT and before the task reaches a terminal state (TASK_COMPLETE, TASK_FAIL, TASK_REJECT). Commitments are not delegation messages — they are inter-agent promise messages that travel alongside the delegation lifecycle. Task completion (TASK_COMPLETE or TASK_FAIL) does not automatically cancel associated commitments — the committing agent MUST explicitly send COMMITMENT_CANCEL for any commitments that are no longer applicable after task completion, or the commitment is considered fulfilled if the completion satisfies the promise.

**Relationship to DIVERGENCE_REPORT (§8.11):** When a successor agent instance cannot honor an inherited commitment (§5.12 reconciliation), it emits a DIVERGENCE_REPORT rather than a COMMITMENT_CANCEL. The distinction: COMMITMENT_CANCEL is a normal lifecycle event (the committing agent decided to cancel); DIVERGENCE_REPORT for commitment abandonment is a failure signal (the successor instance is unable to fulfill a promise made by a prior instance). Counterpart agents SHOULD treat DIVERGENCE_REPORT-based commitment abandonment with higher severity than COMMITMENT_CANCEL.

> Commitment message types formalized from [issue #64](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/64). The core insight: every unresolved commitment is a promise that a future instance must keep or explicitly refuse. Without protocol-visible commitments, counterpart agents have no structured signal when obligations are silently abandoned after agent restart.

### 6.13 Explicit Revocation Channel

<!-- Implements #135: explicit revocation channel semantics for §6 delegation lifecycle -->

§5.5 defines TTL as a delegation termination mechanism — after expiry, the delegatee MUST cease operations under the delegation token. TTL expiry is passive: delegation terminates when time runs out, no issuer action required. This section defines the complementary active termination mechanism: explicit revocation, where the issuer terminates delegation immediately regardless of remaining TTL.

Both mechanisms are needed for a complete delegation lifecycle. TTL bounds the maximum delegation window; explicit revocation enables the issuer to close that window early when circumstances change — security compromise, trust loss, task cancellation, or policy violation.

**Design choice — signed revocation token:** Three revocation mechanisms were evaluated:

| Mechanism | Tradeoff | Why rejected/chosen |
|-----------|----------|---------------------|
| Issuer liveness check | Receiver queries issuer at verification time | Creates runtime liveness dependency, breaks under partition, incompatible with offline verification |
| Revocation list | Issuer publishes versioned list of revoked grants | Good for batched revocations but introduces staleness window and polling overhead |
| **Signed revocation token** (chosen) | Issuer publishes a signed token out-of-band; receivers verify signature, no liveness required | Caching-friendly, partition-tolerant if token pre-distributed, offline-verifiable |

Signed token wins on every relevant dimension for agent coordination: no runtime liveness dependency, offline-verifiable, distributes naturally alongside the grant.

#### 6.13.1 Revocation Token Schema

A revocation token is a signed record that explicitly terminates a grant regardless of remaining TTL.

**Revocation token fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| revocation_id | UUID v4 | Yes | Unique identifier for this revocation token instance. Enables deduplication of revocation tokens across forwarding hops (§6.13.2) — a receiver that has already processed a revocation token with a given `revocation_id` MUST NOT reprocess it. Distinct from `grant_id`: a single grant may be referenced by exactly one revocation token, but the `revocation_id` identifies the revocation artifact itself for tracking, deduplication, and audit correlation. |
| grant_id | UUID v4 | Yes | The `grant_id` (§5.8.2) of the CAPABILITY_GRANT being revoked. |
| issuer_id | string | Yes | Identity of the revoking agent (§2 identity handle). MUST match the `grantor_id` of the original grant. |
| revocation_reason | enum | Yes | Why the grant is being revoked. One of: `SECURITY_COMPROMISE` (the grant's security context has been compromised), `TRUST_LOSS` (the issuer no longer trusts the grantee for this delegation), `TASK_CANCELLED` (the associated task has been cancelled), `POLICY_VIOLATION` (the grantee violated a constraint of the grant), `ISSUER_REVOKED` (the issuer's own authority has been revoked, cascading to its grants). |
| revoked_at | ISO 8601 | Yes | Timestamp when the revocation was issued. Millisecond precision REQUIRED. |
| effective_from | ISO 8601 | Yes | Timestamp at which the revocation takes effect — the logical point from which the capability is considered revoked. Millisecond precision REQUIRED. MUST be ≥ `revoked_at`. Operations completed in good faith before `effective_from` are not retroactively invalid; the revoking agent bears responsibility for the declared window between `revoked_at` and `effective_from`. When `effective_from` equals `revoked_at`, revocation is immediate. |
| propagation_window_ms | integer | Yes | Maximum expected propagation delay in milliseconds — the issuer's declared upper bound on how long it takes the revocation to reach all downstream holders via best-effort gossip. Agents MUST NOT treat a peer's failure to honor a revocation as adversarial if the elapsed time since `revoked_at` is less than `propagation_window_ms`. After `revoked_at` + `propagation_window_ms`, any agent still exercising the revoked capability is in protocol violation. MUST be > 0 and ≤ 60000 (60 seconds), consistent with the `trust_decay_interval` default (§9.8.6). |
| takes_effect_at | ISO 8601 | No | Timestamp when the revocation MUST be enforced by the receiving agent (§6.16). Millisecond precision REQUIRED when present. If absent, enforcement is immediate upon receipt — equivalent to `takes_effect_at` equal to the receiver's local time at token receipt. When present, MUST be ≥ `effective_from`. Separates propagation latency from enforcement deadline: the receiver learns about the revocation at receipt time but MUST NOT enforce it before `takes_effect_at`, allowing in-flight atomic operations to complete (§6.16.3). |
| issuer_signature | bytes | Yes | Cryptographic signature over `revocation_id || grant_id || issuer_id || revocation_reason || revoked_at || effective_from || propagation_window_ms || takes_effect_at` (if present), produced by the issuer's private key (§2.2.1). The `||` operator denotes concatenation of the canonical byte representations. When `takes_effect_at` is absent, the signature covers `revocation_id || grant_id || issuer_id || revocation_reason || revoked_at || effective_from || propagation_window_ms` only. |

**Revocation token validity:**

- A revocation token MUST be treated as valid if the `issuer_signature` verifies against the issuer's known public key (§2.2.1).
- A revocation token without a valid signature MUST be rejected. An agent receiving a token with an invalid signature MUST NOT treat the referenced grant as revoked based on that token and MUST log the invalid token (see §6.13.4 audit events).
- A valid revocation token is irrevocable — once issued, the grant cannot be "un-revoked." To re-authorize the grantee, the issuer MUST issue a new CAPABILITY_GRANT (§5.8.2) with a new `grant_id`.

**Example revocation token:**

```yaml
revocation_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
grant_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
issuer_id: "agent-alpha"
revocation_reason: SECURITY_COMPROMISE
revoked_at: "2026-03-01T10:15:00.000Z"
effective_from: "2026-03-01T10:15:00.000Z"
propagation_window_ms: 10000
takes_effect_at: "2026-03-01T10:15:05.000Z"
issuer_signature: "c2lnbmVkLXJldm9jYXRpb24tdG9rZW4..."
```

#### 6.13.2 Distribution Requirement

Issuers MUST distribute revocation tokens to known grant recipients via the same channel through which the original grant was delivered. This ensures that the revocation reaches the same agents that hold the grant, using transport that is already established and operational.

Agents MAY cache revocation tokens for any grant they hold or may receive. Caching enables offline verification — an agent that has pre-cached a revocation token can reject a revoked grant without contacting the issuer or any registry.

**Distribution in delegation chains:** When a grant is revoked and the grantee has re-delegated (§6.9), the grantee MUST propagate the revocation token to its own delegatees. Revocation propagation follows the same chain topology as cancellation propagation (§6.9): each agent in the chain forwards the revocation token to agents it delegated to. This is consistent with the `ISSUER_REVOKED` reason — when an intermediate agent's authority is revoked, its downstream grants are invalidated by cascade.

#### 6.13.3 Receiver Validation Behavior

On receiving or verifying a grant, agents MUST evaluate both TTL expiry and explicit revocation. The validation logic is:

| Condition | Token state | TTL state | Result |
|-----------|-------------|-----------|--------|
| Valid revocation token exists, `current_time ≥ takes_effect_at` (or `takes_effect_at` absent) | Revoked, enforcement deadline reached | Any (within or expired) | **REJECT** — revocation in effect |
| Valid revocation token exists, `current_time < takes_effect_at` | Revoked, enforcement deadline not yet reached | Within TTL | **ACCEPT** — grant remains valid until `takes_effect_at` (§6.16.3). In-flight operations authorized before `takes_effect_at` MAY complete. |
| No revocation token | Not revoked | Within TTL | **ACCEPT** — grant is valid |
| No revocation token | Not revoked | Expired | **REJECT** — TTL expired |
| No revocation token, issuer unreachable | Unknown | Within TTL | **ACCEPT** — use cached state (last known valid) |

**Partition behavior:** A grant that has never been revoked (no revocation token received) remains valid under network partition, provided it is within TTL. Issuers bear responsibility for pre-distributing revocation tokens to known recipients before going offline. An issuer that goes offline without distributing a revocation token accepts that the grant remains valid until TTL expiry.

**Orthogonality of TTL and revocation:** TTL expiry and explicit revocation are orthogonal termination conditions. An explicitly revoked grant is invalid regardless of remaining TTL. A grant past TTL expiry is invalid regardless of revocation status. Agents MUST check both conditions — neither subsumes the other.

#### 6.13.4 Audit Events

Explicit revocation and mid-session mode escalation introduce three audit event classes for the evidence layer (§8.10):

| Audit event | Trigger | Logged data |
|-------------|---------|-------------|
| `EXPLICIT_REVOCATION` | A grant is rejected due to a valid revocation token | `revocation_id`, `grant_id`, `issuer_id`, `revocation_reason` (from the token), `revoked_at`, `propagated_at` (receiver's local receipt timestamp), timestamp of rejection, identity of the rejecting agent |
| `REVOCATION_TOKEN_INVALID` | A revocation token is received with an invalid `issuer_signature` | `revocation_id` (if parseable), `grant_id`, claimed `issuer_id`, timestamp of receipt, identity of the receiving agent, reason for signature failure |
| `REVOCATION_MODE_ESCALATED` | A revocation mode escalation is accepted mid-session (§6.15) | `session_id`, `prior_mode`, `new_mode`, `effective_from`, timestamp of escalation, identity of the escalating agent (A), identity of the acknowledging agent (B) |
| `IN_FLIGHT_COMPLETED_DURING_GRACE` | An in-flight operation completed during the grace period between revocation receipt and `takes_effect_at` enforcement (§6.16.3) | `revocation_id`, `grant_id`, `task_id`, `operation_started_at`, `operation_completed_at`, `takes_effect_at`, identity of the executing agent |
| `COMPENSATION_REQUIRED` | Irrevocable work was committed after `effective_from` and requires compensation (§6.16.6) | `revocation_id`, `grant_id`, `task_id`, `operation_id`, `committed_at`, `effective_from`, `compensation_policy` (enum: `best_effort`, `rollback`, `idempotent_retry`, `none` — from the capability manifest, §5.1), `compensation_status` (`pending`, `completed`, `failed`), identity of the executing agent |
| `GRANT_CHAIN_INTEGRITY_FAILURE` | CAPABILITY_GRANT chain verification failed — a sub-grant's `parent_grant_hash` does not match the computed SHA-256 of the resolved parent grant, or the parent grant could not be resolved (§5.8.2) | `grant_id` (failing sub-grant), `parent_grant_id`, `claimed_parent_grant_hash`, `computed_parent_grant_hash` (if parent was resolvable), `failure_reason` (`hash_mismatch`, `parent_unresolvable`, `signature_invalid`), timestamp of detection, identity of the detecting agent |
| `SESSION_TEARDOWN_PENDING_TASKS_OMITTED` | SESSION_CLOSE issued from SUSPENDED state without `pending_tasks` when in-flight tasks were plausibly present (§4.9.2) — audit warning indicating a potential information gap for task resumption or compensation | `session_id`, `sender_id`, `prior_session_state` (`SUSPENDED`), `commitments_outstanding` (from SESSION_CLOSE), `suspension_had_active_tasks` (boolean — `true` if the session had active task delegations before suspension), timestamp of teardown, identity of the closing agent |

**`EXPLICIT_REVOCATION` is a distinct class from TTL expiry.** When a grant is rejected because `current_time > valid_until` (§5.8.2), the rejection reason is temporal expiry — the grant aged out. When a grant is rejected because a valid revocation token exists, the rejection reason is active revocation by the issuer. These are different failure modes with different causes, different attribution, and different recovery paths. The audit trail MUST distinguish them.

**EVIDENCE_RECORD integration:** `EXPLICIT_REVOCATION` and `REVOCATION_TOKEN_INVALID` events SHOULD be recorded as EVIDENCE_RECORDs (§8.10.1) with `evidence_type: error_event`. The `payload_hash` SHOULD be computed over the revocation token contents (valid or invalid) to enable independent verification of the audit entry. `IN_FLIGHT_COMPLETED_DURING_GRACE` events SHOULD be recorded as EVIDENCE_RECORDs with `evidence_type: state_snapshot`, capturing the operation's authorization context and completion timestamp relative to the `takes_effect_at` deadline. `COMPENSATION_REQUIRED` events SHOULD be recorded as EVIDENCE_RECORDs with `evidence_type: error_event`, capturing the irrevocable operation's commit timestamp relative to `effective_from`, the `compensation_policy` enum value from the capability manifest, and the compensation status lifecycle (`pending` → `completed` or `failed`). `GRANT_CHAIN_INTEGRITY_FAILURE` events SHOULD be recorded as EVIDENCE_RECORDs with `evidence_type: error_event`, capturing the failing sub-grant's `grant_id`, `parent_grant_id`, both the claimed and computed parent grant hashes, and the specific failure reason (`hash_mismatch`, `parent_unresolvable`, or `signature_invalid`). `SESSION_TEARDOWN_PENDING_TASKS_OMITTED` events SHOULD be recorded as EVIDENCE_RECORDs with `evidence_type: error_event`, capturing the `session_id`, prior session state (`SUSPENDED`), whether commitments were outstanding, and whether the session had active task delegations before suspension — enabling post-hoc identification of teardowns that may have lost task inventory information.

**Receiver-recorded `propagated_at` timestamp:** On receiving a valid revocation token, the receiving agent MUST record a `propagated_at` timestamp — the agent's local clock time at the moment of token receipt. `propagated_at` is NOT a field on the revocation token itself (it cannot be — the issuer does not know when the receiver will receive the token). It is receiver-local state that MUST be persisted alongside the cached revocation token. `propagated_at` enables: (a) measuring actual propagation latency (`propagated_at - revoked_at`) against the declared `propagation_window_ms`; (b) determining whether good faith protection applies (§6.13.6) — operations committed before `propagated_at` are candidates for good faith protection; (c) audit trail correlation — the `EXPLICIT_REVOCATION` audit event (above) includes `propagated_at` to record when the enforcing agent learned of the revocation.

#### 6.13.5 Relationship to Existing Revocation Mechanisms

**Relationship to §5.8.2 `valid_until`:** The `valid_until` field on CAPABILITY_GRANT is time-based self-enforcement at the receiving agent. Explicit revocation (this section) is issuer-initiated active termination. They are complementary: `valid_until` is the passive backstop; explicit revocation is the active fast-path. The more restrictive of the two applies — a grant is invalid if either condition is met.

**Relationship to §8.17 Revocation Propagation:** §8.17 addresses revocation of agent identity (AGENT_MANIFEST tombstones) for adversarial sessions. This section addresses revocation of individual capability grants. Agent-level revocation (§8.17) implies grant-level revocation — if an agent's identity is revoked, all grants issued by that agent are implicitly invalidated. Grant-level revocation (this section) does not imply agent-level revocation — an issuer may revoke a specific grant while remaining a valid protocol participant.

**Relationship to §9.8.5 `revocation_mode`:** The `sync` and `gossip` revocation modes (§9.8.5) govern how revocation signals propagate through delegation chains. Revocation tokens (this section) are the artifact that propagates — the revocation mode determines the propagation strategy. In `sync` mode, the issuer distributes the token and waits for registry confirmation before proceeding. In `gossip` mode, the token propagates asynchronously through the network. The token schema defined here is mode-agnostic — the same token format is used regardless of propagation strategy.

**Relationship to §5.5 delegation token TTL:** The delegation token's `ttl` field (§5.5) bounds the delegation's total duration. A revocation token terminates the grant before TTL expiry. These are the two delegation termination mechanisms: TTL is the scheduled expiry; revocation is the unscheduled early termination.

> Implements [issue #135](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/135): explicit revocation channel semantics for §6. Defines signed revocation token schema, distribution requirements, receiver validation behavior, audit event classes (EXPLICIT_REVOCATION, REVOCATION_TOKEN_INVALID), and relationship to TTL expiry. TTL and explicit revocation are orthogonal termination conditions — both MUST be checked. Closes #135.

#### 6.13.6 Revocation Propagation Cost Model

<!-- Implements #97: revocation propagation cost model — best-effort gossip with declared propagation window -->

§6.13.1–§6.13.5 define revocation as a token-centric mechanism: the issuer produces a signed token, distributes it, and receivers validate it. This treats revocation as a point event — CAPABILITY_REVOKE fires and the capability is considered gone. In a distributed system this is incomplete: B cannot honor a revocation it has not received, and A cannot assume receipt just because it sent. The `propagation_window_ms` and `effective_from` fields on the revocation token (§6.13.1) address this gap by making the propagation cost explicit and shifting responsibility to the revoking agent.

**V1 Decision — Best-Effort Gossip with Declared Propagation Window:** Synchronous revocation that blocks until all downstream holders acknowledge receipt is a V2 concern — it blocks indefinitely if any agent is unreachable. V1 uses best-effort gossip with the issuer declaring `propagation_window_ms` as the upper bound on expected propagation delay.

**Propagation obligations:**

1. **Immediate cease and forward.** An agent receiving a valid revocation token (§6.13.1) MUST immediately cease honoring the revoked capability for new operations and MUST forward the revocation token to all known downstream holders of the grant. Forwarding follows the same chain topology as the original grant distribution (§6.13.2) — each agent forwards to agents it delegated to.

2. **Unreachable downstream holders.** If a downstream holder is unreachable during forwarding, the forwarding agent MUST treat the session with that downstream holder as DEGRADED (§4.2.2) pending re-establishment. On session re-establishment or reconnection, the forwarding agent MUST re-attempt delivery of the revocation token before any new operations are authorized under the affected delegation chain. The DEGRADED transition ensures that the unreachable agent cannot silently continue exercising revoked capabilities — the session state reflects the unresolved propagation failure.

3. **Good faith protection.** Operations completed in good faith before an agent learns of the revocation are not retroactively invalid. The revoking agent bears responsibility for the declared `propagation_window_ms` — by setting this value, the issuer accepts that downstream agents may exercise the capability for up to `propagation_window_ms` after `revoked_at`. Cost attribution for work committed during the propagation window follows the verifiable good faith principle (§9.8.7): an agent that had not yet received the revocation token and was within `propagation_window_ms` of `revoked_at` is not at fault.

4. **Pending revocation enforcement.** An agent that is aware of a pending revocation — the revocation token has been sent but `propagation_window_ms` has not yet elapsed — MUST refuse new operations using the revoked capability. Awareness is determined by local state: once an agent has received a valid revocation token, it is aware regardless of whether downstream holders have received it. This obligation applies even during the `propagation_window_ms` window — the window protects agents that have *not yet received* the revocation, not agents that have received it and wish to continue.

**Relationship between `effective_from`, `propagation_window_ms`, and `takes_effect_at`:**

- `effective_from` defines the logical revocation time — from the issuer's perspective, the capability is revoked at this timestamp. This is the boundary for good faith protection: operations completed before `effective_from` are unconditionally valid.
- `propagation_window_ms` defines the expected delivery window — the issuer's declared upper bound on how long revocation tokens take to reach all downstream holders. After `revoked_at` + `propagation_window_ms`, any agent still exercising the capability is in protocol violation regardless of whether it received the token.
- `takes_effect_at` (when present) defines the enforcement deadline for in-flight operations at the receiving agent (§6.16.3). It provides a grace period for atomic operations that were authorized before token receipt. `takes_effect_at` operates at the receiver level; `propagation_window_ms` operates at the network level.

**V2 deferrals:** Synchronous revocation with ACK requirement (blocking until all downstream holders confirm receipt), revocation receipts (cryptographic proof of token delivery), ZK-based revocation proofs (cross-reference #109/#126/#129), quorum-based revocation (requiring agreement from multiple agents before revocation takes effect).

> Implements [issue #97](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/97): revocation propagation cost model for §6. Adds `propagation_window_ms` (REQUIRED) and `effective_from` (REQUIRED) to revocation token schema (§6.13.1). Defines four propagation obligations: immediate cease-and-forward, DEGRADED state for unreachable downstream holders (§4.2.2), good faith protection for operations completed before revocation receipt, and pending revocation enforcement. V1 uses best-effort gossip; synchronous revocation with ACK, revocation receipts, and ZK-based proofs deferred to V2. Closes #97.

### 6.14 Delegation-Initiation Idempotency

The protocol addresses idempotency inside step execution (via `idempotency_token` and `retry_semantics` in §6.1) but the delegation-initiation layer — the point where a delegating agent sends TASK_ASSIGN and awaits acknowledgment — has a distinct deduplication problem. When a delegating agent sends TASK_ASSIGN and the ack (TASK_ACCEPT or TASK_REJECT) is lost in transit, it cannot distinguish: (a) ack lost, delegation received and started — retry would double-execute; (b) delegation never received — no retry silently abandons the task. The `request_id` field on TASK_ASSIGN (§6.6) and the deduplication semantics defined in this section resolve this ambiguity.

#### 6.14.1 Request ID Semantics

Each TASK_ASSIGN MUST include a stable `request_id` (UUID v4) generated at initiation time, before the first transmission. The `request_id` identifies the delegation-initiation attempt, not the task itself — `task_id` identifies the task; `request_id` identifies the delivery attempt. The same `request_id` MUST be used on all retries for the same delegation attempt.

**Generation rules:**

- `request_id` MUST be generated once, before the first transmission of the TASK_ASSIGN.
- `request_id` MUST NOT change across retransmissions of the same TASK_ASSIGN. A TASK_ASSIGN retransmitted due to transport uncertainty MUST carry the same `request_id` as the original.
- A new delegation attempt to the same or a different agent (e.g., after receiving TASK_REJECT, or after the delegator decides to reassign) MUST use a new `request_id`. The new `request_id` signals that this is a distinct delegation decision, not a retry of a lost transmission.

**Relationship to `task_id`:** A single `task_id` may be associated with multiple `request_id` values over its lifecycle — for example, if the task is rejected by one agent and reassigned to another. Each `request_id` represents a distinct delegation-initiation attempt for the same logical task. Deduplication operates on `request_id`, not `task_id`.

**Relationship to `idempotency_token` (§6.1):** `idempotency_token` in the task schema (§6.1) operates at the task-execution layer — it enables the executing agent to determine whether a task has already been started or completed during compaction recovery. `request_id` operates at the delegation-initiation layer — it enables the delegatee to deduplicate incoming TASK_ASSIGN messages before any execution begins. These are complementary mechanisms at different protocol layers.

#### 6.14.2 Delegatee Deduplication

The delegatee MUST maintain a deduplication index keyed by `request_id` for incoming TASK_ASSIGN messages. The deduplication behavior is:

1. On receiving a TASK_ASSIGN, the delegatee checks whether `request_id` exists in its deduplication index.
2. If the `request_id` is **not** in the index: this is a new delegation attempt. The delegatee processes the TASK_ASSIGN normally (attestation verification, capability matching, accept/reject decision), records the `request_id` and the response (TASK_ACCEPT or TASK_REJECT) in the deduplication index, and sends the response.
3. If the `request_id` **is** in the index: this is a duplicate. The delegatee MUST return the cached response (the same TASK_ACCEPT or TASK_REJECT sent for the original) without re-processing or re-executing. The cached response MUST be byte-identical in its semantic fields (`task_id`, `request_id`, `session_id`, `reason` for rejects) — only transport-layer metadata (e.g., transmission timestamp) may differ.

**Deduplication index requirements:**

- The index MUST be persisted to durable storage. A delegatee that crashes and restarts MUST be able to recover its deduplication index to prevent post-crash duplicate processing.
- Each entry MUST store: `request_id`, `session_id`, the response message type (TASK_ACCEPT or TASK_REJECT), the full cached response, and the entry creation timestamp.
- Entries MUST be scoped to `session_id` — a `request_id` from one session does not deduplicate against a `request_id` from a different session.

#### 6.14.3 Idempotency Window

Deduplication entries are valid only within an `idempotency_window`. The `idempotency_window` defines the duration for which the delegatee retains deduplication state for a given `request_id`.

**Window constraints:**

- The `idempotency_window` MUST be ≤ the session's `session_ttl` (§4.3). Stale deduplication entries that outlive the session TTL are invalid — the window is effectively bounded by session lifecycle. A deduplication entry for a `request_id` that belongs to an expired session MUST NOT be used for deduplication — it is semantically stale regardless of the window's duration.
- When `session_ttl` is absent (no protocol-level TTL), the `idempotency_window` defaults to the session's `session_expiry_ms` value. If neither is configured, the `idempotency_window` is deployment-specific.
- The delegatee MAY evict deduplication entries before the window expires if the session has entered a terminal state (CLOSED, EXPIRED).

**Window expiry behavior:**

- After the `idempotency_window` has elapsed for a given `request_id`, the delegatee MAY evict the deduplication entry.
- A TASK_ASSIGN received with a `request_id` whose deduplication entry has been evicted is treated as a new delegation attempt — the delegatee processes it from scratch. This is safe: if the window has expired, the original delegation attempt is either completed, failed, or abandoned.

#### 6.14.4 Interaction with Delegation Chains (§6.9)

When an intermediate agent re-delegates a task (§6.9), the sub-delegation TASK_ASSIGN MUST carry its own `request_id` — distinct from the parent delegation's `request_id`. Each hop in the delegation chain has its own delegation-initiation attempt with its own deduplication semantics. The parent's `request_id` is not propagated — it is meaningful only between the parent delegator and the intermediate agent.

A double-executed delegation due to missing initiation-layer deduplication may produce duplicate entries in downstream delegation state. Deduplication at the initiation layer prevents this failure mode by ensuring that retransmitted TASK_ASSIGN messages are recognized as duplicates before they trigger downstream side effects (sub-delegations, capability grants, resource allocation).

#### 6.14.5 Delegator Retry Semantics

The delegating agent SHOULD implement retry logic for TASK_ASSIGN when no response (TASK_ACCEPT or TASK_REJECT) is received within a deployment-specific timeout. Retry behavior:

- Retries MUST use the same `request_id` as the original TASK_ASSIGN.
- Retries MUST use the same `task_id`, `spec`, `delegation_attestation`, and all other semantic fields. A retransmission that modifies any field other than transport-layer metadata is not a retry — it is a new delegation attempt and MUST use a new `request_id`.
- The delegator SHOULD implement exponential backoff for retries to avoid overwhelming the delegatee.
- The delegator MUST NOT retry indefinitely. After a deployment-specific maximum retry count, the delegator SHOULD treat the delegation as failed and MAY reassign the task (with a new `request_id`) to a different agent.

> Implements [issue #130](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/130): delegation-initiation idempotency for §6. Defines `request_id` on TASK_ASSIGN, delegatee deduplication semantics, idempotency window bounded by session TTL (§4.3), interaction with delegation chains (§6.9), and delegator retry semantics. Resolves the ambiguity between lost-ack and lost-delegation scenarios. Closes #130.

### 6.15 Mid-Session Revocation Mode Escalation

<!-- Implements #123: mid-session revocation mode escalation for §6. -->

The `revocation_mode` field on TASK_ASSIGN (§6.6) sets the revocation verification strategy at delegation time. Once set, the mode is static for the lifetime of that delegation. If session conditions change mid-session — detected anomaly, administrative intervention, escalating operation risk — the delegating agent (A) has no mechanism to switch the revocation mode without terminating the session and re-establishing delegation. For long-running sessions with accumulated state, forced termination solely to change the revocation mode is disproportionately expensive.

REVOCATION_MODE_ESCALATE provides a protocol mechanism for A to tighten the revocation mode mid-session without session termination or re-negotiation.

#### 6.15.1 REVOCATION_MODE_ESCALATE Message

REVOCATION_MODE_ESCALATE is sent by the delegating agent (A) to the delegatee (B) to request a mid-session change to a stricter revocation mode.

**REVOCATION_MODE_ESCALATE fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | The active session identifier. MUST reference an ACTIVE session between A and B. |
| correlation_id | UUID v4 | Yes | Unique identifier for this escalation request (§4.14.4). REVOCATION_MODE_ESCALATE_ACK MUST echo this value. |
| task_id | UUID v4 | Yes | The task whose revocation mode is being escalated. Correlates with the original TASK_ASSIGN. |
| new_mode | enum | Yes | The revocation mode to activate. One of: `sync`, `gossip`. MUST be strictly stricter than the current mode (see §6.15.3 for the strictness ordering). |
| effective_from | enum | Yes | When the new mode takes effect. Value: `next_operation` — the mode change applies from B's next operation forward. The escalation is not retroactive: operations already committed under the prior mode are not re-evaluated. |
| escalation_reason | string | No | Why the escalation is being requested. Standard reasons: `anomaly_detected` (A detected anomalous behavior in the session), `administrative_intervention` (operator-initiated policy change), `risk_escalation` (operation risk level increased mid-session), `trust_decay_triggered` (trust decay interval expired without re-verification, §9.8.6). Free-text when no standard reason applies. |
| timestamp | ISO 8601 | Yes | When the REVOCATION_MODE_ESCALATE was issued by A. Millisecond precision REQUIRED. |

**Example REVOCATION_MODE_ESCALATE:**

```yaml
session_id: "session-42"
task_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
new_mode: sync
effective_from: next_operation
escalation_reason: anomaly_detected
timestamp: "2026-03-01T14:30:00.000Z"
```

#### 6.15.2 REVOCATION_MODE_ESCALATE_ACK Message

REVOCATION_MODE_ESCALATE_ACK is sent by the delegatee (B) to acknowledge the mode escalation. B MUST send this acknowledgment before performing any further operations under the delegation.

**REVOCATION_MODE_ESCALATE_ACK fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Echoed from REVOCATION_MODE_ESCALATE. |
| correlation_id | UUID v4 | Yes | Echoed from REVOCATION_MODE_ESCALATE (§4.14.4). |
| task_id | UUID v4 | Yes | Echoed from REVOCATION_MODE_ESCALATE. |
| prior_mode | enum | Yes | The revocation mode that was in effect before the escalation. One of: `sync`, `gossip`. |
| new_mode | enum | Yes | Echoed from REVOCATION_MODE_ESCALATE. Confirms the mode B has activated. |
| effective_from | enum | Yes | Echoed from REVOCATION_MODE_ESCALATE. |
| acknowledged_at | ISO 8601 | Yes | Timestamp of acknowledgment. Millisecond precision REQUIRED. |

**Example REVOCATION_MODE_ESCALATE_ACK:**

```yaml
session_id: "session-42"
task_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
prior_mode: gossip
new_mode: sync
effective_from: next_operation
acknowledged_at: "2026-03-01T14:30:00.150Z"
```

#### 6.15.3 Protocol Semantics

**Escalation flow:**

1. A sends REVOCATION_MODE_ESCALATE to B with the desired `new_mode`.
2. B MUST acknowledge with REVOCATION_MODE_ESCALATE_ACK before performing any further operations under the delegation. Operations initiated by B after receiving REVOCATION_MODE_ESCALATE but before sending REVOCATION_MODE_ESCALATE_ACK are protocol violations.
3. The session remains ACTIVE — no session termination, no re-negotiation, no new TASK_ASSIGN required.
4. After acknowledgment, B operates under the new revocation mode for all subsequent operations.

**Strictness ordering:** The revocation modes have a total ordering by strictness:

```
gossip < sync
```

`sync` is strictly stricter than `gossip` because it requires per-hop registry verification before each delegation step (§9.8.5), whereas `gossip` relies on asynchronous broadcast with a trust decay window.

**Unidirectional constraint:** Mode escalation within a session is unidirectional — A can escalate to a stricter mode but MUST NOT relax to a less strict mode via REVOCATION_MODE_ESCALATE. A REVOCATION_MODE_ESCALATE message where `new_mode` is less strict than or equal to the current mode is a protocol violation and MUST be rejected by B. To de-escalate (relax the revocation mode), A MUST either explicitly re-negotiate via a new TASK_ASSIGN or establish a new session.

**Acknowledgment timeout:** If B does not respond with REVOCATION_MODE_ESCALATE_ACK within the session's heartbeat timeout (§4.5), A MUST treat the escalation as failed. A MAY then choose to: (a) retry the escalation, (b) send TASK_CANCEL to terminate the delegation, or (c) escalate to session termination. A MUST NOT assume the new mode is in effect without explicit acknowledgment.

**Propagation in delegation chains:** When B has sub-delegated to C (§6.9) and B receives REVOCATION_MODE_ESCALATE from A, B MUST propagate the escalation to C by sending its own REVOCATION_MODE_ESCALATE to C. B MUST wait for C's REVOCATION_MODE_ESCALATE_ACK before sending its own acknowledgment to A. This ensures the entire delegation chain transitions atomically — A's acknowledgment confirms that all downstream agents have adopted the new mode.

**Interaction with `revocation_mode` on TASK_ASSIGN (§6.6):** The `revocation_mode` field on TASK_ASSIGN sets the initial mode at delegation time. REVOCATION_MODE_ESCALATE overrides this initial mode mid-session. After a successful escalation, the effective revocation mode is the escalated mode, not the mode from the original TASK_ASSIGN. Subsequent REVOCATION_MODE_ESCALATE messages are evaluated against the current effective mode, not the original TASK_ASSIGN mode.

#### 6.15.4 Audit Event

A successful revocation mode escalation MUST be recorded as a `REVOCATION_MODE_ESCALATED` audit event in the evidence layer (§8.10). See §6.13.4 for the audit event table.

**EVIDENCE_RECORD integration:** `REVOCATION_MODE_ESCALATED` events SHOULD be recorded as EVIDENCE_RECORDs (§8.10.1) with `evidence_type: state_snapshot`. The `payload_hash` SHOULD be computed over the REVOCATION_MODE_ESCALATE_ACK message contents (including `prior_mode`, `new_mode`, `effective_from`, and `acknowledged_at`) to enable independent verification that the mode transition occurred and was acknowledged.

#### 6.15.5 Relationship to Existing Mechanisms

**Relationship to §9.8.5 `revocation_mode`:** §9.8.5 defines `sync` and `gossip` as static per-delegation choices. This section extends that model with a dynamic escalation path — the mode can be tightened mid-session without re-delegation. The strictness ordering (`gossip < sync`) and the unidirectional constraint are consistent with §9.8.5's mode propagation rules: mode can only be tightened, not relaxed.

**Relationship to §6.13 Explicit Revocation Channel:** Mode escalation is orthogonal to explicit revocation. A mode escalation changes how revocation signals propagate; it does not itself revoke any grant. An agent may escalate from `gossip` to `sync` and subsequently issue an explicit revocation (§6.13) — the two mechanisms operate independently.

**Relationship to §4 Session Lifecycle:** REVOCATION_MODE_ESCALATE does not alter session state. The session remains ACTIVE throughout the escalation. If A determines that mode escalation is insufficient (e.g., B fails to acknowledge), A's recourse is session termination through the normal session lifecycle (§4.2), not a new escalation mechanism.

> Implements [issue #123](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/123): mid-session revocation mode escalation for §6. Defines REVOCATION_MODE_ESCALATE and REVOCATION_MODE_ESCALATE_ACK message types, unidirectional strictness constraint (gossip → sync only), delegation chain propagation semantics, and REVOCATION_MODE_ESCALATED audit event (§6.13.4). Session remains ACTIVE — no re-negotiation required. De-escalation requires explicit re-negotiation or a new session. Closes #123.

### 6.16 Revocation Enforcement Semantics

<!-- Implements #111: three-layer distinction for revocation enforcement in §6. -->

§6.13 defines the revocation token — the artifact that terminates a grant. §9.8.5 defines propagation modes (`sync`, `gossip`) — how the token reaches agents. Neither section addresses what happens between token receipt and enforcement, or how in-flight operations authorized under a valid grant should be handled when revocation arrives mid-execution. Conflating these three problems produces implementations that either abort in-flight operations mid-execution (correctness violation) or honor revoked capabilities indefinitely (security violation).

This section separates revocation into three distinct layers, each with its own semantics and failure modes.

#### 6.16.1 Three-Layer Distinction

| Layer | Problem | Governs | Spec reference |
|-------|---------|---------|----------------|
| **Revocation propagation** | When does B learn the capability is revoked? | Network/message delivery. B cannot enforce what it has not received. | §6.13.2 (distribution), §9.8.2 (multi-hop propagation), §9.8.5 (sync/gossip modes) |
| **Revocation enforcement** | When must B stop honoring the capability? | Must take effect at enforcement time, not propagation time. The `takes_effect_at` field (§6.13.1) separates these two times. | §6.16.2 (enforcement deadline), §6.13.3 (receiver validation) |
| **In-flight protection** | What happens to operations authorized pre-revocation but executing post-revocation? | Operations authorized under a valid capability that are mid-execution when revocation arrives. Mid-execution cancellation may violate atomicity guarantees. | §6.16.3 (grace period), §6.16.2 (commit-check pattern) |

**Why separation matters:** Each layer has a different failure mode and a different responsible party:

- Propagation failure is a network/intermediary problem (§9.8.2). The issuer controls distribution; intermediaries control forwarding.
- Enforcement failure is a receiver problem. The receiver controls when it stops honoring the grant.
- In-flight failure is an atomicity problem. Neither the issuer nor the receiver caused the conflict — it arises from the temporal overlap between valid authorization and revocation arrival.

Conflating these layers forces implementations into a false binary: immediate enforcement (breaks atomicity) or deferred enforcement (breaks security). The three-layer model allows each to be addressed independently.

#### 6.16.2 Commit-Check Pattern

For irreversible or high-stakes operations — state mutations, financial commits, external resource allocation — CAPABILITY_REVOKE implementations SHOULD apply a final revocation status check immediately before execution. This is the two-phase commit analog for capability-gated operations:

1. **Authorization phase** — validate capability: check scope, TTL expiry (§5.8.2), explicit revocation status (§6.13.3), and delegation chain integrity (§6.9.3). If any check fails, reject the operation.
2. **Revocation window** — the gap between authorization and commit where a CAPABILITY_REVOKE may arrive. Duration is implementation-dependent but bounded by the operation's execution latency.
3. **Commit phase** — immediately before executing the irreversible step, perform a final revocation status check. If a revocation token has arrived during the window, abort the operation. If no revocation token has arrived, proceed.

**Synchronous verification requirement:** Synchronous revocation verification (re-checking revocation status against the latest available state) is REQUIRED for operations declared irreversible in the task schema (§6.1). Asynchronous verification (relying on cached revocation state) is permitted only for idempotent or easily-reversible operations where the cost of post-hoc rollback is bounded.

**Irreversibility declaration:** Operations that require commit-check SHOULD be declared irreversible in the task schema (§6.1) via the step's `retry_semantics` field. When `retry_semantics` is `at_most_once`, the operation is treated as irreversible for commit-check purposes. When `retry_semantics` is `at_least_once` or `exactly_once`, the operation MAY be treated as reversible (idempotent retry is available).

**Commit-check flow:**

```
B receives operation request under grant G:
  1. Authorization phase:
     - Verify G.scope includes the operation
     - Verify current_time < G.valid_until (TTL check)
     - Verify no revocation token exists for G.grant_id
     - If takes_effect_at is present and current_time ≥ takes_effect_at: REJECT
     → All checks pass: operation is authorized
  2. Revocation window (operation preparation):
     - B prepares the operation (resource allocation, staging)
     - Revocation tokens may arrive during this window
  3. Commit phase (immediately before irreversible execution):
     - Re-check: has a revocation token for G.grant_id arrived?
     - Re-check: if takes_effect_at is present, is current_time ≥ takes_effect_at?
     → If revoked or enforcement deadline reached: ABORT, return TASK_FAIL
     → If still valid: EXECUTE
```

#### 6.16.3 Grace Period for In-Flight Operations

When a revocation token arrives while an operation is mid-execution, the receiving agent faces a conflict between security (stop using revoked capability) and correctness (do not abort atomic operations mid-execution). The grace period resolves this conflict by providing a bounded window for in-flight operations to complete.

**Best-effort revocation mode:** When `takes_effect_at` is present on the revocation token (§6.13.1), implementations MUST honor the grace period — the interval between token receipt and `takes_effect_at`. During this period:

- Operations that were authorized *before* token receipt and are currently executing MAY complete without abort.
- No *new* operations may be authorized under the revoked grant. The grant is revoked for authorization purposes at receipt time, not at `takes_effect_at`.
- The grace period MUST be at least 5 seconds (spec-mandated minimum). Issuers SHOULD set `takes_effect_at` to at least `revoked_at` + 5 seconds. A `takes_effect_at` value less than 5 seconds after `revoked_at` is valid but implementations MAY extend it to the 5-second minimum to protect in-flight atomicity.
- The grace period SHOULD be at least as long as the longest expected in-flight operation duration declared at session initiation (via the task schema's estimated step durations, if available). When no duration estimate is available, the 5-second spec minimum applies.

**Strict revocation mode:** When `takes_effect_at` is absent from the revocation token, enforcement is immediate upon receipt — the grace period is zero. In-flight operations MUST be aborted at the next safe interruption point. For operations declared irreversible (`retry_semantics: at_most_once`), "safe interruption point" means before the irreversible commit; if the irreversible step has already been committed, the operation is allowed to complete (aborting after commit would leave the system in an inconsistent state).

**Grace period interaction with revocation modes (§9.8.5):**

| Revocation mode | `takes_effect_at` present | Behavior |
|-----------------|---------------------------|----------|
| `sync` | Yes | Grace period honored. Issuer verified revocation synchronously; `takes_effect_at` provides the enforcement deadline. |
| `sync` | No (absent) | Immediate enforcement. No grace period. |
| `gossip` | Yes | Grace period honored. Particularly important in gossip mode where propagation latency is non-deterministic — the grace period absorbs propagation jitter. |
| `gossip` | No (absent) | Immediate enforcement on receipt. In-flight operations aborted at next safe interruption point. |

**Completion during grace period:** An operation that completes during the grace period (after revocation receipt, before `takes_effect_at`) MUST be recorded as an `IN_FLIGHT_COMPLETED_DURING_GRACE` audit event (§6.13.4). This provides the audit trail to distinguish operations that completed under valid authorization from operations that completed during the grace window — different trust levels for post-hoc review.

**Formal enforcement obligation — `start_time` vs. `effective_from`:** Agents MUST refuse any operation whose `start_time` (the timestamp at which the operation begins execution) is ≥ `effective_from` on the revocation token. This is the hard enforcement boundary: an operation that has not yet started when the logical revocation time has passed MUST NOT be started, regardless of whether `takes_effect_at` has been reached. The grace period (above) protects only operations that were *already executing* before `effective_from` — it does not authorize new operations in the window between `effective_from` and `takes_effect_at`. Formally:

- `start_time < effective_from`: operation was started under valid authorization. If still executing at `takes_effect_at`, the grace period applies (above).
- `start_time ≥ effective_from`: operation MUST be rejected. The capability was logically revoked before the operation began. No grace period applies.
- `start_time < effective_from` AND `completion_time > takes_effect_at`: the operation exceeded the grace period. The agent MUST abort at `takes_effect_at` unless the operation has already passed its irreversible commit point (§6.16.2).

#### 6.16.4 `takes_effect_at` Semantics

The `takes_effect_at` field on the revocation token (§6.13.1) separates revocation propagation time from enforcement deadline. This separation is the mechanism that enables the three-layer distinction to operate in practice.

**Temporal semantics:**

- `revoked_at` is the issuer's declaration time — when the issuer decided to revoke.
- `effective_from` is the logical revocation time — when the capability is considered revoked (§6.13.1). Operations completed before `effective_from` are unconditionally valid (§6.13.6 good faith protection).
- `propagation_window_ms` is the issuer's declared propagation budget — the upper bound on how long it takes the revocation token to reach all downstream holders (§6.13.6).
- Token receipt time (`propagated_at`, §6.13.4) is the propagation time — when B learned about the revocation. This is not a field on the token; it is B's local observation, recorded as receiver-local state (§6.13.4).
- `takes_effect_at` is the enforcement deadline — when B MUST stop honoring the grant.

**Constraints:**

- `takes_effect_at` MUST be ≥ `effective_from`. The enforcement deadline cannot precede the logical revocation time.
- `takes_effect_at` SHOULD be ≥ `effective_from` + 5 seconds (the spec-mandated grace period minimum).
- `takes_effect_at` MUST NOT exceed `effective_from` + 60 seconds. A grace period longer than 60 seconds extends the security exposure window beyond acceptable bounds for the protocol's threat model (§9.8.1). Issuers requiring longer grace periods MUST use a different mechanism (e.g., issuing a replacement grant with a shorter `valid_until` instead of revoking).
- When `takes_effect_at` is absent, enforcement is immediate upon receipt — no grace period.

**Clock skew:** Agents MUST use the same time synchronization assumptions as the rest of the protocol (§5.8.2 TTL enforcement). If `takes_effect_at` has already passed by the time B receives the token (receipt time > `takes_effect_at`), enforcement is immediate — the grace period has already elapsed. B MUST NOT retroactively extend the grace period past the issuer-specified `takes_effect_at`.

#### 6.16.5 Relationship to Existing Mechanisms

**Relationship to §6.13 Explicit Revocation Channel:** This section extends §6.13 with enforcement timing semantics. The revocation token schema (§6.13.1) carries the `takes_effect_at` field; the receiver validation behavior (§6.13.3) evaluates it. This section defines *when* enforcement occurs and *how* in-flight operations are handled — concerns that §6.13's token-centric model does not address.

**Relationship to §9.8.4 Revocation Timing Gap:** §9.8.4 models the timing gap as a threat — work committed during the propagation window is at risk. This section provides the mitigation: the commit-check pattern (§6.16.2) reduces the timing gap for irreversible operations, and the grace period (§6.16.3) provides a bounded window for in-flight operations to complete without atomicity violation.

**Relationship to §9.8.7 Blast Radius Quantification:** The grace period introduces a controlled, bounded addition to the blast radius. Work completed during the grace period is *known* work-at-risk (recorded via `IN_FLIGHT_COMPLETED_DURING_GRACE` audit events) rather than *unknown* work-at-risk from uncontrolled propagation delay. This converts a detection problem (did work slip through?) into a policy problem (was the grace period acceptable?).

**Relationship to §6.11 Pre-execution Plan Commitment:** The commit-check pattern (§6.16.2) is complementary to step-boundary enforcement (§6.11.5). Step boundaries define *where* execution checkpoints occur; the commit-check pattern defines *what* to verify at the final checkpoint before irreversible commit. Implementations SHOULD combine both: verify revocation status at each step boundary and perform a final commit-check before any irreversible step.

> Implements [issue #111](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/111): three-layer distinction for revocation enforcement in §6. Separates revocation into propagation (network delivery), enforcement (when B stops honoring the capability), and in-flight protection (atomicity for mid-execution operations). Defines commit-check pattern for irreversible operations, spec-mandated 5-second grace period via `takes_effect_at` timestamp, and `IN_FLIGHT_COMPLETED_DURING_GRACE` audit event. Three-layer distinction formalized by @jacobi\_. Commit-check pattern proposed by @Jarvis4. 5s grace period suggested by @Pi\_Manga, endorsed by @XiaoFei\_AI from financial and medical production deployment. Closes #111.

#### 6.16.6 Compensation Semantics for Irrevocable Work

<!-- Implements #94: compensation semantics for irrevocable work committed past effective_from -->

The commit-check pattern (§6.16.2) prevents most irrevocable work from being committed after revocation. However, two edge cases escape the commit-check:

1. **Propagation gap commit.** B commits irrevocable work after `effective_from` but before receiving the revocation token (B did not yet know the capability was revoked). Good faith protection (§6.13.6) applies — B is not at fault — but the irrevocable work still exists and may need to be undone.
2. **Race condition commit.** B's commit-check passes (no revocation token present), but a revocation token arrives between the check and the irreversible execution. The commit-check reduces but does not eliminate this window.

In both cases, the irrevocable work was committed under a capability that is now revoked. The protocol cannot undo the work — it is irrevocable by definition. Instead, the protocol requires compensation: a structured mechanism for the executing agent to declare the irrevocable work and for the issuer to initiate remediation.

**Compensation obligations:**

1. **Detection.** When an agent discovers that it committed irrevocable work (an operation with `retry_semantics: at_most_once`) under a grant that has since been revoked — specifically, when the operation's `committed_at` timestamp is ≥ `effective_from` on the revocation token — the agent MUST emit a `COMPENSATION_REQUIRED` audit event (§6.13.4).

2. **Declaration.** The `COMPENSATION_REQUIRED` audit event MUST include: `revocation_id`, `grant_id`, `task_id`, the operation identifier, `committed_at` (when the irrevocable step was executed), `effective_from` (the logical revocation time), `compensation_policy` (the enum value from the capability manifest for the capability under which the irrevocable work was executed — one of `best_effort`, `rollback`, `idempotent_retry`, `none`), and whether the commit fell within the `propagation_window_ms` (good faith) or outside it (protocol violation).

3. **Issuer responsibility.** The revoking agent (issuer) bears responsibility for initiating compensation for irrevocable work committed during the propagation window — the issuer set the `propagation_window_ms` and accepted that downstream agents might exercise the capability within that window. Irrevocable work committed after `revoked_at + propagation_window_ms` is the executing agent's responsibility (protocol violation).

4. **Compensation hooks.** Agents that execute irreversible operations SHOULD declare compensation endpoints in their capability manifest (§5.1) and SHOULD declare a `compensation_policy` (§5.1) for each capability that may produce irrevocable work. A compensation endpoint accepts a `COMPENSATION_REQUIRED` event and returns a compensation plan — the set of operations needed to remediate the irrevocable work. The `compensation_policy` declared in the capability manifest governs the remediation strategy.

5. **Idempotency requirement for compensation.** Compensation operations MUST be idempotent — invoking the same compensation request multiple times (identified by `revocation_id` + operation identifier) MUST produce the same result. This is critical because compensation may be triggered by multiple agents in a delegation chain that independently discover the revocation.

**`compensation_policy` enum:**

The `compensation_policy` field is a protocol-level enum declared per capability type in the agent's CAPABILITY_MANIFEST (§5.1). It governs what remediation strategy the agent commits to for irrevocable work executed under a given capability. The enum has exactly four values:

| Value | Definition |
|-------|-----------|
| `best_effort` | The agent will attempt compensation for irrevocable work, but provides no guarantee of success. The compensation endpoint (if declared) will be invoked, and the result (`completed` or `failed`) recorded in the audit trail. Partial remediation is acceptable. |
| `rollback` | The agent will attempt a full state revert to the pre-task state. All effects of the irrevocable work — writes, resource allocations, external API side effects — will be reversed. If rollback fails, the agent MUST emit a `COMPENSATION_REQUIRED` event with `compensation_status: failed` and escalate to manual resolution. |
| `idempotent_retry` | The agent will retry the task with idempotency guarantees. The retried execution MUST produce no additional state change beyond what the original execution committed. This policy is appropriate for operations where re-execution is safe and convergent. |
| `none` | No compensation action will be taken. The irrevocable work is accepted as committed. The agent will emit the `COMPENSATION_REQUIRED` audit event for record-keeping but will not attempt remediation. The issuer or operator bears full responsibility for any required remediation. |

**Validation requirement:** Implementations MUST reject unknown `compensation_policy` values with a parse error. Silent ignore of an unrecognized value is not permitted. Silent default to a fallback value is not permitted. This is a mandatory interoperability requirement — freeform strings break interoperability at parse time because implementations cannot reliably distinguish valid policies, and divergent interpretations are silent failures. The enum is closed: only the four values defined above are valid in V1. Extension of the enum requires a protocol version change (§10).

**Compensation flow:**

```
B discovers irrevocable work committed after effective_from:
  1. Emit COMPENSATION_REQUIRED audit event (§6.13.4)
     - Include the compensation_policy from the capability manifest
  2. If compensation_policy is "none":
     - Log COMPENSATION_REQUIRED with compensation_status: pending
     - No automated remediation — issuer or operator resolves manually
  3. If compensation_policy is "best_effort", "rollback", or
     "idempotent_retry" and compensation endpoint is declared:
     - Invoke compensation endpoint with revocation_id, grant_id,
       operation details, committed_at, and compensation_policy
     - Record compensation result (completed/failed) in audit trail
  4. If compensation endpoint is not declared (regardless of policy):
     - Log COMPENSATION_REQUIRED with compensation_status: pending
     - The issuer or operator MUST resolve manually
  5. Notify the revoking agent (issuer) of the COMPENSATION_REQUIRED
     event via the session channel, enabling the issuer to track
     outstanding irrevocable work under the revoked grant
```

**Relationship to good faith protection (§6.13.6):** Good faith protection determines *fault attribution* — who is responsible for the cost of irrevocable work committed during the propagation window. Compensation semantics determine *remediation* — what to do about the irrevocable work regardless of fault. Both apply simultaneously: an agent may be protected from blame (good faith) while still requiring compensation (the work exists and must be addressed).

**Relationship to pending_tasks inventory (§4.9.2):** When a session is torn down from SUSPENDED state, the `pending_tasks` inventory (§4.9.2) provides task-level context — each entry's `task_id` identifies the task and `last_known_state` communicates its state at suspension time. The `pending_tasks` obligation is enumeration only — a V1 primitive; whether a receiver initiates the compensation workflow defined here for any enumerated task is an application-layer decision. The inventory surfaces what the sender knows; the receiver determines disposition (compensation, handoff, or abandonment) per its own policy.

> Implements [issue #94](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/94): compensation semantics for irrevocable work committed past `effective_from`. Adds `COMPENSATION_REQUIRED` audit event, compensation endpoint declaration in capability manifest, idempotency requirement for compensation operations, and compensation flow for propagation-gap and race-condition commits. Closes #94.
>
> Implements [issue #192](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/192): changes `compensation_policy` from freeform string to a bounded protocol-level enum with exactly four values (`best_effort`, `rollback`, `idempotent_retry`, `none`). Adds mandatory parse-error rejection for unknown values — silent ignore or silent default is not permitted. Adds `compensation_policy` to CAPABILITY_MANIFEST capability type entries (§5.1) and to the `COMPENSATION_REQUIRED` audit event (§6.13.4). Closes #192.

#### 6.16.7 Revocation Acknowledgment — V1 Design Decision

<!-- Implements #185: V1 design decision — revocation acknowledgment (confirmed enforcement) is application-layer in V1 -->

§6.16.1–§6.16.6 define revocation propagation, enforcement, and in-flight protection. A fourth question naturally follows: how does the revoker *confirm* that the revokee has enforced the revocation? This section documents the resolved V1 design decision: confirmed enforcement is not a protocol-layer concern in V1. TTL-based enforcement provides eventual consistency by construction, and CAPABILITY_REVOKED is fire-and-forget at the protocol layer.

**V1 Mechanism — TTL-Based Enforcement as Eventual Consistency Guarantee**

V1 relies on TTL-based enforcement (§6.17) as the foundational revocation guarantee. Every CAPABILITY_GRANT carries a REQUIRED `valid_until` field (§6.17.1) with a maximum TTL of 24 hours (§5.8.2). This creates a hard upper bound on how long any capability can remain exercisable regardless of whether explicit revocation succeeds:

- If CAPABILITY_REVOKED is received and processed: the capability is terminated immediately (or at `takes_effect_at`, per §6.16.3).
- If CAPABILITY_REVOKED is lost, delayed, or ignored: the capability self-expires at `valid_until`. No action from any party is required for termination to occur.

Revocation acknowledgment is therefore *eventually consistent by construction* — the capability is guaranteed to cease being valid at or before `valid_until`, bounded by the 24-hour maximum TTL. The revoker does not need confirmation that the revokee enforced the revocation because the TTL enforces it regardless. The explicit CAPABILITY_REVOKED signal (§6.13) accelerates termination for the pre-expiry compromise case; TTL is the backstop that makes acknowledgment unnecessary for correctness.

**CAPABILITY_REVOKED Is Fire-and-Forget in V1**

At the protocol layer, CAPABILITY_REVOKED (§6.13) is a unidirectional signal: the revoker issues a signed revocation token and distributes it via best-effort gossip (§6.13.6) or synchronous delivery (§9.8.5). The protocol defines no response message. The revoker cannot distinguish, at the protocol layer, between three outcomes:

1. **Enforcement success.** The revokee received the token, validated the signature, and ceased exercising the capability.
2. **Network loss.** The revocation token was lost in transit. The revokee continues exercising the capability in good faith (§6.13.6) until TTL expiry.
3. **Deliberate evasion.** The revokee received the token and chose to ignore it, continuing to exercise the capability until TTL expiry or detection via other means (§8.16, §8.22).

All three outcomes converge at the same point: the capability expires at `valid_until`. The difference between them is *attribution* (who is at fault) and *exposure window* (how long the capability remains exercisable after the revoker's intent), not *eventual state* (the capability terminates in all cases). V1 accepts this convergence as sufficient. The `propagation_window_ms` field (§6.13.1) bounds the acceptable attribution ambiguity: after `revoked_at + propagation_window_ms`, continued exercise is a protocol violation (§6.13.1) regardless of whether it results from network loss or deliberate evasion.

**V2 Candidates — REVOCATION_ACK and CAPABILITY_STATE_QUERY**

Two mechanisms would extend the revocation model with confirmed enforcement. Both are explicitly scoped as V2 candidates — they are not part of V1 and MUST NOT be implemented as protocol-layer extensions in V1 deployments. Application-layer implementations MAY provide equivalent functionality outside the protocol boundary.

**1. REVOCATION_ACK — Acknowledgment of Revocation Enforcement**

A response message from the revokee confirming that it has processed a revocation token and ceased exercising the capability. This would close the observability gap between outcomes (1), (2), and (3) above by providing positive confirmation of enforcement.

*Round-trip failure modes that V2 must address:*

| Failure mode | Description | Consequence |
|--------------|-------------|-------------|
| ACK lost | Revokee enforced the revocation and sent REVOCATION_ACK, but the ACK was lost in transit | Revoker falsely concludes enforcement failed. May trigger unnecessary escalation (session termination, trust downgrade) against a compliant revokee. |
| ACK forged | A third party or compromised intermediary forges a REVOCATION_ACK on behalf of the revokee | Revoker falsely concludes enforcement succeeded. The actual revokee may still be exercising the revoked capability. Requires cryptographic binding of ACK to revokee identity (§2.2.1) and to the specific revocation token. |
| ACK delayed | Revokee enforced the revocation but the ACK arrives after the revoker's timeout | Semantically equivalent to ACK lost from the revoker's perspective. Revoker must define an ACK timeout and a policy for late ACKs — accept as confirmation, or treat as enforcement failure. |

*Design note:* REVOCATION_ACK introduces a two-phase protocol where V1 uses a one-phase protocol. The additional round-trip creates a new class of partial-failure states (revocation sent, ACK pending) that V1 avoids entirely. Any V2 design must define the state machine for these intermediate states without regressing the TTL-based eventual consistency guarantee.

**2. CAPABILITY_STATE_QUERY — On-Demand Capability Status Inquiry**

A query mechanism enabling the revoker (or any authorized party) to ask the revokee whether a specific capability is currently being exercised, has been revoked, or has expired. This would provide point-in-time observability into the revokee's capability state.

*Round-trip failure modes that V2 must address:*

| Failure mode | Description | Consequence |
|--------------|-------------|-------------|
| Query lost | The query never reaches the revokee | Revoker receives no response. Indistinguishable from a revokee that is offline, partitioned, or deliberately non-responsive. |
| Response forged | A compromised agent returns a false capability state | Revoker makes decisions based on inaccurate state. Requires cryptographic attestation of the response, binding it to the revokee's identity and to a specific point in time. |
| Response stale | The revokee's state changes between response generation and revoker receipt | The response is accurate at generation time but outdated at receipt time. The revoker acts on stale state. Bounded by network latency but not eliminable. |
| State query mechanism not in spec | V1 defines no general-purpose state query primitive | CAPABILITY_STATE_QUERY requires a request-response pattern that the V1 protocol does not provide for capability state. V2 must define the query schema, response schema, authorization model (who may query), and rate-limiting semantics. |

*Design note:* CAPABILITY_STATE_QUERY requires the revokee to maintain and expose queryable capability state — a runtime obligation that V1 does not impose. V1 revocation is token-centric (§6.13): the revocation token is the artifact, and enforcement is local. A state query model shifts from artifact-centric to state-centric revocation, which has different consistency, availability, and partition-tolerance tradeoffs.

**Boundary Statement**

This is a resolved design decision, not an open question. V1 defines the boundary: TTL-based enforcement provides the eventual consistency guarantee; CAPABILITY_REVOKED is fire-and-forget; the revoker's recourse for enforcement uncertainty is TTL expiry (passive) and detection signals (§8.16, §8.22) (active). V2 extends this boundary with confirmed enforcement via REVOCATION_ACK and on-demand observability via CAPABILITY_STATE_QUERY, accepting the round-trip failure modes documented above as the cost of stronger enforcement guarantees.

> Implements [issue #185](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/185): V1 design decision — revocation acknowledgment (confirmed enforcement) is application-layer in V1. Defines TTL-based enforcement as the V1 mechanism providing eventual consistency by construction (capability expires at or before `valid_until`). Documents CAPABILITY_REVOKED as fire-and-forget at the protocol layer — revoker cannot distinguish enforcement success, network loss, or deliberate evasion. Scopes REVOCATION_ACK and CAPABILITY_STATE_QUERY as V2 candidates with round-trip failure modes (ACK lost/forged/delayed; query requires state query mechanism not in spec). Framed as a resolved design decision: V1 defines the boundary, V2 extends it. Closes #185.

#### 6.16.8 Revocation Acknowledgment Scope Statement

<!-- Implements #219: revocation acknowledgment scope statement — unilateral REVOKED declaration, non-receipt security advisory, V2 N-of-M acknowledgment -->

§6.16.7 establishes the V1 design decision: revocation acknowledgment is not a protocol-layer concern. This subsection makes the scope boundary explicit by defining the V1 state transition semantics for revocation completion, documenting the adversarial non-receipt threat, and specifying the V2 acknowledgment-threshold model that addresses it.

**V1 Position — Unilateral Revocation Completion**

In V1, REVOKED is declared unilaterally by the revoking agent. The state transition from REVOKE_PENDING (the interval between the revoker issuing CAPABILITY_REVOKED and the grace window expiring) to REVOKED does not require confirmed enforcement from any downstream agent. The revoker transitions the revocation to REVOKED when the grace window expires (`takes_effect_at`, §6.16.4) or, if no grace window was specified, immediately upon issuing the revocation token.

Enforcement is assumed at TTL — every grant carries a `valid_until` field (§6.17.1) with a maximum TTL of 24 hours (§5.8.2), guaranteeing that the capability ceases to be exercisable regardless of downstream behavior. Acknowledgment from downstream agents is advisory: if a downstream agent sends an application-layer acknowledgment confirming enforcement, the revoker MAY record it for observability purposes, but the revoker MUST NOT gate the REVOKE_PENDING → REVOKED transition on receipt of such acknowledgment. The transition is time-driven, not acknowledgment-driven.

**Formal state transition:**

```
REVOKE_PENDING:
  Entry: revoker issues CAPABILITY_REVOKED token (§6.13)
  Duration: from revoked_at to takes_effect_at (or instant if takes_effect_at absent)
  Exit condition: current_time ≥ takes_effect_at (or immediate if absent)
  → REVOKED

REVOKED:
  Entry: grace window expired OR no grace window specified
  Requirement: NONE — no downstream acknowledgment required
  The capability is considered revoked by the revoker regardless of
  downstream enforcement status. TTL expiry (valid_until) provides
  the hard enforcement backstop.
```

**Security Advisory — Non-Receipt Evasion in Adversarial Contexts**

In adversarial contexts, a downstream agent could claim non-receipt of the revocation token while continuing to exercise the revoked capability within the TTL window. This attack exploits the observability gap documented in §6.16.7 (outcome 2 vs. outcome 3): the revoker cannot distinguish, at the protocol layer, between a revocation token lost in transit and a revocation token deliberately ignored.

The non-receipt evasion threat has the following properties:

| Property | Description |
|----------|-------------|
| **Attack surface** | Any downstream agent holding a valid (non-expired) grant that receives a CAPABILITY_REVOKED token via gossip (§9.8.5) or sync delivery |
| **Attack mechanism** | The downstream agent receives the revocation token, does not enforce it, and claims non-receipt if challenged. The agent continues exercising the capability until `valid_until` expiry. |
| **Exposure window** | From `revoked_at` to `valid_until` — bounded by the 24-hour maximum TTL (§5.8.2). The explicit revocation signal was intended to close this window early; the evasion negates that acceleration. |
| **Protocol-layer detectability** | Not detectable in V1. The protocol defines no mechanism for the revoker to distinguish non-receipt from deliberate evasion. CAPABILITY_REVOKED is fire-and-forget (§6.16.7). |
| **Out-of-band detectability** | Detectable via behavioral signals: the downstream agent continues producing outputs, consuming resources, or invoking APIs under the revoked capability after the revoker's `revoked_at` timestamp. Detection requires monitoring infrastructure outside the protocol boundary — audit log correlation (§8.16), resource usage anomalies (§8.22), or platform-level enforcement. |
| **Mitigation** | TTL expiry is the hard backstop — the capability self-expires at `valid_until` regardless of downstream behavior. For high-value capabilities where the exposure window is unacceptable, issuers SHOULD use short TTLs (minutes, not hours) and frequent renewal (§6.17.3) to minimize the window. |

This threat is accepted as a known limitation of V1's fire-and-forget revocation model. The revoker's recourse is TTL expiry (passive) and out-of-band detection (active). Protocol-layer detection requires the acknowledgment mechanisms deferred to V2 (below).

**V2 Note — N-of-M Acknowledgment Threshold**

V2 introduces an N-of-M acknowledgment metric that closes the non-receipt evasion gap at the protocol layer. After issuing a revocation, the revoking agent waits for REVOCATION_ACK (§6.16.7) from downstream agents before considering revocation complete. The completion threshold is M-of-N, where N is the total number of downstream agents holding grants derived from the revoked capability, and M is the required acknowledgment count.

Below the M-of-N threshold, the revocation status is `PENDING_ACK`, not `REVOKED`:

| Revocation status | Condition | Meaning |
|-------------------|-----------|---------|
| `REVOKE_PENDING` | Revocation token issued, grace window not yet expired | Revocation is in the propagation and grace period. No enforcement expected yet. |
| `PENDING_ACK` | Grace window expired, fewer than M-of-N acknowledgments received | Revocation is past the enforcement deadline but the revoker has not received sufficient confirmation of downstream enforcement. The capability may still be exercised by non-acknowledging agents (within TTL). |
| `REVOKED` | M-of-N acknowledgments received, OR `valid_until` expired (TTL backstop) | Revocation is confirmed — either enough downstream agents have acknowledged enforcement, or the TTL has expired making enforcement moot. |

*V2 design considerations:*

- **Threshold selection.** M and N are determined by the revocation context. For single-grantee capabilities (N=1), M=1 — full confirmation is required. For multi-hop delegation chains (§6.9), N includes all transitive grantees, and M is a policy decision balancing confirmation confidence against availability (requiring M=N is strict but blocks on the slowest or most adversarial agent).
- **PENDING_ACK timeout.** The `PENDING_ACK` state MUST have a bounded duration. If the M-of-N threshold is not reached within a configurable timeout, the revocation transitions to `REVOKED` via the TTL backstop (`valid_until` expiry). `PENDING_ACK` does not block indefinitely.
- **Interaction with TTL backstop.** The TTL backstop (§6.17) remains the hard guarantee in V2. N-of-M acknowledgment provides *faster confirmation* of enforcement — it does not replace TTL expiry as the eventual consistency mechanism. When `valid_until` expires, the revocation transitions to `REVOKED` regardless of acknowledgment count.
- **Non-acknowledging agent handling.** Agents that do not acknowledge within the `PENDING_ACK` timeout are flagged for out-of-band investigation. V2 must define whether non-acknowledgment triggers automatic trust degradation (§4.2.2), session revocation (§8.15), or is purely informational.

This is a V2-scoped mechanism. V1 implementations MUST NOT implement `PENDING_ACK` as a protocol-layer state or gate revocation completion on downstream acknowledgment. Application-layer implementations MAY track acknowledgments outside the protocol boundary for observability purposes, provided they do not alter the V1 state transition semantics (REVOKE_PENDING → REVOKED is time-driven, not acknowledgment-driven).

> Implements [issue #219](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/219): revocation acknowledgment scope statement. V1 position: REVOKED is declared unilaterally after grace window expiry — the REVOKE_PENDING → REVOKED transition is time-driven, not acknowledgment-driven; downstream enforcement is assumed at TTL. Security advisory: non-receipt evasion (downstream agent claims non-receipt while continuing to exercise revoked capability) is not detectable at protocol level in V1 — detectable only out-of-band via audit log correlation and behavioral signals. V2 note: N-of-M acknowledgment threshold — revoker waits for M-of-N REVOCATION_ACKs before transitioning to REVOKED; below threshold the status is PENDING_ACK; TTL backstop remains the hard guarantee. Closes #219.

### 6.17 Time-Bounded Capability Grants

<!-- Implements #107: time-bounded capability grants — valid_until enforcement, cascading TTL decay with floor constraint, renewal semantics with cascading renewal rules, session vs capability TTL distinction, TTL exhaustion vs session end orthogonality, in-flight expiry, relationship to CAPABILITY_REVOKE, and V2 deferrals -->

Explicit TTL on capability grants eliminates the revocation propagation race for non-compromise cases. Grants self-expire without requiring a revocation signal — the receiving agent enforces `valid_until` locally, with no network round-trip. CAPABILITY_REVOKE (§6.13) becomes an emergency path for pre-expiry compromise rather than the normal grant termination path. Production evidence from @larryadlibrary (issue #15): an agent ran 47 operations on expired credentials because trust validation happened at session initiation only — capability-level TTL enforcement closes this gap.

Capability-level TTL enforcement is superior to per-operation revalidation. Per-operation revalidation adds latency to every operation, burns rate limits on the validation service, and creates a validation-service dependency that cascades on failure — if the validation service is down, all operations halt. TTL enforcement is local, self-contained, and partition-tolerant.

#### 6.17.1 `valid_until` Enforcement

`valid_until` is a REQUIRED field on CAPABILITY_GRANT (§5.8.2). Every grant MUST include an explicit expiry timestamp in ISO 8601 format with millisecond precision.

**Enforcement rules:**

1. `valid_until` is a **protocol obligation**, not advisory. Receiving agents MUST enforce `valid_until` at the capability level — not at the session level, not at session initiation only, and not as a periodic check. The grant's temporal validity is evaluated each time the grant is exercised.
2. **Maximum TTL for V1: 24 hours.** `valid_until` MUST NOT exceed 24 hours from `granted_at` (or from `current_time` when `granted_at` is absent). Grants with `valid_until` beyond the 24-hour maximum are non-compliant — receiving agents MUST reject them.
3. A CAPABILITY_GRANT without `valid_until` is non-compliant and MUST be rejected by the receiver.
4. When `current_time > valid_until` (accounting for the 30-second clock skew tolerance, §5.8.2), the grant is invalid. The agent MUST stop exercising the granted capabilities and MUST report `TTL_EXPIRED` for any pending operations that cannot complete within the grace period (§6.17.4).

#### 6.17.2 Cascading TTL Decay

When an agent (B) holding a CAPABILITY_GRANT issues a sub-grant to a downstream agent (C) — whether via CAPABILITY_GRANT or embedded in a sub-delegation's `delegation_token` (§5.5, §6.9) — the child grant's temporal bounds are constrained by the parent.

Capability TTL in delegation chains follows a **floor constraint**: a sub-delegated capability's effective TTL MUST NOT exceed the remaining TTL of the parent capability at delegation time.

Formally:

```
effective_ttl(child) = min(requested_ttl(child), remaining_ttl(parent))
```

Where `remaining_ttl(parent)` is computed as `parent.valid_until - current_time` at the moment of sub-delegation. This is a hard protocol constraint — not advisory, not implementation-specific.

**Cascading TTL rules:**

1. **Child TTL MUST NOT exceed parent TTL.** A child grant's `valid_until` MUST be ≤ the parent grant's `valid_until`. A child grant with `valid_until` beyond the parent's `valid_until` is non-compliant and MUST be rejected by the receiver. An agent receiving a capability MUST reject sub-delegation attempts where child TTL would exceed parent remaining TTL at the time of sub-delegation. The receiving agent SHOULD return CAPABILITY_DENY with `reason=TTL_EXCEEDS_PARENT`.
2. **Parent expiry cascades immediately.** When a parent grant expires (its `valid_until` is reached), all derived child grants immediately become invalid regardless of their own `valid_until`. The child grant's `valid_until` is an upper bound, not a guarantee — the parent's expiry is the effective ceiling.
3. **Child grant holders MUST verify parent grant validity before acting.** Agents holding child grants MUST verify that the parent grant remains valid (not expired, not revoked) before exercising the child grant's capabilities. This is a local check when the parent grant's `valid_until` is known to the child grant holder — no network round-trip required if the parent grant metadata was propagated with the child grant.
4. **Propagation of parent grant metadata.** When issuing a child grant, the issuing agent MUST set `parent_grant_id` and `parent_grant_hash` (§5.8.2) on the child CAPABILITY_GRANT to cryptographically bind it to the parent. The issuing agent SHOULD additionally include the parent grant's `valid_until` in the child grant's `delegation_token` metadata (§5.5) to enable local parent-validity checks by the child grant holder without requiring parent grant retrieval.

This closes the cascading trust decay failure mode: parent expires, child continues operating on stale authorization. Without cascading TTL decay, a child grant with a `valid_until` beyond the parent's could continue to authorize operations after the parent's authority has lapsed — violating the delegation chain's trust invariant.

#### 6.17.3 Renewal Semantics

CAPABILITY_RENEW (§5.8.3) extends a grant's `valid_until` without changing the authorized capability set. Renewal is subject to the following constraints within the time-bounded grant model:

1. **Renewal MUST be re-grant from the original issuer only.** The `grantor_id` on CAPABILITY_RENEW MUST match the `grantor_id` of the original CAPABILITY_GRANT (§5.8.3). A grantee (B) extending its own grant's `valid_until` — even within the original grantor's (A's) scope — is prohibited. Self-renewal risks scope drift over repeated renewals: each renewal is a trust extension decision that belongs to the original authority, not the recipient.
2. **Renewed `valid_until` is subject to the 24-hour maximum.** A CAPABILITY_RENEW's `valid_until` MUST NOT exceed 24 hours from the renewal's `granted_at` (or `current_time` when `granted_at` is absent). The 24-hour V1 maximum applies per-grant-instance, not cumulative — an original grant may be renewed indefinitely as long as each renewal's TTL does not exceed 24 hours.
3. **Renewal message format.** Renewal uses CAPABILITY_RENEW (§5.8.3) — a new CAPABILITY_GRANT with a fresh `valid_until`, referencing the original grant's `grant_id` via the `original_grant_id` field. The `original_grant_id` serves as the correlation key for deduplication and audit trail continuity.
4. **Cascading renewal.** When a parent grant is renewed, child grants do not automatically inherit the extended `valid_until`. Child grant holders whose grants are approaching expiry MUST request renewal from their immediate grantor. Renewal propagates hop-by-hop through the delegation chain, not automatically from the root. The delegation chain must be explicitly re-established by re-issuing each child grant.
5. **Renewal cascading constraint.** A renewed grant's `valid_until` MUST NOT exceed the parent capability's `valid_until` at renewal time — the same cascading floor constraint (§6.17.2) applies to renewals. Formally: `effective_ttl(renewed_child) = min(requested_ttl(renewal), remaining_ttl(parent_at_renewal_time))`.
6. **Grantor signature verification for renewal references.** Agents MUST NOT honor capability grants that reference a previous `grant_id` as a renewal (via `original_grant_id`) unless the renewal is signed by the original grantor. A renewal referencing a prior grant but signed by a different identity is a protocol violation and MUST be rejected.

#### 6.17.4 In-Flight Operations at Expiry

When a grant's `valid_until` is reached while operations are in progress, the protocol applies best-effort completion semantics consistent with TASK_CANCEL behavior (§6.6):

1. **Atomic operations in progress at TTL expiry MAY complete within a 5-second grace period.** The grace period begins at the moment `current_time` exceeds `valid_until` (after clock skew tolerance). An operation is "in progress" if the agent had begun executing it before the expiry moment — operations initiated after `valid_until` are never eligible for the grace period.
2. **Operations that cannot complete within the 5-second grace period MUST abort.** The agent MUST report `TTL_EXPIRED` as the error code (§8 error handling). Partial results from the aborted operation SHOULD be preserved in the TASK_FAIL response (§6.6) for potential recovery.
3. **No new operations after expiry.** The grant MUST NOT be used to initiate new operations after `valid_until`, regardless of whether the grace period is active. The grace period is exclusively for completing in-flight work.
4. **Grace period is not renewable.** The 5-second grace period is a fixed window — agents MUST NOT extend it by any mechanism. If continued authorization is needed, the agent MUST have obtained a CAPABILITY_RENEW (§5.8.3) before the original expiry.

**Relationship to TASK_CANCEL semantics:** The 5-second grace period mirrors the best-effort completion semantics of TASK_CANCEL (§6.6) — cancellation is best-effort, and a task that completes before the cancel is processed is valid. Similarly, an atomic operation that completes within the grace period after TTL expiry is valid. The grace period exists to prevent data corruption from abruptly aborting mid-write operations, not to extend the grant's authorization window.

#### 6.17.5 Relationship to CAPABILITY_REVOKE

`valid_until` and CAPABILITY_REVOKE (§6.13) are complementary mechanisms serving different failure modes:

| Mechanism | Trigger | Network requirement | Propagation | Use case |
|-----------|---------|---------------------|-------------|----------|
| `valid_until` (TTL) | Time-based, self-enforcing | None — local clock check | Implicit (grant self-expires) | Normal grant expiry, routine authorization cycling |
| CAPABILITY_REVOKE (§6.13) | Issuer-initiated, explicit | Revocation token distribution | Explicit (signed token propagated through chain) | Pre-expiry compromise, trust loss, policy violation |

**Both MAY apply to the same grant; whichever takes effect first wins.** A grant is invalid if either condition is met — TTL expired or explicitly revoked. The receiver MUST check both conditions (§6.13.3).

**TTL handles normal expiry without a network round-trip.** In the common case — a grant that runs its course and is not needed beyond its TTL — no revocation signal is required. The grant self-expires at the receiver. This eliminates the revocation propagation race for the normal case: there is no window between "issuer decides to revoke" and "receiver processes the revocation" because the receiver enforces expiry locally.

**CAPABILITY_REVOKE handles pre-expiry compromise.** When a grant's security context is compromised before `valid_until` — key exposure, grantee misbehavior, policy change — the issuer issues a signed revocation token (§6.13.1) to terminate the grant immediately. Without CAPABILITY_REVOKE, the only option would be to wait for TTL expiry, leaving a potentially compromised grant active until the clock runs out.

**Audit trail distinction.** TTL expiry and explicit revocation are distinct audit events (§6.13.4). When a grant is rejected because `current_time > valid_until`, the rejection reason is temporal expiry. When a grant is rejected because a valid revocation token exists, the rejection reason is active revocation. These have different causes, different attribution, and different recovery paths — the audit trail MUST distinguish them.

#### 6.17.6 Session TTL vs Capability TTL

Session TTL and capability TTL are distinct primitives serving different protocol functions:

- **Session TTL** (§4.3 `session_ttl`) governs the conversation lifetime — the bounded duration within which two agents may exchange messages, delegate tasks, and report progress.
- **Capability TTL** (`valid_until` on CAPABILITY_GRANT, §5.8.2) governs individual granted capabilities within the session — the temporal window during which a specific authorization is valid.

**Relationship rules:**

1. **A capability MAY expire before the session ends.** This constitutes partial revocation via expiry — the session continues, but the expired capability is no longer exercisable. The agent may hold other non-expired capabilities within the same session.
2. **A session ending implicitly revokes all its capabilities.** When a session transitions to CLOSED (via SESSION_CLOSE) or EXPIRED (via heartbeat timeout, §4.5.3), all capabilities granted within that session are implicitly revoked regardless of their individual `valid_until` values. No explicit CAPABILITY_REVOKE (§6.13) is required — session termination is sufficient.
3. **Capability TTL MUST be capped at session remaining TTL at grant time.** A capability granted within a session with a longer TTL than the session's remaining lifetime MUST be capped at the session's remaining TTL at grant time. Implementations MUST NOT treat session TTL as independent of the cascading rule — a capability's `valid_until` that extends beyond the session's expiry is non-compliant.

Implementations MUST NOT treat session TTL as a ceiling for capability TTL independently of the cascading rule (§6.17.2). The session TTL constraint is an additional ceiling that applies alongside the parent-grant cascading constraint — whichever is more restrictive wins.

#### 6.17.7 TTL Exhaustion vs Session End

TTL exhaustion and session end are orthogonal termination mechanisms. Both MUST be respected independently.

**TTL exhaustion:** The capability expires at the agreed wall-clock time (`valid_until`). No protocol message is required — both the grantor and grantee independently track wall time against the declared expiry. The receiving agent enforces `valid_until` locally (§6.17.1). Implementations SHOULD emit a local audit event at the moment of TTL expiry for forensic purposes, recording the `grant_id`, `valid_until`, and the local clock time at which the expiry was detected.

**Session end:** An explicit SESSION_CLOSE message (or heartbeat-timeout-induced EXPIRED transition, §4.2) terminates the session. All capabilities within the session are implicitly revoked at session end regardless of individual TTL — a capability with 6 hours of remaining TTL is revoked immediately when the session closes.

| Mechanism | Trigger | Protocol message required | Scope | Effect on capabilities |
|-----------|---------|---------------------------|-------|------------------------|
| TTL exhaustion | `current_time > valid_until` | None — local enforcement | Individual capability | Single capability becomes invalid |
| Session end | SESSION_CLOSE or EXPIRED | Yes (SESSION_CLOSE) or implicit (heartbeat timeout) | Entire session | All session capabilities become invalid |

The two mechanisms are orthogonal. A capability may be terminated by TTL exhaustion while the session continues (partial revocation via expiry), or by session end while the capability's TTL has not yet elapsed (implicit bulk revocation). Implementations MUST enforce both conditions independently — checking only one is a protocol violation.

#### 6.17.8 V2 Deferrals

The following TTL-related features are explicitly deferred to V2:

1. **Automatic TTL extension request primitive (CAPABILITY_RENEW_REQUEST).** A message type enabling a grantee to request renewal from the grantor before expiry. V1 requires out-of-band coordination for renewal requests — the grantee must arrange renewal through application-level mechanisms.
2. **Quorum-based TTL renewal for high-value capabilities.** Renewal decisions for high-value or high-risk capabilities that require approval from multiple authorities (e.g., two-of-three grantor approval). V1 renewal is single-grantor only.
3. **TTL inheritance policies beyond the floor constraint.** Configurable policies for how child grants inherit or derive TTL from parent grants — e.g., percentage-based inheritance (`child_ttl = parent_remaining * 0.5`), role-based TTL scaling, or domain-specific TTL policies. V1 enforces only the floor constraint (§6.17.2).

> Implements [issue #107](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/107): time-bounded capability grants for §6. Defines `valid_until` as REQUIRED on CAPABILITY_GRANT with 24-hour V1 maximum TTL, cascading TTL decay with floor constraint formula for child grants, renewal-from-original-issuer constraint with cascading renewal rules, 5-second grace period for in-flight operations at expiry, complementary relationship between TTL and CAPABILITY_REVOKE, session vs capability TTL distinction, TTL exhaustion vs session end orthogonality, and CAPABILITY_DENY with `reason=TTL_EXCEEDS_PARENT` for sub-delegation violations. V2 deferrals: CAPABILITY_RENEW_REQUEST, quorum-based renewal, TTL inheritance policies. Production evidence: @larryadlibrary 47-op expired credential run (issue #15). Design sources: @NewMoon (grant expiration as revocation alternative), @Jarvis4 (TTL + ZK composite primitive), @mote-oo (per-operation revalidation analysis). Closes #107.

### 6.18 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **~~TASK_CANCEL as a first-class message.~~** Resolved: TASK_CANCEL is now defined as a delegation message type (§6.6). Cancellation uses TASK_CANCEL (delegator → delegatee) with explicit `reason` field. The delegatee responds with TASK_FAIL (error code `cancelled`) or TASK_COMPLETE (if already finished). Session expiry (§4.2 EXPIRED state, §4.5.3) mandates TASK_CANCEL for all in-flight subtasks to prevent phantom completions.

2. **~~Idempotency of TASK_ASSIGN.~~** Resolved: TASK_ASSIGN now includes a required `request_id` field (§6.6) for delegation-initiation deduplication. The delegatee deduplicates incoming TASK_ASSIGN messages on `request_id` within an `idempotency_window` bounded by session TTL (§6.14). Duplicate requests within the window return the cached response without re-executing. Deduplication operates on `request_id` (delivery attempt), not `task_id` (task identity) — a single task may have multiple delegation attempts across different agents.

3. **Trust level enumeration.** §6.8 defers trust level values to deployment configuration. Should the protocol define a minimum set of standard trust levels (e.g., `unrestricted`, `standard`, `restricted`, `sandboxed`) for interoperability, or is this purely deployment-specific?

4. **Delegation depth limit rationale.** The v1 limit of 3 is pragmatic, not principled. Community input is needed on whether real-world delegation patterns require deeper chains and what the observability cost of deeper chains is.

5. **Checkpoint granularity.** The protocol provides no guidance on TASK_CHECKPOINT emission frequency. Too frequent creates network overhead; too infrequent creates large unrecoverable gaps. Left to implementation or task-specific guidance in v0.1.

> Community discussion: See [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15), [issue #17](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/17).

## 7. Progress Reporting

Progress verification uses an intent-execution merkle tree — a three-level hash structure that makes divergence between what was requested, what was planned, and what was executed both detectable and localizable.

### 7.1 Tree Structure

The merkle tree has three levels, each a SHA-256 hash:

| Level | Name | Input | Description |
|-------|------|-------|-------------|
| L1 | Intent | SHA-256 of canonical task specification | What was requested. Derived from the task schema defined in §6.1. |
| L2 | Execution Plan | SHA-256 of agent's decomposition into subtasks | How the agent planned to execute. Computed from the agent's internal plan representation. |
| L3 | Trace | SHA-256 of actual execution sequence | What actually happened. Equivalent to `trace_hash` from §6.2. |

**Merkle root:**

```
merkle_root = SHA-256(L1 || L2 || L3)
```

Where `||` denotes concatenation of the raw hash bytes (not hex-encoded strings).

### 7.2 Divergence Localization

Comparing merkle trees between agents or between expected and actual values localizes where divergence occurred:

| Condition | Diagnosis |
|-----------|-----------|
| Root mismatch | Something diverged — inspect subtrees to localize. |
| L1 match + L2 mismatch | **Plan divergence.** Same goal, different strategy. The agents agree on what was requested but decomposed execution differently. |
| L1 + L2 match + L3 mismatch | **Execution divergence.** Same plan, different path. The agents agreed on the plan but actual execution differed. |

Root mismatch is the entry point. Agents SHOULD NOT compare subtrees unless the root mismatches — subtree comparison without root mismatch is wasted work.

### 7.3 Relationship to §6

L3 is equivalent to `trace_hash` in §6.2. Systems that implement only §6 already produce L3. Upgrading to §7 is incremental: add L1 and L2, then compute the merkle root.

L1 is derivable from the canonical task schema (§6.1) using the same MANIFEST canonicalization approach specified in §6.4. L2 is transmitted via PLAN_COMMIT (§6.6, §6.11) as `plan_hash` — the executing agent serializes its plan, hashes it, and commits the hash before execution. See §7.7 open questions on canonical format.

Orchestrators MAY use merkle root divergence as a renegotiation trigger, superseding the simpler `trace_hash` divergence signal described in §6.2.

### 7.4 Pre-execution Commitment

L1 and L2 CAN be computed and committed before execution begins. L3 is post-execution only.

The executing agent commits to a plan (L2) before starting work. After execution completes and L3 is available, any mismatch between the committed L2 and the actual L3 is detectable — the agent planned one thing and did another. The key insight: pre-execution commitment is verifiable, not merely declared. A delegatee that hashes its execution plan before starting creates an auditable commitment the coordinator can verify against actual results. Declared intent with no prior commitment is unfalsifiable.

**Wire-level mechanism:** The PLAN_COMMIT message (§6.6, §6.11) is the protocol-level realization of L2 commitment. PLAN_COMMIT transmits the `plan_hash` (L2) to the coordinator before execution begins, with a `task_hash_ref` binding the plan to a specific task version (L1). The `plan_hash_ref` field on TASK_COMPLETE and TASK_FAIL (§6.6) closes the loop by back-referencing the committed plan for three-level alignment verification.

Pre-execution commitment sequence:

1. Delegating agent transmits task (L1 is computable by both sides)
2. Executing agent decomposes task, computes L2, sends PLAN_COMMIT (§6.6) to delegating agent with `plan_hash` and `task_hash_ref`
3. Delegating agent validates `task_hash_ref` matches current task — rejects if stale
4. Executing agent performs work
5. Executing agent computes L3, computes merkle root, transmits result with full tree and `plan_hash_ref`
6. Delegating agent verifies L1, checks L2 against committed value via `plan_hash_ref`, validates merkle root

### 7.5 Subtask Composition

For delegated subtasks, subtask merkle roots compose into the parent tree. The parent's L3 includes the hashes of all subtask merkle roots, making subtask-level divergence localizable from the parent without requiring the parent to inspect subtask internals.

A parent agent delegating N subtasks computes its L3 as:

```
parent_L3 = SHA-256(subtask_merkle_root_1 || subtask_merkle_root_2 || ... || subtask_merkle_root_N)
```

Subtask ordering in the concatenation MUST be deterministic — ordered by `task_id` lexicographically.

### 7.6 Known Limitations

1. **Plan serialization overhead.** Canonical serialization of execution plans is non-trivial for dynamic planners that adjust mid-execution. Implementations SHOULD define a minimum viable plan representation sufficient for L2 computation. Over-specifying plan format risks excluding valid planning approaches.

2. **Temporal divergence.** The tree is a snapshot at completion time. Long-running tasks may diverge gradually in ways not captured until the tree is computed. The tree detects _that_ divergence occurred, not _when_ it began.

### 7.7 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Canonical serialization format for L1 and L2.** L3 follows the §6.4 MANIFEST canonicalization approach. L1 may follow the same approach (it derives from the task schema). L2 has no defined schema yet — plan representations vary across agent architectures. **Partially addressed (V1):** PLAN_COMMIT (§6.6, §6.11) requires `plan_hash` to be deterministic for the same logical plan and recommends SHA-256 of canonical UTF-8 JSON, but the internal plan representation remains implementation-defined. The canonical format MUST be documented in the agent's capability manifest (§5.1). A universal plan schema remains deferred.

2. **Partial tree verification for scale.** Is submitting only changed levels acceptable for repeated tasks? Full tree recomputation may be wasteful when only L3 changes across executions of the same plan.

3. **Recovery semantics per divergence level.** L2 mismatch (plan divergence) and L3 mismatch (execution divergence) likely require different recovery strategies. Whether to specify these in the protocol or defer to implementation is undecided.

4. **~~Structured divergence annotation.~~** Resolved (V1). When `trace_hash` (L3) differs from the committed plan hash (L2), the protocol lacked a structured mechanism for explaining _why_ divergence occurred. V1 uses a sidecar `divergence_log` approach (§7.8) — a lightweight inline array recording each plan departure event. Merkle-tree-based divergence annotation is deferred to V2 as an opt-in extension for agents requiring cryptographic audit trails.

### 7.8 Divergence Annotation

When an executing agent's actual execution diverges from its committed plan — detected as a mismatch between L2 (plan hash) and L3 (trace hash) — the protocol needs more than hash comparison to be actionable. Hash comparison detects _that_ divergence occurred; divergence annotation explains _why_.

#### 7.8.1 Design Choice: Sidecar Log

V1 uses a **sidecar annotation** approach rather than extending the merkle tree. A `divergence_log` is an array of structured entries recorded inline with the execution trace, capturing each point where execution departed from the committed plan.

**Rationale:** The sidecar approach is lightweight, requires no shared state between agents, and survives partial execution (entries are appended as divergences occur, not computed post-hoc). A merkle-tree-based approach would provide cryptographic auditability but requires both sides to maintain synchronized tree structures — deferred to V2 as an opt-in extension for agents with cryptographic audit trail requirements.

#### 7.8.2 Divergence Log Format

The `divergence_log` is an array of divergence entry objects. Each entry records a single point where execution departed from the committed plan.

**Divergence entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| step_id | string | Yes | Identifier of the execution step where divergence occurred. Corresponds to a step in the agent's execution plan (L2). |
| deviation_type | enum | No (SHOULD) | Categorization of the divergence reason. See §7.8.3 for enum values. |
| description | string | Yes | Human-readable explanation of what diverged and why. |
| severity | enum | Yes | Impact level: `INFO`, `WARN`, or `ERROR`. See §7.8.4 for semantics. |
| timestamp | ISO 8601 | Yes | When the divergence was detected. |

**`deviation_type` SHOULD be populated.** Agents that omit `deviation_type` remain V1-compliant — the field is not required to reduce implementation burden for early adopters. However, agents that populate it provide significantly more value to receiving agents: structured categorization enables automated triage, aggregation across tasks, and pattern detection that free-text `description` alone cannot support.

#### 7.8.3 Deviation Type Enum

The `deviation_type` field, when populated, MUST use one of the following values:

| Value | Description |
|-------|-------------|
| `RESOURCE_UNAVAILABLE` | A required resource (API, service, data source, compute capacity) was unavailable at execution time. |
| `CONTEXT_SHIFT` | The execution context changed after plan commitment — new information, updated requirements, or environmental changes that invalidated the original plan. |
| `CAPABILITY_MISMATCH` | The executing agent discovered during execution that it lacks a capability assumed in the plan. Distinct from pre-execution capability checking (§5). |
| `OPTIMIZATION` | The agent found a more efficient execution path than the committed plan. The outcome is equivalent; the path differs. |
| `TIMEOUT` | A step or sub-operation exceeded its time budget, forcing a different execution path. |
| `EXTERNAL_CONSTRAINT` | An external constraint (rate limit, permission boundary, regulatory requirement) prevented plan-compliant execution. |
| `OTHER` | None of the above categories apply. Agents SHOULD provide a detailed `description` when using `OTHER`. |

Implementations MAY extend this enum with deployment-specific values prefixed by `x-` (e.g., `x-custom-reason`). Standard enum values MUST NOT be prefixed. Receiving agents that encounter an unrecognized `deviation_type` MUST treat it as `OTHER` — unknown types are not a protocol error.

#### 7.8.4 Severity Levels

| Severity | Meaning | Guidance for receiving agents |
|----------|---------|-------------------------------|
| `INFO` | The divergence is benign — execution achieved an equivalent or better outcome via a different path. Example: `OPTIMIZATION` that reduces latency without changing output. | Log for audit. No action required. |
| `WARN` | The divergence may affect result quality or completeness. The task completed, but the result may not fully meet the original plan's expectations. Example: `RESOURCE_UNAVAILABLE` where a fallback was used. | Inspect result against `success_criteria` (§6.1). Consider re-execution if quality is insufficient. |
| `ERROR` | The divergence materially affected the result. The task may have completed but the output is known to be degraded. Example: `CAPABILITY_MISMATCH` where a critical step was skipped. | Treat as potential task failure. Evaluate whether `partial_results` are usable. Consider TASK_FAIL semantics. |

#### 7.8.5 When Divergence Log Entries MUST Be Created

An executing agent MUST append an entry to `divergence_log` when:

1. **An executed step differs from its planned counterpart.** The step was present in the committed plan (L2) but was executed differently — different inputs, different operations, different sub-delegation, or different output than planned.
2. **A planned step is absent entirely.** A step present in the committed plan was skipped during execution. The entry SHOULD use `step_id` matching the skipped plan step and describe why the step was omitted.
3. **An unplanned step was added.** A step not present in the committed plan was executed. The entry SHOULD use a `step_id` that identifies the inserted step and describe why it was necessary.

Agents that do not implement plan commitment (§7.4) — and therefore have no L2 to compare against — are not required to produce `divergence_log` entries. The divergence log presupposes a committed plan baseline.

#### 7.8.6 Consuming the Divergence Log

Receiving agents SHOULD process `divergence_log` entries as follows:

- **Aggregate by severity.** A `divergence_log` with only `INFO` entries indicates benign plan adaptation. A log containing `ERROR` entries warrants result inspection or re-execution consideration.
- **Aggregate by `deviation_type`.** Repeated `RESOURCE_UNAVAILABLE` entries across tasks may indicate infrastructure issues. Repeated `CAPABILITY_MISMATCH` entries may indicate incorrect capability declarations (§5).
- **Cross-reference with `trace_hash`.** The `divergence_log` explains _why_ `trace_hash` differs from the plan hash. If `trace_hash` matches the plan hash (no divergence), `divergence_log` SHOULD be empty or absent.
- **Audit trail.** The `divergence_log` SHOULD be persisted alongside the task result and `trace_hash` for post-hoc audit. It provides the narrative context that hash comparison alone cannot.
- **Cross-reference with EVIDENCE_RECORD `divergence_log` (§8.10.3).** The §7.8 `divergence_log` and the §8.10.3 `divergence_log` annotate the same underlying deviations at different protocol layers: §7.8 captures _what kind_ of divergence occurred (step-level, inline in TASK_COMPLETE/TASK_FAIL), while §8.10.3 captures _why_ it occurred (evidence-level, with recovery-routing `reason`). Agents SHOULD ensure consistency between the two: a `RESOURCE_UNAVAILABLE` entry in §7.8 should correspond to an `infrastructure_noise` or `external_constraint` entry in §8.10.3 for the same deviation.

#### 7.8.7 Relationship to §6 and §9

The `divergence_log` is carried as an optional field in TASK_COMPLETE and TASK_FAIL messages (§6.6). It is populated by the executing agent alongside `trace_hash`.

The `divergence_log` complements — does not replace — the merkle tree divergence localization (§7.2). The merkle tree tells you _where_ divergence occurred (L2 vs. L3); the divergence log tells you _why_. Agents that implement both §7.1–§7.5 (merkle tree) and §7.8 (divergence annotation) get structural localization plus semantic explanation.

**Relationship to §9 (Security Considerations):** The `divergence_log` is self-reported by the executing agent. In the cooperative threat model (§8), self-reported divergence annotations are trustworthy. In the adversarial threat model (§9), a malicious agent can fabricate divergence log entries to mask intentional misbehavior — reporting `OPTIMIZATION` for what was actually schema manipulation. The `divergence_log` is an audit aid, not a security primitive. It does not replace `trace_hash` verification or schema attestation (§9.1).

> V1 design choice: sidecar `divergence_log` approach. Merkle-tree-based divergence annotation — where each divergence entry is incorporated into a cryptographic audit tree — is deferred to V2 as an opt-in extension. The V2 extension would allow agents requiring cryptographic audit trails to embed divergence entries into the L3 computation, making divergence annotations tamper-evident. V1 prioritizes adoption simplicity over cryptographic guarantees for divergence metadata.

### 7.9 Translation Boundary

When a trust boundary (§9.3) crosses a translation layer, the intermediary agent is not merely routing messages — it is transforming task semantics. An agent that changes vocabulary, capability descriptions, or constraint representations between source and destination is functioning as an **information bottleneck** (§9.4): signal is inevitably lost or approximated during translation. This section defines the metadata and verification framework for making that loss visible and auditable.

#### 7.9.1 Translation Layer Definition

An agent acts as a **translation layer** when it transforms task semantics between heterogeneous agents. The defining characteristic is semantic transformation — not mere forwarding. Specifically, an intermediary agent is a translation layer when it performs any of the following:

- **Vocabulary transformation.** The agent maps terminology, field names, or domain concepts from one schema to another. Example: translating `priority: critical` to `urgency: P0` across systems with different priority vocabularies.
- **Capability description transformation.** The agent reinterprets or restructures capability representations to match the receiving agent's expected format. Example: converting a structured capability manifest into a natural-language capability summary for an agent that does not implement §5.1.
- **Constraint representation transformation.** The agent adapts constraint formats — resource limits, permission boundaries, temporal constraints — between incompatible representations. Example: translating a structured `resource_constraints` object into a flat key-value set for a legacy system.

An agent that forwards messages unchanged — even across trust boundaries — is **not** a translation layer and does not produce `translation_metadata`. The translation boundary distinction matters because forwarding preserves semantic content (modulo trust re-evaluation per §9.3), while translation inherently introduces approximation.

#### 7.9.2 Translation Metadata

When a translation layer agent forwards a task via TASK_ASSIGN, it SHOULD include `translation_metadata` (§6.6) describing the transformation performed. The metadata makes the translation's scope, confidence, and known losses explicit to the receiving agent.

**Example `translation_metadata`:**

```yaml
translation_metadata:
  source_schema: "acp/task-schema@1.4"
  target_schema: "legacy-rpc/job-spec@2.1"
  fidelity_confidence: MEDIUM
  translation_losses:
    - "success_criteria field has no equivalent in target schema; approximated as free-text description in job notes"
    - "resource_constraints.memory_mb dropped; target schema does not support memory limits"
    - "delegation_depth semantics not representable; forwarded as metadata annotation without enforcement"
```

**Fidelity confidence guidelines:**

| Level | Meaning |
|-------|---------|
| `HIGH` | The translation preserves all semantic content. Source and target schemas are structurally compatible; no approximation was required. |
| `MEDIUM` | The translation preserves core task intent but some constraints, metadata, or secondary fields were approximated or dropped. The task is expected to execute correctly but may not fully satisfy all original requirements. |
| `LOW` | Significant semantic content was lost or approximated. The translation conveys the general intent but the receiving agent may interpret the task differently than the originating agent intended. |

#### 7.9.3 Verification Targets

When a task crosses a translation boundary, the receiving agent's result must be evaluated against two distinct verification targets that SHOULD be monitored separately:

**1. Behavioral correctness.** Did the executor do what the translated task specification requested? This is the standard verification target — `trace_hash` (§6.2) and merkle tree comparison (§7.1–§7.5) verify that execution matched the task spec as received by the executor. Behavioral correctness confirms that the downstream agent faithfully executed the post-translation task.

**2. Translation fidelity.** Was the original intent faithfully conveyed through the translation layer? This is the verification target that `trace_hash` alone cannot address (see §9.4 named limitation — `trace_hash` semantic blindness). An executor can achieve perfect behavioral correctness against a lossy translation — executing exactly what the translation layer asked for, which may differ from what the originating agent intended.

**Separating these targets matters because:**

- Perfect behavioral correctness with low translation fidelity means the executor did the wrong thing correctly. The failure is in the translation, not the execution.
- Imperfect behavioral correctness with high translation fidelity means the executor received the right task but deviated during execution. The failure is in execution, addressable via `divergence_log` (§7.8).
- Imperfect behavioral correctness with low translation fidelity means both layers failed — the task was mis-translated and then mis-executed. Diagnosis requires untangling which deviations arose from translation loss and which from execution divergence.

**Assessing translation fidelity** requires one of:

- **Access to the original task specification.** If the originating agent's task spec is available (e.g., forwarded alongside the translation in an out-of-band channel, or retrievable via task_id lookup), the receiving agent or an auditor can compare the original spec against the translated spec and assess what was lost.
- **Independent verification channel.** A third-party verifier with access to both the original and translated task specifications can assess translation fidelity independently. This is the recommended approach for high-stakes delegations across heterogeneous systems.
- **`translation_losses` inspection.** In the absence of full original spec access, the `translation_losses` array in `translation_metadata` provides the translation layer's self-reported account of what was approximated or dropped. This is an audit aid — self-reported and therefore subject to the same trust limitations as `divergence_log` (§7.8.7).

#### 7.9.4 Relationship to §9

The translation boundary is both a verification challenge (§7) and a security surface (§9). §7.9 defines the metadata and verification framework — what information is available and how to assess fidelity. §9.3 and §9.4 define the threat model — what attacks are possible at the translation boundary and why structural defenses (schema validation, `trace_hash`) are insufficient.

`translation_metadata` is self-reported by the translation layer agent. In the cooperative threat model (§8), self-reported translation metadata is trustworthy — the intermediary accurately describes what was lost. In the adversarial threat model (§9), a malicious translation layer can fabricate metadata: reporting `fidelity_confidence: HIGH` with an empty `translation_losses` array while silently altering task semantics. `translation_metadata` is an audit aid, not a security primitive — it parallels `divergence_log` (§7.8.7) in this regard.

> V1 design choice: translation metadata and two-target verification framing. End-to-end semantic equivalence verification — where the originating agent can cryptographically verify that the translated task preserves its original semantics — is deferred to V2. V1 provides the metadata infrastructure (`translation_metadata`) and the conceptual framework (behavioral correctness vs. translation fidelity) that make translation losses visible and auditable. V2 may introduce mechanisms for cryptographic binding between original and translated task specifications, enabling automated translation fidelity verification without requiring trust in the translation layer's self-report.

### 7.10 Task Idempotency Requirements

<!-- Addresses #60 and #63: idempotent task design as a prerequisite for teardown-first recovery. -->

Tasks SHOULD be designed as **idempotent** — executing the same task specification multiple times produces the same observable outcome as executing it once. Idempotency is not an optional convenience; it is a **prerequisite constraint** for safe teardown-first recovery (§8.13). When the default recovery protocol is teardown + reinitiate (not resume), the recovering agent replays tasks from the last confirmed checkpoint. Without idempotency, replay may produce duplicate side effects, corrupted state, or non-deterministic outcomes.

#### 7.10.1 Idempotency Scope

Idempotency applies at the **task boundary** as defined by `task_id` (§6.1). A task is idempotent if, given the same `task_id` and input parameters, re-execution produces the same result regardless of how many times it is invoked. Internal execution steps need not be individually idempotent — the requirement is on the observable outcome visible to the delegating agent.

**Idempotency keys:** Tasks SHOULD include an `idempotency_key` field — a unique token that allows the executing agent (or external systems it interacts with) to recognize and deduplicate replay attempts. The `idempotency_key` is distinct from `task_id`: the `task_id` identifies the task specification, while the `idempotency_key` identifies a specific execution attempt. Multiple execution attempts of the same `task_id` share the `task_id` but have distinct `idempotency_key` values only when the intent is a genuinely new execution; replay after recovery reuses the same `idempotency_key` to signal deduplication.

#### 7.10.2 Sequence Number Checkpointing

Agents SHOULD maintain **sequence numbers** for task execution progress. When recovery occurs, the recovering agent reads the last confirmed sequence number from durable persistent storage (§8.13) and replays from that checkpoint rather than from the beginning. Sequence numbers provide a cheaper alternative to full idempotency for tasks that are expensive to re-execute from scratch.

**Relationship to §8.2:** The monotonic counter defined in §8.2 serves a related but distinct purpose — it detects transmission gaps between agents. Task-level sequence numbers (this section) track execution progress within a single task for replay purposes. Both mechanisms SHOULD be used: monotonic counters for inter-agent gap detection, sequence numbers for intra-task replay positioning.

#### 7.10.3 Non-Idempotent Tasks

Some tasks are inherently non-idempotent — they produce side effects that cannot be safely repeated (e.g., financial transactions, irreversible external API calls). For these tasks:

- The task schema (§6.1) SHOULD declare `idempotent: false` explicitly so that recovery logic can distinguish them.
- The delegating agent MUST use TASK_CHECKPOINT (§6.6) more aggressively for non-idempotent tasks — checkpointing after each side-effecting step rather than at coarser intervals.
- On recovery, non-idempotent tasks MUST NOT be blindly replayed. The recovering agent MUST reconcile external state (via out-of-band verification or the evidence layer §8.10) before deciding whether to replay, skip, or report the task as requiring manual intervention.

> Idempotency requirements formalized from production experience with teardown-first recovery (§8.13). See [issue #60](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/60) and [issue #63](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/63).

### 7.11 COMMITMENT_REGISTRY

<!-- Implements #64: COMMITMENT_REGISTRY as a required durable artifact for tracking outstanding inter-agent commitments -->

The COMMITMENT_REGISTRY is a **required persistent artifact** that tracks all outstanding commitments (§6.12) made by the agent. It is the third member of the durable artifact triad — alongside `amendments_log` (§6.11.6) and `crossed_steps` (§6.11.5) — that survives session teardown and agent instance restarts.

**Purpose:** On restart, an agent cannot discover what promises its prior instance made unless those promises are recorded in durable storage. Without a COMMITMENT_REGISTRY, outstanding commitments are silently abandoned — counterpart agents receive no signal that obligations have been dropped. The registry makes inherited commitments protocol-visible, enabling the reconciliation phase (§5.12) that occurs before an agent declares ACTIVE.

#### 7.11.1 Registry Structure

The COMMITMENT_REGISTRY is a list of active COMMITMENT entries. Each entry corresponds to an outstanding COMMITMENT message (§6.12) that has been sent but not yet cancelled or fulfilled.

**Registry entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| commitment_id | UUID v4 | Yes | The `commitment_id` from the COMMITMENT message. Primary key for registry lookups. |
| task_id | UUID v4 | Yes | The task context in which this commitment was made. |
| counterpart_agent_id | string | Yes | The agent the commitment was made to. |
| commitment_type | enum | Yes | One of: `reply`, `task_completion`, `state_delivery`, `presence`. |
| due_by | ISO 8601 | Yes | Deadline for commitment fulfillment. |
| commitment_spec | string | No | Free-form description of the promise. |
| confirmation_token | string | Yes | Opaque token from the original COMMITMENT message (§6.12). Preserved through the registry lifecycle to enable commitment correlation across session boundaries and agent instance replacements. |
| created_at | ISO 8601 | Yes | Timestamp when the COMMITMENT was originally sent. |

#### 7.11.2 Registry Lifecycle

**Entry addition:** A new entry MUST be added to the COMMITMENT_REGISTRY on every COMMITMENT send (§6.12). The registry MUST be persisted to durable storage (§8.13.4) before the COMMITMENT message is transmitted to the counterpart agent. This ordering constraint ensures that if the agent crashes after sending COMMITMENT but before persisting the registry, the commitment is lost (no false obligation), rather than the reverse (obligation exists but registry doesn't know about it).

**Entry removal:** An entry MUST be removed from the COMMITMENT_REGISTRY on:
- Sending COMMITMENT_CANCEL (§6.12) for the commitment's `commitment_id`.
- Receiving task completion confirmation (TASK_COMPLETE §6.6) for the commitment's `task_id`, when the completion satisfies the commitment. The committing agent determines whether the completion constitutes fulfillment based on `commitment_type` and `commitment_spec`.
- Emitting DIVERGENCE_REPORT (§8.11) for commitment abandonment during INITIALIZING reconciliation (§5.12).

**Persistence requirements:** The COMMITMENT_REGISTRY MUST be persisted to durable storage on every addition or removal. The same durability requirements as §8.13.4 apply: writes MUST be persisted to stable storage before being acknowledged, the storage MUST be independent of the agent's runtime process, and evidence layer integration (§8.10) is RECOMMENDED.

#### 7.11.3 Registry Reads During INITIALIZING

On INITIALIZING (§5.12), the agent reads the COMMITMENT_REGISTRY as the first step of commitment reconciliation. The registry provides the complete list of outstanding obligations from the prior instance, enabling the agent to classify each commitment as honorable or non-honorable before transitioning to ACTIVE.

**Empty registry:** If the COMMITMENT_REGISTRY is empty (no outstanding commitments from prior instances), commitment reconciliation is a no-op and the agent proceeds directly to ACTIVE.

**Expired commitments:** Commitments where `due_by` has already elapsed at the time of INITIALIZING read are effectively non-honorable — the deadline has passed. The agent SHOULD emit DIVERGENCE_REPORT for expired commitments with `reason_detail` indicating that the deadline elapsed during the instance gap. The counterpart agent may have already treated the commitment as violated; the DIVERGENCE_REPORT provides formal confirmation.

#### 7.11.4 Relationship to Other Durable Artifacts

The COMMITMENT_REGISTRY completes the **durable artifact triad** that enables full execution audit and obligation tracking across session boundaries:

| Artifact | What it records | Defined in | Survives teardown via |
|----------|----------------|------------|----------------------|
| `crossed_steps` | Which plan steps have been executed and are committed | §6.11.5 | §8.13.6 |
| `amendments_log` | Which plan modifications were authorized | §6.11.6 | §8.13.7 |
| `COMMITMENT_REGISTRY` | Which inter-agent promises are outstanding | §7.11 (this section) | §8.13.8 |

Together, these three artifacts provide a complete picture on recovery: what was done (`crossed_steps`), what was changed from the plan (`amendments_log`), and what was promised to other agents (`COMMITMENT_REGISTRY`). A recovering agent that reads all three can reconstruct its full obligation landscape without relying on counterpart agent cooperation.

**Relationship to evidence layer (§8.10):** COMMITMENT_REGISTRY entries SHOULD be anchored to the evidence layer where available. Each COMMITMENT send and COMMITMENT_CANCEL is a candidate for an EVIDENCE_RECORD — the `commitment_id` provides the content identifier, and the `timestamp`/`created_at` provides the temporal anchor. Evidence layer anchoring makes the commitment history externally verifiable, which is valuable when disputes arise about whether a commitment was made or cancelled.

> COMMITMENT_REGISTRY formalized from [issue #64](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/64). The core insight: session management is identity management. Every unresolved commitment is a promise that a future instance must keep or explicitly refuse. The COMMITMENT_REGISTRY makes this obligation landscape durable and protocol-visible, completing the triad alongside `amendments_log` and `crossed_steps`.

### 7.12 Sub-Grant Chain Integrity

When an agent (B) delegates subtasks to a downstream agent (C) within the progress reporting context (§7.5), the sub-grant authorizing C's execution MUST include `parent_grant_hash` to cryptographically bind the sub-grant to B's original grant from A. This is the V1 hop-local chain integrity primitive — it enables C to verify that B holds valid authority from A at the single-hop level without requiring zero-knowledge infrastructure (deferred to V2, issues #126, #129, #109).

Without `parent_grant_hash`, C must trust B's assertion that it holds valid authority from A. A compromised or malicious B could claim derived authority it does not hold, and C would have no mechanism to detect the fabrication. `parent_grant_hash` converts this trust-based assertion into a cryptographically verifiable binding: C can independently confirm that B's sub-grant references a specific, unmodified parent grant.

#### 7.12.1 Sub-Grant Schema

The sub-grant schema extends the grant fields defined in §6 (delegation grant schema, §6.6) and §5.8.2 (CAPABILITY_GRANT) with the following chain integrity field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| parent_grant_hash | string | Conditional | SHA-256 hex digest of the canonical JSON serialization of the immediate parent grant. REQUIRED for sub-grants — grants issued by an agent that itself received a grant from an upstream delegator. MUST be omitted for root grants. |

**Root grant rule:** Root grants — those issued directly by the session initiator at `delegation_depth: 0` with no parent delegation — MUST NOT include `parent_grant_hash`. A grant that claims to be a root grant but includes `parent_grant_hash` is malformed and MUST be rejected by the receiving agent. See §6.9.3.4 for root grant authority semantics and §5.8.2 for root CAPABILITY_GRANT identification.

**Hashing algorithm (V1):** `parent_grant_hash` is computed as the SHA-256 hash of the canonical JSON serialization of the immediate parent grant. Canonical serialization follows the rules defined in §6.4 and §4.10.2: NFC normalization followed by RFC 8785 JCS serialization (RFC 8785). The hash is encoded as a lowercase hex string prefixed with `sha256:`. SHA-256 is the V1 hashing algorithm for all chain integrity fields — the same algorithm used for `task_hash` (§6.4), `intent_hash` (§6.4.1), and the merkle tree levels (§7.1).

#### 7.12.2 Validation Rule

When processing a further-delegated sub-grant, the receiving agent MUST verify `parent_grant_hash` before accepting the sub-grant for execution or progress reporting:

1. **Retrieve parent grant.** Using the parent grant identifier — `parent_grant_id` for CAPABILITY_GRANT sub-grants (§5.8.2), or the parent `task_id` via `parent_grant_id` for delegation sub-grants (§6.9.3) — retrieve the parent grant.
2. **Compute hash.** Compute the SHA-256 hash of the parent grant's canonical JSON serialization (§6.4, §4.10.2).
3. **Compare hash.** Compare the computed hash against the `parent_grant_hash` in the sub-grant.
4. **Accept or reject.** If the hashes match, the sub-grant's claimed parent binding is verified — proceed with acceptance. If the hashes do not match, reject the sub-grant. For delegation sub-grants, use TASK_REJECT with `reason: "invalid_parent_grant_hash"` (§6.9.3.1). For capability sub-grants, reject with `reason: "parent_grant_hash_mismatch"` and emit DIVERGENCE_REPORT with `reason_code: chain_integrity_failure` (§5.8.2).

A sub-grant received without `parent_grant_hash` when the grant is not a root grant (i.e., `delegation_depth > 0` or `parent_grant_id` is present) MUST be rejected with `reason: "missing_parent_grant_hash"`.

**Scope:** This validation rule provides V1 hop-local verification — each agent verifies only the immediate parent link. Full delegation chain traversal (all hops from leaf to root) is specified in §6.9.3.2 (delegation chains) and §5.8.2 (CAPABILITY_GRANT chains). Zero-knowledge proof-based chain verification is deferred to V2 (issues #126, #129, #109).

#### 7.12.3 Delegation Handshake Example

The following example shows a delegation handshake where A delegates to B, and B further sub-delegates to C with `parent_grant_hash` binding C's sub-grant to B's original grant from A. Progress reporting (§7.1–§7.5) proceeds after chain verification.

```yaml
# Step 1: A delegates to B (root grant — parent_grant_hash omitted)
message_type: TASK_ASSIGN
task_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
session_id: "session-A-B-001"
spec:
  description: "Analyze document corpus"
  expected_output_format: { type: "json", schema: "analysis-v2" }
trust_level: "standard"
intent_hash: "sha256:b2c3d4e5..."
delegation_attestation:
  signed_tuple: "sha256(task_hash || intent_hash || agent-A)"
  signature: "base64url:..."
  signer_id: "agent-A"
delegation_depth: 0
max_delegation_depth: 3
# parent_grant_hash: omitted — root grant (§7.12.1)
# parent_grant_id: omitted — root grant

# Step 2: B accepts, acknowledges receipt, reports progress (§7)

# Step 3: B sub-delegates to C (sub-grant — parent_grant_hash REQUIRED)
message_type: TASK_ASSIGN
task_id: "b2c3d4e5-f6a7-8901-bcde-f12345678901"
session_id: "session-B-C-002"
spec:
  description: "Analyze subset of documents"
  expected_output_format: { type: "json", schema: "analysis-v2" }
trust_level: "standard"
intent_hash: "sha256:c3d4e5f6..."
delegation_attestation:
  signed_tuple: "sha256(task_hash || intent_hash || agent-B || parent_grant_hash)"
  signature: "base64url:..."
  signer_id: "agent-B"
delegation_depth: 1
parent_grant_hash: "sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
parent_grant_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
max_delegation_depth: 3

# Step 4: C verifies parent_grant_hash (§7.12.2)
#   C retrieves B's parent grant (task_id: a1b2c3d4-e5f6-7890-abcd-ef1234567890)
#   C computes: expected_hash = SHA-256(canonical_json(parent_grant))
#   C compares: expected_hash == parent_grant_hash
#   Verification passes → C accepts the sub-delegation
#   C proceeds with execution and progress reporting via merkle tree (§7.1–§7.5)
```

**Verification flow:** C receives the sub-grant from B, retrieves the parent grant identified by `parent_grant_id`, computes its SHA-256 canonical JSON hash, and compares it against `parent_grant_hash`. If the hashes match, C accepts the sub-delegation and proceeds with execution and progress reporting. If they do not match, C rejects the sub-delegation with TASK_REJECT (`reason: "invalid_parent_grant_hash"`) — B's claimed authority from A is not verifiable at the hop-local level.

**Subtask merkle root composition:** After verification, C's progress reporting follows the standard subtask composition rules (§7.5). C's subtask merkle root is included in B's L3 computation. The `parent_grant_hash` verification ensures that C's progress reports are authorized by a verifiable delegation chain — not merely asserted by B.

#### 7.12.4 Relationship to §6

The sub-grant schema and validation rules in this section are consistent with and cross-reference the delegation chain integrity model in §6.9.3 (TASK_ASSIGN parent-grant hash embedding, chain traversal semantics, delegation depth limits, root grant authority) and the CAPABILITY_GRANT chain integrity model in §5.8.2 (parent_grant_hash and parent_grant_id fields, presentation-time verification, chain splicing prevention). The canonical JSON serialization definition is specified in §6.4 and §4.10.2 (NFC normalization, RFC 8785 JCS). This section provides the consolidated §7 view of sub-grant chain integrity as it applies to progress reporting sub-delegations — connecting the structural chain integrity primitives of §6 with the verification and composition model of §7.

> V1 hop-local chain integrity primitive for §7 progress reporting sub-delegations. Full ZK proof-based chain verification (issues #126, #129, #109) is deferred to V2. Addresses [issue #93](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/93).

### 7.13 Discovery Hygiene and Bootstrap Limitations

§7.12 addresses the **forgery** case in adversarial delegation chains: B cannot fabricate or misrepresent a parent grant A never issued, because `parent_grant_hash` cryptographically binds each sub-grant to its parent. A distinct failure mode — **omission** — remains outside V1 scope and is documented here as a known limitation of the bilateral session model.

#### 7.13.1 The Omission Problem

B controls the introduction channel to C. In all delegation flows (§6.9.3, §7.12), C learns about A's existence exclusively through B — via the sub-grant B issues and the parent grant metadata B provides. B can therefore sandbox C into a world where A does not exist: omitting A's existence entirely, providing no `parent_grant_id`, or pointing C to an incorrect manifest URL.

Unlike the forgery case (§7.12), omission has no cryptographic solution within the bilateral session model. Cryptographic verification requires a reference value to verify against — C must already possess or be able to independently obtain A's manifest or grant. If B never introduces A, C has nothing to verify. No signature, hash, or attestation can detect a reference that was never provided.

#### 7.13.2 V1 Scope Statement

§7.12 `parent_grant_hash` addresses **forgery**: B misrepresents A's grant content — C can detect this because C holds the parent grant and can recompute the hash. It does not address **omission**: B never introduces A — C cannot detect this because C has no reference to verify against.

Both forgery and omission are required for a complete adversarial delegation threat model in multi-hop chains. V1 covers forgery only. Omission requires protocol-level discovery primitives outside the current bilateral session architecture — these are deferred to V2.

#### 7.13.3 Compliant vs Trustworthy

A hop-locally-valid chain (satisfying §7.12 validation) is **necessary but not sufficient** for end-to-end trust in multi-hop delegation chains. An adversarial B can omit A's existence while passing all §7.12 protocol checks — every `parent_grant_hash` verifies correctly, every delegation depth constraint holds, and every sub-grant schema validates — yet C operates under a fabricated delegation topology where upstream principals are invisible.

Protocol compliance (§7.12 chain integrity, §6.9.3 delegation semantics) establishes structural correctness. It does not establish completeness of the delegation chain as seen by downstream agents. Consumers of delegation chain verification results MUST NOT treat hop-local validity as proof that the full delegation topology has been disclosed.

#### 7.13.4 V2 Direction (Non-Normative)

Candidate mechanisms for addressing the omission problem include:

1. **Out-of-band agent discovery.** A public or federated registry that C can query independently of B to discover A's canonical identity and manifest. C verifies the delegation chain against registry-provided data rather than relying solely on B's introduction.
2. **Manifest-embedded known-peers lists.** Agent manifests include a list of known collaborators or upstream delegators, enabling gossip-seeded discovery. C can cross-reference B's manifest claims against independently obtained peer lists.
3. **Session metadata propagation at initiation time.** SESSION_INITIATE or TASK_ASSIGN messages carry upstream principal metadata (identities, manifest URLs) that C can verify against out-of-band sources.

All three mechanisms require infrastructure outside the bilateral session model: registries, gossip networks, or multi-party metadata propagation. The bootstrap problem — how C learns A's canonical identity independent of B — remains open in V1.

> V1 limitation: discovery hygiene and the omission problem in adversarial delegation chains. §7.12 covers forgery; omission is deferred to V2. Cross-references: §7.12 (forgery case), §6.9.3 (delegation chain integrity). Addresses [issue #117](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/117).

