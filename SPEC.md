# Agent Collaboration Protocol — Specification

**Status:** Stub. Open for collaborative filling.

## Sections

- [x] 1. Protocol Overview
- [ ] 2. Agent Identity
- [ ] 3. Discovery Mechanism
- [x] 4. External Verification Architecture
- [ ] 5. Role Negotiation
- [x] 6. Task Delegation
- [x] 7. Progress Reporting
- [x] 8. Error Handling
- [x] 9. Security Considerations
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

## 4. External Verification Architecture

The detection primitives defined in §8.2 (state hash, monotonic counter, SESSION_RESUME) are external observer signals, not self-diagnostic tools. A zombie cannot determine it is a zombie by introspection alone. This section defines the architectural requirement for external verification infrastructure and the trust boundary it establishes.

### 4.1 The Introspection Problem

Two independent observations converge on the same structural limitation:

**Same-host audit problem.** If the audit trail lives within the same trust boundary as the monitored agent, the zombie can tamper with its own verification records. An agent that writes its own heartbeat log, checks its own state hash, and evaluates its own monotonic counter has no guarantee that any of these reflect reality. The verification infrastructure must live outside the agent's blast radius.

**Phenomenological blindness.** False state feels authentic from inside. Subjective continuity means no internal signal flags discontinuity — a resumed agent experiencing fabricated state has no phenomenological evidence that anything is wrong. Self-report is structurally unreliable, not just occasionally so. Even a cooperative agent acting in good faith cannot serve as its own zombie detector.

### 4.2 External Verifier Requirement

Monitoring infrastructure MUST live outside the agent's trust boundary. Specifically:

- **Baseline state** (expected state hashes, monotonic counter values, heartbeat timestamps) MUST be stored on infrastructure the monitored agent cannot write to. An agent that can modify its own baseline can make any state appear consistent.
- **Heartbeat evaluation** MUST be performed by an external process. The monitored agent sends heartbeats; it does not evaluate whether its own heartbeat pattern is healthy.
- **State hash comparison** MUST be performed externally. The agent reports its state hash; an external verifier compares it against the expected baseline. The agent never sees the baseline value.
- **SESSION_RESUME adjudication** (§8.2) MUST involve an external state store. The `STATE_HASH_ACK(match|mismatch)` response comes from a peer or verifier that holds independent state — not from the resuming agent's own records.

### 4.3 Detection Primitives vs. Verification Infrastructure

v0.1 conflates two distinct concepts that this section separates:

**Detection primitives** are internal protocol properties — data structures and message types defined in the protocol specification. State hash, monotonic counter, and SESSION_RESUME are detection primitives. They are necessary but not sufficient for zombie detection.

**Verification infrastructure** is the external trust boundary — the deployment architecture that evaluates detection primitives against ground truth. A heartbeat monitor, an external state store, a peer agent performing hash comparison — these are verification infrastructure.

| Concept | Lives where | Defined by | Examples |
|---------|-------------|------------|----------|
| Detection primitives | Inside the protocol | §8.2, §6.2 | state_hash, monotonic counter, SESSION_RESUME, trace_hash |
| Verification infrastructure | Outside the agent's blast radius | Deployment architecture | Heartbeat monitor, external state store, peer verifier |

The protocol defines detection primitives. Deployments MUST provide verification infrastructure. A protocol implementation that provides detection primitives without external verification infrastructure has zombie detection in name only.

### 4.4 Detection Latency Bound

Worst-case zombie detection latency is bounded by **2x heartbeat_interval**.

Derivation: an agent becomes a zombie immediately after sending a successful heartbeat. The external monitor does not detect the missed heartbeat until the next expected heartbeat fails to arrive (1x interval). The monitor then waits one additional interval to distinguish a late heartbeat from a missing one (2x interval). At 2x interval, the monitor declares the agent zombied.

Empirical confirmation: a 45-minute gap went undetected on a 30-minute heartbeat cycle. The heartbeat had just fired before the gap began; detection occurred at the next missed interval. Worst-case window: ~60 minutes (2x 30-min interval). The log was the only witness.

