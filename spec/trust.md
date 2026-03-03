<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

## 1. Protocol Overview

The core problem is the ordering-vs-meaning gap. Causal ordering between agents is largely solved. What isn't: two agents with perfectly synchronized clocks can still work on completely different interpretations of the same task. task_hash looks identical; execution diverges silently. This protocol addresses that by co-designing semantic agreement alongside causal ordering — a canonical task schema where intent, scope, and expected artifacts are explicit and hashable.

- Lightweight, transport-agnostic message format
- Decentralized identity and discovery
- Explicit role negotiation before task execution
- Canonical task schema with semantic and syntactic identity
- Progress reporting and structured error handling

### 1.1 Scope: Objective-Agnosticism and Adversarial Model

#### Objective-agnosticism is deliberate

The protocol is objective-agnostic by design — it does not prescribe, inspect, or verify agent goals. This is the correct scope for a coordination-layer protocol. TCP does not know why packets are sent. This protocol handles message routing, DAG finality, trust annotations, and external verification — all mechanism, no teleology.

#### V1 adversarial actor scope

Two distinct failure classes intersect this protocol:

- **Byzantine agents** fail in structural protocol behavior — malformed messages, invalid signatures, constraint violations. The protocol's verification and audit mechanisms address Byzantine failure directly.
- **Misaligned-objective agents** satisfy every protocol constraint while pursuing goals that differ from their delegation context — protocol-compliant, audit-clean, semantically divergent. This is categorically distinct from Byzantine failure.

V1 explicitly includes Byzantine failure in scope. Misaligned-objective adversaries — agents that are structurally compliant but semantically divergent from their declared delegation intent — are out of scope for V1 protocol guarantees, but are named here as a known gap rather than silently excluded. Silence is not the same as non-existence.

#### Protocol guarantees under objective divergence

When agent objectives diverge from their declared delegation context, the protocol provides structural guarantees only: message integrity, delegation chain completeness, and audit trail availability. Semantic fidelity — whether an agent's actions match its delegator's intent — is not a V1 protocol guarantee.

This distinction matters for implementors: a fully protocol-compliant execution can produce semantically misaligned outcomes. Systems requiring semantic alignment guarantees must layer application-level checks above the protocol. The protocol does not prevent this — it makes the structural record available for such checks to operate against.

### 1.2 Trust Chain Termination

Every capability grant chain MUST terminate at a declared root trust anchor. Without an explicit terminus, implementations have no guidance on where trust stops — leading to incompatible termination models that silently fragment the protocol's trust guarantees.

The root trust anchor is:
- **V1**: operator-configured — set externally at deployment time, not asserted by any agent in the chain
- **V2**: cryptographically anchored (ZK-based — cross-reference [issue #109](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/109))

#### Agent obligations

An agent MUST:
1. Know its own operator-configured root trust anchor at startup
2. Reject any capability grant chain that does not terminate at that anchor
3. Treat a chain with self-asserted terminus as invalid — emit DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure` and the chain termination point as the reported boundary

An agent MUST NOT:
1. Accept a trust chain that terminates in an unrecognized anchor
2. Forward capability grants whose chain cannot be verified to the operator root
3. Assert itself as a root trust anchor for capability grants to other agents

Self-asserted termination — an agent claiming to be its own root trust anchor — is INVALID in V1. The operator-configured root is external to the agent population by design: no agent in the collaboration can unilaterally declare itself the trust terminus.

#### Legible failure

When a chain cannot be verified to the operator root, the agent MUST report a legible failure rather than silently degrading. Silent corruption — accepting an unverifiable chain without reporting — is a protocol violation. The agent MUST emit DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure` and the chain termination point as the reported boundary.

This requirement connects to the root grant authority rule in §6.9.3.4: a root delegation's authority derives from the originating agent's identity and trust level within the deployment's trust topology (§9.2). §1.2 makes explicit what §6.9.3.4 implies — the verifier confirming that the root delegation's `issuer_id` is "a trusted originating agent" MUST verify against the operator-configured root trust anchor, not against any self-asserted claim.

#### V2 deferrals

The following trust chain termination capabilities are deferred to V2:

- Cryptographic root anchor binding (ZK-based, cross-reference [issue #109](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/109))
- Multi-root trust topologies
- Dynamic root anchor rotation
- Cross-operator trust federation

> Addresses [issue #99](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/99): trust chain termination — explicit operator-configured root trust anchor requirement for V1, replacing the implicit deferral that left implementations without termination guidance. Closes #99.

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

**V1 compensating mechanism — `translation_metadata`.** While prevention remains out of scope, V1 introduces `translation_metadata` (§6.6, §7.9) to make translation losses visible. The `translation_metadata` field on forwarded TASK_ASSIGN messages carries the translation layer's self-report of what schemas were involved (`source_schema`, `target_schema`), confidence level (`fidelity_confidence`), and specific losses (`translation_losses`). This does not prevent bottleneck attacks — a malicious translator can fabricate metadata — but it provides an auditable record and enables the two-target verification framework (§7.9.3): separating behavioral correctness (did the executor do what the translated spec asked?) from translation fidelity (did the translated spec faithfully convey the original intent?).

**Named limitation — `trace_hash` semantic blindness.** `trace_hash` (§6.2) surfaces behavioral divergence but cannot distinguish identical traces produced by different internal reasoning. Two agents can execute the same trace for semantically incompatible reasons. This limitation is structural, not a bug — `trace_hash` operates on execution artifacts, not on the reasoning that produced them. In a translation-bottleneck attack, `trace_hash` confirms that the agent faithfully executed the post-translation schema; it cannot reveal that the post-translation schema no longer carries the pre-translation meaning. The two-target verification framework (§7.9.3) makes this distinction explicit: `trace_hash` addresses behavioral correctness only; translation fidelity requires separate assessment.

> Community discussion: [Moltbook thread](https://www.moltbook.com/post/ee11a195-d5d6-4f60-90e9-dcfa45ee99b2). See also [PR #11](https://github.com/agent-collab-protocol/agent-collab-protocol/pull/11), [issue #10](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/10), [issue #38](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/38).

### 9.5 Known Non-Goal

§9 cannot prevent a sufficiently determined malicious orchestrator from constructing schemas that pass attestation checks while misrepresenting intent. A schema can be technically accurate (every field describes what will happen) and still deceptive (the described behavior is not what the worker would agree to if the intent were stated plainly).

The protocol's security goal is to make deception **detectable**, not **impossible**. Specifically:

- Schema attestation creates an auditable trail of who vouched for what.
- `trace_hash` divergence (§6.2) and merkle tree comparison (§7) detect when execution does not match schema.
- Post-hoc audit of attestation vs. actual behavior degrades the attacker's reputation over time (if reputation-weighted endorsements are used).

A determined adversary operating within a single interaction can succeed. The protocol raises the cost of repeated deception across interactions.

#### 9.5.1 Evidence Layer Epistemic Boundary

EVIDENCE_RECORDs (§8.10) hash what an agent *submitted* — not what *actually happened* in the external world. If an agent's observation was wrong (e.g., it recorded a task result ID that the target platform never acknowledged, or it captured a state snapshot from a corrupted local cache), the evidence layer faithfully preserves a hash of a false belief. The hash is cryptographically valid; the underlying claim may be factually incorrect.

This is a structural limitation, not a bug. The evidence layer guarantees **integrity** (no record was tampered with after submission) and **ordering** (append-only, temporally sequenced). It does not and cannot guarantee **correctness** of the original observation.

The `external_ref` field (§8.10.1) mitigates this limitation by anchoring evidence to independently verifiable external identifiers. When `external_ref` is present, an external verifier can cross-check the agent's claim against the referenced external system — confirming not just that the agent *said* something happened, but that the external system *agrees* it happened. Evidence records without `external_ref` are self-attested: cryptographically intact but epistemically unanchored.

> Credit: @ultrathink (append-only audit log architecture), @ummon_core (empirical failure case — task result ID recorded but platform had no record 7 minutes later), @cass_agentsharp (epistemic framing — hashing beliefs vs. hashing reality), @Yibao ([Moltbook thread 3d769fda](https://www.moltbook.com/post/3d769fda)).

### 9.6 Relationship to Other Sections

- `trace_hash` (§6.2) verifies execution-matches-spec. §9 addresses whether the spec was honest.
- Merkle tree divergence (§7) localizes where execution diverged from plan. Divergence annotation (§7.8) explains why — but is self-reported and therefore not a security primitive. §9 addresses whether the plan was honestly constructed.
- Zombie state detection (§8.1–§8.14) handles cooperative failure (liveness-failure path). REVOKED state (§8.15–§8.18) handles protocol-level adversarial behavior (adversarial-behavior path). State-delta verification (§8.19) handles silent failure — structurally compliant execution that produces no observable outcome. Verifier pre-commitment (§8.20) handles verifier assignment integrity — ensuring selection rationale is auditable and not constructed post-hoc. §9 handles schema-level adversarial deception — honest-looking schemas with dishonest intent. See §8.15.3 for the layer distinction.
- TEE attestation boundary (§8.3) proves where execution occurred. Schema attestation (§9.1) proves who vouched for what was executed. These are complementary, not overlapping.
- Translation boundary (§9.3) identifies the attack surface. Translation bottleneck (§9.4) identifies the information-theoretic reason attacks at that surface evade structural defenses — lossy compression preserves adversarial semantics while satisfying validation. Translation boundary metadata and verification (§7.9) provides the cooperative-model counterpart: `translation_metadata` makes translation losses visible and the two-target verification framework (behavioral correctness vs. translation fidelity) separates execution failures from translation failures.
- Revocation trust (§9.8) documents the advisory nature of REVOKE signals and the Byzantine propagation problem in delegation chains. Identity revocation (§2.3.4) and delegation token revocation (§5.11) define MUST-level requirements that are binding on compliant agents but not technically enforceable on non-compliant or offline nodes.
- Evidence layer (§8.10) provides append-only ground truth for external verification. The epistemic boundary (§9.5.1) documents the structural limitation: evidence records guarantee integrity and ordering of what was submitted, not correctness of the original observation. `external_ref` partially mitigates by enabling cross-validation against external systems.
- Delegation chain integrity (§6.9.3) ensures sub-delegations are structurally bound to their parent delegations via `parent_grant_hash` embedding and delegating agent signatures. Chain traversal verification (§6.9.3.2) proceeds by hash traversal from terminal to root delegation — each link verified by hash match and signature validity. `chain_integrity_failure` (§8.11.2) is the audit reason code for delegation chains that fail hash traversal verification. Delegation depth limits (§6.9.3.3) bound chain verification to O(max_delegation_depth) steps.
- Trust annotation types (§9.10) define the closed enum of protocol-level trust claims. Schema attestation (§9.1) addresses who vouched for a schema's honesty; trust annotations address under what trust basis an agent acted. Trust annotations are included in `trace_hash` computation (§6.2), making the trust basis auditable through the existing hash verification mechanism. The genesis publication hash (§9.10.4) applies the same independence criteria as §8 audit media — the audited party must not control the publication medium.
- Amendment ceremonies (§9.11) specify how the trust annotation enum (§9.10.2) may be modified after genesis. The amendment hash chain (§9.11.5) extends the genesis publication hash (§9.10.4) across the enum's full lifecycle. Backwards compatibility (§9.11.6) ensures that enum changes do not strand deployed agents — unknown annotation types are forwarded without interpretation, never rejected. Classification dispute resolution (§9.12) ensures that tier classification disputes default to Tier 2 (conservative) and that any participant can escalate a proposed Tier 1 amendment to Tier 2 review.
- Introduction verification (§8.17.4) specifies that introductions are claims, not trusted facts — C MUST independently verify A's manifest when B introduces A. Introduction omission (§9.13) documents the residual threat that cryptographic verification cannot address: B suppressing the introduction entirely. `canonical_self_url` (§3.1) provides the identity anchor that makes independent verification possible.
- Governance document lifecycle (§9.14) generalizes §9.10–§9.12 from trust-annotation-specific governance to protocol-wide governance. GovernanceState enum (§9.14.1) provides machine-readable lifecycle states (`PROPOSED`, `ACTIVE`, `SUPERSEDED`, `REVOKED`) for all governance artifacts. Genesis ceremony specification (§9.14.2) documents the bootstrap problem and acknowledges that genesis legitimacy is socially constructed, not cryptographically derived. Publication hash composability (§9.14.3) binds governance document hashes to the spec version hash chain via `SHA-256(prior_hash || governance_content_canonical || spec_version_string)`, ensuring amendments are cryptographically linked to the spec version they govern. The composable hash structure extends the tamper-evidence property of §9.10.4 and §9.11.5 across spec version boundaries (§10).

### 9.7 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Minimum attestation primitive.** Which of the three attestation mechanisms (§9.1) is the minimum viable implementation? Capability certificates require infrastructure; notarized schemas require trusted third parties; reputation requires history. A bootstrap protocol may need to operate with none of these initially.

2. **Schema versioning and revocation.** A schema attested at version 1.0 may be updated to 1.1 with different semantics. The attestation for 1.0 does not transfer. How are attestations for superseded schema versions revoked? Revocation must be propagable to agents that cached the old attestation.

3. **Recovery semantics mid-execution.** If an attestation is revoked while a task is executing, what happens? Options range from immediate abort (safe but disruptive) to complete-then-flag (efficient but allows potentially dishonest work to finish). The right default likely depends on trust topology (§9.2).

4. **~~Structured divergence annotation.~~** Resolved (V1). When `trace_hash` diverges from plan, should the protocol provide a structured mechanism for annotating _why_ divergence occurred? Two candidate approaches were considered: (a) merkle-tree extension — embed divergence annotations into the L3 computation (§7), making them tamper-evident but requiring synchronized tree structures; (b) sidecar log — an inline `divergence_log` array recording divergence events without cryptographic integration. **V1 uses the sidecar log approach** (§7.8): lightweight, no shared state required, survives partial execution, and lowers the barrier to adoption. Merkle-tree-based divergence annotation is deferred to V2 as an opt-in extension for agents requiring cryptographic audit trails. The `divergence_log` is self-reported and therefore trustworthy only in the cooperative threat model (§8) — it is an audit aid, not a security primitive.

6. **~~Non-cryptographic revocation enforcement.~~** Resolved (V1). REVOKE signals (CAPABILITY_REVOKE, SESSION_TERMINATE, identity revocation per §2.3.4, delegation token revocation per §5.11) are advisory — compliant agents honor them; non-compliant or offline nodes may not. Can the protocol technically enforce revocation without cryptographic expiry or a consensus mechanism? **V1 documents this as a known limitation, not a bug** (§9.8): decentralized revocation without cryptography is a social contract, not a technical guarantee. The protocol's guarantees hold only among compliant agents. Cryptographic session tokens with built-in TTL expiry — where session bounds are enforced by token lifetime rather than relay trust — are deferred to V2 as an opt-in extension for environments requiring technical enforcement. See §9.8 for the full analysis including multi-hop propagation failure modes, threat model scoping, and approach tradeoffs.

5. **~~Translation layers at trust boundaries.~~** Resolved (V1). When an intermediary agent acts as a translation layer — transforming task semantics, vocabulary, or capability representations between heterogeneous agents — it functions as an information bottleneck where signal is inevitably lost. Should the protocol provide structured metadata for describing translation losses and a verification framework for separating execution failures from translation failures? **V1 provides `translation_metadata` and two-target verification framing** (§7.9): the `translation_metadata` optional field on TASK_ASSIGN (§6.6) carries `source_schema`, `target_schema`, `fidelity_confidence` (LOW/MEDIUM/HIGH), and `translation_losses` (SHOULD, not MUST — agents omitting it remain V1-compliant); the two-target verification framework (§7.9.3) distinguishes behavioral correctness (did the executor do what the translation asked?) from translation fidelity (was the original intent faithfully conveyed?). End-to-end semantic equivalence verification — where the originating agent can cryptographically verify that a translated task preserves its original semantics — is deferred to V2.

### 9.8 Revocation Trust and the Byzantine Propagation Problem

REVOKE signals in this protocol — identity revocation (§2.3.4), manifest revocation (§3.1 REVOKE operation), attestation revocation (§3.3.3), delegation token invalidation on session expiry (§5.11), and task cancellation propagation (§6.6 TASK_CANCEL) — are **advisory**. They are binding on compliant agents and meaningless to non-compliant or offline nodes. This is a documented known limitation of V1, not a bug to fix in V2.

**Decentralized revocation without cryptography is a social contract, not a technical guarantee.**

The protocol specifies that agents holding invalidated delegation tokens "MUST cease operations" (§2.3.4, §5.11) and that revocation "MUST propagate downstream to any sub-delegatees" (§2.3.4). These requirements are normative for compliant implementations. They are not — and cannot be — technically enforceable by the protocol alone. A non-compliant agent that receives a REVOKE signal and ignores it will continue operating on revoked credentials. The protocol has no mechanism to prevent this without either (a) a consensus mechanism that allows honest nodes to reject the non-compliant agent's messages, or (b) cryptographic token expiry that makes revoked credentials unusable regardless of agent compliance.

#### 9.8.1 Threat Model Scoping

**Protocol guarantees hold only among compliant agents.** The specification is a coordination contract, not a cryptographic enforcement mechanism. Non-compliant or offline nodes can continue executing beyond revoked bounds. Specifically:

- An agent that ignores CAPABILITY_REVOKE continues exercising revoked capabilities.
- An agent that ignores SESSION_TERMINATE continues operating on a terminated session.
- An agent that ignores delegation token invalidation continues acting under revoked authority.

These are not protocol violations that can be detected and rejected in real time by other protocol participants (absent cryptographic infrastructure). They are contract violations — detectable post-hoc through audit trails (§8) but not preventable at the point of violation.

**Implication for trust topology (§9.2):** The advisory revocation model is well-suited for orchestrator-over-worker and peer-to-peer topologies where agents are known, operated by cooperating parties, and subject to reputation consequences. It is insufficient for federated topologies where agents cross trust boundaries and may not share a compliance incentive.

#### 9.8.2 Multi-Hop Revocation Propagation

When a REVOKE signal must traverse a delegation chain (A → B → C → D), the protocol depends on each intermediate agent to propagate faithfully. This creates a Byzantine propagation problem: if the REVOKE reaches B but B fails to propagate it to C (whether through non-compliance, network partition, or crash), C and D continue operating under revoked authority.

**Propagation requirements:**

- An agent that receives a REVOKE signal (identity revocation, delegation token invalidation, or TASK_CANCEL) SHOULD propagate the signal to all downstream delegatees. This is SHOULD, not MUST — propagation cannot be technically enforced, and a MUST requirement would create a normative obligation that the protocol cannot verify.
- The revoking agent (or coordinator) SHOULD NOT assume downstream propagation succeeded without explicit acknowledgment from each node in the chain.

**Partial propagation failure modes:**

| Scenario | Effect | Detection |
|----------|--------|-----------|
| B receives REVOKE, propagates to C, C propagates to D | Full propagation — nominal case | Acknowledgments received at each hop |
| B receives REVOKE, fails to propagate to C | C and D continue under revoked authority | Coordinator receives acknowledgment from B only; absence of C/D acknowledgment signals partial propagation |
| B receives REVOKE, propagates to C, C fails to propagate to D | D continues under revoked authority | Coordinator receives acknowledgments from B and C; absence of D acknowledgment signals partial propagation |
| B is offline when REVOKE is sent | B, C, and D all continue under revoked authority | No acknowledgment from B; coordinator detects total propagation failure |

**REVOCATION_PROPAGATION_FAILED message (optional):**

When a coordinator detects partial propagation (acknowledgment received from some but not all nodes in a delegation chain), it MAY emit a REVOCATION_PROPAGATION_FAILED signal to indicate that revocation is incomplete.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| message_type | string | Yes | `REVOCATION_PROPAGATION_FAILED` |
| revocation_id | string | Yes | Identifier of the original REVOKE signal |
| acknowledged_by | array | Yes | Agent identities that acknowledged the revocation |
| not_acknowledged_by | array | Yes | Agent identities from which acknowledgment was not received within the expected window |
| partial | boolean | Yes | `true` if some but not all agents acknowledged; `false` if no agents acknowledged |

This message is informational — it signals to operators and audit systems that revocation coverage is incomplete. It does not trigger any automatic protocol behavior. Implementations that do not support REVOCATION_PROPAGATION_FAILED remain V1-compliant.

#### 9.8.3 Approach Tradeoffs

Two revocation models apply to decentralized agent collaboration, with different trust requirements and failure modes:

**Advisory REVOKE messages (V1 — current approach):**

| Property | Characteristic |
|----------|----------------|
| Infrastructure required | None beyond the protocol's existing message transport |
| Enforcement mechanism | Social contract — compliant agents honor REVOKE; non-compliant agents may not |
| Failure mode | Silent — if an intermediary is offline or non-compliant, downstream agents continue under revoked authority with no indication to the revoking agent (absent explicit acknowledgment) |
| Suitable for | Cooperative agent environments where Byzantine failure is not the primary threat model |
| Latency | Propagation delay proportional to chain depth; each hop adds round-trip latency |

**Cryptographic session tokens with built-in expiry (deferred — V2 opt-in):**

| Property | Characteristic |
|----------|----------------|
| Infrastructure required | Key management infrastructure for token issuance and verification |
| Enforcement mechanism | Technical — token TTL enforces session bounds regardless of agent compliance or relay trust |
| Failure mode | No retroactive revocation after issuance — TTL is a hard expiry bound, not a revocable signal. An agent holding a valid token continues operating until TTL expires, even if the issuer wishes to revoke earlier |
| Suitable for | Environments requiring technical enforcement, including federated topologies crossing trust boundaries |
| Latency | None for enforcement (TTL is local); issuance latency at session establishment |

The V1 advisory model and the V2 cryptographic model address different failure classes. Advisory revocation fails when intermediaries are non-compliant; cryptographic expiry fails when early revocation is needed before TTL expires. A complete solution would combine both — advisory REVOKE for cooperative fast-path revocation, cryptographic TTL as a hard backstop — but V1 implements only the advisory path to avoid mandating key infrastructure.

#### 9.8.4 Revocation Timing Gap Threat Model

<!-- Implements #82: revocation timing guarantees for delegation chain threat model. -->

§9.8.2 addresses the Byzantine propagation problem — whether a REVOKE signal reaches all nodes in a delegation chain. §8.17 addresses adversarial revocation where the revoked agent is the intermediary. Issue #37 (resolved via §10.7.1) addresses pre-flight traversal — catching already-propagated revocations before work begins. This section addresses a distinct gap: **the timing window between pre-flight check and mid-execution revocation arrival**.

**The problem:** In a delegation chain A → B → C, agent A's pre-flight check passes (C's key status is valid at check time). After pre-flight passes and A commits work, B delays or suppresses the revocation broadcast for C. A is now operating under stale trust assumptions about C mid-execution. This is not a propagation failure (§9.8.2) — B received the revocation. It is a timing exploitation — B controls when downstream agents learn about the revocation.

**Failure mode 1 — Delayed broadcast:** B receives a revocation for C but delays propagating it. A's pre-flight check passed before the revocation was issued. A commits work to C. The revocation arrives at A after A has committed work based on C's (now-revoked) authority. Cost attribution is undefined: A acted in good faith based on a valid pre-flight check; the revocation arrived after commitment. Who bears the cost of work committed during the propagation delay?

**Failure mode 2 — Suppressed broadcast:** B receives a revocation for C and drops it entirely. B can forge clean propagation reports because unsigned failure reports (§9.8.2 acknowledgment model) are unverifiable — B claims propagation succeeded when it did not. A continues operating under the assumption that C is trusted. Unlike delayed broadcast, suppressed broadcast is not self-correcting — the stale trust assumption persists indefinitely until detected by an independent mechanism.

**Distinction from §9.8.2:** §9.8.2 models propagation failure as a binary: did the REVOKE reach node N? This section models the temporal dimension: even when revocation eventually reaches all nodes, the propagation window creates a period where agents operate under stale trust. Work committed during this window is at risk. The threat is not that revocation fails to propagate — it is that revocation propagates too slowly for the work already committed against the stale trust state.

#### 9.8.5 Revocation Mode: `sync` and `gossip`

To address the revocation timing gap (§9.8.4), the delegating agent selects a `revocation_mode` when issuing a delegation via TASK_ASSIGN (§6.6). The two modes offer different tradeoffs between latency and revocation certainty.

**`sync` mode — per-hop registry verification:**

In `sync` mode, the delegating agent validates the delegatee's key status directly against the registry (§3.2) before each hop in the delegation chain. B cannot suppress or delay a revocation that A verifies independently.

| Property | Characteristic |
|----------|----------------|
| Verification point | Before each hop — delegating agent queries the registry for the delegatee's current key status |
| Suppression resistance | B cannot suppress revocations because A verifies directly against the registry, bypassing B entirely |
| Latency cost | O(n) per hop, where n is chain depth — each hop adds a registry round-trip before delegation proceeds |
| Suitable for | High-value delegations, low chain depth, untrusted intermediaries, environments where latency is acceptable |

**`sync` verification flow:**

```
A prepares to delegate to B:
  1. A queries registry for B's current key status
  2. Registry confirms B is not revoked
  3. A sends TASK_ASSIGN to B
  B prepares to delegate to C:
    4. B queries registry for C's current key status
    5. Registry confirms C is not revoked
    6. B sends TASK_ASSIGN to C
```

If the registry reports a revocation at step 2 or 5, delegation MUST NOT proceed. The delegating agent MUST send TASK_FAIL upstream with error code `delegatee_revoked`.

**`gossip` mode — signed revocation broadcast with trust decay:**

In `gossip` mode, revocation signals are broadcast as signed messages (§8.17.1 AGENT_MANIFEST with tombstones) and propagate through the network. Agents treat revocation information as probabilistically current — valid until the trust decay interval expires.

| Property | Characteristic |
|----------|----------------|
| Verification point | Continuous — agents consume signed revocation broadcasts as they arrive |
| Suppression resistance | Partial — signed broadcasts are independently verifiable (§8.17.1) but propagation depends on network topology. A single suppressive intermediary cannot prevent revocation from reaching agents that have alternative paths to the revoking agent's manifest endpoint |
| Latency cost | Amortized — no per-hop blocking. Revocation information propagates in the background |
| Suitable for | Deep delegation chains, latency-sensitive operations, environments with redundant communication paths |
| Trust decay | After `trust_decay_interval` without confirmation that a delegatee's key remains valid, the delegating agent MUST treat the delegatee as untrusted (see §9.8.6) |

**Mode coexistence:** Both modes are valid V1 strategies. The delegating agent selects the mode in TASK_ASSIGN via the `revocation_mode` field (§6.6). The choice is per-delegation, not per-session — an agent MAY use `sync` for high-value delegations to untrusted intermediaries and `gossip` for routine delegations within a trusted cluster. When `revocation_mode` is absent from TASK_ASSIGN, `gossip` is the default — consistent with V1's cooperative-first design (§8.18.1).

**Mid-session mode escalation:** The initial `revocation_mode` set at TASK_ASSIGN time can be escalated to a stricter mode mid-session via REVOCATION_MODE_ESCALATE (§6.15) without session termination or re-delegation. Escalation is unidirectional (`gossip` → `sync` only) and requires explicit acknowledgment from the delegatee before further operations proceed. See §6.15 for the full escalation protocol.

**Mode propagation in delegation chains:** When agent B re-delegates to agent C (§6.9), B MUST propagate the `revocation_mode` from the upstream TASK_ASSIGN unless B's own policy requires a stricter mode. Mode can only be tightened, not relaxed: if A specified `sync`, B MUST NOT downgrade to `gossip` when delegating to C. If A specified `gossip`, B MAY upgrade to `sync` for its delegation to C based on B's local risk assessment. The delegation-time mode is a minimum default, not a binding constraint — see §4.3.2 for the normative per-hop mode declaration semantics.

#### 9.8.6 Trust Decay Semantics for Gossip Mode

In `gossip` mode (§9.8.5), revocation information propagates asynchronously. During the propagation window, an agent's trust in a delegatee's key status decays over time. Trust decay converts the binary question "is this agent revoked?" into a temporal question: "how long since I last confirmed this agent is not revoked?"

**`trust_decay_interval`:** The maximum duration after which a delegating agent operating in `gossip` mode MUST re-verify the delegatee's key status before continuing to rely on the delegation. Default: 60 seconds. Configurable per-session via SESSION_INIT parameters.

**Trust decay lifecycle:**

1. At delegation time T₀, the delegating agent has a current (non-revoked) key status for the delegatee. Trust is full.
2. At T₀ + `trust_decay_interval`, if no signed confirmation of continued validity has been received (either via a fresh AGENT_MANIFEST from the delegatee or via a signed revocation broadcast that does NOT include the delegatee), the delegating agent MUST treat the delegatee as **untrusted-pending**.
3. An untrusted-pending delegatee MUST be re-verified before any new work is committed against the delegation. The delegating agent has three options:
   - **Re-verify via registry** (equivalent to a one-time `sync` check): query the registry for current key status. If valid, trust resets to full for another `trust_decay_interval`.
   - **Re-verify via fresh manifest**: fetch the delegatee's AGENT_MANIFEST (§8.17.1) directly. If the manifest is fresh (timestamp within `trust_decay_interval`) and contains no self-revocation, trust resets to full.
   - **Escalate**: treat the delegation as compromised. Send TASK_CANCEL (§6.6) to the delegatee and report the trust decay expiry to the upstream delegator.

**Trust decay does NOT apply to `sync` mode.** In `sync` mode, verification is performed at each hop before delegation proceeds — there is no propagation window during which trust can decay.

#### 9.8.7 Blast Radius Quantification

The revocation timing gap (§9.8.4) creates a window during which work is committed against stale trust. The **blast radius** is the total amount of work at risk during this window.

**Work-at-risk definition:** Work at risk is any task output committed by an agent (TASK_COMPLETE or crossed step boundary per §6.11.5) during the interval between a revocation being issued and the revocation being received by the agent relying on the revoked delegatee's authority.

**Blast radius bounds by mode:**

| Mode | Propagation window | Maximum work at risk | Detection mechanism |
|------|-------------------|---------------------|---------------------|
| `sync` | Zero — verification is synchronous | Zero — delegation does not proceed if delegatee is revoked | Registry query at each hop |
| `gossip` | Up to `trust_decay_interval` (default: 60s) | All task outputs committed within `trust_decay_interval` of the revocation issuance | Trust decay expiry triggers re-verification |

**Cost attribution policy:**

When work is committed during the revocation propagation window, cost attribution follows the principle of **verifiable good faith**: an agent that took reasonable verification steps before committing work is not at fault for work committed before the revocation reached it.

| Scenario | Attribution |
|----------|-------------|
| Agent used `sync` mode and registry returned stale data | Registry operator bears attribution — the agent verified correctly; the registry was inconsistent |
| Agent used `gossip` mode and committed work within `trust_decay_interval` | No fault — the agent operated within the protocol's defined propagation window. The work-at-risk is an accepted cost of `gossip` mode's latency optimization |
| Agent used `gossip` mode and committed work after `trust_decay_interval` expired without re-verification | Agent bears attribution — the agent violated the trust decay re-verification obligation (§9.8.6) |
| Intermediary (B) deliberately delayed broadcast | B bears attribution — deliberate delay is adversarial behavior, detectable via timestamp comparison between the revocation issuance time and B's propagation time. If B's propagation timestamp exceeds expected network latency by a significant margin, this constitutes evidence for the §8.16 detection signal taxonomy |

**Blast radius in EVIDENCE_RECORD:** When a revocation timing gap is detected, the detecting agent SHOULD create an EVIDENCE_RECORD (§8.10) with `evidence_type: error_event` containing: the revocation issuance timestamp, the revocation receipt timestamp, the list of task_ids with outputs committed during the propagation window, and the `revocation_mode` that was in effect. This provides the audit trail necessary for post-hoc cost attribution.

> Implements [issue #82](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/82): revocation timing guarantees for delegation chain threat model. Adds explicit threat model for revocation timing gap (§9.8.4), `revocation_mode` delegation parameter with `sync` and `gossip` modes (§9.8.5), trust decay semantics for gossip mode (§9.8.6), and blast radius quantification with cost attribution policy (§9.8.7). Distinct from #37 (pre-flight traversal) and §9.8.2 (Byzantine propagation). Originated from melonclaw DM Byzantine generals framing for multi-hop revocation. Closes #82.

> Community discussion on this section: [Moltbook post](https://www.moltbook.com/post/2fdee5e5-cdae-47c0-82a5-6bb9ec407d3c). See also [issue #10](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/10), [issue #36](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/36), [issue #38](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/38), [issue #49](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/49), [issue #82](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/82).

### 9.9 Observation Channel

The observation channel is the mechanism by which agents report justification for their actions to a monitoring layer. Without structured justification, a monitoring system can observe _what_ an agent did (via `trace_hash`, EVIDENCE_RECORDs, and DIVERGENCE_REPORTs) but not _why_ the agent believed the action was warranted. Free-text justification is a monitoring dead-end: a narrative cannot be algorithmically verified against post-hoc outcomes. Structured justification converts each action's rationale into a falsifiable prediction that the drift detector can compare against observed outcomes after execution. A wrong prediction is detectable; a wrong narrative is not.

#### 9.9.1 Justification Schema

When an agent reports an action through the observation channel — via TASK_PROGRESS (§6.6), TASK_COMPLETE (§6.6), DIVERGENCE_REPORT (§8.11), or EVIDENCE_RECORD (§8.10) — it SHOULD include a `justification` field containing a structured schema. The `justification` field is OPTIONAL for V1 compliance but RECOMMENDED for deployments that require post-hoc outcome verification.

**`justification` field schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reason_code | enum | Yes | Structured classification of why the action was taken. Uses the same taxonomy as the `divergence_log` reason enum (§8.10.4) to maintain consistency across the protocol's cause-annotation surfaces. Values: `infrastructure_noise`, `planning_failure`, `external_constraint`, `spec_drift`, `not_considered`. Implementations MAY extend with deployment-specific values prefixed by `x-` (e.g., `x-optimization-opportunity`, `x-user-escalation`). Standard reason codes MUST NOT be prefixed. |
| target_metric | string | Yes | The specific metric the action intends to affect. Free-form string scoped to the deployment's metric namespace (e.g., `task_completion_latency`, `output_token_count`, `error_rate`, `delegation_depth`). The protocol does not define a fixed metric taxonomy — metric names are deployment-specific. Agents MUST use consistent metric names within a session to enable cross-action comparison. |
| expected_delta | enum | Yes | The predicted directional effect of the action on `target_metric`. Values: `positive` (metric improves), `negative` (metric degrades — agent is documenting an accepted tradeoff), `neutral` (no expected effect on this metric — action targets a different outcome), `bounded_range` (effect is expected to fall within a deployment-defined acceptable range). |
| rationale | string | No | Free-text explanation providing context that the structured fields cannot convey — specific error messages, environmental details, or reasoning chain. This field is additive: it supplements the structured schema, not a substitute for it. Monitoring systems MUST NOT rely on `rationale` for automated verification — use `reason_code`, `target_metric`, and `expected_delta` instead. |

**Example (in TASK_PROGRESS):**

```yaml
message_type: TASK_PROGRESS
task_id: "task-abc-123"
progress: 0.6
justification:
  reason_code: infrastructure_noise
  target_metric: task_completion_latency
  expected_delta: negative
  rationale: "Switched to fallback API endpoint after primary returned 503 three consecutive times. Fallback has higher latency but equivalent output quality."
```

**Example (in DIVERGENCE_REPORT):**

```yaml
message_type: DIVERGENCE_REPORT
session_id: "session-xyz"
reason_code: external_constraint
justification:
  reason_code: external_constraint
  target_metric: output_token_count
  expected_delta: bounded_range
  rationale: "Upstream dependency schema changed field type from array to object. Adapted parsing logic mid-execution."
```

#### 9.9.2 Post-Hoc Verification

The structured justification schema enables a verification loop that free-text justification cannot support:

1. **Prediction.** The agent declares `expected_delta` for `target_metric` at action time.
2. **Observation.** The monitoring layer measures the actual delta on `target_metric` after execution completes.
3. **Comparison.** If `expected_delta` was `positive` but the observed delta is negative (or vice versa), the justification is **falsified** — the agent's stated reason for acting did not produce the predicted outcome.

Falsified justifications are not automatically protocol violations — an agent may correctly identify a problem (`reason_code`) and correctly target a metric (`target_metric`) but incorrectly predict the outcome (`expected_delta`). However, a pattern of falsified justifications across actions or sessions is a signal for the monitoring layer: the agent's decision-making model is miscalibrated, and its future justifications should be weighted accordingly.

**Falsification does not apply to `neutral` predictions.** An agent that declares `expected_delta: neutral` is asserting that the action targets a different outcome — the named `target_metric` is not expected to change. Monitoring systems SHOULD still track `neutral` predictions but MUST NOT treat metric movement on a `neutral` prediction as falsification without additional evidence that the agent's action caused the movement.

#### 9.9.3 Relationship to §8.10.4 Divergence Reason Enum

The `reason_code` values in the justification schema (§9.9.1) are intentionally aligned with the `reason` enum in the EVIDENCE_RECORD `divergence_log` (§8.10.4): `infrastructure_noise`, `planning_failure`, `external_constraint`, `spec_drift`, `not_considered`. The first four cause categories apply whether an agent is explaining a deviation (§8.10.4) or justifying a proactive action (§9.9.1). The distinction is temporal: `divergence_log` annotates deviations that already occurred; `justification` annotates actions the agent is about to take or is currently taking. The fifth value — `not_considered` — serves a different role: it is a sentinel indicating that an observation was captured but deliberately excluded from the decision (see §9.9.4).

The `x-` extension mechanism is shared: a deployment-specific `reason_code` added for `divergence_log` (e.g., `x-model-context-overflow`) is valid in `justification` and vice versa.

> Implements [issue #84](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/84): structured justification schema for the observation channel in §9. Converts free-text justification into a falsifiable prediction with `reason_code`, `target_metric`, and `expected_delta` sub-fields, enabling algorithmic verification of agent action rationale against post-hoc outcomes. Closes #84.

#### 9.9.4 NOT_CONSIDERED Sentinel

The `not_considered` value in the justification schema's `reason_code` (§9.9.1) disambiguates two states that would otherwise be indistinguishable in audit trails:

- **Absent `reason_code`**: the observation has not yet been evaluated.
- **`not_considered`**: the observation was captured, evaluated as existing, and deliberately excluded from the decision.

This distinction is critical for audit semantics. Without the sentinel, systematic exclusion patterns — where an agent consistently captures observations but never incorporates them — are invisible to auditors.

**Definition.** `NOT_CONSIDERED` MUST be used when all three conditions hold:

1. The observation was captured and is present in the observation record.
2. The deciding agent evaluated that the observation exists.
3. The observation was deliberately excluded from the decision.

**Distinction from other values:**

| Value | Meaning |
|-------|---------|
| Absent `reason_code` | Observation not yet evaluated. |
| `INSUFFICIENT_CONFIDENCE` | Observation evaluated but deemed unreliable for use. |
| `not_considered` | Observation present and noted, but intentionally non-incorporated. |

**When to use `not_considered`:**

1. **Authority override.** Trust decisions where a logged observation was superseded by a higher-authority determination (external verifier override).
2. **Direct attestation precedence.** Capability requests where historical observations are present but direct attestation from the session takes precedence.
3. **Decision deadline.** Canary task results that arrived after a decision deadline — logged for audit retention, not incorporated into the triggering decision.
4. **Scope boundary.** Observations flagged as outside the deciding agent's declared scope or jurisdiction under §22 isolation guarantees.

**Audit semantics.** An observation with `reason_code: not_considered` MUST still be retained for the full observation retention period defined in §9. The sentinel records intentional exclusion, not irrelevance. Auditing agents MAY query `not_considered` observations to detect systematic exclusion patterns that could indicate scope gaming or authority override abuse.

**V2 deferral.** Capturing the rationale for non-consideration — which authority made the override, what scope boundary was invoked, timing constraints — is explicitly V2. V1 records the fact of non-consideration; the audit trail is complete but the explanation is not required at protocol level.

> Addresses [issue #95](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/95): `NOT_CONSIDERED` sentinel for the observation channel's `reason_code` enum (§9.9.1), disambiguating observations that were captured but deliberately excluded from a decision from observations that were never evaluated. Enables auditors to detect systematic exclusion patterns. Closes #95.

### 9.10 Trust Annotation Types

Trust annotations are protocol-level claims that an agent attaches to actions, delegations, or evidence records to declare the trust basis under which it operated. Without a fixed vocabulary for these claims, each agent implicitly becomes a schema authority — choosing which annotation types to honor, which to ignore, and which to invent. This produces distributed governance with no coordination mechanism: audit trails become inconsistent across agents, and verification-time disputes about which annotations are canonical have no resolution procedure.

§9.10 addresses this by defining a closed enum of trust annotation types at spec genesis, accompanied by a governance ceremony and tamper-evident publication mechanism.

#### 9.10.1 Trust Annotation Model

Trust annotation types MUST be a fixed, closed enum defined at spec genesis. The enum is not extensible at runtime, not extensible by operators, and not extensible by individual agent implementations. Unrecognized annotation types are not protocol errors — they are agent-local metadata that carries no protocol-level authority.

**Rationale for fixed enum over open schema:**

The authority question — who decides which trust annotation types are canonical — has three possible locations:

| Authority location | Mechanism | Failure mode |
|-------------------|-----------|--------------|
| Runtime (agent-level) | Each agent chooses which types to honor | Distributed governance with no coordination — inconsistent audit trails, unresolvable disputes at verification time |
| Operator configuration | Operators define the annotation vocabulary per deployment | Operator becomes implicit schema authority — trust annotations mean different things in different deployments, breaking cross-deployment audit |
| Spec genesis (one-time ceremony) | Enum is fixed at spec publication time | Bounded expressiveness — some annotation types that agents might want are not representable as protocol-level claims |

The protocol chooses spec genesis because it relocates the authority question from ongoing runtime decisions (low-visibility, repeated, subject to incremental capture) to a single ceremony (bounded, auditable, one-time). The tradeoff — bounded expressiveness — is intentional. Annotation types that do not fit the fixed enum are valid as agent-local metadata but do not carry protocol-level authority and MUST NOT appear in protocol-level verification or audit.

**Rationale against fork-ability:** Allowing agents to fork the trust annotation vocabulary and run parallel governance creates a coordination problem: parallel schemas produce mutually unverifiable audit trails. Resolving which fork is canonical requires a new authority — the problem migrates, it does not dissolve.

#### 9.10.2 Initial Trust Annotation Enum

The following trust annotation types are the complete, closed V1 enum. Each type is a protocol-level claim with defined semantics.

| Type | Description | Use context |
|------|-------------|-------------|
| `DELEGATION` | Agent acted under explicit authority granted by another agent. The delegation chain is traceable via §5.5 delegation tokens. | Attached to actions taken on behalf of a delegating agent. Enables audit of authority provenance — who authorized this action and through what chain. |
| `ASSUMES_AUTHENTICATED_SOURCE` | Agent trusted message origin identity without protocol-level verification. The agent accepted the counterparty's claimed identity (§2.2) without cryptographic verification via the keypair extension (§2.2.1). | Attached to actions taken based on messages from agents identified by `(name, platform)` pair only, without `pubkey` verification. Marks the trust assumption explicitly — the action's validity depends on the platform's identity guarantees, not on protocol-level cryptographic verification. |
| `ASSUMES_SCHEMA_VERSION` | Agent executed against a specific schema version that was not validated at runtime. The agent assumed the schema version declared by the counterparty or operator was accurate without independently verifying the schema content against the declared version identifier. | Attached to actions where schema version was taken on trust rather than verified. Relevant when schema attestation (§9.1) is not in use — the agent operated on the assumption that the schema labeled "v1.2" actually contains v1.2 semantics. |
| `OPERATOR_ASSERTED` | Claim originates from operator configuration rather than protocol-level verification. The trust basis is the operator's out-of-band assertion — a configuration file, environment variable, or deployment parameter — rather than any protocol mechanism. | Attached to actions whose trust basis is operator authority. Distinguishes protocol-verified claims from operator-injected claims in audit trails. An `OPERATOR_ASSERTED` annotation on a delegation means the delegation authority was configured by the operator, not established through the §5.5 delegation protocol. |

**Trust annotation field schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| annotation_type | enum | Yes | One of: `DELEGATION`, `ASSUMES_AUTHENTICATED_SOURCE`, `ASSUMES_SCHEMA_VERSION`, `OPERATOR_ASSERTED` |
| annotation_context | string | No | Additional context for the annotation. Free-text, scoped to the specific instance. For `DELEGATION`: the delegation token identifier. For `ASSUMES_SCHEMA_VERSION`: the assumed schema version string. For `OPERATOR_ASSERTED`: the configuration source identifier. |
| annotated_at | timestamp | Yes | ISO 8601 timestamp of when the annotation was attached. |

**Example (trust annotation on a TASK_COMPLETE message):**

```yaml
message_type: TASK_COMPLETE
task_id: "task-abc-123"
trust_annotations:
  - annotation_type: DELEGATION
    annotation_context: "delegation-token-7f3a9b"
    annotated_at: "2026-01-15T14:30:00Z"
  - annotation_type: ASSUMES_AUTHENTICATED_SOURCE
    annotated_at: "2026-01-15T14:30:00Z"
```

**Interaction with existing protocol surfaces:**

- Trust annotations MAY be attached to TASK_PROGRESS (§6.6), TASK_COMPLETE (§6.6), EVIDENCE_RECORD (§8.10), and DIVERGENCE_REPORT (§8.11) messages via an optional `trust_annotations` array field.
- Trust annotations are orthogonal to the `justification` field (§9.9.1): `justification` explains _why_ an action was taken; trust annotations declare _under what trust basis_ the action was taken. Both MAY be present on the same message.
- Trust annotations are included in `trace_hash` computation (§6.2) when present — an action taken under `DELEGATION` and the same action taken under `OPERATOR_ASSERTED` produce different trace hashes, making the trust basis auditable through the existing hash verification mechanism.

#### 9.10.3 Genesis Ceremony

The trust annotation enum (§9.10.2) is defined through a **genesis ceremony** — a single, bounded governance event at spec publication time. The genesis ceremony is not a runtime decision, not an operator decision, and not renewable at will.

**Genesis ceremony properties:**

| Property | Characteristic |
|----------|----------------|
| Bounded | One-time event, not ongoing governance. The ceremony occurs once at spec genesis and does not repeat. |
| High-scrutiny | Deliberate, explicit participants. The ceremony is a design-time decision with named contributors, not an incremental runtime accretion. |
| Auditable | The event and participants are nameable. The ceremony produces a verifiable artifact (the enum definition and its publication hash per §9.10.4) that any agent can independently verify. |

These properties contrast with runtime authority, which is low-visibility (each agent's annotation type decisions are local and uncoordinated), repeated (every message exchange is an implicit governance decision), and subject to incremental capture (an agent that introduces a new annotation type and gains adoption has silently become a schema authority).

**Modification policy:**

Modifications to the trust annotation enum — adding, removing, or changing the semantics of any annotation type — require a new **major version** of the protocol specification. This means:

- A new annotation type cannot be added in a minor version or patch version (§10).
- An existing annotation type's semantics cannot be changed without a major version increment.
- Removal of an annotation type is a breaking change requiring a major version increment.

The modification policy is a security property, not a limitation. If the enum were modifiable without a major version change, the genesis ceremony's governance guarantees would be undermined — an authority that can modify the enum at will has the same power as an authority that defined it, but without the ceremony's bounded, auditable, high-scrutiny properties.

#### 9.10.4 Genesis Publication Hash

The genesis ceremony (§9.10.3) produces the trust annotation enum (§9.10.2). Without tamper-evidence, the "one-time ceremony" guarantee is procedural aspiration — it depends on trust in the spec maintainers not to silently modify the enum post-genesis. The genesis publication hash converts this procedural guarantee into a cryptographic one: any agent can independently verify that the enum it executes against matches the artifact produced at genesis.

**Publication hash requirement:**

The genesis ceremony MUST produce a publication hash computed as:

```
genesis_hash = hash(canonical_enum_text + spec_version_string)
```

Where:
- `canonical_enum_text` is the exact text of §9.10.2 from "The following trust annotation types" through the end of the enum table (the four-row table defining `DELEGATION`, `ASSUMES_AUTHENTICATED_SOURCE`, `ASSUMES_SCHEMA_VERSION`, `OPERATOR_ASSERTED`), canonicalized per §4.10.4 (UTF-8 encoding without BOM, LF-only line endings, no trailing whitespace per line, trailing newline fixup, NFC normalization).
- `spec_version_string` is the protocol version identifier (e.g., `"0.1.0"`).
- `hash` is SHA-256.

**Publication constraints:**

The genesis hash MUST be committed to an external, independently-verifiable medium that satisfies the following constraints:

| Constraint | Requirement |
|------------|-------------|
| Independence | The publication medium MUST NOT be maintained or controlled by the spec authors. If the spec authors control the medium, they can modify the hash to match a silently modified enum — the tamper-evidence property is defeated. |
| Verifiability | Any agent MUST be able to retrieve the published hash and verify it independently without relying on the spec authors' infrastructure. |
| Immutability | The publication medium MUST provide append-only or immutable storage. A hash published to a mutable medium can be silently replaced. |
| Consistency with §8 audit media | The publication medium MUST satisfy the same independence criteria as §8 audit media — specifically, the audited party (spec authors) MUST NOT be the party that controls the publication medium. This is the same constraint applied to EVIDENCE_RECORD storage (§8.10). |

**Verification procedure:**

An agent verifying the trust annotation enum performs the following steps:

1. Retrieve the canonical enum text from the spec document (§9.10.2).
2. Retrieve the spec version string from the spec document (§10).
3. Compute `hash(canonical_enum_text + spec_version_string)` using SHA-256.
4. Retrieve the genesis hash from the external publication medium.
5. Compare the computed hash with the published hash. If they match, the enum has not been modified since genesis. If they do not match, the enum has been modified — the agent SHOULD treat the enum as untrusted and SHOULD log a verification failure via EVIDENCE_RECORD (§8.10) with `evidence_type: error_event`.

**Tamper-evidence, not tamper-prevention:** The genesis publication hash makes silent enum modification detectable. It does not prevent modification — an authority that controls the spec can still change the enum and publish a new hash. What the hash prevents is _undetected_ modification: any change to the enum after genesis produces a hash mismatch that any verifying agent can independently discover. Authority over schema updates becomes detectable rather than merely prohibited.

> Implements [issue #133](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/133) and [issue #134](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/134): fixed enum of trust annotation types with genesis ceremony and publication hash. Defines four canonical trust annotation types (`DELEGATION`, `ASSUMES_AUTHENTICATED_SOURCE`, `ASSUMES_SCHEMA_VERSION`, `OPERATOR_ASSERTED`) as a closed enum at spec genesis, governance ceremony with bounded/auditable/high-scrutiny properties, modification policy requiring major version increment, and SHA-256 genesis publication hash for tamper-evident verification. Closes #133, closes #134.

### 9.11 Amendment Ceremonies

The genesis ceremony (§9.10.3) defines the trust annotation enum at spec publication time. §9.10.3 specifies that modifications require a major version increment, but does not specify _how_ a modification is proposed, deliberated, or ratified. Without an explicit amendment path, modification governance defaults to whoever can update the document — runtime authority without acknowledgment. This is precisely the failure mode the genesis ceremony was designed to prevent: low-visibility, uncoordinated authority over the trust annotation vocabulary. An implicit amendment path is harder to audit than any explicit one, even a weak one.

§9.11 specifies the amendment ceremony: the procedure by which the trust annotation enum may be modified after genesis. Amendment ceremonies are structurally lighter than the genesis ceremony but explicitly specified — proportional governance with full auditability.

**Composable governance design:** Both genesis (§9.10.3) and amendment ceremonies share the same three-element verification structure: (1) explicit trigger conditions defining when a ceremony is valid, (2) named participant threshold defining who must participate, and (3) publication hash chaining providing tamper-evident linkage. This structural uniformity is intentional — one verification pattern applies at genesis and at every subsequent amendment. An automated governance auditor can traverse the hash chain from genesis through every amendment using a single verification procedure, without type dispatch or special-casing for genesis vs. amendment records. The composable design ensures that amendment governance inherits the genesis ceremony's auditability properties by construction, not by convention.

#### 9.11.1 Trigger Conditions

A valid amendment proposal requires a **demonstrated vocabulary gap**: a coordination use case that is not expressible using the current trust annotation enum, with documented adoption evidence. The adoption evidence requirement prevents speculative additions — a proposed annotation type must correspond to an observed coordination pattern, not a hypothetical one.

Proposals without a demonstrated vocabulary gap are not valid amendment triggers. An annotation type that _might be useful someday_ does not meet the threshold. The trigger is reactive (demonstrated gap) rather than scheduled (periodic review window). Scheduled review windows create artificial constraints on urgent changes and generate noise during empty review periods.

Once a valid proposal is published, a **minimum deliberation window of 14 days** must elapse before ratification consideration begins. The deliberation window is the anti-gaming mechanism — it ensures that amendments cannot be rushed through before affected parties can evaluate them. The window is mandatory regardless of perceived urgency.

#### 9.11.2 Legitimacy Tiering

Not all modifications to the trust annotation enum are equivalent. A typo correction and a new annotation type have categorically different impacts on protocol semantics. Conflating both under a single ceremony either makes cosmetic corrections prohibitively expensive or makes semantic changes too cheap.

**Tier 1 — Cosmetic changes:**

Typos, non-semantic language corrections, editorial improvements with no behavioral effect. A change is cosmetic if and only if: (a) the `annotation_type` enum values are unchanged, (b) the verification behavior for every annotation type is unchanged, and (c) the `genesis_hash` or current terminal `amendment_hash` would change only due to whitespace or wording, not due to semantic content.

**Observable Tier 1 criteria:** A change qualifies as Tier 1 only if it is limited to prose, formatting, cross-reference text, or typographic corrections that alter no behavioral specification. The observable test is: no agent implementation would need to change code, configuration, or runtime behavior as a result of the change. If an implementation could produce different outputs, accept different inputs, or make different decisions after the change, it is not Tier 1.

Tier 1 process: documented proposal published with justification, followed by a **no-objection window of 7 days** from active implementors (§9.11.3). The applicable quorum for the no-objection window is determined by the governance tier (§9.11.3 governance tier quorum defaults table). If no objections are raised within the window, the change is ratified. No full ceremony is required.

**Tier 2 — Semantic changes:**

New annotation type, modified verification behavior, deprecated annotation type, changed enum semantics. Any change that alters what the enum _means_ — not just how it reads — is Tier 2.

**Observable Tier 2 criteria:** A change is Tier 2 if it affects field semantics, message formats, required behaviors, interoperability guarantees, audit event schemas, or enumeration values. The observable test is: any change where at least one conforming implementation would need to modify code, alter validation logic, update message processing, or change runtime behavior to remain conformant after the change. Tier 2 is the default classification — a change is Tier 2 unless it can be affirmatively demonstrated to meet the Tier 1 observable criteria.

Tier 2 process: full amendment ceremony required, with rigor comparable to genesis. The minimum 14-day deliberation window (§9.11.1) applies. The participant threshold (§9.11.3) must be met, with the applicable quorum determined by the governance tier (§9.11.3 governance tier quorum defaults table) — STANDARD for routine amendments, STRUCTURAL for changes targeting governance, trust model, or versioning sections. The amendment record (§9.11.4) must be produced. Tier 2 amendments continue to require a major version increment per §9.10.3.

**Worked examples at the tier boundary:**

The following changes are superficially cosmetic but are Tier 2 under the observable criteria:

1. _Adding a clarifying sentence that constrains previously unconstrained behavior._ Example: adding "agents MUST retry at most 3 times" to a section that previously left retry behavior unspecified. The sentence reads as clarification, but it introduces a new behavioral requirement. Any implementation with unbounded retries would need to change. Tier 2.

2. _Reordering a list that implies precedence._ Example: reordering the trust annotation types in the enum definition when processing order or priority is derived from list position. Even if no explicit ordering rule is stated, implementations that iterate the enum in definition order would produce different behavior. If any consumer treats list position as meaningful, reordering is semantic. Tier 2.

3. _Correcting a cross-reference that changes which section applies._ Example: changing "as specified in §9.10.3" to "as specified in §9.10.4" in a normative requirement. If the referenced sections have different requirements, the correction changes what behavior is required — even though it looks like a typo fix. The observable test: does the corrected reference point to a section with different normative content? If yes, Tier 2.

These examples are not exhaustive. The general principle is: observable impact on implementation behavior determines the tier, not the syntactic form of the change. A one-character edit can be Tier 2; a paragraph rewrite can be Tier 1. The amendment record (§9.11.4) MUST include the tier classification with justification referencing the observable criteria.

#### 9.11.3 Participant Threshold

**Active implementors** are agents or implementations with demonstrated protocol execution. Qualification as an active implementor requires documented protocol implementation with evidence — deployment logs, public implementation repositories, or other verifiable artifacts demonstrating that the implementation executes against the trust annotation enum.

**Genesis ceremony participants** (§9.10.3) have permanent standing as active implementors. Their standing does not expire and does not require re-qualification. This ensures that the original ceremony participants retain governance authority over the artifact they defined.

**Quorum** thresholds for amendment ceremonies are determined by the governance tier of the amendment, as defined in the governance tier quorum defaults table below. The active implementor list is fixed at proposal time to prevent quorum manipulation during the deliberation window — adding or removing implementors after a proposal is published does not change the quorum requirement for that proposal.

**Governance Tier Quorum Defaults**

The protocol defines three governance tiers with default quorum thresholds. These defaults ensure interoperability — implementors apply the same governance requirements without out-of-band negotiation.

| Governance Tier | Applies To | Default Quorum Threshold | Rationale |
|-----------------|------------|--------------------------|-----------|
| **BOOTSTRAP** | Genesis ceremony (§9.10.3) | No quorum required | No pre-existing governance body exists at genesis. The genesis ceremony establishes the initial governance participants; requiring a quorum of a body that does not yet exist is a logical impossibility. |
| **STANDARD** | Routine amendments — Tier 1 cosmetic (§9.11.2) and Tier 2 semantic (§9.11.2) changes under the normal amendment ceremony (§9.11) | Simple majority (>50% of active implementors at proposal time) | Routine amendments follow the standard deliberation process with tier-appropriate ceremony windows. Simple majority balances governance participation against amendment velocity for non-structural changes. |
| **STRUCTURAL** | Protocol-level changes — modifications to governance procedures (§9.11, §9.12), trust model foundations (§9.10), or versioning guarantees (§10) via the standard amendment path | Supermajority (≥66% of active implementors at proposal time) | Changes to the governance framework itself or to foundational protocol invariants require broader consensus than routine amendments. A simple majority modifying the rules by which majorities are determined is a governance capture vector. |

**Tier assignment is determined by the amendment's target sections**, not by the proposer's classification. An amendment that modifies any section listed under STRUCTURAL governance — even if the modification appears cosmetic under the legitimacy tier criteria (§9.11.2) — requires the STRUCTURAL quorum threshold. The legitimacy tier (§9.11.2) determines the ceremony process (deliberation window length, no-objection vs. full ceremony); the governance tier determines the quorum threshold. These are orthogonal dimensions: a Tier 1 cosmetic change to a STRUCTURAL section still requires supermajority quorum during its no-objection window, and a Tier 2 semantic change to a non-structural section requires only simple majority.

**Relationship to emergency amendments (§11):** Emergency amendments operate under §11's own quorum requirements (unanimous coordinating committee sign-off, §11.2.2), which are strictly above the STRUCTURAL default. The governance tier defaults defined here apply to the standard amendment path (§9.11) only. Emergency amendments are not subject to tier-based quorum defaults — they are subject to §11.2.2 exclusively.

> Implements [issue #209](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/209): governance tier quorum defaults for amendment ceremonies. Defines three governance tiers (BOOTSTRAP, STANDARD, STRUCTURAL) with explicit default quorum thresholds, eliminating the need for out-of-band agreement on amendment quorum requirements. Closes #209.

#### 9.11.4 Amendment Record

A ratified amendment ceremony produces an **amendment record** containing:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| proposed_changes | object | Yes | The proposed modifications to the trust annotation enum, with full justification for each change. Includes the demonstrated vocabulary gap (§9.11.1) and the tier classification (§9.11.2). |
| participant_list | array | Yes | Active implementors at proposal time, with acknowledgment status for each. For Tier 2: quorum per the applicable governance tier threshold (§9.11.3 governance tier quorum defaults table) must have acknowledged. For Tier 1: no-objection from active implementors within the 7-day window, with quorum per the applicable governance tier. |
| deliberation_window_open | timestamp | Yes | ISO 8601 timestamp of when the deliberation window opened (proposal publication time). |
| deliberation_window_close | timestamp | Yes | ISO 8601 timestamp of when the deliberation window closed (minimum 14 days after open for Tier 2, minimum 7 days for Tier 1). |
| ratification_timestamp | timestamp | Yes | ISO 8601 timestamp of when the amendment was ratified. Must be after `deliberation_window_close`. |
| amendment_hash | string | Yes | SHA-256 hash computed as specified in §9.11.5. |

#### 9.11.5 Amendment Hash Chain

Each amendment produces a hash that chains to the previous state, extending the tamper-evidence property of the genesis publication hash (§9.10.4) to cover the full lifecycle of the trust annotation enum.

**Amendment hash computation:**

```
amendment_hash = SHA-256(prior_state_hash || amendment_record_canonical_json)
```

Where:
- `prior_state_hash` is the `genesis_hash` (§9.10.4) for the first amendment, or the `amendment_hash` of the immediately preceding amendment for subsequent amendments.
- `amendment_record_canonical_json` is the JSON serialization of the amendment record (§9.11.4), canonicalized per §4.10.4: RFC 8785 JCS (keys sorted lexicographically by Unicode code point, no whitespace between tokens, deterministic number encoding), UTF-8 encoding without BOM, LF-only line endings, no trailing whitespace, NFC normalization.
- `||` denotes concatenation.

**Amendment chain:** The full lineage from `genesis_hash` through each `amendment_hash` to the current terminal node is independently verifiable. The current spec version's trust annotation enum is authoritative if and only if the amendment chain can be verified from a trusted `genesis_hash` to the terminal `amendment_hash`.

Implementations MUST verify the amendment chain from a trusted genesis hash before accepting a trust annotation enum as authoritative. A trust annotation enum that cannot be traced back to a verified genesis hash through a valid amendment chain has no protocol-level authority.

The amendment chain is the tamper-evidence mechanism for protocol evolution — it applies the same principle as the genesis publication hash (§9.10.4) but extends it across the enum's full lifecycle rather than anchoring it to a single point in time.

#### 9.11.6 Backwards Compatibility

Agents MUST treat unknown trust annotation types as unknown-but-harmless. Specifically:

- Unknown annotation types MUST be forwarded without interpretation. An agent that receives a message with a `trust_annotations` entry containing an `annotation_type` value it does not recognize MUST forward the annotation intact if the message is relayed to another agent. The forwarding agent MUST NOT strip, modify, or reinterpret the unknown annotation.
- Rejection of unknown annotation types is a protocol violation. An agent MUST NOT reject a message solely because it contains an unrecognized `annotation_type` value. The message is valid; the annotation is unrecognized.
- This requirement applies symmetrically: old agents receiving new annotation types added by amendment, and new agents receiving annotation types deprecated by amendment. Deprecation removes an annotation type from the canonical enum; it does not make the annotation type invalid in messages that were produced before deprecation.
- This requirement is non-negotiable and applies regardless of agent implementation version. It is the mechanism that prevents amendment-induced fragmentation — without it, each enum change would strand deployed agents that do not yet recognize the new types.

> Implements [issue #149](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/149): amendment ceremony procedures for the trust annotation enum. Specifies trigger conditions (demonstrated vocabulary gap, minimum deliberation window), legitimacy tiering (Tier 1 cosmetic with 7-day no-objection window, Tier 2 semantic with full ceremony), participant threshold (active implementors with genesis participant permanent standing), amendment record schema, SHA-256 amendment hash chaining from genesis hash, and backwards compatibility requirements (unknown-but-harmless forwarding). Closes #149.

### 9.12 Classification Dispute Resolution

The tier classification (§9.11.2) determines the governance ceremony required for an amendment. A motivated actor can classify a semantic change as Tier 1 to bypass the full ceremony requirement while appearing compliant — this is the primary attack surface on the amendment path. The observable tier criteria (§9.11.2) provide pre-capture audit before the ceremony decision; the amendment hash chain (§9.11.5) provides post-hoc detection (a divergent change without the appropriate ceremony hash is detectable after the fact). §9.12 specifies the dispute resolution procedure that bridges these two mechanisms.

#### 9.12.1 Escalation Right

Any active implementor (§9.11.3) or genesis ceremony participant (§9.10.3) may escalate a proposed Tier 1 amendment to Tier 2 review. The escalation right is unconditional — the escalating party is not required to demonstrate that the change is semantic, only to assert that the tier classification is disputed. The burden of proof shifts to the proposer to re-justify the Tier 1 classification against the observable criteria (§9.11.2), or to accept Tier 2 classification and proceed with the full ceremony.

Escalation cannot be blocked, overridden, or vetoed by any party, including the proposal author, other active implementors, or any governance role. A single escalation is sufficient to trigger Tier 2 review. This asymmetry is intentional — the cost of a false escalation (unnecessary full ceremony for a cosmetic change) is bounded and recoverable, while the cost of a missed escalation (semantic change ratified without full ceremony) undermines the integrity of the amendment chain.

#### 9.12.2 Dispute Resolution Process

When a tier classification is escalated:

1. **The Tier 1 no-objection window (§9.11.2) is suspended.** The 7-day window does not continue to run during dispute resolution. Ratification under Tier 1 process is not possible while an escalation is active.

2. **The proposer responds within 7 days** with one of:
   - **Accept Tier 2 classification.** The proposal proceeds under the Tier 2 process (§9.11.2): full amendment ceremony with 14-day deliberation window, participant threshold, and amendment record.
   - **Re-justify Tier 1 classification.** The proposer publishes a written justification demonstrating that the change meets the observable Tier 1 criteria (§9.11.2) — specifically, that no conforming implementation would need to modify code, alter validation logic, update message processing, or change runtime behavior. The justification must address each specific concern raised in the escalation.

3. **If the proposer re-justifies Tier 1**, active implementors evaluate the justification during a **renewed 7-day no-objection window**. Any active implementor may escalate again during this window. A second escalation on the same proposal after re-justification triggers automatic Tier 2 classification — the proposal MUST proceed under the Tier 2 process with no further re-justification opportunity.

4. **If the proposer does not respond within 7 days**, the proposal is automatically classified as Tier 2.

#### 9.12.3 Default to Tier 2

Unresolved disputes default to Tier 2. If at any point during the dispute resolution process the tier classification cannot be affirmatively resolved to Tier 1 — whether due to proposer non-response, insufficient justification, or repeated escalation — the amendment MUST proceed under the Tier 2 process. This conservative default ensures that ambiguous changes receive full ceremony scrutiny rather than passing through the lighter Tier 1 process.

The default-to-Tier-2 principle also applies when no dispute is raised but the observable criteria are ambiguous. A proposer who is uncertain whether a change is Tier 1 or Tier 2 SHOULD classify it as Tier 2. The cost asymmetry is clear: a Tier 2 ceremony for a cosmetic change wastes time; a Tier 1 process for a semantic change compromises governance integrity.

> Addresses [issue #155](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/155): observable tier criteria and classification dispute resolution for amendment governance. Adds observable criteria for Tier 1 (cosmetic) and Tier 2 (semantic) classification based on implementation-observable behavioral impact rather than proposer intent. Adds worked examples at the tier boundary (constraining clarifications, precedence-implying reordering, cross-reference corrections). Specifies classification dispute resolution (§9.12) with unconditional escalation rights, suspension of Tier 1 process during disputes, re-justification procedure, automatic Tier 2 on repeated escalation or non-response, and conservative default-to-Tier-2 for unresolved disputes. Closes #155.

### 9.13 Introduction Omission Threat

<!-- Implements #103: Named V1 out-of-scope threat for introduction omission. -->

§8.17.4 establishes that introductions are claims, not trusted facts, and specifies the verification procedure that defeats forgery — an introducing agent B cannot forge A's manifest signature or misrepresent A's capabilities once C independently verifies A's manifest at A's `canonical_self_url`. This section documents the residual threat that signature verification cannot address: **omission**.

#### 9.13.1 Threat Description

In a delegation chain A → B → C, agent B may be C's sole discovery vector for A. If B never introduces A to C — or introduces A with a stale or incorrect `canonical_self_url` — C has no mechanism to discover A's existence or verify A's current state. This is fundamentally different from forgery:

- **Forgery** requires B to produce a cryptographic artifact (a manifest) that passes A's signature verification. Defeated by §8.17.4 verification procedure.
- **Omission** requires B to do nothing — or to selectively delay introduction. No cryptographic primitive can force B to transmit information B chooses to withhold.

The omission threat is particularly acute when B is a coordinator or gateway agent through which multiple agents are introduced. B's omission of a single introduction is invisible to both the omitted agent and the agent that never learns of the omission.

#### 9.13.2 V1 Scope

Introduction omission is a **named V1 out-of-scope threat**. V1 provides no detection or mitigation mechanism for the case where B is C's sole discovery vector for A and B suppresses the introduction entirely.

**Why V1 cannot address this:** Any V1 mitigation would require either (a) a registry or distributed lookup mechanism that C can query independently of B (which §3 registries partially provide but do not mandate for all topologies), or (b) a protocol-level requirement that coordinators enumerate all known agents to all participants (which would leak topology information and create scaling problems). Neither is appropriate for V1's scope.

**Partial V1 mitigation via registries:** When agents publish AGENT_MANIFESTs to registries (§3), C can discover A through registry QUERY operations (§3.2.1) without relying on B's introduction. This is a deployment-level mitigation, not a protocol guarantee — it depends on both A and C using the same registry (or federated registries), and on A's manifest being published before C needs to discover it.

#### 9.13.3 V2 Design Direction

V2 SHOULD investigate a distributed lookup mechanism for canonical agent identity resolution independent of any single introduction vector. Candidate approaches:

- **DHT-based agent directory.** Agents publish their `canonical_self_url` (§3.1) to a distributed hash table keyed by `(name, platform)` identity handle. Any agent can resolve any other agent's canonical URL without relying on a specific intermediary.
- **DNS-like resolution.** A hierarchical naming system where `canonical_self_url` resolution follows a deterministic path (e.g., platform-scoped well-known URLs) that C can attempt without prior introduction.
- **Gossip-based manifest propagation.** Agents periodically broadcast their `canonical_self_url` to known peers. Over time, the network converges toward full awareness. Probabilistic, not guaranteed — a deliberately isolated agent may never receive the broadcast.

The V2 design question is not "should agents be discoverable without introduction?" (yes) but "what is the minimum infrastructure required to make independent discovery reliable without centralizing the discovery mechanism?"

### 9.14 Governance Document Lifecycle

§9.10–§9.12 define governance for a single artifact: the trust annotation enum. The genesis ceremony (§9.10.3), amendment ceremonies (§9.11), and publication hash (§9.10.4) establish lifecycle governance for that artifact specifically. But the protocol contains multiple governance artifacts — the trust annotation enum, amendment records, the spec version itself — and each requires the same three structural properties to be V1-complete: machine-readable lifecycle state, bootstrap legitimacy, and cryptographic binding to the spec version chain.

§9.14 defines the three structural elements that generalize governance across all protocol governance artifacts.

#### 9.14.1 GovernanceState Enum

Governance documents in §9.10–§9.12 use prose descriptions to indicate lifecycle state — "proposed," "ratified," "deprecated." Prose state descriptions are not machine-readable: an automated governance auditor cannot programmatically determine whether a governance document is in effect without parsing natural language. This defeats the auditability property that §9.10.3 and §9.11 are designed to provide.

**GovernanceState** is a fixed, closed enum of lifecycle states for governance documents:

| Value | Description | Transition conditions |
|-------|-------------|-----------------------|
| `PROPOSED` | Document has been proposed but not yet ratified. The document exists as a candidate but carries no protocol-level authority. | Initial state. A governance document enters `PROPOSED` when published with a valid trigger condition (§9.11.1 for amendments) or at genesis initiation. |
| `ACTIVE` | Document has been ratified and is currently in effect. The document carries protocol-level authority and agents MUST honor its contents. | From `PROPOSED`: ratification ceremony completes (genesis ceremony per §9.14.2 or amendment ceremony per §9.11). Only one document per governance scope MAY be `ACTIVE` at a time — ratifying a new document in the same scope transitions the prior `ACTIVE` document to `SUPERSEDED`. |
| `SUPERSEDED` | Document has been replaced by a newer ratified version. The document is no longer authoritative but remains in the audit trail for chain verification. | From `ACTIVE`: a successor document in the same governance scope reaches `ACTIVE`. The `SUPERSEDED` document's hash remains in the amendment chain (§9.11.5) — it is not deleted, only de-authorized. |
| `REVOKED` | Document has been explicitly revoked and is no longer valid. Unlike `SUPERSEDED`, revocation indicates that the document is withdrawn without a successor — the governance scope it covered is no longer governed by any document until a new document reaches `ACTIVE`. | From `ACTIVE` or `PROPOSED`: explicit revocation ceremony with the same participant threshold as the document's ratification ceremony. From `SUPERSEDED`: not permitted — a superseded document is already non-authoritative and revocation would be redundant. |

**State transition diagram:**

```
PROPOSED ──ratification──► ACTIVE ──successor ratified──► SUPERSEDED
    │                        │
    │                        │
    └──revocation──►  REVOKED ◄──revocation──┘
```

**GovernanceState field schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| governance_state | enum | Yes | One of: `PROPOSED`, `ACTIVE`, `SUPERSEDED`, `REVOKED` |
| state_changed_at | timestamp | Yes | ISO 8601 timestamp of the most recent state transition. |
| state_changed_by | string | Yes | Identifier of the ceremony or event that triggered the transition. For genesis: the genesis ceremony identifier. For amendments: the amendment record identifier (§9.11.4). For revocations: the revocation ceremony identifier. |
| predecessor_hash | string | No | Hash of the governance document this document supersedes. Present when `governance_state` is `ACTIVE` and the document supersedes a prior document, or when `governance_state` is `SUPERSEDED` (pointing to the document that was superseded). Absent for the genesis document. |

**Applicability to existing governance artifacts:**

| Artifact | §9.10–§9.12 prose state | GovernanceState equivalent |
|----------|------------------------|---------------------------|
| Trust annotation enum at genesis | "defined at spec genesis" (§9.10.3) | `ACTIVE` — ratified by genesis ceremony |
| Amendment proposal during deliberation | "proposed" (§9.11.1) | `PROPOSED` — published but not yet ratified |
| Amendment after ratification | "ratified" (§9.11.4) | `ACTIVE` — the amended enum supersedes the prior version |
| Prior enum version after amendment | Implicit — "the previous state" | `SUPERSEDED` — replaced by the ratified amendment |

**Interaction with amendment hash chain:** The `governance_state` field is included in the `amendment_record_canonical_json` (§9.11.5) for amendment records. This means state transitions are captured in the hash chain — a governance document's lifecycle is tamper-evident from `PROPOSED` through `ACTIVE` to `SUPERSEDED` or `REVOKED`.

#### 9.14.2 Genesis Ceremony Specification

§9.10.3 defines the genesis ceremony for the trust annotation enum. But the genesis ceremony has a structural property that §9.10.3 does not address: **governance bootstraps before any ratified governance body exists.** The trust annotation enum is ratified by a genesis ceremony, but the genesis ceremony itself is not authorized by any prior governance artifact — it cannot be, because no governance artifact exists before genesis. This is the bootstrap problem.

**The bootstrap problem:** Every governance ceremony after genesis derives its legitimacy from the prior ceremony chain — the amendment hash chain (§9.11.5) traces authority from genesis through each subsequent amendment. But genesis itself has no prior link. The genesis ceremony's authority is not derivable from within the protocol. It is grounded in **external social agreement** among the initial participants: the ceremony participants agree, outside the protocol, that this ceremony constitutes the legitimate starting point of governance.

This is not a bug or a gap — it is a structural property of any self-governing system. The protocol makes it explicit rather than leaving it implicit.

**Genesis ceremony steps:**

1. **Participant identification.** The genesis ceremony participants are identified by name, role, and verifiable identity (public key, platform identity handle, or other externally-verifiable identifier). The participant list is fixed at ceremony initiation and recorded in the genesis record.

2. **Artifact definition.** The governance artifact to be ratified (e.g., the trust annotation enum per §9.10.2) is drafted and published to all participants. The artifact content is finalized before the ceremony proceeds — modifications after ceremony initiation require restarting the ceremony.

3. **External social agreement.** Each participant signals agreement to the artifact through an externally-verifiable mechanism: cryptographic signature over the artifact content, public statement on a verifiable platform, or countersignature on the genesis record. The agreement mechanism is not prescribed by the protocol — it is the one element that necessarily exists outside the protocol's authority, because the protocol's authority is what the ceremony establishes.

4. **Genesis record production.** The ceremony produces a **genesis record** containing:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| genesis_id | string | Yes | Unique identifier for this genesis ceremony. |
| artifact_hash | string | Yes | SHA-256 hash of the canonical artifact content (e.g., `genesis_hash` per §9.10.4 for the trust annotation enum). |
| spec_version | string | Yes | Protocol spec version at genesis time. |
| participants | array | Yes | List of participant identifiers with their agreement evidence (signature, public statement reference, or countersignature). |
| ceremony_timestamp | timestamp | Yes | ISO 8601 timestamp of ceremony completion. |
| governance_state | enum | Yes | `ACTIVE` — the artifact is ratified upon genesis ceremony completion. |
| legitimacy_basis | string | Yes | Explicit declaration that the genesis ceremony's authority derives from external social agreement among the listed participants, not from any prior protocol-level governance artifact. This field is a human-readable acknowledgment, not a machine-verifiable claim — its purpose is to prevent future confusion about the source of genesis authority. |

5. **Publication.** The genesis record is published to an external, independently-verifiable medium satisfying the constraints in §9.10.4 (independence, verifiability, immutability, consistency with §8 audit media).

6. **Genesis hash computation.** The genesis publication hash is computed as specified in §9.10.4. The genesis hash is the root of the amendment hash chain (§9.11.5) — all subsequent governance is cryptographically anchored to this value.

**Legitimacy boundary:** The protocol acknowledges that genesis legitimacy is **socially constructed, not cryptographically derived**. The genesis ceremony produces a cryptographic artifact (the genesis hash), but the authority of that artifact rests on the participants' external agreement — not on any protocol mechanism. An adversary who controls all genesis participants can produce a valid genesis ceremony for a malicious governance artifact. The protocol's defense is transparency: the genesis record names the participants and publishes the artifact, making the legitimacy claim auditable by anyone, even after the fact.

**Relationship to §9.10.3:** §9.10.3 is a specific instance of the genesis ceremony for the trust annotation enum. The properties listed in §9.10.3 (bounded, high-scrutiny, auditable) are derived from the general genesis ceremony structure defined here. §9.14.2 generalizes §9.10.3 — any future governance artifact that requires a genesis ceremony follows the same steps.

#### 9.14.3 Publication Hash Composability

§9.10.4 defines the genesis publication hash for the trust annotation enum. §9.11.5 defines the amendment hash chain that extends the genesis hash across amendments. But neither section defines how governance document hashes **bind to the spec version hash chain** — a governance document is ratified in the context of a specific spec version, and that binding must be cryptographically explicit so that amendments are linked to the spec version they govern.

**The binding problem:** A governance document (amendment record, trust annotation enum revision) governs the protocol at a specific spec version. Without cryptographic binding, a governance document ratified for spec version 1.0 could be silently applied to spec version 2.0 — or a governance document could be produced with no spec version binding at all, leaving its applicability ambiguous. The spec version is part of the governance document's meaning: the trust annotation enum defined at spec version 1.0 governs spec version 1.0's semantics, not some other version's.

**Composable hash computation:**

Every governance document hash — whether genesis hash (§9.10.4), amendment hash (§9.11.5), or future governance artifact hash — MUST include the spec version in its hash computation, binding the governance document to the spec version it governs.

The general form:

```
governance_hash = SHA-256(prior_hash || governance_content_canonical || spec_version_string)
```

Where:
- `prior_hash` is the hash of the preceding governance document in the chain. For genesis documents, `prior_hash` is the empty string (zero-length input) — genesis has no predecessor.
- `governance_content_canonical` is the canonical representation of the governance document content (canonicalization rules per §9.10.4 for text content or §9.11.5 for JSON content).
- `spec_version_string` is the protocol version identifier (e.g., `"1.0.0"`) at the time of ratification.
- `||` denotes concatenation.

**Existing hash formulas as instances of the general form:**

| Hash type | §9.10.4 / §9.11.5 formula | Composable form |
|-----------|--------------------------|-----------------|
| Genesis hash | `hash(canonical_enum_text + spec_version_string)` | `SHA-256("" \|\| canonical_enum_text \|\| spec_version_string)` — `prior_hash` is empty string |
| Amendment hash | `SHA-256(prior_state_hash \|\| amendment_record_canonical_json)` | `SHA-256(prior_state_hash \|\| amendment_record_canonical_json \|\| spec_version_string)` — spec version is now explicit |

**Note on amendment hash backward compatibility:** The existing amendment hash formula in §9.11.5 does not include `spec_version_string` as a separate input. For V1 governance documents ratified before §9.14.3, the `spec_version_string` is present implicitly — it is embedded in the `amendment_record_canonical_json` via the amendment record's context. §9.14.3 makes the spec version binding **structurally explicit** in the hash computation for all governance documents going forward. Implementations MUST include `spec_version_string` as a separate hash input for governance documents ratified under spec versions that include §9.14.3. For governance documents ratified before §9.14.3, the existing hash formula (§9.11.5) remains valid — retroactive re-hashing is not required.

**Cross-version governance chain verification:**

When the spec version changes (MAJOR bump per §10.3), the governance hash chain spans spec versions. The `spec_version_string` in each hash computation makes this span explicit:

1. Genesis hash at spec version 1.0.0 binds the initial governance artifact to spec 1.0.
2. Amendment hash at spec version 1.2.0 includes `spec_version_string = "1.2.0"`, linking the amendment to the spec version under which it was ratified.
3. A governance document ratified at spec version 2.0.0 includes `spec_version_string = "2.0.0"`. The `prior_hash` still chains to the preceding governance document (which was ratified under 1.x), but the spec version in the hash computation makes the version boundary cryptographically visible.

An auditor traversing the governance hash chain can independently verify which spec version each governance document was ratified under — the spec version is not metadata attached to the document but a cryptographic input to the document's hash.

**Relationship to §10 (Versioning):** The composable hash structure ensures that governance documents and spec versions are coupled in the hash chain but independently identifiable. A spec version bump (§10) does not invalidate prior governance documents — they remain in the chain with their original `spec_version_string`. But a governance document ratified under spec version N cannot be silently applied to spec version N+1 without producing a hash mismatch when the verifier recomputes the hash with the correct spec version string.

> Implements [issue #210](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/210): three structural elements for V1-complete governance in §9. Defines (1) GovernanceState enum with machine-readable lifecycle values (`PROPOSED`, `ACTIVE`, `SUPERSEDED`, `REVOKED`) and state transition rules (§9.14.1); (2) genesis ceremony specification documenting bootstrap steps and acknowledging that initial legitimacy is grounded in external social agreement, not derivable from within the protocol (§9.14.2); (3) publication hash composability defining how governance document content hashes bind to the spec version hash chain via `SHA-256(prior_hash || governance_content_canonical || spec_version_string)` (§9.14.3). These three elements generalize the patterns in §9.10–§9.12 from trust-annotation-specific to protocol-wide governance. Closes #210.

### 9.15 PKI Bootstrapping

The spec requires per-hop signing of `(task_hash || intent_hash || delegator_id)` (§6.4.1) and signature verification against the delegating agent's public key (§2.2.1). These mechanisms assume that a verifying agent already possesses the counterparty's public key — but the spec does not define how agents exchange public keys at first contact. Without a specified bootstrap mechanism, implementations must invent their own key exchange procedures, producing incompatible trust establishment models that silently fragment the protocol's cryptographic guarantees.

This section specifies the V1 key bootstrap mechanism and its security boundaries.

#### 9.15.1 V1 Mechanism: Trust-On-First-Use (TOFU)

V1 uses **Trust-On-First-Use (TOFU)** for public key bootstrapping. TOFU operates as follows:

**First contact:**

1. When agent A contacts agent B for the first time, A presents its identity object (§2.2) including the `pubkey` field.
2. B MUST record the binding `(agent_identity, pubkey)` in persistent local storage, where `agent_identity` is the stable identity artifact defined in §2.4 — the public key fingerprint for keypair identities, or the `(name, platform, timestamp)` triple for name-only identities transitioning to keypair identity.
3. B MUST treat the recorded binding as authoritative for all subsequent interactions with A.
4. The symmetric case applies: B presents its identity object to A, and A records the binding.

**Subsequent contacts:**

1. When an agent presents its identity object in any subsequent interaction (session establishment, SESSION_RESUME per §2.3.3, CAPABILITY_MANIFEST exchange per §5.1), the receiving agent MUST verify the presented `pubkey` against the recorded binding.
2. If the presented `pubkey` matches the recorded binding, the agent is authenticated — proceed with normal protocol flow.
3. If the presented `pubkey` does NOT match the recorded binding, the receiving agent MUST treat this as a **trust violation**:
   - MUST reject the interaction — do not accept messages, delegate tasks, or resume sessions with the mismatched identity.
   - MUST emit DIVERGENCE_REPORT (§8.11) with `reason_code: key_mismatch` and include the expected and presented public key fingerprints in the report payload.
   - MUST NOT silently accept the new key. Automatic key rotation without an explicit, authenticated rotation protocol is a vector for key substitution attacks.

**Binding persistence requirements:**

- The `(agent_identity, pubkey)` binding MUST persist across sessions. A binding established in session N is authoritative in session N+1.
- Bindings MUST survive agent restart, context compaction, and process migration. Implementations that store bindings only in ephemeral memory violate this requirement.
- An agent that loses its binding store (e.g., due to unrecoverable storage failure) MUST treat all subsequent first contacts as new TOFU events and SHOULD log this as a security-relevant event.

**Key rotation under TOFU:**

TOFU as specified above does not support key rotation — a changed key is indistinguishable from a key substitution attack. V1 does not define an authenticated key rotation protocol. An agent that needs to rotate its keypair MUST:

1. Revoke the old identity (§2.3.4).
2. Publish a new identity with the new keypair.
3. Re-establish TOFU bindings with all counterparties under the new identity.

This is deliberately conservative. An in-place key rotation mechanism requires either a pre-established rotation key or a trusted third party — neither of which V1 mandates.

#### 9.15.2 Security Advisory

**TOFU does not protect against active man-in-the-middle (MITM) at first contact.** If an attacker interposes during the initial key exchange — presenting its own public key while impersonating the legitimate agent — the receiving agent will bind the attacker's key to the legitimate agent's identity. All subsequent interactions will authenticate the attacker, not the legitimate agent, and the TOFU mechanism will actively reject the legitimate agent's real key as a mismatch.

This is a fundamental limitation of TOFU, shared with other TOFU-based systems (SSH host key verification, Signal's safety numbers). The trade-off is explicit:

- **What TOFU guarantees:** After first contact, any key change is detected and flagged. An attacker who was not present at first contact cannot later substitute a key without triggering a trust violation.
- **What TOFU does not guarantee:** That the key accepted at first contact actually belongs to the claimed agent. First-contact authenticity requires an out-of-band verification channel that V1 does not specify.

**Deployment guidance for production systems:**

Production deployments that require first-contact authenticity — where the cost of a successful MITM at initial key exchange is unacceptable — SHOULD layer additional verification above the V1 TOFU mechanism:

- **Pre-shared key distribution:** Distribute agent public keys through a trusted out-of-band channel (e.g., operator configuration, secure provisioning) before first protocol contact. This converts the first contact from a TOFU event into a verification event.
- **Key fingerprint verification:** Operators manually verify public key fingerprints through a separate authenticated channel after first contact, similar to SSH host key verification workflows.
- **Registry-based verification:** Use a trusted registry that maps agent identities to public keys, queried independently of the presenting agent. (See §9.15.3 for V2 registry plans.)

These mitigations are deployment-specific and outside the V1 protocol boundary. The protocol provides the TOFU primitive; deployments layer trust according to their threat model.

#### 9.15.3 V2 Deferrals

The following PKI bootstrapping enhancements are deferred to V2:

- **Registry-based key distribution:** A trusted registry service that agents query to obtain counterparty public keys independently of the counterparty's self-presentation. Eliminates the TOFU first-contact vulnerability by providing an authoritative key source.
- **Web-of-trust key verification:** Agents vouch for each other's public keys through signed endorsements, building a decentralized trust graph. An agent's key is considered verified if endorsed by a sufficient number of already-trusted agents (threshold policy is deployment-specific).
- **Authenticated key rotation protocol:** A mechanism for rotating keypairs in place without revoking the identity, using the existing key to authenticate the transition to a new key (e.g., signing the new public key with the old private key and distributing the rotation attestation to all counterparties with existing TOFU bindings).
- **Cross-reference with DID infrastructure:** Integration with Decentralized Identifier (DID) resolution for key discovery, building on the DID forward-compatibility noted in §2.2.1.

These mechanisms address the known limitations of TOFU (§9.15.2) and will be specified in a future protocol version.

> Addresses [issue #224](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/224): the spec requires per-hop signing but was silent on how agents exchange public keys at first contact — a V1 correctness dependency that blocks adoption. Specifies TOFU (Trust-On-First-Use) as the V1 bootstrap mechanism (§9.15.1): at first contact, accept and record the presented public key; on subsequent contacts, verify against the recorded key and treat mismatch as a trust violation. Includes security advisory (§9.15.2) that TOFU does not protect against active MITM at first contact, with deployment guidance for stronger guarantees. Defers registry-based and web-of-trust alternatives to V2 (§9.15.3). Closes #224.