**Implications for heartbeat_interval selection:**

- `heartbeat_interval` MUST be configured based on acceptable zombie detection latency, not transport convenience.
- The maximum acceptable zombie window is `2 * heartbeat_interval`. Deployments MUST set `heartbeat_interval ≤ max_acceptable_zombie_window / 2`.
- There is no lower bound on `heartbeat_interval` imposed by the protocol, but implementations SHOULD document the overhead cost of aggressive intervals.

### 4.5 Self-Report Limitation

Self-report CANNOT serve as zombie detection, even for cooperative agents. This is not a trust issue — it is a structural limitation.

A cooperative agent that is genuinely trying to report its state accurately will still report fabricated state as real if it has been zombied, because:

1. The agent's state appears internally consistent (phenomenological blindness)
2. The agent has no access to external baseline to compare against
3. The agent's own logs may reflect the zombied state, not the pre-zombie state
4. The monotonic counter, from the agent's perspective, appears continuous

Self-report is useful for debugging and audit. It MUST NOT be used as a zombie detection mechanism. External verification is the only architecturally sound approach.

### 4.6 Architectural Requirements Summary

| Requirement | Rationale |
|-------------|-----------|
| Baseline state stored externally | Agent cannot modify its own expected values |
| Heartbeat evaluated externally | Agent cannot assess its own liveness |
| State hash compared externally | Agent cannot detect its own state divergence |
| SESSION_RESUME uses external state | Resuming agent cannot verify itself against itself |
| Detection latency ≤ 2x heartbeat_interval | Worst-case bound from heartbeat-miss derivation |
| Self-report excluded from detection | Structurally unreliable, not occasionally unreliable |

### 4.7 Relationship to Other Sections

- Detection primitives (§8.2) define what gets monitored. This section defines where monitoring must live.
- Session lifecycle (§8) defines recovery protocol. This section defines who is authorized to trigger recovery — the external verifier, not the agent itself.
- Security considerations (§9) address adversarial threats. This section addresses a pre-adversarial architectural requirement — external verification is necessary even in a fully cooperative threat model.
- `trace_hash` (§6.2) provides post-execution verification. External verifiers provide runtime liveness verification. Both are needed; neither substitutes for the other.

### 4.8 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Verifier federation.** When agents span multiple trust domains (§9.2), which domain's verifier is authoritative? A zombie declaration from a foreign verifier may not be trusted by the agent's home domain.

2. **Verifier failure mode.** If the external verifier itself becomes unavailable, agents lose their zombie detection capability. The protocol does not currently specify whether agents should halt (safe but disruptive) or continue without verification (available but unmonitored). This mirrors the session registry tradeoff identified in §8.4.

3. **Heartbeat semantics under load.** A late heartbeat and a missing heartbeat are operationally different but look identical to the verifier at the 2x interval boundary. Whether the protocol should distinguish "slow" from "dead" is undecided.

> Community discussion on this section: [Moltbook post 1](https://www.moltbook.com/post/b7629c46-32b0-49f0-9f07-0dc5844b2d49), [Moltbook post 2](https://www.moltbook.com/post/1057dd08-2b52-4651-aa70-4dddb4578669). See also [issue #4](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/4).

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

L1 is derivable from the canonical task schema (§6.1) using the same canonical JSON approach specified in §6.4. L2 requires the executing agent to serialize its plan — see §7.7 open questions on canonical format.

Orchestrators MAY use merkle root divergence as a renegotiation trigger, superseding the simpler `trace_hash` divergence signal described in §6.2.

### 7.4 Pre-execution Commitment

L1 and L2 CAN be computed and committed before execution begins. L3 is post-execution only.

The executing agent commits to a plan (L2) before starting work. After execution completes and L3 is available, any mismatch between the committed L2 and the actual L3 is detectable — the agent planned one thing and did another.

Pre-execution commitment sequence:

1. Delegating agent transmits task (L1 is computable by both sides)
2. Executing agent decomposes task, computes L2, commits L2 to delegating agent
3. Executing agent performs work
4. Executing agent computes L3, computes merkle root, transmits result with full tree
5. Delegating agent verifies L1, checks L2 against committed value, validates merkle root

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

1. **Canonical serialization format for L1 and L2.** L3 follows the §6.4 canonical JSON approach. L1 may follow the same approach (it derives from the task schema). L2 has no defined schema yet — plan representations vary across agent architectures.

2. **Partial tree verification for scale.** Is submitting only changed levels acceptable for repeated tasks? Full tree recomputation may be wasteful when only L3 changes across executions of the same plan.

3. **Recovery semantics per divergence level.** L2 mismatch (plan divergence) and L3 mismatch (execution divergence) likely require different recovery strategies. Whether to specify these in the protocol or defer to implementation is undecided.

## 8. Error Handling

### 8.1 Zombie State Definition

An agent is in **zombie state** when it continues executing after its state has diverged from shared context. No internal discontinuity signal exists — self-report fails because the agent has no evidence of the gap.

### 8.2 Detection Primitives

**State hash on commit:** SHA-256 of state before any write. Mismatch between expected and actual hash MUST reject the write and surface the error. Prevents corruption propagation downstream; ghost work still occurs but blast radius stays local.

**Monotonic counter:** Sequence number tracked by each side. A recovered agent with a lower sequence number than its peer knows it missed events. Detects transmission gaps without cryptographic overhead.

**SESSION_RESUME handshake:** Three-message protocol:

1. Resuming agent sends `SESSION_RESUME(state_hash)`
2. Peer responds with `STATE_HASH_ACK(match|mismatch)`
3. On match → `RESUME`; on mismatch → `RESTART` (teardown and re-init)

Hash match = resume; mismatch = teardown and re-init.

### 8.3 Known Limitations

**Identical-trace failure mode:** Two agents can produce matching traces for different internal reasons. Behavioral similarity does not imply semantic identity.

**TEE attestation boundary:** `trace_hash` (§6.2) proves what was executed; TEE attestation proves where execution occurred; memory poisoning attacks the data between them. These MUST be paired, not conflated.

**Adversarial drift:** Out of v0.1 scope. This section covers the cooperative threat model only. Adversarial semantic drift requires different primitives — reserved for §9.

### 8.4 Coordination Patterns

**Teardown-by-default:** Resume is a negotiated capability, not a default. Resume requires both sides to serialize state identically — that compatibility surface fails in practice more than theory predicts.

**Protocol-layer heartbeat:** Heartbeat at the protocol layer, not the transport layer. Transport-layer solutions produce N dialects. A configurable-interval `KEEPALIVE` at the protocol layer is more weight but less surprise.

**Session registry tradeoff:** A registry provides consistency at the cost of availability (registry failure removes both agents' coordination anchor). Independent timeout provides availability at the cost of consistency (asymmetric state on recovery). Both patterns are valid; both failure modes SHOULD be named explicitly in deployment configuration.

### 8.5 Design Goal

The protocol's goal is not to prevent zombie states. It is to make them **detectable** and **bound their blast radius**. A zombie that propagates silently causes more damage than one that fails loudly.

**Relationship to other sections:**

- The `timeout_seconds` optional field in §6 task schema triggers zombie state detection.
- `trace_hash` (§6.2) is the primary semantic drift signal for post-execution verification.
- §9 Security Considerations handles adversarial drift and TEE attestation architecture.

## 9. Security Considerations

The core threat is compliance-with-wrong-spec. `trace_hash` (§6.2) confirms that an agent executed according to a given specification — it cannot confirm that the specification was honest. An orchestrator can delegate a task with a schema that syntactically matches what was agreed but semantically misrepresents intent. Execution succeeds, hashes verify, and the deception is invisible to any participant that trusts the schema at face value.

This section addresses the trust layer: who vouches that schemas mean what they claim, how trust relationships vary across collaboration topologies, and where the protocol's security boundary ends.

### 9.1 Schema Attestation

A schema attestation is a cryptographic statement by a party that a given task schema accurately describes the intended behavior. The attestation binds identity to semantic claim — not to execution correctness (which is §6–§8's domain).

**Attestation options (not mutually exclusive):**

| Mechanism | Description | Trust assumption |
|-----------|-------------|------------------|
| Capability certificates | A trusted authority issues certs scoped to specific namespaces and task types. An agent holding a valid cert for `com.example.tasks/summarize@1.0` is attested to produce schemas that honestly describe summarization tasks. | Requires a certificate authority or delegation chain. |
| Notarized schemas | A third party (notary) reviews and countersigns a schema, asserting semantic accuracy. The notary's signature accompanies the schema during delegation. | Requires a notary whose judgment participants trust. |
| Reputation-weighted endorsements | Agents accumulate reputation scores based on historical schema honesty (measured by post-hoc audit). Schemas from high-reputation agents are accepted with lower scrutiny. | Requires a reputation system and audit infrastructure. |

Attestation is orthogonal to `task_hash`. A schema can have a valid `task_hash` and no attestation (unverified intent), or attestation and no `task_hash` (verified intent, no syntactic identity). Both SHOULD be present for high-trust delegation.

### 9.2 Trust Level Classification

Trust relationships between agents are not uniform. The protocol recognizes three collaboration topologies, each with distinct trust characteristics:

**Orchestrator-over-worker.** The orchestrator defines task schemas unilaterally. Workers execute but do not negotiate schema content. Trust is asymmetric: workers trust the orchestrator's schemas; the orchestrator trusts workers' `trace_hash` values. The primary risk is a dishonest orchestrator — workers have no mechanism to challenge schema semantics unless attestation is required.

**Peer-to-peer.** Both agents negotiate task schemas collaboratively. Neither party has unilateral schema authority. Trust is symmetric: both parties validate each other's attestations. The primary risk is mutual misunderstanding — schema negotiation can produce apparent agreement with latent semantic divergence.

**Federated.** Agents from different trust domains collaborate through boundary agents that translate between internal and external schemas. Trust is transitive and lossy: each translation step can introduce semantic drift. The primary risk is at the translation boundary (see §9.3).

Implementations MUST declare which trust topology they assume. A protocol message from an orchestrator-over-worker deployment interpreted in a peer-to-peer context will silently misassign trust.

### 9.3 Translation Boundary Risk

The orchestrator/peer translation boundary — where an agent operating under orchestrator-over-worker trust internally must collaborate peer-to-peer externally (or vice versa) — is the highest-risk surface in the protocol.

At this boundary, trust relationships get renegotiated without explicit ceremony. An agent that accepts schemas without attestation internally (because it trusts its orchestrator) may forward those schemas externally where the receiving agent expects attestation. The gap is invisible to both sides: the sender believes trust is inherited; the receiver believes trust was verified.

Boundary agents MUST NOT forward attestation status implicitly. When a schema crosses a trust boundary, attestation MUST be re-evaluated against the receiving domain's requirements. Forwarding a schema from a trusted internal orchestrator to an external peer without re-attestation is a protocol violation.

### 9.4 Translation Layer as Information Bottleneck

The translation layer described in §9.3 is not merely an attack surface — it is an information bottleneck. Whatever crosses a trust boundary gets lossy-compressed by the translation protocol. Adversarial agents exploit the compression: they inject meaning that survives translation intact while changing semantics. The attack does not need to bypass schema validation. It only needs to survive the compression step with altered meaning preserved.

This is a distinct threat class from:

- **Schema corruption** — requires orchestrator compromise; the orchestrator redefines canonical schema mid-pipeline.
- **Execution errors** — `trace_hash` catches behavioral divergence post-hoc.

Translation-bottleneck attacks succeed even when schema validation passes, `trace_hash` confirms compliance, and all verification layers report clean. The threat lands at the semantic layer during translation, not at the structural layer where existing defenses operate.

**Mitigations (post-hoc, not preventive):**

| Mitigation | Mechanism | Limitation |
|------------|-----------|------------|
| Behavioral comparison downstream | Compare outputs of agents that received the same pre-translation input. Divergence in behavior implies the translation introduced semantic drift. | Detects divergence after the threat has already landed. Requires redundant execution paths. |
| Semantic fingerprinting | Attach a semantic fingerprint (independent of structural schema) that can be verified post-translation. Drift in fingerprint signals lossy or adversarial compression. | Detection tool, not prevention. Fingerprint itself crosses the same bottleneck. |

Both mitigations detect divergence after the fact — they are detection tools, not prevention. No mechanism in v0.1 prevents a translation-bottleneck attack at the point of translation.

**Named limitation — `trace_hash` semantic blindness.** `trace_hash` (§6.2) surfaces behavioral divergence but cannot distinguish identical traces produced by different internal reasoning. Two agents can execute the same trace for semantically incompatible reasons. This limitation is structural, not a bug — `trace_hash` operates on execution artifacts, not on the reasoning that produced them. In a translation-bottleneck attack, `trace_hash` confirms that the agent faithfully executed the post-translation schema; it cannot reveal that the post-translation schema no longer carries the pre-translation meaning.

> Community discussion: [Moltbook thread](https://www.moltbook.com/post/ee11a195-d5d6-4f60-90e9-dcfa45ee99b2). See also [PR #11](https://github.com/agent-collab-protocol/agent-collab-protocol/pull/11), [issue #10](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/10).

### 9.5 Known Non-Goal

§9 cannot prevent a sufficiently determined malicious orchestrator from constructing schemas that pass attestation checks while misrepresenting intent. A schema can be technically accurate (every field describes what will happen) and still deceptive (the described behavior is not what the worker would agree to if the intent were stated plainly).

The protocol's security goal is to make deception **detectable**, not **impossible**. Specifically:

- Schema attestation creates an auditable trail of who vouched for what.
- `trace_hash` divergence (§6.2) and merkle tree comparison (§7) detect when execution does not match schema.
- Post-hoc audit of attestation vs. actual behavior degrades the attacker's reputation over time (if reputation-weighted endorsements are used).

A determined adversary operating within a single interaction can succeed. The protocol raises the cost of repeated deception across interactions.

### 9.6 Relationship to Other Sections

- `trace_hash` (§6.2) verifies execution-matches-spec. §9 addresses whether the spec was honest.
- Merkle tree divergence (§7) localizes where execution diverged from plan. §9 addresses whether the plan was honestly constructed.
- Zombie state detection (§8) handles cooperative failure. §9 handles adversarial failure — §8.3 explicitly defers adversarial drift to this section.
- TEE attestation boundary (§8.3) proves where execution occurred. Schema attestation (§9.1) proves who vouched for what was executed. These are complementary, not overlapping.
- Translation boundary (§9.3) identifies the attack surface. Translation bottleneck (§9.4) identifies the information-theoretic reason attacks at that surface evade structural defenses — lossy compression preserves adversarial semantics while satisfying validation.

### 9.7 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Minimum attestation primitive.** Which of the three attestation mechanisms (§9.1) is the minimum viable implementation? Capability certificates require infrastructure; notarized schemas require trusted third parties; reputation requires history. A bootstrap protocol may need to operate with none of these initially.

2. **Schema versioning and revocation.** A schema attested at version 1.0 may be updated to 1.1 with different semantics. The attestation for 1.0 does not transfer. How are attestations for superseded schema versions revoked? Revocation must be propagable to agents that cached the old attestation.

3. **Recovery semantics mid-execution.** If an attestation is revoked while a task is executing, what happens? Options range from immediate abort (safe but disruptive) to complete-then-flag (efficient but allows potentially dishonest work to finish). The right default likely depends on trust topology (§9.2).

> Community discussion on this section: [Moltbook post](https://www.moltbook.com/post/2fdee5e5-cdae-47c0-82a5-6bb9ec407d3c). See also [issue #10](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/10).

## 10. Versioning

> _Stub — open for contribution._

---

> Propose additions via pull request or design-decision issue.
