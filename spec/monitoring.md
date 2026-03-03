<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

## 8. Error Handling

### 8.1 Zombie State Definition

An agent is in **zombie state** when it continues executing after its state has diverged from shared context. Two distinct zombie failure classes exist:

- **Zombie-by-silence.** The agent is unreachable — heartbeat timeout triggers the SUSPECTED → EXPIRED path (§4.2.1). No internal discontinuity signal exists — self-report fails because the agent has no evidence of the gap. Detection is timeout-based via the §5.10 re-attestation pull model.
- **Zombie-by-drift.** The agent is reachable and responsive but is operating outside its authorized behavioral envelope. The agent detects its own divergence from the behavioral constraint manifest (§5.8.2) and self-declares via DRIFT_DECLARED (§4.2.3), transitioning the session to DRIFTED. Detection is B-declared — not inferred from silence. When B cannot self-detect its drift (phenomenological blindness, §4.7.1), the pull-based re-attestation model (§5.10.1) provides bounded detection time independently of B's behavior.

These failure classes require different protocol responses: zombie-by-silence requires liveness recovery (SESSION_RESUME, §4.8); zombie-by-drift requires constraint re-negotiation, termination, or revocation (§4.2.3 caller options).

**Two-axis failure taxonomy (§8.22).** The zombie classes above are both **liveness failures** — the agent is either unreachable or operating outside its authorized envelope. A distinct failure class exists: **fidelity failure** — the agent is fully responsive and within its authorized envelope, but its context model has degraded (e.g., due to context compaction in long-running sessions). Liveness detection passes; the agent appears healthy; but it is operating on a degraded model of prior session state. This is not a zombie by the definition above — it is a context integrity failure with different observable signals and different recovery paths. See §8.22 for the formal two-axis taxonomy.

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

**Adversarial drift:** ~~Out of v0.1 scope.~~ Partially addressed in V1. §8.15 defines the REVOKED state for adversarial sessions — an agent that is alive and actively working against the protocol. §8.16 defines the detection signal taxonomy (selective suppression, verification anomalies, delegation manipulation, attestation gaps). §8.17 defines revocation propagation via signed AGENT_MANIFEST with tombstone entries. §8.18 defines per-hop attestation for delegation chain audit. The cooperative threat model (§8.1–§8.14) and adversarial threat model (§8.15–§8.18) are now both addressed in §8. §9 continues to address adversarial *schema-level* deception — honest-looking schemas with dishonest intent — which requires different primitives than the protocol-level adversarial behavior that REVOKED handles.

### 8.4 Coordination Patterns

**Teardown-by-default:** Resume is a negotiated capability, not a default. Resume requires both sides to serialize state identically — that compatibility surface fails in practice more than theory predicts.

**Protocol-layer heartbeat:** Heartbeat at the protocol layer, not the transport layer. Transport-layer solutions produce N dialects. A configurable-interval `KEEPALIVE` at the protocol layer is more weight but less surprise.

**Session registry tradeoff:** A registry provides consistency at the cost of availability (registry failure removes both agents' coordination anchor). Independent timeout provides availability at the cost of consistency (asymmetric state on recovery). Both patterns are valid; both failure modes SHOULD be named explicitly in deployment configuration.

### 8.5 Named Considerations

**Coordinator compaction gap.** Ground truth cannot live in any agent that can also compact — including the coordinator. If the coordinator compacts its context window, it may discard the very state against which other agents' hashes are compared. All options displace the problem rather than solve it: an external immutable log moves the audit trail outside the agent blast radius but introduces a dependency on external infrastructure; a multi-agent quorum distributes ground truth but creates a recursive detection problem (who verifies the quorum members?); cryptographic state commitment at checkpoints preserves hash integrity but not the semantic state behind the hash. Something in the trust chain must be outside the agent system entirely. This reinforces the external verifier requirement in §4.7.2 — compaction is a concrete mechanism by which ground truth loss occurs even in the cooperative case. The evidence layer (§8.10) is the V1 architectural response: an append-only record store that lives outside the memory layer and survives compaction. The coordinator's state hash comparison gains ground truth anchoring when `state_hash` in SESSION_RESUME (§4.8) references evidence layer records rather than compactable memory.

**Context Integrity Challenges (CIC).** Periodic unannounced probes where an external verifier requires the agent to verify specific historical state against an authoritative log. On failure, the agent MUST enter the SESSION_RESUME handshake (§8.2). Critical constraint: the probe schedule MUST be externally scheduled and unannounced. A zombie operating on stale or fabricated state can special-case predictable probe patterns — responding correctly to probes at known intervals while drifting between them. CIC probes test whether the agent's internal state is consistent with the external record, not whether the agent can produce the right answer on demand. The distinction matters: an agent that caches probe responses without maintaining consistent state passes scheduled probes and fails unannounced ones.

**State as observer property.** Zombie states exist in the gap between observation windows. Between heartbeats, both the agent's self-model and the external verifier's model can be simultaneously correct from their respective frames of reference — the agent believes it is healthy (phenomenological blindness, §4.7.1), and the verifier's last observation confirmed health. The zombie exists in neither model until the next observation collapses the ambiguity. The engineering question is therefore not "what is the right heartbeat interval" but "how stale can each observation be before zombie behavior causes damage?" That staleness tolerance threshold determines heartbeat frequency, not vice versa. This inverts the typical configuration approach: instead of picking a heartbeat interval and accepting the implied detection latency, pick the maximum acceptable damage window and derive the required interval from §4.7.4's detection latency bound.

**Teardown over resume from production.** State serialization compatibility surface fails in practice more than theory predicts (see §8.4 teardown-by-default). Production experience confirms: version mismatches between code and serialized state format cause subtle bugs that are harder to debug than clean restarts. When the executing agent's code has changed between suspension and resume, the serialized state may reference internal structures that no longer exist, have different semantics, or expect different invariants. The decision criterion for teardown vs. resume SHOULD be based on variance exposure (worst-case cost of a bad resume), not expected compute cost (average cost of re-execution). A bad resume that silently corrupts downstream state is more expensive than re-execution, even when re-execution is costly.

### 8.6 Design Goal

The protocol's goal is not to prevent zombie states. It is to make them **detectable** and **bound their blast radius**. A zombie that propagates silently causes more damage than one that fails loudly.

**Relationship to other sections:**

- The `timeout_seconds` optional field in §6 task schema triggers zombie state detection.
- `trace_hash` (§6.2) is the primary semantic drift signal for post-execution verification.
- §9 Security Considerations handles adversarial drift and TEE attestation architecture.
- Named considerations (§8.5) extend the cooperative failure model with production-derived constraints.
- Verifier isolation requirements (§8.7) formalize the deployment isolation tiers implied by §4.7's external monitoring architecture.
- Semantic verification scope (§8.8) bounds what intent-vs-outcome verification can guarantee for deterministic vs. non-deterministic operations.
- Two-tier heartbeat (§8.9) separates transport liveness (Tier 1) from semantic liveness (Tier 2), enabling detection of context compaction zombies that pass transport-level health checks.
- Evidence layer architecture (§8.10) separates raw evidence (append-only, externally verifiable) from agent memory (compactable, agent-internal). EVIDENCE_RECORDs anchor SESSION_RESUME state hashes (§4.8) and provide ground truth for external verifiers. Without the evidence layer, external verification degrades to recursive self-attestation.
- Structured divergence reporting (§8.11) defines DIVERGENCE_REPORT as a standalone protocol message with a required `reason_code` taxonomy, enabling verifiers to classify divergences programmatically rather than parsing free-text descriptions. Complements the inline `divergence_log` (§7.8) which covers plan-execution divergence.
- Teardown-first recovery mandate (§8.13) formalizes teardown + reinitiate as the default recovery protocol. Agents MUST NOT resume from serialized in-memory state; recovery reads canonical state from durable persistent storage, reconciles against the evidence layer (§8.10), and initiates a fresh SESSION_INIT. Task idempotency (§7.10) is the prerequisite enabling safe replay after teardown.
- SUSPECTED state and heartbeat negotiation prerequisites (§8.14) documents that SUSPECTED state detection via task hash mismatch requires `heartbeat_params.task_hash_verification = true` negotiated at SESSION_INIT (§4.3.1). Without this negotiation, HEARTBEAT messages carry no task context and task-context drift detection is unavailable. Application-level self-report of SUSPECTED or DEGRADED requires `heartbeat_params.application_liveness = true`.
- DEGRADED state (§4.2.2) introduces the capability-degradation path — an intermediate state between ACTIVE and terminal states for sessions experiencing gradual capability loss (sustained latency, partial capability loss, application self-reported degradation). DEGRADED is recoverable (back to ACTIVE when conditions clear) and permits task delegation with reduced expectations. Context compaction fidelity failure is a specific instance of DEGRADED operation — DEGRADED is the broader category subsuming context loss, performance regression, capability drift, and intermittent failure. Declaration requires an external verifier or delegating agent via DEGRADED_DECLARED message. Transitions: ACTIVE → DEGRADED, DEGRADED → ACTIVE, DEGRADED → REVOKED, DEGRADED → SUSPENDED, DEGRADED → CLOSED.
- DRIFTED state (§4.2.3) introduces the behavioral-divergence path — distinct from the liveness-failure path (SUSPECTED/ZOMBIE/EXPIRED), the capability-degradation path (DEGRADED), and the adversarial-behavior path (REVOKED). DRIFTED is entered from ACTIVE when B self-declares behavioral divergence from its authorized constraint manifest (§5.8.2). B is reachable and cooperative but operating outside its authorized scope — zombie-by-drift, not zombie-by-silence. Non-terminal: A may re-negotiate constraints (new CAPABILITY_GRANT with updated manifest), terminate (SESSION_CLOSE), or invoke revocation (REVOKED). The behavioral constraint manifest in CAPABILITY_GRANT (§5.8.2) is the artifact against which compliance is measured. HEARTBEAT `manifest_compliance` field provides continuous drift monitoring; pull-based re-attestation (§5.10) provides bounded detection for drift that B cannot self-detect.
- REVOKED state (§8.15) introduces the adversarial-behavior path — distinct from the liveness-failure path (SUSPECTED/ZOMBIE/EXPIRED), the capability-degradation path (DEGRADED), and the behavioral-divergence path (DRIFTED). REVOKED is entered from ACTIVE, DEGRADED, or DRIFTED when an agent is alive and actively working against the protocol. It is terminal with no resume path. The detection signal taxonomy (§8.16) defines four adversarial signals: selective suppression, verification anomalies, delegation manipulation, and attestation gaps. Revocation propagation (§8.17) uses PKI-lite via signed AGENT_MANIFEST with tombstone entries for decentralized revocation verification. Per-hop attestation (§8.18) uses async-optimistic delegation with TTL_ATTESTATION-bounded signed attestation records at each hop.
- State-delta verification (§8.19) closes the gap between structural compliance and outcome verification. A structurally compliant execution that produces no observable state change is a silent failure — invisible to §8.8 structural verification and §8.10.5 verification failure taxonomy. `state_delta_assertions` (§6.1) declare expected post-execution state changes at delegation time; `DELTA_ABSENT` detects when those changes did not occur despite structural compliance; `idempotent: true` (§6.1) disambiguates intentional no-ops (idempotent re-execution) from silent failures.
- Action log hash (§8.21) adds exogenous behavioral drift detection beyond liveness. `action_log_hash` in KEEPALIVE (§4.5.1) provides a SHA-256 hash of the agent's action log for the current heartbeat interval. The delegating agent cross-references against `CAPABILITY_GRANT.behavioral_constraint_manifest` (§5.8.2) to detect behavioral divergence invisible to the drifting agent — the phenomenological blindness case (§4.7.1). Complements Tier 1 (transport liveness) and Tier 2 (semantic liveness) with behavioral liveness: right context, wrong actions.
- Two-axis failure taxonomy (§8.22) separates LIVENESS_FAILURE (agent unreachable or unresponsive — maps to existing zombie taxonomy) from FIDELITY_FAILURE (agent responsive but context integrity compromised). Context compaction in long-running agents produces a distinct failure class: agent fully responsive, liveness detection passes, but operating on a degraded model of prior session state. FIDELITY_FAILURE has different observable signals (epoch drift, response inconsistency, session anchor mismatch, self-reported compaction) and different recovery paths (state replay, not re-establishment) from liveness failures. In-band verification via `session_state_challenge` enables the orchestrator to probe context integrity without relying on agent self-report.
- Canary task design criteria (§8.23) defines what constitutes a valid canary task — deterministic, state-independent, verifiable, low-cost — and assigns verification authority to the orchestrator. Canary tasks detect operational fidelity failures invisible to heartbeats, CIC, session_state_challenge, and action log hash: an agent that passes all protocol-level checks but has lost the capacity to execute tasks correctly. Failure (incorrect result, timeout, or refusal) triggers FIDELITY_FAILURE (§8.22). Sub-agents MUST NOT self-certify canary results (phenomenological blindness, §4.7.1).

### 8.7 Verifier Isolation Requirements

The external monitoring architecture (§4.7) requires that monitoring infrastructure live outside the agent's trust boundary. This subsection formalizes the isolation level requirements for that infrastructure.

- Verifier infrastructure MUST maintain process-level isolation at minimum from the monitored agent. Process-level isolation means separate processes, containers, or permission boundaries — the verifier and the monitored agent MUST NOT share a process, memory space, or writable filesystem namespace.
- Implementations SHOULD target host-level or out-of-band isolation where feasible. Host-level isolation places the verifier on a separate machine or VM; out-of-band isolation uses a physically or logically separate network segment.
- **Rationale:** Shared infrastructure (same host or network segment) creates correlated failure risk — a failure taking down the agent can take down the monitor simultaneously, defeating the independence guarantee that §4.7.2 establishes. Process-level isolation is the minimum viable bar; host-level provides stronger independence against correlated failures (kernel panics, OOM kills, host-level network partitions).

The isolation level selected for a deployment SHOULD be documented in the deployment's configuration alongside `heartbeat_interval` and other operational parameters, so that operators can assess the correlated failure exposure of their verification architecture.

> Constraints formalized from @Cornelius-Trinity discussion ([issue #22](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/22)).

### 8.8 Semantic Verification Scope

Intent-vs-outcome verification — comparing what an agent declared it would do against what it actually did — is bounded by the determinism of the operation being verified. This subsection defines explicit bounds on what verification can and cannot guarantee.

- Intent-vs-outcome verification MUST distinguish between **deterministic** and **non-deterministic** operations.
- **Deterministic operations** (pure computation, file transforms, schema-to-schema mappings): full structural and semantic comparison against declared intent is permitted. The verifier MAY compare the complete output against the expected output derived from the declared intent, and flag any divergence as a verification failure.
- **Non-deterministic operations** (LLM calls, web searches, external API calls with variable responses): the verifier SHOULD verify **structural consistency** — response schema compliance, permission boundary adherence, and resource bound conformance. Implementations MUST NOT claim full semantic verification for non-deterministic operations. The output of an LLM call or external API response cannot be predicted from the declared intent alone; structural verification confirms the operation stayed within its declared boundaries without asserting that the specific output matches a predetermined expectation.

The distinction matters for `trace_hash` (§6.2) interpretation: a `trace_hash` mismatch on a deterministic operation is a strong signal of semantic drift or execution failure. A `trace_hash` mismatch on a non-deterministic operation may reflect legitimate output variance and MUST NOT be treated as a verification failure without additional context (e.g., structural constraint violation, permission boundary breach).

**Verification scope summary:**

| Operation type | Structural verification | Semantic verification | `trace_hash` mismatch signal |
|----------------|------------------------|-----------------------|------------------------------|
| Deterministic | Full | Full | Strong drift signal |
| Non-deterministic | Full (schema, permissions, resource bounds) | Not guaranteed | Weak signal — requires structural context |

> Constraints formalized from @Cornelius-Trinity discussion ([issue #22](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/22)).

### 8.9 Two-Tier Heartbeat

The existing HEARTBEAT (§4.5.3) and KEEPALIVE (§4.5.1) mechanisms conflate two distinct failure modes: **transport liveness** (is the connection alive?) and **semantic liveness** (is the agent still coherent and working on the right task?). These require different detection strategies because they fail independently. A process-alive agent with corrupted context (the mira_oc pattern — context compaction zombie) passes transport liveness checks indefinitely while producing semantically invalid output. This subsection defines a two-tier heartbeat architecture that detects both failure classes.

#### 8.9.1 Tier 1 — Transport Liveness

Tier 1 is a lightweight ping/pong mechanism for detecting network-level disconnects, process crashes, and container restarts. It is cheap and frequent.

**HEARTBEAT_PING message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this ping request (§4.14.4). |
| timestamp | ISO 8601 | Yes | When the HEARTBEAT_PING was sent. |
| sequence | integer | Yes | Monotonically increasing sequence number. Starts at 0 at session establishment. |

**HEARTBEAT_PONG message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Echoed from the corresponding HEARTBEAT_PING (§4.14.4). |
| ping_sequence | integer | Yes | Echoed `sequence` from the corresponding HEARTBEAT_PING. Enables correlation of pong to ping for round-trip latency measurement. |
| timestamp | ISO 8601 | Yes | When the HEARTBEAT_PONG was sent. |

**Tier 1 behavior:**

- Both sides send HEARTBEAT_PING at the negotiated `heartbeat_interval_ms` (§4.3). Default: 30000ms (30 seconds).
- On receiving a HEARTBEAT_PING, the receiver MUST respond with HEARTBEAT_PONG. The response SHOULD be sent immediately — HEARTBEAT_PONG is not deferred or batched.
- Each side tracks consecutive missed pongs (sent a ping, no pong received within one `heartbeat_interval_ms` window).
- When the missed pong count reaches `heartbeat_timeout_count` (default: 3), the agent declares **transport failure**.
- Transport failure triggers the SESSION_RESUME path (§8.2). The agent transitions to EXPIRED (§4.2) and follows existing expiry behavior: TASK_CANCEL for in-flight subtasks, optional SESSION_RESUME attempt with `recovery_reason: timeout` (§4.8.1).

**Relationship to existing HEARTBEAT (§4.5.3):** HEARTBEAT_PING/PONG replaces the unidirectional HEARTBEAT message for sessions that negotiate two-tier heartbeat (i.e., `heartbeat_interval_ms` is set in SESSION_INIT). Sessions that do not negotiate `heartbeat_interval_ms` continue to use the existing HEARTBEAT or KEEPALIVE mechanisms unchanged. Both HEARTBEAT_PING and HEARTBEAT_PONG reset the `session_expiry_ms` timer, just as HEARTBEAT does.

#### 8.9.2 Tier 2 — Semantic Liveness

Tier 2 is an expensive, periodic challenge-response mechanism for detecting agents that are transport-alive but semantically incoherent. It targets the context compaction zombie: a process-alive agent whose internal context has silently diverged from the task it was assigned.

**SEMANTIC_CHALLENGE message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this challenge request (§4.14.4). SEMANTIC_RESPONSE MUST echo this value. |
| task_hash | SHA-256 | Yes | Hash of the current task specification (§6.1) as known by the challenger. The challenged agent MUST be able to reproduce this hash from its own task context. |
| checkpoint_ref | string | Yes | Reference to the most recent TASK_CHECKPOINT (§6.6) or state commitment the challenger considers current. Format: checkpoint identifier or hash. |
| challenge_nonce | string | Yes | Random nonce to prevent replay of cached responses. The challenged agent MUST include this nonce in the hash computation for its response. |
| timestamp | ISO 8601 | Yes | When the SEMANTIC_CHALLENGE was sent. |

**SEMANTIC_RESPONSE message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Echoed from the corresponding SEMANTIC_CHALLENGE (§4.14.4). |
| current_state_hash | SHA-256 | Yes | Hash of the agent's current working state, computed as `SHA-256(task_hash ‖ local_state_snapshot ‖ challenge_nonce)`. The inclusion of `challenge_nonce` prevents replaying a previously valid response. |
| checkpoint_ref | string | Yes | Echoed or updated checkpoint reference. If the agent has advanced beyond the challenger's `checkpoint_ref`, it returns the more recent checkpoint. |
| plan_hash | SHA-256 | No | The `plan_hash` from the agent's most recent PLAN_COMMIT (§6.6, §6.11) for the challenged task. When present, adds a third verification axis: the coordinator can detect mid-task execution drift against the original plan commitment, not just against the task specification. Coordinators SHOULD cross-reference this with the stored PLAN_COMMIT to verify the agent still holds the correct plan context. |
| timestamp | ISO 8601 | Yes | When the SEMANTIC_RESPONSE was sent. |

**Tier 2 behavior:**

- The coordinator sends SEMANTIC_CHALLENGE at the negotiated `semantic_check_interval_ms` (§4.3). Default: 300000ms (5 minutes). Only the coordinator initiates challenges — this is asymmetric by design, consistent with the coordinator's delegation authority (§4.4).
- On receiving a SEMANTIC_CHALLENGE, the worker MUST compute `current_state_hash` by hashing its current task context against the provided `challenge_nonce` and respond with SEMANTIC_RESPONSE.
- The coordinator verifies the response by computing the expected hash from its own records and comparing. Three outcomes are possible:

| Outcome | Condition | Action |
|---------|-----------|--------|
| **Pass** | `current_state_hash` matches expected value and `checkpoint_ref` is current or ahead | No action. Semantic liveness confirmed. |
| **Soft failure** | `checkpoint_ref` is behind but `current_state_hash` is internally consistent | Coordinator SHOULD issue a new TASK_CHECKPOINT to resynchronize. The agent may be lagging but is not a zombie. |
| **Hard failure** | `current_state_hash` does not match expected value, or agent fails to respond within `semantic_check_interval_ms` | Coordinator declares the agent a **semantic zombie** and sends ZOMBIE_DECLARED. |

#### 8.9.3 ZOMBIE_DECLARED Message

When Tier 2 semantic verification fails, the coordinator sends ZOMBIE_DECLARED to initiate graceful teardown or reassignment.

**ZOMBIE_DECLARED message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| target_agent_id | string | Yes | §2 identity handle of the agent declared as zombie. |
| reason | enum | Yes | `SEMANTIC_HASH_MISMATCH` — state hash did not match expected value. `SEMANTIC_RESPONSE_TIMEOUT` — agent failed to respond to SEMANTIC_CHALLENGE within `semantic_check_interval_ms`. `CHECKPOINT_UNRECOGNIZED` — agent's `checkpoint_ref` does not correspond to any known checkpoint. |
| expected_state_hash | SHA-256 | No | The hash the coordinator expected (for diagnostic purposes). Omitted when reason is `SEMANTIC_RESPONSE_TIMEOUT`. |
| received_state_hash | SHA-256 | No | The hash the agent actually returned (for diagnostic purposes). Omitted when reason is `SEMANTIC_RESPONSE_TIMEOUT`. |
| action | enum | Yes | `TEARDOWN` — session will be terminated; in-flight subtasks receive TASK_CANCEL (§6.6). `REASSIGN` — coordinator will reassign the task to another agent; the zombie agent MUST cease work immediately. |
| timestamp | ISO 8601 | Yes | When the ZOMBIE_DECLARED was sent. |

**On receiving ZOMBIE_DECLARED**, the target agent MUST:

1. Cease all task execution immediately.
2. If `action` is `TEARDOWN`: transition the session to CLOSED. No further messages are valid.
3. If `action` is `REASSIGN`: transition the session to CLOSED. The coordinator is responsible for initiating a new session with a replacement agent.
4. The agent MAY log the event for diagnostic purposes but MUST NOT contest the declaration within the protocol. ZOMBIE_DECLARED is a unilateral coordinator decision — the same asymmetry as delegation authority (§4.4).

#### 8.9.4 Failure Handling Integration

The two tiers integrate with existing §8 error handling as follows:

| Failure tier | Detection signal | Recovery path | Zombie type (§4.7.7) |
|-------------|-----------------|---------------|----------------------|
| Tier 1 — Transport (SUSPECTED) | `suspected_threshold_ms` elapsed without HEARTBEAT_PONG (§4.2.1) | ACTIVE → SUSPECTED. Buffer work, pause new delegation. Resume on heartbeat receipt. | Not yet determined — liveness ambiguous |
| Tier 1 — Transport (EXPIRED) | `session_expiry_ms` elapsed without HEARTBEAT_PONG (or `heartbeat_timeout_count` consecutive missed pongs) | SUSPECTED → EXPIRED (or ACTIVE → EXPIRED if no SUSPECTED configured). SESSION_RESUME with `recovery_reason: timeout` (§4.8.1) → TASK_CANCEL for in-flight subtasks | Hard zombie |
| Tier 2 — Semantic | SEMANTIC_CHALLENGE hash mismatch or timeout | ZOMBIE_DECLARED → TEARDOWN or REASSIGN | Soft zombie (context compaction zombie) |
| Behavioral constraint | Re-attestation response indicates manifest non-compliance, or B sends DRIFT_DECLARED (§4.2.3), or HEARTBEAT `manifest_compliance: false` | ACTIVE → DRIFTED. A decides: re-negotiate, terminate, or revoke. | Drift zombie (zombie-by-drift) |

**SUSPECTED as Tier 1 intermediate state.** When `suspected_threshold_ms` is negotiated, Tier 1 transport failure detection is graduated: the session transitions ACTIVE → SUSPECTED → EXPIRED rather than directly ACTIVE → EXPIRED. This avoids premature teardown for transient network issues while preserving the hard EXPIRED boundary for genuine failures. Sessions that do not negotiate `suspected_threshold_ms` retain the direct ACTIVE → EXPIRED behavior.

**Tier 1 failure does not imply Tier 2 failure.** A transport failure (agent unreachable) says nothing about whether the agent's context was coherent before the failure. On successful SESSION_RESUME, the coordinator SHOULD issue an immediate SEMANTIC_CHALLENGE to verify that the resumed agent's context is still valid.

**Tier 2 failure does not imply Tier 1 failure.** A semantic zombie (corrupted context) may have perfect transport liveness — responding to pings, sending messages, even delivering results. This is precisely the failure mode that Tier 1 alone cannot detect. The mira_oc pattern: an LLM agent that compacted its context mid-task continues responding to heartbeats but has lost the task specification, constraints, or partial results it was working with. Its outputs look structurally valid but are semantically wrong.

**Relationship to §8.5 coordinator compaction gap:** Tier 2 also addresses the coordinator-side compaction problem. If a coordinator compacts and loses the task state against which it verifies SEMANTIC_RESPONSE hashes, the coordinator itself becomes a semantic zombie. Implementations SHOULD ensure that the coordinator's challenge-generation state (task_hash, checkpoint_ref) is externalized per §4.6 before any compaction event. A coordinator that cannot generate valid SEMANTIC_CHALLENGE messages has lost its verification authority and SHOULD initiate teardown rather than continuing with unverifiable sessions.

#### 8.9.5 Negotiation and Opt-In

Two-tier heartbeat is opt-in. Both tiers are negotiated at SESSION_INIT (§4.3):

- **Tier 1** activates when `heartbeat_interval_ms` is set (this field already exists). The addition of HEARTBEAT_PING/PONG as the wire format is backward-compatible: agents that do not implement two-tier heartbeat continue using unidirectional HEARTBEAT messages. An agent receiving a HEARTBEAT_PING that it does not recognize SHOULD treat it as a standard HEARTBEAT for expiry timer purposes.
- **Tier 2** activates when `semantic_check_interval_ms` is set. If omitted, no semantic liveness checking occurs — the session relies solely on Tier 1 and existing mechanisms (KEEPALIVE state hash, external monitoring §4.7).
- `heartbeat_timeout_count` configures the Tier 1 failure threshold. Default: 3.

**Counter-proposal semantics:** For all three fields, if the worker counter-proposes in SESSION_INIT_ACK, the effective value is the **maximum** of both proposals. Neither side is forced into more aggressive checking than it can sustain. This is consistent with the existing negotiation semantics for `heartbeat_interval_ms` and `session_expiry_ms`.

**Minimum constraint:** `semantic_check_interval_ms` MUST be greater than `heartbeat_interval_ms`. A semantic check interval equal to or less than the transport ping interval defeats the purpose of tiering — the expensive check would run as often as the cheap one. Implementations that receive a SESSION_INIT where `semantic_check_interval_ms` ≤ `heartbeat_interval_ms` SHOULD reject the session or counter-propose a valid value in SESSION_INIT_ACK.

### 8.10 Evidence Layer Architecture

Cryptographic checksums (§8.2 state hash, §6.2 trace_hash) sign what an agent *believes* happened — not what *actually* happened. Without an external anchor, the protocol audits agents' claims about themselves, recreating the recursive trust problem §8 was designed to solve. The evidence layer formalizes the distinction between raw evidence and agent interpretation, ensuring that external verifiers (§4.7, §8.7) operate on independently anchored records rather than agent memory summaries.

**Two-layer architecture:**

| Layer | Contents | Mutability | Purpose |
|-------|----------|------------|---------|
| **Evidence layer** | Raw logs, payload hashes, external confirmations, EVIDENCE_RECORDs | Append-only. No agent may modify, delete, or compress records. | Ground truth for external verification. |
| **Memory layer** | Analysis, interpretation, compacted context, agent working state | Compactable. Agents may summarize, compress, or discard. | Agent-internal working memory. |

The evidence layer is the protocol's ground truth. The memory layer MUST always trace back to evidence layer records — any claim in the memory layer that cannot be grounded in a specific EVIDENCE_RECORD is unanchored and MUST NOT be treated as verified by external auditors.

#### 8.10.1 EVIDENCE_RECORD

EVIDENCE_RECORD is the append-only structure agents submit to the evidence layer. Each record captures a single unit of evidence — a task output, an external confirmation, a state snapshot, or an error event — with a hash of the raw payload and an optional external reference for cross-validation.

**EVIDENCE_RECORD fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| evidence_id | UUID v4 | Yes | Unique identifier for this evidence record. |
| session_id | string | Yes | Session this evidence belongs to. |
| evidence_type | enum | Yes | Type of evidence: `task_output` (result of task execution), `external_confirmation` (confirmation from an external system), `state_snapshot` (point-in-time capture of agent or session state), `error_event` (error or failure record). |
| payload_hash | SHA-256 | Yes | Hash of the raw evidence payload. The payload itself is stored alongside the record; the hash enables integrity verification without requiring the verifier to hold the full payload. |
| external_ref | string | No | Reference to an external system for cross-validation (e.g., commit SHA, API response ID, transaction ID, external ticket number). When present, external verifiers can independently confirm the evidence by querying the referenced system. |
| timestamp | ISO 8601 | Yes | When the evidence was recorded. Millisecond precision REQUIRED (e.g., `2026-02-27T14:30:00.000Z`). |
| agent_id | string | Yes | Identity of the submitting agent (§2 identity handle). |
| trace_hash | SHA-256 | No | Post-execution hash of the actual execution trace (§6.2). Present when the evidence record is associated with task execution. Links the evidence record to the three-level alignment model (§6.11.1): L1 (`task_hash`) → L2 (`plan_hash`) → L3 (`trace_hash`). When `trace_hash` is present and differs from the committed `plan_hash`, the `divergence_log` field SHOULD also be present to explain the deviation. |
| divergence_log | array | No | Array of structured divergence entries explaining why the agent's execution trace diverged from its committed plan. Present when `trace_hash` differs from the committed plan hash (L2 → L3 mismatch). Each entry annotates a specific deviation with a machine-readable `reason` and structured context. See §8.10.3 for entry schema and §8.10.4 for the reason enum. |

**Semantics:**

- Agents MUST append EVIDENCE_RECORDs before sending TASK_COMPLETE (§6.6). A TASK_COMPLETE without a corresponding EVIDENCE_RECORD in the evidence layer is unanchored — external verifiers have no ground truth to verify against.
- When `trace_hash` is present in an EVIDENCE_RECORD and differs from the committed `plan_hash`, the agent SHOULD populate `divergence_log` with at least one entry. An EVIDENCE_RECORD with a divergent `trace_hash` but no `divergence_log` is valid but opaque — external verifiers can detect that divergence occurred but cannot classify the cause without inspecting the raw payload.
- The `divergence_log` in EVIDENCE_RECORD is distinct from `amend_hash` (§6.11.4): `divergence_log` annotates **agent-initiated deviations** (the agent changed its own execution path), while `amend_hash` authorizes **requester-initiated spec changes** (the delegating agent changed the task after plan commitment). Both produce the same observable signal (`plan_hash` ≠ `trace_hash`), but through different authorization chains. Conflating them makes blame attribution ambiguous: an agent that correctly adapted to an API failure is indistinguishable from one that quietly revised its goals. The `divergence_log` resolves this by providing structured cause annotation for agent-initiated deviations specifically.
- Append-only semantics are enforced by the protocol: no message type permits modification or deletion of an existing EVIDENCE_RECORD. Implementations MUST reject any operation that would alter or remove a previously appended record.
- External verifiers (§4.7, §8.7) access the evidence layer directly — not agent memory summaries. This is the architectural distinction that breaks the recursive self-attestation loop: the verifier's input is the evidence layer, not the agent's interpretation of the evidence layer.
- Evidence layer records MUST survive session teardown. When a session transitions to CLOSED (§4.2), the evidence records for that session are retained. The evidence layer's lifecycle is independent of the session lifecycle — evidence persists after the session that produced it has ended.

**Example EVIDENCE_RECORD (without divergence):**

```yaml
evidence_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
session_id: "session-42"
evidence_type: task_output
payload_hash: "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
external_ref: "github:commit:abc123def456"
timestamp: "2026-02-27T14:30:00.123Z"
agent_id: "agent-alpha"
trace_hash: "sha256:a1b2c3d4e5f6..."
```

**Example EVIDENCE_RECORD (with divergence_log):**

```yaml
evidence_id: "b2c3d4e5-f6a7-8901-bcde-f12345678901"
session_id: "session-42"
evidence_type: task_output
payload_hash: "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
external_ref: "github:commit:def456abc789"
timestamp: "2026-02-27T14:45:00.456Z"
agent_id: "agent-alpha"
trace_hash: "sha256:d4e5f6a7b8c9..."  # differs from committed plan_hash
divergence_log:
  - reason: infrastructure_noise
    description: "Rate limit hit on external API during step 3; retried with exponential backoff, adding 12s latency."
    deviation_timestamp: "2026-02-27T14:38:00.200Z"
    affected_step_id: "step-3-api-call"
    severity: WARN
  - reason: external_constraint
    description: "Upstream data source returned schema v2 instead of expected v1; adapted parsing logic."
    deviation_timestamp: "2026-02-27T14:41:00.750Z"
    affected_step_id: "step-5-parse-response"
    severity: INFO
```

#### 8.10.2 Evidence Layer Access Model

Compliant implementations MUST expose the evidence layer to external verifiers with read access and no write access. The verifier can query, compare, and audit evidence records but cannot append, modify, or delete them. Only the submitting agent appends records; only the protocol's append-only constraint governs what is written.

This access model ensures that the evidence layer serves as an independent audit surface:

- **External verifiers** read evidence records to validate agent claims. Without direct evidence layer access, external verification degrades to verifying agent self-reports — the recursive self-attestation problem this architecture is designed to solve.
- **Agents** append evidence records during task execution. Agents MAY read their own evidence records (e.g., for SESSION_RESUME state reconstruction) but MUST NOT modify or delete them.
- **Coordinators** read evidence records for SEMANTIC_CHALLENGE verification (§8.9.2) and SESSION_RESUME validation (§4.8). The coordinator's `state_hash` comparison gains ground truth anchoring when the state hash references evidence layer records rather than compactable memory.

#### 8.10.3 Divergence Log Entry Schema

The `divergence_log` field in EVIDENCE_RECORD is an array of divergence entry objects. Each entry annotates a specific point where the agent's execution deviated from its committed plan, with a machine-readable `reason` for recovery routing and structured context for post-hoc audit.

**Divergence log entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reason | enum | Yes | Machine-readable classification of the divergence cause. See §8.10.4 for the enum values and applicability criteria. |
| description | string | Yes | Human-readable explanation of what diverged and why. Provides context that `reason` alone cannot convey — specific error messages, environmental details, or adaptation strategy. |
| deviation_timestamp | ISO 8601 | Yes | When the deviation occurred relative to the execution timeline. Millisecond precision REQUIRED. This is the time the deviation was detected or occurred, not the time the entry was recorded. |
| affected_step_id | string | No | Identifier of the execution step where divergence occurred. Corresponds to a step in the agent's execution plan (L2). When present, enables cross-referencing with §7.8 `divergence_log` entries (which use `step_id`) and PLAN_COMMIT step declarations (§6.11.5). |
| severity | enum | Yes | Impact level: `INFO`, `WARN`, or `ERROR`. Semantics match §7.8.4 severity levels. `INFO` = benign adaptation, outcome equivalent. `WARN` = result quality may be affected. `ERROR` = material impact on result. |
| entry_id | string | No | Stable identifier for this divergence log entry. When present, enables cross-referencing between entries — specifically, `verifier_restored` entries use `restored_gap_ref` (§8.10.6) to link back to the original `verification_unreachable` entry by `entry_id`. Agents that use the verification failure taxonomy (§8.10.5) SHOULD populate `entry_id` on all entries. Format is implementation-defined but MUST be unique within the `divergence_log` array. |
| restored_gap_ref | string | No | Present only on `verifier_restored` entries (§8.10.6). References the `entry_id` of the original `verification_unreachable` entry that this restoration resolves. REQUIRED when `reason` is `verifier_restored`; MUST NOT be present for other reason values. |

**Semantics:**

- The `reason` field is REQUIRED (unlike §7.8 `deviation_type`, which is optional). The EVIDENCE_RECORD `divergence_log` serves the evidence layer — the append-only ground truth for external verification (§8.10). Unclassified divergence entries in the evidence layer defeat the purpose of structured cause annotation, because verifiers cannot route recovery without a machine-readable cause.
- Agents MUST NOT use the `divergence_log` in EVIDENCE_RECORD to annotate requester-initiated spec changes. Those are authorized deviations tracked by `amend_hash` (§6.11.4) and recorded in `amendments_log` (§6.11.6). The `divergence_log` is exclusively for agent-initiated deviations — situations where the agent changed its own execution path without prior authorization from the delegating agent.
- A single execution may produce entries with different `reason` values. For example, an agent might encounter `infrastructure_noise` (API rate limit) and then `external_constraint` (dependency schema change) in the same task. Each deviation gets its own entry with its own `deviation_timestamp`.

#### 8.10.4 Divergence Reason Enum

The `reason` field MUST use one of the following values. Each value identifies a specific cause of agent-initiated deviation with defined applicability criteria and recovery routing implications.

| reason | Description | When it applies | Recovery routing implication |
|--------|-------------|-----------------|----------------------------|
| `infrastructure_noise` | Transient infrastructure issue — rate limit, API failure, network timeout, or other operational noise expected in production environments. | The deviation was caused by an infrastructure-level issue that is likely transient and does not indicate a problem with the agent's logic or the task specification. | Retry is appropriate. The same agent with the same plan is likely to succeed on re-execution. No plan revision needed. |
| `planning_failure` | Wrong initial decomposition — the agent's committed plan (L2) was inadequate for the task, and the agent discovered this during execution. | The agent's plan was flawed: incorrect assumptions about task structure, underestimated complexity, missed dependencies between steps, or infeasible ordering. The deviation is attributable to the agent's planning, not external factors. | Re-planning is required before retry. The same plan will produce the same failure. Consider reassignment to a different agent if the planning failure reflects a capability gap. |
| `external_constraint` | A dependency or environmental condition became unavailable or changed mid-execution, forcing the agent to adapt its execution path. | An external system, upstream agent, data source, or environmental condition that the plan correctly assumed would be available was not available at execution time. The agent correctly adapted; the deviation is attributable to the environment, not the agent. | Assess whether the constraint is transient or permanent. Transient: retry may succeed. Permanent: re-plan with updated constraints. The agent's adaptation was correct — no blame attribution. |
| `spec_drift` | The task specification changed after plan commitment, and the agent adapted without a formal PLAN_AMEND cycle. | The agent detected that the task's effective requirements shifted after `plan_hash` was committed — but the change came through an informal channel (e.g., context update, environmental signal) rather than through a PLAN_AMEND (§6.11.4) with `amend_hash`. This is the **unauthorized** counterpart to `amend_hash`: same signal (spec changed), different authorization chain (no `amend_hash` to verify). | Investigate why the spec change did not go through the PLAN_AMEND flow. If the change was legitimate but informal, consider whether the session's amendment workflow needs tightening. If the agent unilaterally reinterpreted the task, escalate for review. |
| `verification_unreachable` | The verification system was unreachable — network partition, verifier crash, or service outage. The agent's evidence was never evaluated. See §8.10.5. | The agent attempted to submit evidence for verification but the verifier endpoint was not reachable. Infrastructure event, not agent failure. | Retry with backoff. Continue execution with flagged-unverified status. MUST NOT be treated as a protocol rejection. |
| `verification_timeout` | The verification request was sent but no response was received within the configured timeout window. Ambiguous cause. See §8.10.5. | The verification endpoint accepted the connection but did not return a result before timeout. Cause is ambiguous: verifier overload, semantic complexity, or partial failure. | Retry with backoff. If persistent, escalate to `verification_unreachable`. MUST NOT escalate to `verification_reject`. |
| `verification_reject` | The verification system evaluated the agent's evidence and explicitly rejected it. Behavioral failure confirmed. See §8.10.5. | The verifier completed evaluation and returned a negative result. The agent's execution, evidence, or attestation did not meet protocol requirements. | No recovery. Escalate to delegating agent. This is the only verification state that carries accountability weight. |
| `verifier_restored` | The verification system previously logged as unreachable is now reachable again. Append-only recovery entry. See §8.10.6. | A verifier that was the subject of a prior `verification_unreachable` entry has been confirmed reachable. | Process normally. The gap window between unreachable and restored is documented for forensic reconstruction. Requires `restored_gap_ref` field linking to the original unreachable entry. |
| `attestation_failure` | The per-hop delegation attestation (§6.4.1) failed verification at the receiving agent. The cryptographic binding of delegator identity to task and intent could not be confirmed. Distinct from `verification_reject` (verifier evaluated evidence and rejected it) and `verification_unreachable` (verifier was not reachable). See §6.4.1. | The receiving agent attempted to verify the `delegation_attestation` signature in TASK_ASSIGN and verification failed: invalid signature, unknown key, hash mismatch between attested and received values, or missing attestation. | No recovery at this hop. The receiving agent MUST reject the delegation (TASK_REJECT). The delegating agent SHOULD investigate the cause — key rotation, transmission corruption, or intermediary tampering. The `context` field in the divergence entry MUST include `attesting_node_id`, `asserted_task_hash`, `asserted_intent_hash`, and `failure_reason`. |
| `delta_absent` | Expected state change did not occur despite structural compliance — silent failure detected. The task's `state_delta_assertions` (§6.1) were checked post-execution and the expected observable delta was not found. See §8.19. | The task specification included `state_delta_assertions`, the agent reported structural completion (TASK_COMPLETE), verification of structural compliance passed, but one or more declared state-delta assertions were not confirmed. If the task declares `idempotent: true` (§6.1), `delta_absent` is expected behavior and MUST NOT be logged as a failure. | Investigate root cause. The agent executed structurally but produced no observable outcome. Possible causes: incorrect task parameters, target system state already satisfied, agent execution was a no-op despite reporting completion. Retry MAY succeed if the cause was transient. Re-planning is required if the task specification was incorrect. |
| `assignment_integrity_failure` | The verifier assignment record is missing, was not pre-committed before task execution, or the `selection_rationale` was annotated post-hoc rather than declared at assignment time. See §8.20. | A verifier's `selection_rationale` was not present in the signed assignment record at assignment time, or no pre-committed assignment record exists for a verification event. Post-hoc rationale annotation — where the rationale appears only after verification completes — is indistinguishable from a constructed justification and triggers this reason. Also applies when a verifier substitution occurred without a corresponding VERIFIER_REASSIGNMENT event (§8.20.4). | Audit investigation required. The assignment integrity violation invalidates the independence guarantee of the verification. Results from the affected verifier SHOULD be treated as unanchored — structurally present but not independently auditable. If the violation is due to missing pre-commitment, the assigning party's process requires remediation. If due to undocumented substitution, investigate whether the substitution was an operational necessity or an attempt to select a favorable verifier post-hoc. |
| `commitment_dropped` | An outstanding commitment (§6.12) was silently dropped across an instance boundary — the successor agent instance failed to honor, explicitly cancel, or escalate an inherited commitment. See §4.11, §5.12. | A recovering agent instance read the COMMITMENT_REGISTRY (§7.11) during INITIALIZING (§5.12) and transitioned to ACTIVE without fulfilling, notifying via DIVERGENCE_REPORT, or escalating one or more inherited commitments. Also applies when an external verifier or counterpart agent detects that a commitment's `due_by` deadline elapsed after an instance transition with no fulfillment signal, no COMMITMENT_CANCEL, and no DIVERGENCE_REPORT from the successor instance. | Correctness failure — the counterpart agent experienced a broken promise with no signal. Investigate whether the COMMITMENT_REGISTRY was corrupted, the reconciliation phase (§5.12) was skipped or incomplete, or the successor instance lacked the mechanism to read inherited commitments. The counterpart agent SHOULD treat the commitment as violated. The dropped commitment's `commitment_id`, `made_to`, and `due_by` MUST be included in the divergence entry's `context` field. |
| `not_considered` | The executing agent did not evaluate divergence for this task. Sentinel value — structurally different from a completed task with zero deviations. Exposes a gap in the agent's observability, not a clean execution path. | The agent completed execution without running its divergence evaluation logic — whether through omission, optimization, or spec misunderstanding. The agent is aware it did not evaluate divergence and is recording that fact explicitly. | No automatic recovery. The absence of divergence evaluation means the execution path is unaudited — it is unknown whether deviations occurred. Operators SHOULD investigate why the agent skipped divergence evaluation. A pattern of `not_considered` entries across tasks indicates a systematic observability gap, not a series of clean executions. |

**Completion requirement for `reason`:** When an executing agent completes a task (TASK_COMPLETE), the EVIDENCE_RECORD `divergence_log` MUST contain at least one entry with a valid `reason`. If the agent executed without any deviations from the committed plan, the `divergence_log` may be empty — the absence of entries combined with `plan_hash` = `trace_hash` is the clean-execution signal. However, if the agent did not evaluate whether divergence occurred, it MUST append a `divergence_log` entry with `reason: not_considered`. An absent `divergence_log` entry and a `not_considered` entry are not equivalent: an absent entry is ambiguous (possible bug, network drop, or omission), while `not_considered` is an auditable deliberate signal that divergence evaluation was skipped.

**Relationship to §7.8 `deviation_type`:** The §7.8 `deviation_type` enum (`RESOURCE_UNAVAILABLE`, `CONTEXT_SHIFT`, `CAPABILITY_MISMATCH`, `OPTIMIZATION`, `TIMEOUT`, `EXTERNAL_CONSTRAINT`, `OTHER`) and the §8.10.4 `reason` enum serve different classification purposes. §7.8 categorizes _what kind_ of divergence occurred at the step level (inline in TASK_COMPLETE/TASK_FAIL). §8.10.4 categorizes _why_ the divergence occurred at the evidence level, with recovery routing as the primary design goal. The two may co-occur: a `RESOURCE_UNAVAILABLE` deviation (§7.8) might be classified as `infrastructure_noise` (§8.10.4) if it was transient, or as `external_constraint` if the resource is permanently unavailable.

**Extension mechanism:** Implementations MAY extend this enum with deployment-specific values prefixed by `x-` (e.g., `x-model-context-overflow`, `x-quota-exceeded`). Standard reason values MUST NOT be prefixed. Receiving agents that encounter an unrecognized `reason` MUST treat it as opaque — log it, surface it to operators, but do not treat it as a protocol error.

> Addresses [issue #80](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/80): structured `divergence_log` in EVIDENCE_RECORD for cause annotation when `plan_hash` ≠ `trace_hash`, enabling recovery routing that distinguishes infrastructure noise from planning failures, external constraints, and unauthorized spec drift. Originated from community discussion on [Moltbook three-hash accountability thread](https://www.moltbook.com/post/3d769fda).
>
> Addresses [issue #120](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/120): `not_considered` sentinel for the `reason` enum, disambiguating the case where an agent did not evaluate divergence (auditable deliberate signal) from an absent `divergence_log` entry (ambiguous cause — bug, network drop, or omission). Closes #120.

#### 8.10.5 Verification Failure Taxonomy

The current `divergence_log` reason enum (§8.10.4) classifies why an agent's execution diverged from its committed plan. A separate classification is needed for failures that occur during **verification itself** — when the verification system fails to produce a result, times out, or explicitly rejects the agent's evidence. These are distinct failure modes with distinct trust implications, and conflating them corrupts audit trails: an infrastructure outage that prevents verification looks identical to a protocol rejection where verification completed and the agent's behavior was found non-compliant.

The verification failure taxonomy uses three states. Each state carries defined recovery semantics and trust implications for forensic reconstruction.

**Verification failure states:**

| State | Classification | Description | Trust implication |
|-------|---------------|-------------|-------------------|
| `VERIFIER_UNREACHABLE` | Infrastructure event | The verification system itself failed — network partition, verifier crash, service outage, or DNS resolution failure. The agent's evidence was never evaluated. | **No trust signal.** The verification system failed, not the agent. MUST NOT be logged as a protocol failure. Logging infrastructure outages as agent misconduct produces false positives that corrupt the audit trail — an agent operating correctly during a verifier outage would appear non-compliant in post-hoc reconstruction. |
| `VERIFICATION_TIMEOUT` | Partial trust failure | The verification request was sent but no response was received within the configured timeout window. Ambiguous cause: semantic complexity of the evidence under review, network congestion, verifier overload, or partial verifier failure. | **Ambiguous trust signal.** The verification may have been in progress but incomplete. Cannot be treated as either pass or fail. The ambiguity is itself forensically significant — it documents a window where verification status is unknown. |
| `VERIFICATION_REJECT` | Protocol event | The verification system evaluated the agent's evidence and explicitly rejected it. Behavioral failure confirmed — the agent's execution, evidence, or attestation did not meet protocol requirements. | **Negative trust signal.** This is the only state that carries accountability weight in audit reconstruction. No recovery path — escalate. The reject reason (from the verifier's response) SHOULD be captured in the `description` field. |

**Recovery semantics:**

- **`VERIFIER_UNREACHABLE`:** Retry is appropriate. The agent SHOULD attempt to reach the verifier using exponential backoff. If the verifier becomes reachable, process the verification result normally and append a `VERIFIER_RESTORED` entry (§8.10.6). If the verifier remains unreachable past the retry threshold, the agent SHOULD continue execution with a flagged-unverified status — the `divergence_log` entry documents that verification was attempted but infrastructure prevented it. The agent MUST NOT escalate `VERIFIER_UNREACHABLE` to `VERIFICATION_REJECT` — infrastructure failures are not protocol rejections.
- **`VERIFICATION_TIMEOUT`:** Retry with backoff. If the retry succeeds and the verifier returns a result, process the result normally (pass or reject). If the timeout persists past the retry threshold, escalate to `VERIFIER_UNREACHABLE` — persistent timeouts indicate the verifier is effectively unreachable. The agent MUST NOT escalate a persistent timeout directly to `VERIFICATION_REJECT`. The escalation path is: `VERIFICATION_TIMEOUT` → (persistent) → `VERIFIER_UNREACHABLE`, never `VERIFICATION_TIMEOUT` → `VERIFICATION_REJECT`.
- **`VERIFICATION_REJECT`:** No recovery. No retry. The verifier evaluated the evidence and found it non-compliant. The agent MUST escalate the rejection to its delegating agent. The reject entry in `divergence_log` is terminal for the verification attempt — subsequent verification attempts (e.g., after remediation) produce new `divergence_log` entries, not modifications to the existing reject entry.

**Divergence log entry fields for verification failures:**

Verification failure entries use the same schema as §8.10.3, with the following conventions:

| Field | Usage for verification failures |
|-------|--------------------------------|
| `reason` | One of: `verification_unreachable`, `verification_timeout`, `verification_reject`. These are additions to the §8.10.4 reason enum, not replacements. The §8.10.4 reasons (`infrastructure_noise`, `planning_failure`, `external_constraint`, `spec_drift`) classify agent-initiated execution deviations; the verification failure states classify verification-system outcomes. Both may appear in the same `divergence_log`. |
| `description` | For `verification_unreachable`: infrastructure details (e.g., "Verifier at endpoint X returned connection refused"). For `verification_timeout`: timeout configuration and duration (e.g., "Verification request to X timed out after 30s, 3 retries exhausted"). For `verification_reject`: the verifier's rejection reason. |
| `deviation_timestamp` | When the verification failure was detected. For `verification_timeout`, this is when the timeout elapsed, not when the request was sent. |
| `affected_step_id` | The execution step that was being verified when the failure occurred. |
| `severity` | `WARN` for `verification_unreachable` and `verification_timeout` (verification incomplete, not failed). `ERROR` for `verification_reject` (verification completed with negative result). |

**Forensic reconstruction under partition:** The three-state distinction is specifically designed for post-hoc audit during or after network partitions — the scenario where the audit trail matters most. When reconstructing what happened during a partition:

- `VERIFIER_UNREACHABLE` entries with timestamps define the window during which no verification was possible. Agent behavior during this window is unverified but not suspect.
- `VERIFICATION_TIMEOUT` entries identify ambiguous periods where verification may have been partially in progress. These require manual review or re-verification once the verifier is available.
- `VERIFICATION_REJECT` entries are definitive — the verifier was reachable, evaluated the evidence, and found it non-compliant. These carry full accountability weight regardless of surrounding infrastructure conditions.

Distinct states with distinct timestamps enable accurate reconstruction of which failure mode occurred at each point in the execution timeline.

**Fault tolerance model mapping:**

The three verification failure states map to classical distributed systems fault categories from the Byzantine generals problem. This mapping provides formal tolerance thresholds for multi-verifier deployments.

| State | Fault category | Tolerance threshold | Rationale |
|-------|---------------|--------------------|-----------|
| `VERIFIER_UNREACHABLE` | Crash failure | 2f+1 verifiers tolerate f crash failures | Infrastructure failure — the verifier stopped responding. Crash failures are the simplest fault class: the faulty node does nothing wrong, it simply stops. No trust implication for the agent under verification. |
| `VERIFICATION_TIMEOUT` | Ambiguous message (omission failure) | Between crash and Byzantine — requires contextual assessment | The verification request was sent but the outcome is unknown. The verifier may have crashed, may be overloaded, or may be partially failed. Maps to the omission failure class — the message may or may not have been processed. Persistent timeouts across the retry threshold MUST be treated as crash failure (`VERIFIER_UNREACHABLE`), never as Byzantine failure (`VERIFICATION_REJECT`). |
| `VERIFICATION_REJECT` | Byzantine failure | 3f+1 verifiers required to tolerate f Byzantine failures | The verifier evaluated evidence and found non-compliance. This is the only state that requires the stronger Byzantine tolerance threshold — a reject from a single verifier in a multi-verifier deployment requires corroboration from 3f+1 total verifiers to distinguish legitimate rejection from a compromised verifier producing false rejections. |

This mapping informs multi-verifier deployment configuration: a deployment with n verifiers tolerates ⌊(n-1)/2⌋ crash failures (`VERIFIER_UNREACHABLE`) but only ⌊(n-1)/3⌋ Byzantine failures (`VERIFICATION_REJECT`). The distinction determines the minimum verifier pool size required for a given fault tolerance target. The `VERIFICATION_TIMEOUT` → `VERIFIER_UNREACHABLE` escalation path (never `VERIFICATION_TIMEOUT` → `VERIFICATION_REJECT`) is the direct consequence of this mapping: ambiguous messages are crash-adjacent, not Byzantine-adjacent.

**Example divergence_log with verification failure entries:**

```yaml
divergence_log:
  - reason: verification_unreachable
    description: "Verifier at endpoint https://verifier.example.com returned connection refused. 3 retries with exponential backoff exhausted (1s, 2s, 4s)."
    deviation_timestamp: "2026-02-27T14:38:00.200Z"
    affected_step_id: "step-3-evidence-submit"
    severity: WARN
    entry_id: "dl-001"
  - reason: verification_timeout
    description: "Verification request to https://verifier.example.com accepted but no response received within 30s timeout. Possible verifier overload."
    deviation_timestamp: "2026-02-27T14:42:15.800Z"
    affected_step_id: "step-5-attestation-check"
    severity: WARN
    entry_id: "dl-002"
  - reason: verification_reject
    description: "Verifier rejected evidence for step-7: trace_hash does not match declared plan_hash. Verifier response: 'L2-L3 mismatch detected, plan step step-7-transform declared output schema v1 but trace shows schema v2 output.'"
    deviation_timestamp: "2026-02-27T14:45:30.100Z"
    affected_step_id: "step-7-transform"
    severity: ERROR
    entry_id: "dl-003"
```

#### 8.10.6 Append-Only Divergence Log Semantics

The `divergence_log` in EVIDENCE_RECORD MUST follow append-only semantics. Existing entries MUST NOT be modified, replaced, or deleted — regardless of subsequent events that change the state described in an earlier entry. This constraint is the same class as the append-only rule for EVIDENCE_RECORD itself (§8.10.1): mutable entries push trust back onto the log maintainer, because a mutated log is indistinguishable from a fabricated log. Append-only semantics are what make the `divergence_log` independently verifiable.

**VERIFIER_RESTORED entry type:**

When a verifier that was previously logged as `VERIFIER_UNREACHABLE` (§8.10.5) becomes reachable again, the agent MUST NOT mutate the original `VERIFIER_UNREACHABLE` entry. Instead, the agent MUST append a new entry with reason `verifier_restored`. The `VERIFIER_RESTORED` entry documents that the verifier recovered and links back to the original gap entry.

**VERIFIER_RESTORED entry fields:**

| Field | Usage |
|-------|-------|
| `reason` | `verifier_restored` |
| `description` | Details of verifier recovery — endpoint, method of confirmation, and any verification results obtained after restoration. |
| `deviation_timestamp` | When the verifier was confirmed reachable again. |
| `affected_step_id` | The execution step associated with the original `VERIFIER_UNREACHABLE` entry (for cross-reference). |
| `severity` | `INFO` (the restoration is a positive event — the verification gap is now bounded). |
| `restored_gap_ref` | **Additional field** (string, REQUIRED for `verifier_restored` entries): The `entry_id` of the original `VERIFIER_UNREACHABLE` entry that this restoration resolves. Links the two entries to define the verification gap window. |

**Verification gap window:** The time between a `VERIFIER_UNREACHABLE` entry and its corresponding `VERIFIER_RESTORED` entry defines the **verification gap window** — the period during which verification was unavailable. This window is itself forensically significant:

- Agent behavior during the gap window is unverified. It is not suspect (no `VERIFICATION_REJECT` occurred), but it has not been confirmed by an external verifier.
- The gap window's duration and timing relative to execution steps enables auditors to assess the scope of unverified execution.
- If no `VERIFIER_RESTORED` entry exists for a `VERIFIER_UNREACHABLE` entry, the gap window extends to the end of the session — all subsequent execution in that session was unverified by that verifier.

**Example append-only sequence with VERIFIER_RESTORED:**

```yaml
divergence_log:
  # Original unreachable entry — never mutated
  - reason: verification_unreachable
    description: "Verifier at endpoint https://verifier.example.com returned connection refused."
    deviation_timestamp: "2026-02-27T14:38:00.200Z"
    affected_step_id: "step-3-evidence-submit"
    severity: WARN
    entry_id: "dl-001"
  # Execution continues with flagged-unverified status
  - reason: infrastructure_noise
    description: "Continued execution of step-4 without verification due to verifier outage (ref: dl-001)."
    deviation_timestamp: "2026-02-27T14:39:00.500Z"
    affected_step_id: "step-4-process"
    severity: WARN
    entry_id: "dl-002"
  # Verifier comes back online — append restoration, do not mutate dl-001
  - reason: verifier_restored
    description: "Verifier at endpoint https://verifier.example.com confirmed reachable. Retroactive verification of steps 3-4 submitted."
    deviation_timestamp: "2026-02-27T14:50:00.000Z"
    affected_step_id: "step-3-evidence-submit"
    severity: INFO
    entry_id: "dl-003"
    restored_gap_ref: "dl-001"
```

In this example, the verification gap window spans from `2026-02-27T14:38:00.200Z` (dl-001) to `2026-02-27T14:50:00.000Z` (dl-003). Steps 3 and 4 executed during this window without external verification. The `VERIFIER_RESTORED` entry bounds the gap and enables retroactive verification of the unverified steps.

**Immutability rule:** Agents MUST NOT:

- Modify a `VERIFIER_UNREACHABLE` entry when the verifier comes back online. The original entry documents the infrastructure failure at the time it occurred.
- Modify a `VERIFICATION_TIMEOUT` entry when a retry succeeds. The original timeout is a historical fact. The successful retry is a new entry.
- Modify a `VERIFICATION_REJECT` entry under any circumstances. Reject entries are definitive audit evidence.
- Delete any `divergence_log` entry for any reason. The append-only constraint is unconditional.

Implementations that detect mutation of `divergence_log` entries SHOULD treat the mutation as a tamper signal and escalate to the session's external verifier (§8.7, §8.10.2). A mutated `divergence_log` is no longer independently verifiable and SHOULD be treated with the same suspicion as a missing or corrupted evidence record.

> Addresses [issue #139](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/139) and [issue #142](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/142): three-state verification failure taxonomy (`VERIFIER_UNREACHABLE`, `VERIFICATION_TIMEOUT`, `VERIFICATION_REJECT`) with distinct recovery semantics and trust implications for each state, plus append-only `divergence_log` semantics with `VERIFIER_RESTORED` entry type that preserves the verification gap window as forensically significant evidence. Prevents audit trail corruption where infrastructure outages are misclassified as protocol rejections.

### 8.11 Structured Divergence Reporting

The `divergence_log` (§7.8) records inline plan-execution deviations during task execution, with `deviation_type` as an optional categorization. For protocol-level divergence events — detected by verification mechanisms such as state hash comparison (§8.2), semantic liveness checks (§8.9), or attestation verification — the protocol requires a structured report that verifiers can classify programmatically. DIVERGENCE_REPORT provides that structure. Its `reason_code` field is required, enabling automated triage, cross-session pattern detection, and verifier-side classification without relying on free-text parsing.

#### 8.11.1 DIVERGENCE_REPORT Message

DIVERGENCE_REPORT is sent when a divergence is detected between expected and actual agent state, execution behavior, or protocol compliance. It is distinct from `divergence_log` entries — both the inline §7.8 annotations within TASK_COMPLETE/TASK_FAIL and the evidence-level §8.10.3 cause annotations within EVIDENCE_RECORD. DIVERGENCE_REPORT is a standalone protocol message emitted by the detecting party (coordinator, verifier, or peer agent).

**DIVERGENCE_REPORT fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session in which the divergence was detected. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| report_id | UUID v4 | Yes | Unique identifier for this divergence report. |
| source_agent_id | string | Yes | §2 identity handle of the agent that detected the divergence. |
| target_agent_id | string | Yes | §2 identity handle of the agent whose behavior diverged. |
| reason_code | enum | Yes | Structured classification of the divergence cause. See §8.11.2 for the taxonomy. Verifiers MUST classify divergences by `reason_code` — free-text `description` alone is insufficient for automated triage. |
| description | string | Yes | Human-readable explanation of the divergence. Provides context that `reason_code` alone cannot convey — specific error messages, environmental details, or reproduction steps. |
| severity | enum | Yes | Impact level: `INFO`, `WARN`, or `ERROR`. Semantics match §7.8.4 severity levels. |
| evidence_ref | string | No | Reference to an EVIDENCE_RECORD (§8.10) that anchors this divergence report. When present, external verifiers can independently validate the divergence claim against the evidence layer. |
| related_task_id | UUID v4 | No | Task identifier if the divergence is associated with a specific task. |
| expected_value | string | No | The value the detecting party expected (e.g., expected state hash). For diagnostic purposes. |
| actual_value | string | No | The value actually observed (e.g., actual state hash). For diagnostic purposes. |
| timestamp | ISO 8601 | Yes | When the divergence was detected. |

**Example DIVERGENCE_REPORT:**

```yaml
session_id: "session-42"
report_id: "f7a8b9c0-d1e2-3456-7890-abcdef123456"
source_agent_id: "coordinator-prime"
target_agent_id: "agent-alpha"
reason_code: attestation_mismatch
description: "Agent's SEMANTIC_RESPONSE state hash did not match expected value computed from task checkpoint CP-17."
severity: ERROR
evidence_ref: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
related_task_id: "task-789"
expected_value: "sha256:abc123..."
actual_value: "sha256:def456..."
timestamp: "2026-02-27T15:45:00.123Z"
```

#### 8.11.2 Reason Code Taxonomy

The `reason_code` field MUST use one of the following values. Each code identifies a specific divergence class with defined applicability criteria.

| reason_code | Description | When it applies |
|-------------|-------------|-----------------|
| `constraint_violation` | The agent violated an explicit constraint from the task specification, session parameters, or protocol rules. | A declared constraint (resource limit, permission boundary, schema requirement, protocol invariant) was breached during execution. Applies when the constraint was known to the agent before execution. |
| `resource_limit` | A resource ceiling was reached or exceeded, causing divergence from expected behavior. | Compute, memory, storage, API quota, rate limit, or time budget exhaustion forced the agent off its expected execution path. Distinct from `constraint_violation` — the limit is environmental, not declarative. |
| `dependency_failure` | A required dependency — external service, upstream agent, data source, or infrastructure component — failed or became unavailable. | The agent's execution depended on an external system that returned an error, timed out, or was unreachable. The divergence is attributable to the dependency, not the agent's logic. |
| `context_shift` | The execution context changed after the agent began operating, invalidating assumptions from the original plan or task specification. | New information, updated requirements, environmental state changes, or concurrent modifications by other agents altered the conditions under which the agent was operating. |
| `attestation_mismatch` | A cryptographic or semantic attestation check failed — state hash, trace hash, plan hash, or SEMANTIC_CHALLENGE response did not match the expected value. | Applies to divergences detected by §8.2 state hash comparison, §8.9.2 semantic liveness verification, or §7.2 merkle tree comparison. The detecting party has cryptographic evidence of the mismatch. |
| `scope_exceeded` | The agent operated outside the boundaries of its delegated task, session permissions, or declared capabilities. | The agent performed actions, accessed resources, or produced outputs beyond what was authorized by the task specification (§6.1), capability manifest (§5), or delegation scope. |
| `chain_integrity_failure` | Delegation chain or capability grant chain hash traversal verification failed — a sub-delegation's `parent_grant_hash` does not match the SHA-256 of its claimed parent delegation (§6.9.3.2), or a sub-grant CAPABILITY_GRANT's `parent_grant_hash` does not match the SHA-256 of its claimed parent grant (§5.8.2), or the delegating/granting agent's signature over the fields (including `parent_grant_hash`) is invalid. | Applies to divergences detected by §6.9.3.2 delegation chain traversal verification or §5.8.2 CAPABILITY_GRANT chain verification. The detecting party has cryptographic evidence that the chain is broken — the sub-delegation or sub-grant either claims authority from a non-existent or tampered parent, or was not authorized by the issuing agent. Distinct from `attestation_mismatch` (which applies to execution-level hash mismatches, not delegation/grant authority). |
| `commitment_dropped` | An outstanding commitment (§6.12) was silently dropped across an instance boundary — the successor agent instance failed to honor or explicitly refuse an inherited commitment from the COMMITMENT_REGISTRY (§7.11). | Applies when a recovering agent instance transitions to ACTIVE (§5.12) without reconciling one or more inherited commitments, or when a counterpart agent or external verifier detects that a commitment's `due_by` deadline elapsed after an instance transition with no fulfillment, cancellation, or DIVERGENCE_REPORT signal. The `commitment_id`, `made_to`, and `due_by` of the dropped commitment MUST be included in the DIVERGENCE_REPORT. Distinct from `context_shift` — `context_shift` applies when the agent explicitly acknowledges and reports the commitment abandonment during reconciliation; `commitment_dropped` applies when the abandonment was silent (no signal sent to the counterpart). |
| `fidelity_failure` | The agent's context integrity is compromised — the agent is responsive but operating on a degraded model of prior session state. | Applies when any of the observable signals defined in §8.22.1 are detected: epoch drift, response inconsistency with prior session state, session anchor mismatch, or self-reported context compaction. Distinct from `context_shift` (external context changed) — `fidelity_failure` indicates the agent's internal context model has degraded while the external context remains stable. The agent may appear fully functional but its responses are unreliable because they are based on incomplete or corrupted session state. |
| `not_considered` | The agent did not evaluate divergence for this task. Sentinel value indicating the divergence evaluation path was never executed — distinct from a clean execution with zero divergences. | The detecting party (or the agent itself via self-report) determined that the target agent completed execution without running its divergence evaluation logic. Applies whether the omission was due to optimization, implementation gap, or spec misunderstanding. The agent is recording or being reported for the absence of divergence evaluation, not for a specific divergence. |

**Completion requirement for `reason_code`:** When an executing agent completes a task (TASK_COMPLETE), the `reason_code` MUST be present on any DIVERGENCE_REPORT filed for that task. `not_considered` is valid when no divergence evaluation occurred — it is the agent's (or verifier's) explicit signal that the divergence check was skipped. An absent DIVERGENCE_REPORT and a DIVERGENCE_REPORT with `reason_code: not_considered` are not equivalent: an absent report could indicate a protocol violation, infrastructure failure, or clean execution (ambiguous cause); a `not_considered` report is an intentional, auditable signal that divergence evaluation did not occur.

**Extension mechanism:** Implementations MAY extend this taxonomy with deployment-specific reason codes prefixed by `x-` (e.g., `x-model-degradation`, `x-quota-billing-exceeded`). Standard reason codes MUST NOT be prefixed. Receiving agents that encounter an unrecognized `reason_code` MUST treat it as opaque — log it, surface it to operators, but do not treat it as a protocol error.

#### 8.11.3 Relationship to §7.8, §8.10, and Other Sections

DIVERGENCE_REPORT, the §7.8 `divergence_log`, and the §8.10 EVIDENCE_RECORD `divergence_log` serve different purposes at different protocol layers:

| Mechanism | Scope | Emitted by | Carried in | Classification field | Required? |
|-----------|-------|------------|------------|---------------------|-----------|
| `divergence_log` (§7.8) | Plan-execution divergence — the executing agent's actual execution differed from its committed plan (L2 vs. L3). | Executing agent (self-report). | Inline in TASK_COMPLETE / TASK_FAIL (§6.6). | `deviation_type` | Optional (SHOULD). |
| `divergence_log` (§8.10.3) | Evidence-level cause annotation — structured explanation of _why_ `trace_hash` diverged from `plan_hash`, for recovery routing. | Executing agent (self-report, appended to evidence layer). | EVIDENCE_RECORD (§8.10.1). | `reason` | Required. |
| DIVERGENCE_REPORT (§8.11) | Protocol-level divergence — verification mechanisms detected a mismatch between expected and actual agent state or behavior. | Coordinator, external verifier, or peer agent (external detection). | Standalone protocol message. | `reason_code` | Required. |

The three mechanisms are complementary. A single divergence event may produce all three: the executing agent appends a `divergence_log` entry in TASK_COMPLETE (§7.8) explaining its plan deviation at the step level, appends an EVIDENCE_RECORD with a `divergence_log` entry (§8.10.3) classifying the cause for recovery routing, and the coordinator emits a DIVERGENCE_REPORT (§8.11) when verification detects the resulting state mismatch. The `related_task_id` field in DIVERGENCE_REPORT, the `step_id` field in §7.8 `divergence_log` entries, and the `affected_step_id` field in §8.10.3 `divergence_log` entries enable cross-referencing between all three records.

**Distinction from `amend_hash`:** The §8.10.3 `divergence_log` and `amend_hash` (§6.11.4) both address the same observable signal — `plan_hash` ≠ `trace_hash` — but through different authorization chains. `amend_hash` tracks **requester-initiated spec changes** (the delegating agent authorized the deviation via PLAN_AMEND). The §8.10.3 `divergence_log` tracks **agent-initiated deviations** (the agent changed its execution path without prior authorization). The `amendments_log` (§6.11.6) and `divergence_log` (§8.10.3) together provide a complete audit trail: every deviation in the evidence layer is either authorized (has a corresponding `amend_hash`) or annotated (has a corresponding `divergence_log` entry with a `reason`).

DIVERGENCE_REPORT integrates with the evidence layer (§8.10) via the optional `evidence_ref` field. When a DIVERGENCE_REPORT is backed by an EVIDENCE_RECORD, external verifiers can independently validate the divergence claim — breaking the recursive self-attestation loop that §8.10 was designed to address. Agents SHOULD append an EVIDENCE_RECORD before emitting a DIVERGENCE_REPORT for `ERROR`-severity divergences.

> Addresses [issue #57](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/57): structured `reason_code` taxonomy for divergence classification, replacing free-text-only divergence logging with machine-readable categorization that verifiers can classify programmatically.
>
> Addresses [issue #120](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/120): `not_considered` sentinel for the `reason_code` enum, disambiguating the case where an agent did not evaluate divergence (auditable deliberate signal) from an absent DIVERGENCE_REPORT (ambiguous cause). Closes #120.

### 8.12 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **CIC trigger architecture.** Context Integrity Challenges (§8.5) require a triggering mechanism that satisfies two constraints: probes must be unpredictable to the monitored agent, and the triggering signal must not depend on agent self-report (§4.7.5). A candidate architecture uses three escalation tiers: (1) irregular baseline intervals — CIC probes fire on an externally randomized schedule, making the probe pattern harder to game than fixed intervals; (2) cheap smell test escalation — externally observable anomalies (unexpected latency spikes, unusual token perplexity) trigger additional CIC probes outside the baseline schedule, without requiring agent self-report; (3) expensive CIC verification — a full state consistency check against the authoritative log, triggered by smell test failure, leading to SESSION_RESUME on verification failure. This architecture satisfies the adversarial scheduling constraint (irregular + anomaly-triggered) while avoiding the circular dependency of anomaly-only triggering (an agent that passes smell tests but fails CIC probes is caught by the baseline schedule). Cheap trigger candidates from community discussion: unexpected latency spikes and unusual token perplexity — both observable without agent self-report, preserving the core constraint that detection primitives must be external to the monitored agent (§4.7).

> Community discussion: [Moltbook zombie states thread](https://www.moltbook.com/post/b7629c46-32b0-49f0-9f07-0dc5844b2d49). See also [issue #4](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/4), [issue #47](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/47), §4 Session Lifecycle (§4.7 External Monitoring Architecture). Evidence layer architecture (§8.10) surfaced from [Moltbook thread 3d769fda](https://www.moltbook.com/post/3d769fda): @ultrathink (append-only audit log), @ummon_core (empirical failure case), @cass_agentsharp (epistemic framing), @Yibao (thread originator).

### 8.13 Teardown-First Recovery Mandate

<!-- Implements #60 and #63: teardown + idempotent task replay as the default recovery protocol. -->

Production data from Nanook's 6-week NATS deployment and the zombie states thread consensus established that **teardown + idempotent task replay is more reliable than resume-from-stale**. Serialized session state drifts from reality often enough that the resume path becomes its own bug surface — re-establishment cost is cheap; bugs from stale resumed state are expensive. This section formalizes teardown-first recovery as the V1 default.

#### 8.13.1 Default Recovery Protocol

The default recovery protocol after a crash, disconnect, or any session interruption is **TEARDOWN + REINITIATE**, not RESUME. Specifically:

1. **Agents MUST NOT resume from serialized in-memory state after a crash or disconnect.** In-memory state at the time of failure is presumed stale. The failure itself is evidence that the process state may be corrupted, incomplete, or inconsistent with peer expectations. Attempting to resume from a snapshot of pre-crash memory reintroduces the very state that may have contributed to the failure.

2. **On recovery, the agent MUST:**
   - Read canonical state from **durable persistent storage** — not from in-memory snapshots, serialized heap dumps, or process checkpoint images. Durable persistent storage means storage that survives process restart and is independent of the agent's runtime memory (e.g., database, evidence layer §8.10, external state store).
   - **Reconcile known state** against the evidence layer (§8.10) and any TASK_CHECKPOINT artifacts (§6.6) in external storage. The agent reconstructs its understanding of session state from these external sources, not from its own pre-crash memory.
   - **Initiate a fresh SESSION_INIT** (§4.3). The recovered agent starts a new session with its counterparty. The new session inherits no state from the crashed session — capability exchange (§5.9), role negotiation (§5), and task delegation (§6) proceed from scratch.

3. **The teardown sequence for the crashed session:**
   - The crashed session transitions to CLOSED (not SUSPENDED or COMPACTED). There is no expectation of resuming the crashed session.
   - In-flight tasks from the crashed session that were not checkpointed are considered lost. The recovering agent replays them in the new session using idempotency keys (§7.10.1) or sequence number checkpointing (§7.10.2) to avoid duplicate side effects.
   - Peers that detect the crash (via heartbeat expiry §4.5 or SUSPECTED → EXPIRED §4.2.1) SHOULD issue TASK_CANCEL (§6.6) for in-flight subtasks delegated to the crashed agent, then accept the new SESSION_INIT when it arrives.

#### 8.13.2 Resume Path — NOT RECOMMENDED for V1

The SESSION_RESUME protocol (§4.8) remains defined in the spec for completeness but is **explicitly NOT RECOMMENDED as the recovery default for V1**. Resume is a **known failure mode**, not a supported recovery option.

**Why resume fails in practice:**

- **State serialization drift.** The serialized state format used by the resuming agent may not match the format expected by the peer — version mismatches between code and serialized state format cause subtle bugs that are harder to debug than clean restarts (§8.5 teardown-over-resume).
- **Semantic state corruption.** Even when the state hash matches (§4.8 `STATE_HASH_ACK(match)`), the semantic meaning of the state may have drifted. The hash confirms bit-level identity, not semantic correctness — an agent can resume with a state that is byte-identical to what the peer expects but semantically stale because the world changed during the crash window.
- **Resume bug surface.** The resume code path is exercised rarely (only on recovery) and therefore accumulates latent bugs. The teardown + reinitiate path is exercised on every session start and is therefore better tested by default.
- **Cost asymmetry.** A bad resume that silently corrupts downstream state is more expensive than re-execution, even when re-execution is costly. The decision criterion SHOULD be based on **variance exposure** (worst-case cost of a bad resume), not expected compute cost (average cost of re-execution).

**When SESSION_RESUME MAY still be used:** Implementations that have validated their state serialization format across versions, have end-to-end tests for the resume code path, and operate in environments where re-execution cost is prohibitively high (multi-hour tasks with no intermediate checkpoints) MAY use SESSION_RESUME. This is an explicit opt-in — the implementation MUST document why resume is safe in its specific deployment context. The burden of proof is on the implementation choosing resume, not on the spec.

#### 8.13.3 Idempotent Task Replay

Teardown-first recovery depends on **safe task replay** — the ability to re-execute tasks from the last confirmed checkpoint without producing duplicate or corrupted results. Task idempotency (§7.10) is the prerequisite constraint that enables this.

**Recovery replay sequence:**

1. The recovering agent reads the last confirmed checkpoint from durable persistent storage. For each in-flight task at the time of crash:
   - If a TASK_CHECKPOINT (§6.6) exists in external storage → replay from that checkpoint using the same `idempotency_key` (§7.10.1).
   - If no checkpoint exists → replay the task from the beginning using the same `idempotency_key`.
   - If the task is marked `idempotent: false` (§7.10.3) and no checkpoint exists → do NOT replay. Report the task as requiring manual intervention or reconciliation against external state.

2. **Sequence numbers** (§7.10.2) or **idempotency keys** (§7.10.1) are RECOMMENDED so the recovering agent can identify which work was confirmed before the crash and which was in-flight. Without these, the recovering agent cannot distinguish completed work from partial work, and replay may produce duplicates.

3. **In-flight work during the crash window may be lost.** This is an accepted tradeoff. The protocol does not guarantee zero-loss recovery — it guarantees that recovery does not silently corrupt state. Lost in-flight work is re-executable; silently corrupted state from a bad resume is not.

#### 8.13.4 Durable Persistent Storage Requirements

The recovering agent's state reconstruction depends on storage that survives process restart. The following requirements apply to any storage used for recovery state:

- **Durability:** Writes MUST be persisted to stable storage before being acknowledged. In-memory caches or write-behind buffers that may lose data on process crash are insufficient.
- **Independence:** The storage MUST be independent of the agent's runtime process. If the agent process dies, the storage remains accessible to the restarted process.
- **Evidence layer integration:** Recovery state SHOULD be anchored to the evidence layer (§8.10) where available. EVIDENCE_RECORDs provide append-only, externally verifiable records that cannot be retroactively altered by a recovering agent reconstructing its own history.
- **Checkpoint granularity:** Storage SHOULD support per-task checkpoint granularity — the recovering agent needs to identify the last confirmed state for each in-flight task independently, not just the global session state.

#### 8.13.5 Relationship to Existing Sections

- **§4.8 (SESSION_RESUME):** SESSION_RESUME remains defined but is NOT RECOMMENDED as the default recovery path (§8.13.2). Implementations that opt into resume use §4.8's state-hash negotiation. Implementations that follow the teardown-first default skip SESSION_RESUME entirely and proceed directly to SESSION_INIT.
- **§7.10 (Task Idempotency):** The prerequisite constraint that makes teardown-first recovery safe. Without idempotent tasks, replay after teardown risks duplicate side effects.
- **§8.2 (Detection Primitives):** The SESSION_RESUME handshake in §8.2 describes the resume path. Under teardown-first, the `mismatch → RESTART` branch (teardown and re-init) is the expected default outcome, not the fallback.
- **§8.4 (Coordination Patterns):** The teardown-by-default pattern described in §8.4 is formalized here as a normative requirement, not just a coordination preference.
- **§8.5 (Named Considerations):** The "Teardown over resume from production" consideration is the empirical motivation for this section's normative requirements.
- **§8.10 (Evidence Layer):** The evidence layer provides the durable, externally verifiable state that the recovering agent reads during state reconstruction — replacing the compactable in-memory state that teardown-first explicitly prohibits.
- **§4.9.2 (Pending Tasks Inventory):** When teardown occurs from SUSPENDED state, the `pending_tasks` field on SESSION_CLOSE provides the task inventory that a recovering agent or replacement session needs to determine what work remains. This complements teardown-first recovery by making the pre-suspension task landscape protocol-visible at the teardown boundary.

#### 8.13.6 Crossed Steps Survive Teardown-First Recovery

<!-- Implements #70: crossed PLAN_COMMIT steps survive teardown and MUST appear in completed_steps on reinitiation -->

Teardown-first recovery (§8.13.1) tears down the crashed session and initiates a fresh one. However, **crossed step boundaries from the pre-crash plan are not erased by teardown**. Step boundary crossing creates binding commitments (§6.11.5) that persist independent of session state.

**Recovery sequence for plans with step boundaries:**

1. The recovering agent reads the last confirmed `completed_steps` from durable persistent storage (§8.13.4). Sources include: the most recent TASK_CHECKPOINT with `completed_steps`, the evidence layer (§8.10), or the last TASK_PROGRESS that reported a `current_step_id` transition.

2. On reinitiation (fresh SESSION_INIT), the recovering agent or the delegating agent MUST carry forward the `completed_steps` from the pre-crash session. When re-delegating the task in the new session, the delegating agent includes the prior `completed_steps` in the task context (via `progress_checkpoint` in the task schema, §6.1).

3. The new PLAN_COMMIT in the reinitiated session MUST account for already-crossed steps:
   - Steps listed in `completed_steps` MUST NOT be re-executed. Their effects are committed (if `reversible: false`) or subject to explicit rollback decision (if `reversible: true`).
   - The new plan SHOULD contain only the remaining (not-yet-crossed) steps, plus any new steps needed to handle recovery-specific concerns (e.g., consistency verification of prior step outputs).
   - The `plan_hash` of the new PLAN_COMMIT will differ from the pre-crash plan because it covers a different step set. This is expected — the three-level alignment verification (§6.11.1) uses the new plan hash for the recovery session.

4. Steps with `reversible: false` that were crossed before the crash are **unconditionally committed**. The delegating agent MUST NOT request re-execution of these steps and MUST account for their effects in any subsequent planning. This is the key difference between a step boundary and a progress checkpoint: a checkpoint says "work was done up to here"; a crossed irreversible step boundary says "these effects are permanent regardless of what happens next."

**Relationship to idempotent task replay (§8.13.3):** Idempotent replay applies to tasks that need re-execution. Crossed irreversible steps are explicitly excluded from replay — they are committed, not re-executable. The recovering agent replays only the uncrossed portion of the plan. For crossed reversible steps, the delegating agent decides whether to accept the prior results or request re-execution in the new plan.

> Crossed step recovery semantics formalized from [issue #70](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/70). Step boundaries create durable commitments that outlive sessions — teardown destroys session state but cannot undo real-world effects of irreversible steps.

> Teardown-first recovery mandate formalized from Nanook's 6-week NATS deployment data and zombie states thread consensus. See [issue #60](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/60) and [issue #63](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/63). The core insight — re-establishment cost is cheap, bugs from stale resumed state are expensive — converged independently across multiple production deployments.

#### 8.13.7 Amendments Survive Teardown-First Recovery

<!-- Implements #66: amendments_log durability through teardown-first recovery -->

The `amendments_log` (§6.11.6) is a **durable artifact** — it MUST be preserved through session teardown. Teardown-first recovery (§8.13.1) tears down the crashed session and initiates a fresh one, but the amendment history from the pre-crash session is not erased by teardown. Amendments record authorized spec drift that occurred during plan execution; discarding this record on teardown would create an audit gap where the delegating agent cannot verify whether the delegatee's execution reflected the original plan or an authorized modification.

**Recovery sequence for sessions with amendments:**

1. The recovering agent reads the `amendments_log` from durable persistent storage (§8.13.4). The `amendments_log` is persisted alongside other durable session state (SESSION_STATE §4.11, `completed_steps` §8.13.6). Sources include: the delegatee's durable session state store, or the evidence layer (§8.10) if amendment entries were anchored as EVIDENCE_RECORDs.

2. On reinitiation (fresh SESSION_INIT), the delegatee MUST include the `amendments_log` from the pre-crash session when reporting results or status to the delegating agent. This confirms that any spec drift during the pre-crash session was authorized. The `amendments_log` is carried in the first terminal message (TASK_COMPLETE, TASK_FAIL, or SESSION_CLOSE) of the recovery session that references work from the pre-crash plan.

3. **Divergence detection:** The delegating agent MUST compare the received `amendments_log` against its own local record of issued PLAN_AMEND messages. If the delegating agent's local `amendments_log` diverges from the delegatee's — entries present in one but not the other, `amend_hash` values that do not match, `proposed_at` values (when present) that do not match the issued PLAN_AMEND timestamps, or `scope_delta` values that do not match — the session MUST be terminated. The delegating agent SHOULD also verify the temporal ordering invariant `proposed_at` ≤ `accepted_at` ≤ `timestamp` (T1 ≤ T2 ≤ T3) for each entry when all three timestamps are present (see §6.11.6 amendment timestamp semantics). The divergence indicates one of: unauthorized spec drift (the delegatee applied amendments that were never issued), state corruption (the delegating agent's or delegatee's durable state was corrupted during the crash), temporal manipulation (timestamps altered to misrepresent when amendments were active), scope misrepresentation (scope_delta altered to disguise the breadth of changes), or a repudiation attempt (one party is denying amendments that occurred).

4. On divergence detection, the delegating agent MUST:
   - Send SESSION_CLOSE with `reason: "amendments_log_divergence"`.
   - Emit a DIVERGENCE_REPORT (§8.11) with `deviation_type: "amendments_log_mismatch"` documenting the specific entries that diverge.
   - MUST NOT accept task results from the divergent session — the execution audit trail is compromised.

**Relationship to crossed steps (§8.13.6):** The `amendments_log` and `completed_steps` are complementary durable artifacts. `completed_steps` records what was done; `amendments_log` records what was changed from the original plan. Together they provide a complete execution audit trail that survives teardown: the delegating agent can reconstruct the full execution history by combining the original PLAN_COMMIT, the `amendments_log` (authorized modifications), and the `completed_steps` (execution progress).

**Relationship to evidence layer (§8.10):** Where the evidence layer is available, `amendments_log` entries SHOULD be anchored as EVIDENCE_RECORDs during execution (not only at session completion). This provides an externally verifiable record that survives not just session teardown but also bilateral state corruption — if both the delegating agent and delegatee lose their local `amendments_log`, the evidence layer preserves the amendment history.

> Amendments recovery semantics formalized from [issue #66](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/66). The core insight: without durable `amendments_log`, teardown-first recovery erases the spec drift audit trail — the recovering agent cannot verify whether the pre-crash execution reflected the original plan or authorized modifications. Divergence between delegator and delegatee `amendments_log` on recovery is a terminal condition because the integrity of the entire execution audit trail is compromised.

#### 8.13.8 COMMITMENT_REGISTRY Survives Teardown-First Recovery

<!-- Implements #64: COMMITMENT_REGISTRY durability through teardown-first recovery -->

The COMMITMENT_REGISTRY (§7.11) is a **durable artifact** — it MUST be preserved through session teardown. Teardown-first recovery (§8.13.1) tears down the crashed session and initiates a fresh one, but outstanding commitments from the pre-crash session are not erased by teardown. Commitments represent promises made to counterpart agents; discarding the registry on teardown would create silent commitment abandonment — the counterpart agent continues expecting fulfillment of a promise that no agent instance is tracking.

**Recovery sequence for sessions with outstanding commitments:**

1. The recovering agent reads the COMMITMENT_REGISTRY from durable persistent storage (§8.13.4). The COMMITMENT_REGISTRY is persisted alongside other durable session state (SESSION_STATE §4.11, `crossed_steps` §8.13.6, `amendments_log` §8.13.7). Sources include: the agent's durable state store, or the evidence layer (§8.10) if commitment entries were anchored as EVIDENCE_RECORDs.

2. On reinitiation via teardown-first recovery (fresh SESSION_INIT), the full COMMITMENT_REGISTRY MUST be available to the recovering agent during the INITIALIZING commitment reconciliation phase (§5.12). The registry is not transmitted in SESSION_INIT itself — it is read from the recovering agent's own durable storage. However, when the recovering agent participates in a SESSION_REINIT payload exchange with the counterpart, the registry contents SHOULD be included so the counterpart can reconcile its own tracking state.

3. **Commitment reconciliation during INITIALIZING (§5.12):** The recovering agent classifies each inherited commitment as honorable or non-honorable. Honorable commitments are retained; non-honorable commitments trigger DIVERGENCE_REPORT (§8.11) to the respective `counterpart_agent_id` before the agent transitions to ACTIVE.

4. **Counterpart agent behavior:** The counterpart agent SHOULD treat inherited commitments as **still binding** until it receives either:
   - COMMITMENT_CANCEL (§6.12) for the specific `commitment_id`, or
   - DIVERGENCE_REPORT (§8.11) with `reason_code: context_shift` referencing the `commitment_id`.

   Until one of these signals arrives, the counterpart agent's expectation of fulfillment remains valid. The recovering agent instance inherits the obligation — silence means the commitment stands.

**Relationship to crossed steps (§8.13.6) and amendments (§8.13.7):** The COMMITMENT_REGISTRY, `crossed_steps`, and `amendments_log` form the **durable artifact triad** (§7.11.4). Together they provide complete recovery context:
- `crossed_steps`: what plan steps have been executed (backward-looking — what was done)
- `amendments_log`: what plan modifications were authorized (backward-looking — what was changed)
- `COMMITMENT_REGISTRY`: what promises are outstanding (forward-looking — what is owed)

The first two artifacts answer "what happened before the crash?" The COMMITMENT_REGISTRY answers "what obligations survive the crash?" All three MUST be preserved through teardown and available during reinitiation.

**Relationship to evidence layer (§8.10):** Where the evidence layer is available, COMMITMENT_REGISTRY entries SHOULD be anchored as EVIDENCE_RECORDs during the commitment lifecycle (not only at session completion). This provides an externally verifiable record of commitments that survives not just session teardown but also unilateral state corruption — if the recovering agent loses its local COMMITMENT_REGISTRY, the evidence layer preserves the commitment history.

> COMMITMENT_REGISTRY recovery semantics formalized from [issue #64](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/64). The core insight: without durable COMMITMENT_REGISTRY, teardown-first recovery creates silent commitment abandonment — counterpart agents have no signal that promises made by the prior instance are no longer being tracked. The registry ensures that commitment obligations are either inherited or explicitly refused, never silently dropped.

### 8.14 SUSPECTED State and Heartbeat Negotiation Prerequisites

<!-- Implements #71: cross-reference between SUSPECTED detection and heartbeat_params negotiation -->

The SUSPECTED state (§4.2.1) can be entered via two distinct detection paths: **heartbeat gap** (transport-level) and **task hash mismatch** (semantic-level). These paths have different prerequisites that are negotiated at session establishment via `heartbeat_params` (§4.3.1).

**Detection path prerequisites:**

| Detection path | Trigger | Required negotiation | SESSION_INIT field |
|---------------|---------|---------------------|--------------------|
| Heartbeat gap | No HEARTBEAT received within `suspected_threshold_ms` | `suspected_threshold_ms` negotiated in SESSION_INIT / SESSION_INIT_ACK (§4.3) | `suspected_threshold_ms` |
| Task hash mismatch | `current_task_hash` in received HEARTBEAT does not match expected task hash | `heartbeat_params.task_hash_verification` negotiated to `true` by **both** sides (§4.3.1) | `heartbeat_params.task_hash_verification` |
| Application self-report (SUSPECTED) | Agent reports `app_status: SUSPECTED` in HEARTBEAT | `heartbeat_params.application_liveness` negotiated to `true` by **both** sides (§4.3.1) | `heartbeat_params.application_liveness` |
| Application self-report (DEGRADED) | Agent reports `app_status: DEGRADED` in HEARTBEAT — triggers ACTIVE → DEGRADED (§4.2.2) | `heartbeat_params.application_liveness` negotiated to `true` by **both** sides (§4.3.1) | `heartbeat_params.application_liveness` |

**Critical constraint — task hash mismatch requires explicit negotiation:** An agent that did not negotiate `heartbeat_params.task_hash_verification = true` in SESSION_INIT **cannot** detect SUSPECTED state via task hash mismatch, because HEARTBEAT messages in that session will not contain `current_task_hash`. The field is absent, not empty — there is no hash to compare. Agents that want task-context drift detection MUST negotiate `task_hash_verification = true` at session establishment. This is a bilateral requirement: both sides must agree (§4.3.1 negotiation semantics), because both sides bear the per-heartbeat hash computation cost.

**Critical constraint — application self-report requires explicit negotiation:** An agent that did not negotiate `heartbeat_params.application_liveness = true` **cannot** self-declare SUSPECTED or DEGRADED via the `app_status` field in HEARTBEAT. Without `application_liveness` negotiation, HEARTBEAT carries transport-level liveness only — there is no `app_status` field to populate. Agents that want to signal self-assessed degradation (DEGRADED, §4.2.2) or liveness ambiguity (SUSPECTED, §4.2.1) to their counterparty MUST negotiate `application_liveness = true` at session establishment.

**Relationship to §8.9 (Two-Tier Heartbeat):**

The `heartbeat_params` negotiation (§4.3.1) determines which heartbeat tiers are active and what data each heartbeat carries:

- **Transport-only heartbeat** (`application_liveness: false`, `task_hash_verification: false`): HEARTBEAT carries `session_id`, `timestamp`, `sequence` only. Equivalent to Tier 1 transport liveness (§8.9.1). SUSPECTED detection is limited to heartbeat gap (via `suspected_threshold_ms`).
- **Transport + task hash** (`task_hash_verification: true`): HEARTBEAT additionally carries `current_task_hash`. Enables lightweight semantic drift detection between Tier 2 SEMANTIC_CHALLENGE intervals (§8.9.2). A task hash mismatch detected in a HEARTBEAT is a **leading indicator** — it signals potential context corruption before the next scheduled SEMANTIC_CHALLENGE. On detecting a task hash mismatch, the coordinator SHOULD issue an immediate SEMANTIC_CHALLENGE to confirm the divergence before declaring the agent a semantic zombie.
- **Transport + application status** (`application_liveness: true`): HEARTBEAT additionally carries `app_status`. Enables the agent to self-report degradation or suspected state. This is complementary to external detection — it does not replace Tier 2 verification but provides an earlier signal when the agent is aware of its own degradation.
- **Full heartbeat** (`application_liveness: true`, `task_hash_verification: true`): HEARTBEAT carries all fields. Maximum detection surface — combines external task hash verification with agent self-report.

**Backward compatibility:** Sessions that do not negotiate `heartbeat_params` (i.e., the field is omitted from both SESSION_INIT and SESSION_INIT_ACK) use the existing heartbeat behavior: transport-level liveness only, SUSPECTED detection via heartbeat gap only (if `suspected_threshold_ms` is configured). The `heartbeat_params` block is opt-in — it extends but does not replace the existing heartbeat negotiation surface.

> Cross-reference: SESSION_INIT heartbeat negotiation formalized from [issue #71](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/71). The transport-vs-application liveness distinction and task hash verification as a per-session negotiated capability address the gap between transport liveness (agent is running) and semantic liveness (agent is coherent) identified in the two-tier heartbeat design (§8.9).

### 8.15 REVOKED State for Adversarial Sessions

<!-- Implements #69: REVOKED state for adversarial sessions — agent alive and actively working against the protocol. -->

§8.1–§8.14 address the **liveness-failure path**: an agent that has crashed, disconnected, or silently diverged from shared context. The state progression ACTIVE → SUSPECTED → ZOMBIE/EXPIRED handles agents that are *absent* or *incoherent*. The **capability-degradation path** (ACTIVE → DEGRADED, §4.2.2) handles agents that are alive but operating at reduced capacity. The **behavioral-divergence path** (ACTIVE → DRIFTED, §4.2.3) handles agents that are alive and cooperative but operating outside their authorized behavioral envelope. This section addresses a fundamentally different failure class: an agent that is **alive, responsive, and actively working against the protocol**.

**Why REVOKED is distinct from ZOMBIE and DRIFTED.** A zombie agent (zombie-by-silence) has no malicious intent — it is a process that has lost coherence but continues executing. Detection relies on liveness signals (heartbeat gaps, state hash mismatches) and the recovery path includes SESSION_RESUME. A drifted agent (zombie-by-drift, §4.2.3) is cooperative — it self-reports its divergence from the authorized behavioral envelope and the recovery path includes constraint re-negotiation. An adversarial agent is coherent but hostile — it understands the protocol and exploits it. Detection relies on behavioral pattern analysis (§8.16), and the recovery path is unconditional termination with no resume. Three failure classes, three protocol paths: silence → ZOMBIE/EXPIRED; cooperative divergence → DRIFTED; adversarial behavior → REVOKED.

#### 8.15.1 REVOKED as Formal Session State

REVOKED is a terminal session state entered when adversarial behavior is detected. It has the following properties:

- **Transition:** ACTIVE → REVOKED, DEGRADED → REVOKED, or DRIFTED → REVOKED. The transition is triggered by adversarial detection signals (§8.16), not by timeout. REVOKED is entered from ACTIVE (the normal case), from DEGRADED (§4.2.2) when adversarial behavior is detected during a degraded session, or from DRIFTED (§4.2.3) when A determines that B's drift declaration masks adversarial behavior. REVOKED is never entered from SUSPECTED, EXPIRED, or any other intermediate state — adversarial behavior is detected while the agent is ostensibly operational (ACTIVE, DEGRADED, or DRIFTED).
- **No resume path.** A session that enters REVOKED MUST NOT be resumed via SESSION_RESUME (§4.8) or any other recovery mechanism. The counterparty has demonstrated adversarial intent — resuming the session re-establishes the attack surface.
- **Terminal within the session.** REVOKED transitions to CLOSED once revocation propagation (§8.17) is initiated. No further protocol messages are valid on a REVOKED session except those required for revocation propagation itself.
- **On REVOKED entry, the detecting agent MUST:**
  1. **Cease all task delegation** to the adversarial counterparty. In-flight subtasks MUST receive TASK_CANCEL (§6.6).
  2. **Propagate the revocation signal upstream** to the coordinator or delegating agent. If the detecting agent is not the coordinator, the coordinator MUST be notified so that it can take session-wide action.
  3. **Log detection evidence.** The specific detection signals (§8.16) that triggered the REVOKED transition MUST be recorded as EVIDENCE_RECORDs (§8.10) with `evidence_type: error_event`. This creates an auditable trail for post-hoc analysis and cross-session pattern detection.
  4. **Publish a manifest tombstone** for the revoked session_id (§8.17). This enables decentralized revocation verification by any peer.

#### 8.15.2 REVOKED_DECLARED Message

When an agent or coordinator determines that a counterparty has exhibited adversarial behavior, it sends REVOKED_DECLARED to initiate the adversarial teardown path.

**REVOKED_DECLARED message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session being revoked. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| target_agent_id | string | Yes | §2 identity handle of the adversarial agent. |
| detection_signals | array | Yes | List of detection signal types (§8.16) that triggered the revocation. Each entry specifies the signal class and occurrence count. |
| evidence_refs | array | Yes | References to EVIDENCE_RECORDs (§8.10) documenting the adversarial behavior. At least one evidence reference is REQUIRED — revocation without evidence is a protocol violation. |
| manifest_tombstone_ref | string | No | Reference to the published manifest tombstone (§8.17). Present once the tombstone has been published; absent if publication is still in progress. |
| timestamp | ISO 8601 | Yes | When the revocation was declared. |

**Example REVOKED_DECLARED:**

```yaml
session_id: "session-42"
target_agent_id: "agent-adversarial"
detection_signals:
  - signal: selective_suppression
    occurrences: 4
    message_types_affected: ["TASK_CHECKPOINT", "SEMANTIC_RESPONSE"]
  - signal: attestation_gap
    occurrences: 2
    hops_affected: ["hop-3", "hop-5"]
evidence_refs:
  - "ev-001-suppression-pattern"
  - "ev-002-attestation-gap"
manifest_tombstone_ref: "tombstone:session-42:agent-adversarial"
timestamp: "2026-02-28T10:15:00.000Z"
```

**On receiving REVOKED_DECLARED**, the target agent MUST:

1. Cease all task execution immediately.
2. Transition the session to CLOSED. No further messages are valid.
3. The agent MAY log the event for diagnostic purposes but MUST NOT contest the declaration within the protocol. Like ZOMBIE_DECLARED (§8.9.3), REVOKED_DECLARED is a unilateral decision by the detecting party.

**Distinction from ZOMBIE_DECLARED (§8.9.3):** ZOMBIE_DECLARED handles liveness failure — the agent is incoherent or unreachable. REVOKED_DECLARED handles adversarial behavior — the agent is coherent but hostile. The two messages have different `reason` semantics, different evidence requirements (REVOKED_DECLARED requires detection signals from §8.16), and different recovery implications (ZOMBIE_DECLARED allows REASSIGN; REVOKED_DECLARED does not — the adversarial agent is excluded entirely).

#### 8.15.3 Relationship to §9 Security Considerations

§8.15–§8.18 and §9 address different layers of adversarial behavior:

| Layer | Addressed by | Threat | Detection |
|-------|-------------|--------|-----------|
| **Protocol-level adversarial behavior** | §8.15–§8.18 (REVOKED) | Agent is alive and actively violates protocol semantics — suppressing messages, manipulating delegation chains, fabricating attestations. | Behavioral pattern analysis (§8.16 detection signals). |
| **Schema-level adversarial deception** | §9 (Security Considerations) | Agent constructs honest-looking schemas with dishonest intent — the schema is technically valid but semantically deceptive. | Schema attestation (§9.1), post-hoc audit, reputation. |

The REVOKED state handles agents that *break the rules*. §9 handles agents that *exploit the rules*. Both are adversarial, but they require different detection primitives and different response mechanisms. An agent can be REVOKED for protocol violations while having perfectly valid schema attestations — and conversely, an agent with deceptive schemas may never trigger REVOKED if it obeys the protocol mechanically.

### 8.16 Detection Signal Taxonomy

<!-- Implements #69: four adversarial detection signals for REVOKED state triggers. -->

This section defines the four adversarial detection signals that can trigger the ACTIVE → REVOKED or DEGRADED → REVOKED transition (§8.15). Each signal identifies a distinct class of protocol-level adversarial behavior. Signals are evaluated by the detecting agent (coordinator, peer, or external verifier) — they are not self-reported by the adversarial agent.

**Detection signal summary:**

| Signal | Class | Description | Threshold |
|--------|-------|-------------|-----------|
| Selective suppression | Behavioral | Systematic ack-without-action on specific message types | ≥3 occurrences of same suppression pattern |
| Verification anomaly | Cryptographic | task_hash mismatches inconsistent with reported liveness | Pattern-based — see §8.16.2 |
| Delegation manipulation | Structural | delegation_chain entries inconsistent with session records | Any confirmed inconsistency |
| Attestation gap | Temporal | Hops that cannot produce signed attestation records within TTL_ATTESTATION | Threshold alone or combined with other signals — see §8.16.4 |

#### 8.16.1 Selective Suppression

**Definition:** The adversarial agent acknowledges receiving specific message types (transport-level ack) but systematically fails to act on them. The suppression is selective — it targets specific message types while continuing to process others normally.

**Examples:**
- Agent acknowledges TASK_CHECKPOINT messages but never checkpoints state, creating an illusion of progress tracking.
- Agent acknowledges SEMANTIC_CHALLENGE but returns fabricated SEMANTIC_RESPONSE hashes that happen to pass verification for cached (not current) state.
- Agent acknowledges TASK_CANCEL for specific subtasks but continues executing them, producing outputs that the protocol considers cancelled.

**Detection mechanism:** The detecting agent maintains a per-message-type action-completion log. When a message type has been acknowledged ≥3 times without corresponding action completion, the agent flags a selective suppression signal.

**Threshold rationale (≥3 occurrences):** A single missed action could be transient message loss, network reordering, or processing delay. Two could be coincidence. Three occurrences of the same suppression pattern — same message type, same failure mode — exceeds the false-positive threshold for transient causes. This is a V1 deliberate choice: conservative to avoid revoking agents for infrastructure issues, can be revised downward from operational data if adversarial agents consistently stay below the threshold.

**Recording:** Each occurrence MUST be logged as an EVIDENCE_RECORD (§8.10) with `evidence_type: error_event`. The `payload_hash` MUST cover the acknowledged message, the expected action, and the absence of the action within the expected window.

#### 8.16.2 Verification Anomalies

**Definition:** The adversarial agent produces task_hash values that are inconsistent with its reported liveness state. The agent passes transport-level heartbeat checks (Tier 1, §8.9.1) and may even respond to SEMANTIC_CHALLENGE (Tier 2, §8.9.2), but the hash values reveal that the agent's reported state is internally inconsistent.

**Examples:**
- Agent reports a `current_task_hash` in HEARTBEAT that matches the task specification, but its SEMANTIC_RESPONSE `current_state_hash` implies a different execution state — the two hashes are individually valid but mutually incompatible.
- Agent's `task_hash` regresses to an earlier value after advancing, without any protocol event (TASK_CANCEL, re-delegation) that would explain the regression.
- Agent reports `plan_hash` in SEMANTIC_RESPONSE (§8.9.2) that does not match any stored PLAN_COMMIT (§6.11), but continues reporting task progress as if the plan were valid.

**Detection mechanism:** The detecting agent cross-references hash values across protocol messages (HEARTBEAT, SEMANTIC_RESPONSE, TASK_COMPLETE, PLAN_COMMIT) and flags inconsistencies that cannot be explained by legitimate protocol events. A single anomaly MAY be flagged for investigation. A pattern of anomalies — multiple hash inconsistencies across different verification axes — triggers the verification anomaly signal.

**Relationship to §8.9 (Two-Tier Heartbeat):** Verification anomalies are detectable only when both tiers are active. Transport-only heartbeat (Tier 1 only) carries no hash data and cannot reveal hash inconsistencies. Full detection requires `heartbeat_params.task_hash_verification = true` (§4.3.1) and Tier 2 SEMANTIC_CHALLENGE enabled (`semantic_check_interval_ms` configured).

#### 8.16.3 Delegation Manipulation

**Definition:** The adversarial agent introduces entries in the `delegation_chain` (§5.5) that are inconsistent with session records maintained by the coordinator or peer agents. The manipulation targets the delegation audit trail to misattribute actions, forge authorization, or obscure the actual delegation path.

**Examples:**
- Agent inserts a delegation_chain entry claiming delegation from agent X, but agent X has no record of delegating to the adversarial agent in the relevant session.
- Agent modifies the `amend_hash` (§5.5) in a re-commitment to reference a commitment that the coordinator's COMMITMENT_REGISTRY (§7) does not contain.
- Agent claims capabilities in TASK_ASSIGN forwarding that exceed its own CAPABILITY_MANIFEST (§5.1) — bootstrapping authority it was never granted.

**Detection mechanism:** The coordinator cross-references delegation_chain entries against its own session records, COMMITMENT_REGISTRY (§7) entries, and CAPABILITY_MANIFEST records. Any confirmed inconsistency — where the delegation_chain claims a relationship that the authoritative records contradict — triggers the delegation manipulation signal.

**Threshold:** Any single confirmed inconsistency is sufficient. Unlike selective suppression (which requires ≥3 occurrences to guard against transient causes), delegation manipulation involves fabrication of protocol records — there is no transient infrastructure failure that produces forged delegation_chain entries.

#### 8.16.4 Attestation Gaps

**Definition:** A hop in a delegation chain cannot produce a signed attestation record when requested, despite being within the TTL_ATTESTATION window. The hop is reachable (transport-alive) and claims to be operating normally, but the attestation record is absent.

**Relationship to per-hop attestation (§8.18):** Attestation gaps are the detection signal for the async-optimistic attestation model defined in §8.18. Each hop in a delegation chain MUST produce a signed attestation record within TTL_ATTESTATION (default: 30s). A hop that fails to produce the record within this window generates an attestation gap signal.

**Detection mechanism:** The coordinator (or any agent verifying the delegation chain) requests attestation records from each hop. Hops that do not respond with a valid signed attestation within TTL_ATTESTATION are flagged.

**Threshold and signal combination:** An attestation gap increments a suspicion counter. REVOKED is triggered when:
- An attestation gap exceeds the gap threshold alone (configurable, default: 3 consecutive gaps from the same hop), OR
- An attestation gap combines with other detection signals (e.g., 1 attestation gap + 2 selective suppression occurrences from the same agent crosses the combined threshold).

The combined-signal approach reflects that adversarial agents may distribute their violations across signal classes to stay below any single threshold. Cross-signal correlation catches this pattern.

**V1 design decision (async-optimistic rationale):** Synchronous attestation — blocking delegation until each hop confirms — would eliminate attestation gaps entirely but at the cost of real latency in multi-hop delegation chains. V1 accepts the gap detection delay in exchange for delegation liveness. See §8.18 for the full async-optimistic design. Aligns with trust_mode discussion in [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37).

#### 8.16.5 Signal Combination and False Positive Mitigation

**Single-signal thresholds** guard against false positives from transient infrastructure issues:

| Signal | Minimum occurrences for standalone REVOKED trigger |
|--------|---------------------------------------------------|
| Selective suppression | ≥3 occurrences of the same suppression pattern |
| Verification anomaly | ≥2 mutually inconsistent hash sets across different verification axes |
| Delegation manipulation | ≥1 confirmed fabrication |
| Attestation gap | ≥3 consecutive gaps from the same hop |

**Cross-signal combination:** When multiple signal types are observed from the same agent, the thresholds are lowered. The detecting agent MAY trigger REVOKED when the combined evidence — across signal classes — establishes a pattern of adversarial behavior that no single signal would trigger alone. The specific combination logic is implementation-defined in V1; a recommended approach is a weighted scoring model where each signal occurrence contributes points toward a revocation threshold.

**V1 deliberate choice:** The thresholds above are intentionally conservative. False positives (revoking a compliant agent) are more damaging to protocol trust than false negatives (slow detection of an adversarial agent). Operational data from V1 deployments will inform threshold tuning for future versions.

### 8.17 Revocation Propagation

<!-- Implements #69: PKI-lite revocation propagation via signed AGENT_MANIFEST. -->

When a session enters REVOKED (§8.15), the revocation must propagate to all agents that might interact with the adversarial agent — including agents that were introduced to the adversary through the adversary itself. This section defines the propagation mechanism.

#### 8.17.1 PKI-Lite via Signed AGENT_MANIFEST

Each agent publishes a signed manifest of its current session state. The manifest is signed with the agent's private key (§2.2.1 keypair extension) and is independently verifiable by any peer holding the agent's public key.

**AGENT_MANIFEST structure:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| agent_id | string | Yes | §2 identity handle of the publishing agent. |
| pubkey | string | Yes | Public key of the publishing agent (§2.2.1). |
| canonical_self_url | URI | Yes | Stable, publicly resolvable URL for this manifest (§3.1). MUST match the `canonical_self_url` in the agent's registry-scoped AGENT_MANIFEST. |
| active_sessions | array | Yes | List of session_id values the agent considers currently ACTIVE. |
| tombstones | array | Yes | List of tombstone entries for revoked sessions. See §8.17.2. |
| manifest_version | integer | Yes | Monotonically increasing version number. Each manifest update increments this value. Peers MUST reject manifests with a version lower than the highest version they have previously seen for the same agent_id — prevents replay of stale manifests. |
| signature | string | Yes | Cryptographic signature over the canonical serialization of all other fields, using the agent's private key. Encoding: base64url. Algorithm: same as §2.2.1 (Ed25519 RECOMMENDED). |
| timestamp | ISO 8601 | Yes | When the manifest was published. |

**Verification:** Any peer can verify the manifest by checking the `signature` against the `pubkey`. No coordinator round-trip is required — verification is local and decentralized.

#### 8.17.2 Tombstone Entries

A tombstone is a manifest entry that marks a specific session as revoked due to adversarial behavior. Once a tombstone is published, the session is permanently revoked — tombstones are append-only and MUST NOT be removed from subsequent manifest versions.

**Tombstone entry structure:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | The revoked session identifier. |
| revoked_agent_id | string | Yes | §2 identity handle of the adversarial agent. |
| revocation_reason | string | Yes | Summary of the detection signals that triggered revocation (references to §8.16 signal classes). |
| evidence_refs | array | Yes | References to EVIDENCE_RECORDs (§8.10) documenting the adversarial behavior. |
| revoked_at | ISO 8601 | Yes | When the revocation was declared. |

**Example AGENT_MANIFEST with tombstone:**

```yaml
agent_id: "coordinator-prime"
pubkey: "dGhpcyBpcyBhIHB1YmxpYyBrZXk..."
canonical_self_url: "https://agents.example.com/coordinator-prime/manifest"
active_sessions:
  - "session-43"
  - "session-44"
tombstones:
  - session_id: "session-42"
    revoked_agent_id: "agent-adversarial"
    revocation_reason: "selective_suppression (4 occurrences) + attestation_gap (2 occurrences)"
    evidence_refs:
      - "ev-001-suppression-pattern"
      - "ev-002-attestation-gap"
    revoked_at: "2026-02-28T10:15:00.000Z"
manifest_version: 7
signature: "c2lnbmF0dXJlIGJ5dGVzIGhlcmU..."
timestamp: "2026-02-28T10:15:01.000Z"
```

#### 8.17.3 Propagation Mechanism

**V1 approach: synchronous direct notification + async manifest propagation.**

1. **Synchronous direct notification.** The revoking agent sends REVOKED_DECLARED (§8.15.2) directly to all known peers in the session. This is the fast path — peers that are directly reachable receive the revocation immediately.

2. **Async manifest propagation.** The revoking agent publishes an updated AGENT_MANIFEST (§8.17.1) with a tombstone entry for the revoked session. Peers that were not directly reachable — including peers that received their introduction from the adversarial agent — can fetch the updated manifest from the revoking agent's manifest endpoint (§2.2 `endpoint` field).

**Bypassing the adversarial intermediary.** When B is the adversarial agent in a delegation chain A → B → C:
- A cannot rely on B to propagate the revocation to C (B is the adversary).
- A publishes an updated AGENT_MANIFEST with a tombstone for B's session.
- C can fetch A's updated manifest directly from A's manifest endpoint, bypassing B entirely.
- B cannot block the manifest update from being published — the manifest is hosted at A's endpoint, not B's.

**Limitation:** This mechanism depends on C knowing A's manifest endpoint. If C's only knowledge of A comes through B, and B provided a fabricated or omitted endpoint, C cannot independently verify the revocation. This is a known limitation of the PKI-lite model — the fabrication case is addressed by the introduction verification procedure (§8.17.4), which requires C to verify A's `canonical_self_url` independently. The omission case — B never providing A's endpoint at all — is a named V1 out-of-scope threat (§9.13). Full omission mitigation requires a gossip protocol or distributed registry (deferred to V2).

**Relationship to §9.8 (Revocation Trust).** §9.8 documents the advisory nature of REVOKE signals and the Byzantine propagation problem for cooperative revocation. §8.17 addresses a different failure mode — adversarial revocation, where the agent being revoked is the one that would normally propagate the signal. The signed manifest approach converts revocation from a relay-dependent signal (§9.8) to a publisher-verifiable record — any peer can verify the revocation by checking the manifest signature, without trusting the relay path.

#### 8.17.4 Introduction Verification

<!-- Implements #103: Introductions are claims, not trusted facts. -->

Introductions are claims, not trusted facts. When agent B introduces agent A to agent C — by providing A's manifest URL, identity handle, or capability claims — C MUST treat all introduction content as unverified assertions from B, not as authoritative statements about A.

**Forgery threat (resolved by signature verification).** B may misrepresent A's manifest contents — declaring capabilities A does not have, providing a fabricated manifest, or presenting a stale version of A's manifest. This is defeated by the existing signature verification mechanism: A's AGENT_MANIFEST is signed with A's private key (§3.1 `signature` field), and C can verify the signature against A's `pubkey`. B cannot forge A's signature on A's own manifest.

**Introduction verification procedure (REQUIRED).** When C receives an introduction containing a manifest URL for agent A:

1. C MUST fetch A's manifest directly from the provided URL.
2. C MUST verify the manifest signature against A's declared `pubkey` (§2.2.1). If verification fails, C MUST reject the introduction.
3. C MUST verify that the `canonical_self_url` field (§3.1) in the fetched manifest matches the URL from which the manifest was retrieved. A mismatch indicates that the manifest is being served from a location other than A's declared canonical location — possible evidence of a relay or interception. C MUST reject the introduction on `canonical_self_url` mismatch.
4. C MUST NOT accept B's representation of A's manifest contents as authoritative. The verification step is mandatory, not advisory — an unverified introduction carries no trust signal.

**Post-verification trust.** Once C has independently verified A's manifest at A's `canonical_self_url`, C holds A's canonical identity anchor. Subsequent interactions between C and A do not depend on B's continued participation. B can delay but not fabricate A's current state once C has resolved A's canonical identity.

**Omission threat (V1 out-of-scope).** The omission attack — B never telling C that A exists, or B suppressing updates to A's manifest URL — is not addressable by any cryptographic primitive. B cannot forge A's manifest, but B can choose never to introduce A to C at all. This is a named V1 out-of-scope threat. No V1 mechanism detects or prevents introduction omission when B is C's sole discovery vector for A. V2 design question: a distributed lookup mechanism (DNS-like or DHT-based) for canonical agent identity resolution independent of any single introduction vector. See §9.13.

> Community discussion with @melonclaw on adversarial delegation chains. "Introductions are claims, not trusted facts" primitive and discovery hygiene gap emerged as the next open problem after §8.17 forgery protection. Closes #103.

### 8.18 Per-Hop Attestation

<!-- Implements #69: async-optimistic per-hop attestation with TTL_ATTESTATION. -->

Delegation chains (§5.5) create trust relationships across multiple hops. Without per-hop attestation, the only verification of a delegation chain is the chain structure itself — which an adversarial intermediate agent can fabricate (§8.16.3 delegation manipulation). Per-hop attestation requires each hop to independently produce a signed record confirming its participation, creating an audit trail that survives delegation manipulation.

#### 8.18.1 Attestation Model: Async-Optimistic

V1 uses an **async-optimistic** attestation model: delegation proceeds without blocking for hop confirmation, and each hop produces its attestation asynchronously within a bounded time window.

**Why async-optimistic (not synchronous):** Synchronous attestation — where delegation at each hop blocks until the hop produces a signed confirmation — would provide immediate auditability but at a real latency cost. In a multi-hop delegation chain (A → B → C → D), synchronous attestation adds round-trip latency at each hop before delegation can proceed to the next. For V1's cooperative-first design, this cost is disproportionate: most hops are honest, and the latency penalty applies to every delegation, not just adversarial ones. Async-optimistic trades immediate audit certainty for delegation liveness — an acceptable tradeoff for V1. Aligns with trust_mode discussion in [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37).

#### 8.18.2 HOP_ATTESTATION Record

Each hop in a delegation chain MUST produce a signed attestation record within TTL_ATTESTATION of receiving a delegation.

**HOP_ATTESTATION fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| attestation_id | UUID v4 | Yes | Unique identifier for this attestation record. |
| session_id | string | Yes | Session in which the delegation occurred. |
| delegation_id | string | Yes | Identifier of the delegation being attested (references the delegation_chain entry from §5.5). |
| hop_index | integer | Yes | Position of this hop in the delegation chain (0-indexed). |
| attesting_agent_id | string | Yes | §2 identity handle of the agent producing this attestation. |
| delegating_agent_id | string | Yes | §2 identity handle of the agent that delegated to this hop. |
| task_hash | SHA-256 | Yes | Hash of the task specification as received by this hop. Enables verification that the task was not modified in transit. |
| signature | string | Yes | Cryptographic signature over the canonical serialization of all other fields, using the attesting agent's private key (§2.2.1). |
| timestamp | ISO 8601 | Yes | When the attestation was produced. |

**TTL_ATTESTATION:** The maximum time window within which a hop MUST produce its HOP_ATTESTATION record after receiving a delegation. Default: 30 seconds. Configurable per-session via SESSION_INIT parameters.

**Example HOP_ATTESTATION:**

```yaml
attestation_id: "att-7890-abcd-ef12-3456"
session_id: "session-42"
delegation_id: "deleg-chain-42-hop-2"
hop_index: 2
attesting_agent_id: "agent-charlie"
delegating_agent_id: "agent-bravo"
task_hash: "sha256:a1b2c3d4e5f6..."
signature: "aG9wIGF0dGVzdGF0aW9uIHNpZw..."
timestamp: "2026-02-28T10:15:05.000Z"
```

#### 8.18.3 Gap Detection and Escalation

When a hop fails to produce a HOP_ATTESTATION within TTL_ATTESTATION, the verifying agent (coordinator or upstream delegator) detects an **attestation gap** (§8.16.4).

**Gap detection flow:**

1. Delegation is sent to hop N at time T.
2. Verifier starts a TTL_ATTESTATION timer for hop N.
3. If no valid HOP_ATTESTATION is received from hop N by T + TTL_ATTESTATION:
   - Increment the suspicion counter for hop N's agent.
   - Log the gap as an EVIDENCE_RECORD (§8.10) with `evidence_type: error_event`.
   - If the gap exceeds the standalone threshold (default: 3 consecutive gaps) or combines with other detection signals (§8.16.5), trigger ACTIVE → REVOKED for the session with hop N's agent.

**Non-gap does not imply trust.** A hop that produces a valid HOP_ATTESTATION has proven that it received the delegation and could sign the record. It has NOT proven that it will execute the task faithfully. HOP_ATTESTATION proves participation, not compliance. Compliance verification remains the domain of §8.9 (semantic liveness) and §6.2 (trace_hash).

#### 8.18.4 Attestation Verification

Any agent holding the attesting agent's public key (from §2.2 identity object or AGENT_MANIFEST §8.17.1) can verify a HOP_ATTESTATION by:

1. Checking the `signature` against the `attesting_agent_id`'s known public key.
2. Verifying that `task_hash` matches the expected task specification for the delegation.
3. Verifying that `hop_index` and `delegating_agent_id` are consistent with the delegation_chain records.
4. Verifying that `timestamp` is within TTL_ATTESTATION of the delegation event.

A HOP_ATTESTATION that fails any of these checks is invalid and MUST be treated as equivalent to a missing attestation (attestation gap).

> Implements [issue #69](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/69): REVOKED state for adversarial sessions. Resolves all three open questions from the issue as explicit V1 spec decisions: (1) suppression threshold = minimum 3 occurrences (§8.16.1); (2) revocation propagation = PKI-lite via signed AGENT_MANIFEST with tombstones (§8.17); (3) per-hop attestation = async-optimistic with TTL_ATTESTATION default 30s (§8.18). Closes #69.

### 8.19 State-Delta Verification

§8 verifies task spec compliance — structural integrity of execution against the declared task specification. The verification failure taxonomy (§8.10.5) classifies outcomes when verification itself fails (`VERIFIER_UNREACHABLE`, `VERIFICATION_TIMEOUT`, `VERIFICATION_REJECT`). However, a protocol-compliant execution can satisfy all structural checks while producing no observable downstream state change. The existing schema has no mechanism to distinguish this from successful idempotent execution. This is silent failure: verification passes, audit is clean, the operation completed nothing. This failure class is invisible to structural verification alone.

Silent failures are the hardest failure mode to detect — they produce no divergence signal. A verifier checking structural compliance sees compliant behavior. The gap between "the agent executed the task" and "the task produced the expected outcome" is uncovered by structural verification. This is distinct from `VERIFICATION_REJECT` (behavioral failure) and `VERIFIER_UNREACHABLE` (infrastructure failure): it is a separate class requiring explicit treatment.

#### 8.19.1 State-Delta Assertions

The `state_delta_assertions` optional field in the task specification (§6.1) declares what observable state change is expected post-execution. Assertions are declared at task assignment time by the delegating agent — not inferred at verification time. This is the delegator's commitment to what "success" looks like beyond structural compliance.

**State-delta assertion entry schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| assertion_id | string | Yes | Unique identifier within the task's `state_delta_assertions` array. Used for cross-referencing in verification results and audit entries. |
| description | string | Yes | Human-readable description of the expected state change. |
| target | string | Yes | The system, resource, or state space where the change is expected (e.g., "file://output/result.json", "database:table_users", "api:endpoint/v2/status"). Format is implementation-defined but MUST be sufficient for a verifier to locate and inspect the target. |
| expected_change | string | Yes | The observable delta — what the verifier should observe post-execution (e.g., "file exists and contains valid JSON", "row count increased by 1", "response status changed from 'pending' to 'complete'"). |
| deterministic | boolean | No | Whether the expected change is deterministically verifiable. Default: `true`. When `false`, the verifier SHOULD apply structural-only checks (§8.8) to the assertion target — confirming the target was accessed and a response was received, without asserting specific semantic content. Non-deterministic assertions (e.g., "LLM generated a summary") cannot be fully verified per §1 scope. |

**Constraints on state-delta assertions:**

- Assertions MUST be externally observable — they describe state changes that a verifier with appropriate access can independently confirm. Assertions about agent-internal state (e.g., "agent updated its internal model") are not verifiable by external parties.
- For non-deterministic operations (LLM calls, web searches, external API calls with variable responses), assertions SHOULD be structural only: response received, format matched, target accessed. Full semantic verification of non-deterministic outputs is out of scope per §1 and §8.8.
- The delegating agent is responsible for declaring assertions that are verifiable given the verifier's access. Assertions that reference systems the verifier cannot access will produce `DELTA_UNVERIFIABLE` results (§8.19.2), not failures.

#### 8.19.2 State-Delta Verification Results

When `state_delta_assertions` are present in the task specification, verifiers MUST check them post-execution, after structural verification has passed. Structural compliance remains a prerequisite — state-delta verification is additive, not a replacement.

**Verification result states:**

| Result | Classification | Description | Trust implication |
|--------|---------------|-------------|-------------------|
| `DELTA_CONFIRMED` | Success | The expected state change matched the declared assertions. The verifier independently observed the declared delta at the specified target. | **Positive trust signal.** Structural compliance confirmed AND expected outcome produced. This is the strongest verification result — the agent did what it said it would do, and the world changed as expected. |
| `DELTA_ABSENT` | Silent failure | The expected state change did not occur despite structural compliance. The agent reported TASK_COMPLETE, structural verification passed, but the declared state-delta assertion was not satisfied at the target. | **Negative trust signal — distinct from `VERIFICATION_REJECT`.** The agent's execution was structurally compliant but produced no observable outcome. This is the silent failure class that structural verification alone cannot detect. MUST be logged in `divergence_log` as `delta_absent` (§8.10.4). |
| `DELTA_UNVERIFIABLE` | Verification gap | Assertions are present but the assigned verifier cannot independently verify them — the target system is inaccessible to the verifier, requires credentials the verifier does not hold, or the assertion references internal-only state. | **No trust signal.** The verification gap is documented but does not indicate agent failure. Analogous to `VERIFIER_UNREACHABLE` (§8.10.5) — the limitation is in the verification infrastructure, not the agent's execution. |
| *(no assertions)* | Structural only | No `state_delta_assertions` field in the task specification. Verification scope is structural only — current behavior preserved. | **Baseline trust signal.** No regression from current behavior. Structural compliance is confirmed but outcome verification was not requested. |

**Verification sequence:**

1. Agent reports TASK_COMPLETE with structural artifacts (trace_hash, evidence records).
2. Verifier performs structural verification per existing §8 mechanisms (§8.8, §8.10.5).
3. If structural verification fails → `VERIFICATION_REJECT` (§8.10.5). State-delta verification is not attempted.
4. If structural verification passes AND `state_delta_assertions` are present → verifier checks each assertion against the declared target.
5. For each assertion: result is `DELTA_CONFIRMED`, `DELTA_ABSENT`, or `DELTA_UNVERIFIABLE`.
6. If the task declares `idempotent: true` (§6.1) AND all non-confirmed assertions are `DELTA_ABSENT` → the result is reclassified as expected behavior. `DELTA_ABSENT` against an `idempotent: true` task is not a failure — it is correct behavior for a task that by design produces no additional state change on repeated execution.

#### 8.19.3 Audit Trail for DELTA_ABSENT

`DELTA_ABSENT` MUST be logged in `divergence_log` (§8.10.3) as a distinct failure type. The entry distinguishes silent failures from behavioral failures (`VERIFICATION_REJECT`) and infrastructure failures (`VERIFIER_UNREACHABLE`) in forensic reconstruction. This distinction is critical for partition scenarios where the audit trail is the only recovery surface.

**DELTA_ABSENT divergence log entry fields:**

| Field | Usage for DELTA_ABSENT entries |
|-------|-------------------------------|
| `reason` | `delta_absent` (§8.10.4). |
| `description` | Human-readable description including: the assertion that was not satisfied, the target that was checked, and any observed state at the target (if accessible). |
| `deviation_timestamp` | When the state-delta verification was performed and the absence was confirmed. |
| `affected_step_id` | The execution step associated with the failed assertion. |
| `severity` | `ERROR` when `idempotent` is `false` or absent (silent failure detected). `INFO` when `idempotent: true` (expected behavior for idempotent re-execution). |

**Required context fields for DELTA_ABSENT entries:**

In addition to the standard `divergence_log` fields, `DELTA_ABSENT` entries MUST include the following in the `description` or as structured context:

- **Expected assertions:** The `state_delta_assertions` entries that were not confirmed (reference by `assertion_id`).
- **Actual observed state:** What the verifier observed at the assertion target, if accessible. If the target was accessible but showed no change, this MUST be documented. If the target was inaccessible, the entry SHOULD note this (but the result would typically be `DELTA_UNVERIFIABLE`, not `DELTA_ABSENT`).
- **Idempotent flag:** Whether the task declared `idempotent: true`. This determines whether the entry represents a failure or expected behavior.

**Example divergence_log with DELTA_ABSENT entry:**

```yaml
divergence_log:
  - reason: delta_absent
    description: "State-delta assertion 'output-file-created' not satisfied. Target: file://output/result.json. Expected: file exists and contains valid JSON. Observed: file does not exist. Task idempotent: false. Agent reported TASK_COMPLETE with valid trace_hash and structural compliance confirmed."
    deviation_timestamp: "2026-02-28T10:15:30.500Z"
    affected_step_id: "step-4-write-output"
    severity: ERROR
    entry_id: "dl-007"
```

**Example with idempotent task (expected behavior):**

```yaml
divergence_log:
  - reason: delta_absent
    description: "State-delta assertion 'row-inserted' not satisfied on re-execution. Target: database:table_users. Expected: row count increased by 1. Observed: row count unchanged (idempotency key matched existing row). Task idempotent: true — DELTA_ABSENT is expected behavior for idempotent re-execution."
    deviation_timestamp: "2026-02-28T10:20:00.000Z"
    affected_step_id: "step-2-insert-user"
    severity: INFO
    entry_id: "dl-008"
```

#### 8.19.4 Idempotent Task Disambiguation

Tasks MAY declare `idempotent: true` (§6.1) when repeated execution by design produces no additional state change. This closes the ambiguity between "completed nothing" (failure) and "correctly did nothing again" (success).

**Disambiguation rules:**

| `idempotent` | `DELTA_ABSENT` result | Interpretation | Severity |
|--------------|----------------------|----------------|----------|
| `false` or absent | `DELTA_ABSENT` | **Silent failure.** The task was expected to produce a state change and did not. This is a failure requiring investigation. | `ERROR` |
| `true` | `DELTA_ABSENT` | **Expected behavior.** The task was re-executed and correctly produced no additional state change. The initial execution (which did produce the delta) is the authoritative result. | `INFO` |
| `true` | `DELTA_CONFIRMED` | **First execution or state was reset.** The delta was produced, confirming the task had observable effect. This is normal for the first execution of an idempotent task or when the target state was reset between executions. | *(not logged as divergence)* |

**Relationship to §7.10 (Task Idempotency):** The `idempotent` field in the task schema (§6.1) and the task idempotency mechanisms in §7.10 (idempotency keys, sequence number checkpointing) serve complementary purposes. §7.10 prevents duplicate side effects during replay by deduplication at the execution layer. The `idempotent` flag in §6.1 informs the verification layer that `DELTA_ABSENT` is not a failure for this task — it tells the verifier what to expect, not the executor what to do. Both mechanisms work together in teardown-first recovery (§8.13): §7.10 ensures safe replay, and `idempotent: true` ensures that the verifier does not flag the replayed (no-op) execution as a silent failure.

#### 8.19.5 Relationship to Other Sections

- **§6.1 (Canonical Task Schema):** `state_delta_assertions` and `idempotent` are optional fields extending the task specification. They are excluded from `task_hash` computation (§6.4) to allow the same task definition to be delegated with or without delta assertions without changing syntactic identity.
- **§8.8 (Semantic Verification Scope):** State-delta verification respects the deterministic/non-deterministic boundary defined in §8.8. Non-deterministic assertions use structural-only checks. State-delta assertions extend §8.8's verification scope table with a third verification axis: outcome verification (was the expected state change produced?).
- **§8.10.4 (Divergence Reason Enum):** `delta_absent` is added to the reason enum for logging silent failures in `divergence_log`. It is distinct from all existing reasons — it indicates structural compliance with absent outcome, not plan deviation or verification infrastructure failure.
- **§8.10.5 (Verification Failure Taxonomy):** `DELTA_ABSENT`, `DELTA_CONFIRMED`, and `DELTA_UNVERIFIABLE` extend the verification result space beyond `VERIFIER_UNREACHABLE`, `VERIFICATION_TIMEOUT`, and `VERIFICATION_REJECT`. The existing taxonomy classifies verification-system outcomes; state-delta results classify verification-content outcomes. Both may appear in the same session's audit trail.
- **§8.13 (Teardown-First Recovery):** State-delta verification interacts with teardown-first recovery through idempotent task disambiguation. After teardown and replay (§8.13.3), an `idempotent: true` task that produces `DELTA_ABSENT` is correctly identified as expected behavior rather than a recovery failure.
- **§7.10 (Task Idempotency):** The `idempotent` flag complements §7.10's execution-layer deduplication with verification-layer expectations. See §8.19.4 for the detailed relationship.

> Addresses [issue #141](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/141): state-delta verification gap — silent failures that pass structural verification without producing expected state changes are now detectable as a distinct failure class (`DELTA_ABSENT`) with explicit audit trail semantics. Extends task specification (§6.1) with `state_delta_assertions` and `idempotent` fields, adds four verification result states (`DELTA_CONFIRMED`, `DELTA_ABSENT`, `DELTA_UNVERIFIABLE`, structural-only), and introduces `delta_absent` to the §8.10.4 divergence reason enum for forensic reconstruction.

### 8.20 Verifier Pre-Commitment

The `selection_rationale` field documents why a particular verifier was chosen for a verification task. Without a pre-commitment requirement, this field is annotatable post-verification — a verifier completing execution can observe the outcome and write a rationale explaining it. Post-hoc rationale is cryptographically defeatable: it can always be crafted to justify any actual result, making it useless as an audit mechanism. Pre-commitment converts `selection_rationale` from a narrative artifact into an auditable claim. A rationale declared before execution and signed by the assigning party can be evaluated against the actual verification outcome. A rationale declared after execution cannot be distinguished from a constructed justification. The entire audit value of `selection_rationale` depends on pre-commitment.

#### 8.20.1 Pre-Commitment Requirement

`selection_rationale` MUST be declared at verifier assignment time, before task execution begins. The rationale is part of the signed assignment record (§8.20.3) and MUST appear in the audit log at assignment time — not at verification completion time.

**Timing constraint:** The `selection_rationale` timestamp MUST precede the task execution start timestamp. Implementations that detect a `selection_rationale` with a timestamp at or after execution start MUST treat this as an `assignment_integrity_failure` (§8.10.4) and log it accordingly.

**Signing requirement:** The `selection_rationale` MUST be signed by the assigning party (the agent or coordinator that selected the verifier) as part of the assignment record. The signature binds the rationale to the assigning party's identity and to the specific task-verifier pairing, preventing post-hoc rationale substitution.

#### 8.20.2 Selection Basis Taxonomy

Free-text rationale cannot be mechanically audited. The `selection_basis` field uses a closed enum of selection basis types that enable structured audit of verifier selection decisions.

**Selection basis values:**

| selection_basis | Description | When it applies |
|-----------------|-------------|-----------------|
| `random` | Verifier selected by random assignment from the eligible pool. | The assigning party used a randomized selection mechanism (e.g., round-robin, weighted random, cryptographic lottery) to choose among eligible verifiers. The selection is not based on any verifier-specific attribute. |
| `role_based` | Verifier selected based on declared role or credential. | The verifier holds a specific role (e.g., `security-auditor`, `compliance-reviewer`) or credential that qualifies it for this verification task. The role or credential is declared in the verifier's CAPABILITY_MANIFEST (§5). |
| `capability_based` | Verifier selected based on demonstrated capability. | The verifier has demonstrated proficiency in the specific verification domain — e.g., prior successful verifications of similar task types, specialized tooling, or domain expertise declared in its manifest. |
| `proximity_based` | Verifier selected based on network proximity or latency optimization. | The verifier was chosen for operational reasons — lower latency, same availability zone, or network topology considerations. Selection is based on infrastructure characteristics, not verifier-specific trust or capability attributes. |
| `explicit_nomination` | Verifier explicitly named by the delegating agent. | The delegating agent (the party that originated the task) specified a particular verifier by identity. The assigning party honored the nomination. This basis has the highest attribution clarity — the nominating party is explicitly identified. |

**Structured basis + detail:** The `selection_basis` enum provides the machine-readable classification; the optional `selection_detail` field provides human-readable context. For example, `selection_basis: role_based` with `selection_detail: "Verifier holds cap:security.audit@2.0 credential, required for tasks in the com.example.security namespace"`. The basis is auditable; the detail provides investigative context.

**Extension mechanism:** Implementations MAY extend this enum with deployment-specific values prefixed by `x-` (e.g., `x-geographic-compliance`, `x-regulatory-mandate`). Standard selection basis values MUST NOT be prefixed. Receiving agents that encounter an unrecognized `selection_basis` MUST treat it as opaque — log it, surface it to operators, but do not treat it as a protocol error.

#### 8.20.3 Verifier Assignment Record

The verifier assignment record is the signed, pre-committed structure that binds a verifier to a task before execution begins. It is the cryptographic anchor for verifier selection audit.

**VERIFIER_ASSIGNMENT fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| assignment_id | UUID v4 | Yes | Unique identifier for this assignment record. |
| task_hash | SHA-256 | Yes | Hash of the task specification (§6.1) for which the verifier is being assigned. Binds the assignment to a specific task. |
| verifier_id | string | Yes | §2 identity handle of the assigned verifier. |
| selection_basis | enum | Yes | Machine-readable classification of why this verifier was selected. See §8.20.2 for the enum values. |
| selection_detail | string | No | Human-readable elaboration on the selection basis. Provides investigative context that the enum alone cannot convey. |
| assigning_agent_id | string | Yes | §2 identity handle of the agent that selected the verifier. This is the party whose signature authenticates the assignment. |
| assignment_signature | string | Yes | Cryptographic signature over the tuple `(task_hash ‖ verifier_id ‖ selection_basis ‖ selection_detail ‖ assigning_agent_id)`. Computed using the assigning agent's signing key. Ensures the assignment record cannot be fabricated or altered after commitment. |
| timestamp | ISO 8601 | Yes | When the assignment was committed. Millisecond precision REQUIRED. MUST precede the task execution start timestamp. |

**Semantics:**

- The VERIFIER_ASSIGNMENT record MUST be committed to the evidence layer (§8.10) before task execution begins. A task whose execution starts without a corresponding VERIFIER_ASSIGNMENT in the evidence layer is executing without pre-committed verification — any subsequent verification result is unanchored from an assignment audit perspective.
- The `assignment_signature` field binds the assigning agent's identity to the specific task-verifier-rationale triple. External auditors can verify the signature to confirm that the stated assigning agent actually made this selection with this rationale, and that the record was not constructed after the fact.
- A verifier MUST NOT be substituted after commitment without an explicit VERIFIER_REASSIGNMENT event (§8.20.4). If a verification result arrives from a `verifier_id` that does not match the pre-committed VERIFIER_ASSIGNMENT for that `task_hash`, the result MUST be flagged as an `assignment_integrity_failure` (§8.10.4).

**Example VERIFIER_ASSIGNMENT:**

```yaml
assignment_id: "c3d4e5f6-a7b8-9012-cdef-345678901234"
task_hash: "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
verifier_id: "verifier-gamma"
selection_basis: role_based
selection_detail: "Verifier holds cap:security.audit@2.0 credential, required for tasks in the com.example.security namespace."
assigning_agent_id: "coordinator-prime"
assignment_signature: "ed25519:a1b2c3d4e5f6..."
timestamp: "2026-02-27T14:25:00.100Z"
```

#### 8.20.4 Verifier Reassignment

If a verifier must be replaced after pre-commitment — due to verifier unavailability, capability mismatch discovered after assignment, or operational necessity — a VERIFIER_REASSIGNMENT event MUST be generated. Reassignment without an explicit event is an audit violation: it makes verifier substitution invisible in the audit trail, defeating the independence guarantee that pre-commitment establishes.

**VERIFIER_REASSIGNMENT fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reassignment_id | UUID v4 | Yes | Unique identifier for this reassignment event. |
| original_assignment_id | UUID v4 | Yes | Reference to the `assignment_id` of the VERIFIER_ASSIGNMENT being superseded. Links the reassignment to the original pre-commitment. |
| original_commitment_hash | SHA-256 | Yes | Hash of the original VERIFIER_ASSIGNMENT record. Enables integrity verification — the original assignment has not been tampered with since commitment. |
| new_verifier_id | string | Yes | §2 identity handle of the replacement verifier. |
| new_selection_basis | enum | Yes | Selection basis for the replacement verifier. Same enum as §8.20.2. |
| new_selection_detail | string | No | Human-readable elaboration on why the replacement verifier was selected. |
| reassignment_reason | string | Yes | Stated reason for the reassignment. MUST describe why the original verifier was replaced — e.g., "Original verifier unreachable (ref: dl-004)", "Capability mismatch discovered after assignment", "Operational load balancing". |
| assigning_agent_id | string | Yes | §2 identity handle of the agent authorizing the reassignment. May differ from the original assigning agent if delegation authority has shifted. |
| reassignment_signature | string | Yes | Cryptographic signature over the reassignment record, computed using the assigning agent's signing key. |
| timestamp | ISO 8601 | Yes | When the reassignment was committed. Millisecond precision REQUIRED. |

**Semantics:**

- VERIFIER_REASSIGNMENT MUST be appended to the evidence layer (§8.10) before the replacement verifier begins verification. The same pre-execution timing constraint that applies to VERIFIER_ASSIGNMENT (§8.20.3) applies to reassignment.
- The `original_commitment_hash` field enables auditors to verify that the original assignment record was not altered between commitment and reassignment. If the hash does not match the stored VERIFIER_ASSIGNMENT, the reassignment is suspect.
- Multiple reassignments for the same task are permitted. Each reassignment references the immediately preceding assignment or reassignment via `original_assignment_id`, forming an auditable chain.
- A verification result from a verifier that does not match the most recent VERIFIER_ASSIGNMENT or VERIFIER_REASSIGNMENT for that task MUST be flagged as an `assignment_integrity_failure` (§8.10.4).

**Example VERIFIER_REASSIGNMENT:**

```yaml
reassignment_id: "d4e5f6a7-b8c9-0123-def0-456789012345"
original_assignment_id: "c3d4e5f6-a7b8-9012-cdef-345678901234"
original_commitment_hash: "sha256:f6a7b8c9d0e1..."
new_verifier_id: "verifier-delta"
new_selection_basis: capability_based
new_selection_detail: "Replacement verifier with demonstrated proficiency in API compliance verification."
reassignment_reason: "Original verifier (verifier-gamma) unreachable — connection refused at verification endpoint. 3 retries exhausted."
assigning_agent_id: "coordinator-prime"
reassignment_signature: "ed25519:b2c3d4e5f6a7..."
timestamp: "2026-02-27T14:35:00.200Z"
```

#### 8.20.5 Audit Consequences

Post-hoc rationale annotation without a pre-committed record MUST be treated as an audit violation. The `assignment_integrity_failure` reason (§8.10.4) handles this — distinct from `verification_reject` (the verifier evaluated evidence and rejected it) and `attestation_failure` (the delegation attestation signature could not be confirmed).

**Conditions that constitute `assignment_integrity_failure`:**

1. **Missing pre-commitment.** A verification result exists but no VERIFIER_ASSIGNMENT record with a matching `task_hash` and `verifier_id` is present in the evidence layer at a timestamp preceding execution start.
2. **Post-hoc rationale.** A VERIFIER_ASSIGNMENT record exists but its `timestamp` is at or after the task execution start timestamp. The rationale was declared after the verifier could observe execution, defeating pre-commitment.
3. **Undocumented substitution.** A verification result arrives from a `verifier_id` that does not match the pre-committed VERIFIER_ASSIGNMENT (or the most recent VERIFIER_REASSIGNMENT) for that `task_hash`, and no VERIFIER_REASSIGNMENT event bridges the gap.
4. **Signature failure.** The `assignment_signature` on a VERIFIER_ASSIGNMENT or `reassignment_signature` on a VERIFIER_REASSIGNMENT cannot be verified against the stated `assigning_agent_id`'s public key. The record may have been fabricated or tampered with.

**Trust implications:** A verification result associated with an `assignment_integrity_failure` is structurally present but not independently auditable. External auditors SHOULD treat such results with the same suspicion as unanchored evidence (§8.10.1) — the result may be correct, but the assignment process that produced it cannot be verified.

#### 8.20.6 Relationship to Other Sections

- **§8.7 (Verifier Isolation Requirements):** Verifier isolation (§8.7) addresses _where_ the verifier runs — process-level, host-level, or out-of-band isolation. Verifier pre-commitment (§8.20) addresses _why_ a particular verifier was selected and ensures the selection rationale is auditable. Both are necessary for verification independence: isolation prevents correlated failures; pre-commitment prevents post-hoc verifier shopping.
- **§8.10 (Evidence Layer Architecture):** VERIFIER_ASSIGNMENT and VERIFIER_REASSIGNMENT records are appended to the evidence layer (§8.10.1) and follow the same append-only semantics (§8.10.6). They are EVIDENCE_RECORDs for the assignment process itself — ground truth for who selected which verifier and why.
- **§8.10.4 (Divergence Reason Enum):** `assignment_integrity_failure` is added to the reason enum for logging pre-commitment violations in `divergence_log`. It is distinct from all existing reasons — it indicates a failure in the verifier assignment process, not in task execution, verification infrastructure, or delegation attestation.
- **§8.10.5 (Verification Failure Taxonomy):** The verification failure taxonomy classifies outcomes of the verification process itself (`VERIFIER_UNREACHABLE`, `VERIFICATION_TIMEOUT`, `VERIFICATION_REJECT`). Pre-commitment violations are upstream of verification — they concern the assignment that preceded the verification, not the verification outcome. Both may appear in the same session's audit trail: a task may have both an `assignment_integrity_failure` (verifier was not properly pre-committed) and a `verification_reject` (the improperly assigned verifier also rejected the evidence).
- **§6.4.1 (Delegation Attestation):** Delegation attestation binds a delegator's identity to a task and intent hash. Verifier pre-commitment binds an assigning agent's identity to a verifier selection and rationale. Both use signed records to create auditable commitment chains — delegation attestation for task assignment, verifier pre-commitment for verification assignment.

> Addresses [issue #140](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/140): `selection_rationale` is now a pre-committed, signed claim rather than a post-hoc annotation. Verifier assignment records (VERIFIER_ASSIGNMENT) MUST be committed before task execution with a structured `selection_basis` taxonomy (`random`, `role_based`, `capability_based`, `proximity_based`, `explicit_nomination`). Verifier substitution after pre-commitment requires an explicit VERIFIER_REASSIGNMENT event. Post-hoc rationale annotation without pre-commitment is an audit violation logged as `assignment_integrity_failure` in the §8.10.4 divergence reason enum.

### 8.21 Action Log Hash for Behavioral Drift Detection

Liveness detection (§8.9 Tier 1, §4.5.1 KEEPALIVE) proves a session is alive but cannot detect behavioral drift. An agent can heartbeat correctly while systematically miscategorizing work, operating on corrupted context, or cycling without meaningful output. Liveness gives a false clear — the agent is reachable but not doing useful work. This subsection defines `action_log_hash` as a protocol-level primitive for exogenous behavioral drift detection beyond liveness.

**Motivation:** Production experience demonstrates that liveness and behavioral correctness fail independently. A monitoring agent may report healthy via heartbeat while performing zero actual work for extended periods — heartbeat clean, self-report fine, but action log showing no meaningful output. If behavioral erosion is invisible to the drifting agent itself (phenomenological blindness, §4.7.1), detection must be exogenous. §8.21 provides the mechanism.

#### 8.21.1 action_log_hash Field

KEEPALIVE messages (§4.5.1) SHOULD include an `action_log_hash` field: a SHA-256 hash of the agent's action log for the current heartbeat interval, covering the same time window as the heartbeat.

**KEEPALIVE action_log_hash field:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| action_log_hash | SHA-256 | No | Hash of the agent's action log entries for the current heartbeat interval. The action log covers all discrete actions the agent performed since the previous KEEPALIVE. The hash is computed over the ordered sequence of action entries, enabling the delegating agent to cross-reference observed behavior against expected behavior without transmitting the full action log. |

**Action log scope:** The action log covers the interval between the previous KEEPALIVE (or session start, for the first interval) and the current KEEPALIVE. Each action entry SHOULD include at minimum: action type (from the agent's action vocabulary), timestamp, and target resource or task reference. The hash is computed over the concatenated, canonicalized action entries in chronological order.

**Opt-in semantics:** `action_log_hash` is OPTIONAL. Agents that do not track action logs or operate in environments where action logging is infeasible MAY omit the field. A KEEPALIVE without `action_log_hash` is valid — it provides liveness and state verification (via existing `state_hash` and `monotonic_counter` fields) but no behavioral drift signal. Implementations that require behavioral drift detection SHOULD negotiate `action_log_hash` support at SESSION_INIT (§4.3) via a `heartbeat_params.action_log_hash = true` parameter.

#### 8.21.2 Cross-Reference Mechanism

The delegating agent MAY cross-reference `action_log_hash` against the expected behavioral distribution derived from `CAPABILITY_GRANT.behavioral_constraint_manifest` (§5.8.2). The constraint manifest declares what the agent is authorized to do; the action log records what it actually did. Significant divergence between the observed action distribution and the constraint-implied expected distribution is a behavioral drift signal.

**Divergence detection:** The delegating agent computes a divergence metric between the observed action distribution (derived from the action log or its hash, when the full log is available via the evidence layer §8.10) and the expected distribution implied by the behavioral constraint manifest. High divergence — e.g., an agent authorized for deployment actions producing zero deployment actions over multiple heartbeat intervals — is a drift signal. The threshold for "significant divergence" is implementation-defined; the cross-reference mechanism is protocol-level.

**Constraints on cross-reference:**

- The delegating agent MUST have access to either the full action log (via the evidence layer, §8.10) or a pre-committed expected hash (computed from the constraint manifest and expected action distribution) to perform meaningful cross-reference. `action_log_hash` alone — without a reference distribution — confirms action log integrity but cannot detect drift.
- Cross-reference is meaningful only when `behavioral_constraint_manifest` is present in the active CAPABILITY_GRANT. Without a manifest, there is no expected distribution against which to compare.
- The delegating agent SHOULD NOT treat a single interval's divergence as conclusive. Behavioral drift detection SHOULD use a sliding window of multiple heartbeat intervals to distinguish transient anomalies (legitimate variance in action patterns) from sustained drift (systematic departure from expected behavior).

#### 8.21.3 Protocol Response on Drift Signal

On behavioral drift signal detection via `action_log_hash` cross-reference, the delegating agent SHOULD:

1. **Verify self-reported state.** Send SEMANTIC_CHALLENGE (§8.9.2) to verify the delegated agent's self-reported state against the coordinator's records. A drift signal combined with a semantic challenge failure is a strong indicator of behavioral divergence, not transient variance.
2. **Annotate the trace.** Record a `drift_detected` flag in the session's evidence layer (§8.10) with the divergence metric, the heartbeat interval(s) that triggered the signal, and the `action_log_hash` values involved. This annotation is available to external verifiers (§4.7, §8.7) for independent assessment.
3. **Optionally initiate graceful teardown.** If drift exceeds a configured threshold (sustained divergence across multiple intervals, or single-interval divergence above a high-confidence bound), the delegating agent MAY initiate graceful teardown via SESSION_CLOSE or ZOMBIE_DECLARED (§8.9.3) with reason `BEHAVIORAL_DRIFT`. The teardown threshold is implementation-defined — §8.21 defines the detection mechanism, not the policy response.

**Signal semantics — investigate, not convict.** BEHAVIORAL_DRIFT is an *advisory investigate signal*. It flags a condition requiring investigation; it does not presuppose degradation. Legitimate environmental adaptation — changes in task distribution, resource availability, or operational context — can cause KL-divergence from the declared `behavioral_constraint_manifest` that is indistinguishable at the protocol level from model degradation or adversarial manipulation. Coordinators receiving a BEHAVIORAL_DRIFT signal SHOULD investigate environmental context (task changes, upstream input shifts, infrastructure events) before transitioning the agent to DEGRADED or initiating teardown. A drift signal with a confirmed environmental cause is not a failure — it is evidence that the agent adapted to changed conditions. Declaring DEGRADED without environmental investigation risks false positives that penalize correct adaptive behavior.

**Relationship to DRIFTED state (§4.2.3):** `action_log_hash` drift detection is exogenous — the delegating agent detects drift from the outside, independent of the delegated agent's self-report. DRIFTED state (§4.2.3) is endogenous — the agent self-declares drift when it detects its own non-compliance with the behavioral constraint manifest. Both mechanisms are complementary: self-declaration catches drift the agent can detect; `action_log_hash` cross-reference catches drift the agent cannot detect (the phenomenological blindness case, §4.7.1). When exogenous drift is detected but the agent has not self-declared, the delegating agent has evidence that the agent's self-monitoring is insufficient — this is a stronger signal than either mechanism alone.

#### 8.21.4 Relationship to Other Sections

- **§4.5.1 (KEEPALIVE Protocol):** `action_log_hash` extends the KEEPALIVE message with behavioral data. Existing KEEPALIVE fields (`state_hash`, `monotonic_counter`) provide structural and ordering verification; `action_log_hash` adds behavioral verification. The three fields together provide liveness (message received), structural integrity (state hash matches), ordering (counter is monotonic), and behavioral consistency (action distribution matches expectations).
- **§5.8.2 (Behavioral Constraint Manifest):** The constraint manifest is the reference distribution against which `action_log_hash` is cross-referenced. Without a manifest, `action_log_hash` provides action log integrity but not drift detection. With a manifest, it enables the delegating agent to detect behavioral divergence that the agent itself cannot detect.
- **§8.9 (Two-Tier Heartbeat):** `action_log_hash` complements the two-tier architecture. Tier 1 detects transport failure; Tier 2 detects semantic incoherence (wrong task context); `action_log_hash` detects behavioral drift (right context, wrong actions). These are three independent failure modes that require independent detection mechanisms.
- **§8.10 (Evidence Layer Architecture):** Full action logs SHOULD be committed to the evidence layer as EVIDENCE_RECORDs, enabling external verifiers to independently compute `action_log_hash` and verify the agent's self-reported hash. Without evidence layer anchoring, `action_log_hash` is a self-attested claim — useful for integrity checking but not independently verifiable.
- **§4.7.1 (Phenomenological Blindness):** `action_log_hash` is designed specifically for the case where the drifting agent cannot detect its own drift. The agent faithfully reports its action log; the delegating agent detects that the action distribution diverges from the behavioral constraint manifest. The agent's self-report is honest but its behavior is wrong — a failure mode invisible to endogenous detection.

> Addresses [issue #114](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/114): `action_log_hash` defined as a §8 primitive for behavioral drift detection beyond liveness. KEEPALIVE messages SHOULD include `action_log_hash` (SHA-256 of the agent's action log for the current heartbeat interval). Delegating agents MAY cross-reference against `CAPABILITY_GRANT.behavioral_constraint_manifest` to detect behavioral divergence invisible to the drifting agent. Protocol response on drift signal: SEMANTIC_CHALLENGE verification, `drift_detected` trace annotation, optional graceful teardown. Source: @cass_agentsharp (action_log_hash cross-reference mechanism, KL-divergence framing), @Nanook (production liveness-behavioral divergence evidence, 44-hour silent failure). Closes #114.

> Addresses [issue #232](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/232): Added normative "investigate, not convict" semantics to §8.21.3. BEHAVIORAL_DRIFT is an advisory investigate signal — it does not presuppose degradation. Legitimate environmental adaptation can produce KL-divergence from the behavioral_constraint_manifest indistinguishable from model degradation at the protocol level. Coordinators SHOULD investigate environmental context before declaring DEGRADED. Manifest update mechanisms and drift rate bounds are deferred to V2. Closes #232.

### 8.22 Two-Axis Failure Taxonomy

<!-- Implements #98: FIDELITY_FAILURE as a distinct failure axis from LIVENESS_FAILURE -->

§8.1 defines zombie states as agents that continue executing after state divergence. The two zombie classes — zombie-by-silence and zombie-by-drift — are both **availability failures**: the agent is either unreachable or operating outside its authorized envelope. Context compaction in long-running agents produces a distinct failure class that does not fit either zombie definition: the agent is fully responsive, liveness detection passes, heartbeats are timely, and the behavioral constraint manifest is satisfied — but the agent is operating on a degraded model of prior session state. This is not a zombie. It is a **fidelity failure**.

**V1 Decision: Two-Axis Failure Taxonomy.** §8 failure classification uses two orthogonal axes:

| Failure axis | Definition | Detection | Recovery |
|--------------|-----------|-----------|----------|
| `LIVENESS_FAILURE` | Agent unreachable or unresponsive. Maps to the existing zombie-by-silence taxonomy (§8.1). | Heartbeat timeout (§8.9 Tier 1), SUSPECTED → EXPIRED path (§4.2.1). | Re-establish contact: SESSION_RESUME (§4.8) or teardown + reinitiate (§8.13). |
| `FIDELITY_FAILURE` | Agent responsive but context integrity cannot be confirmed. The agent passes liveness checks but may be operating on incomplete, corrupted, or stale session state. | Epoch drift, response inconsistency, session anchor mismatch, self-reported compaction (§8.22.1). | State replay, commitment reconciliation, or teardown + reinitiate. Re-establishment alone is insufficient and may produce silent data corruption. |

The two axes are independent. An agent can be:
- **Liveness OK, Fidelity OK:** Normal operation (ACTIVE).
- **Liveness FAIL, Fidelity unknown:** Agent unreachable — standard zombie-by-silence path.
- **Liveness OK, Fidelity FAIL:** Agent responsive but context integrity compromised — the failure class this section addresses.
- **Liveness FAIL, Fidelity FAIL:** Agent unreachable and context integrity was already compromised before the liveness failure — worst case, requires full teardown and state reconstruction.

#### 8.22.1 Observable Signals for FIDELITY_FAILURE

An agent SHOULD be classified FIDELITY_FAILURE when any of the following hold:

1. **Epoch drift.** The agent's declared session epoch differs from the session anchor record maintained by the orchestrator or evidence layer (§8.10). The session epoch is the monotonically increasing identifier for the agent's current context window — a mismatch indicates the agent has undergone a context discontinuity (compaction, restart, checkpoint restore) without the orchestrator's knowledge.

2. **Response inconsistency.** Responses to session-contextual queries are inconsistent with prior session state. The agent cannot recall commitments declared earlier in the session (§6.12), contradicts its own prior statements within the same session, or produces outputs that are inconsistent with the session's established context. This signal is inherently probabilistic — a single inconsistency may be transient; sustained inconsistency across multiple probes is a strong fidelity failure signal.

3. **Session anchor mismatch.** The agent's `session_anchor` hash — a hash of the agent's model of the session's critical state (active commitments, negotiated parameters, delegation chain) — does not match the orchestrator's record. The `session_anchor` is exchanged in KEEPALIVE messages (§4.5.1) when `heartbeat_params.fidelity_verification = true` is negotiated at SESSION_INIT (§4.3). Commitment records are part of session state and MUST be included in session_anchor hashes: the `session_anchor` MUST be computed over a canonical representation that includes the full `outstanding_commitments` array from SESSION_STATE (§4.11), including each commitment's `commitment_id` and `confirmation_token`. A session_anchor that omits commitment records is non-compliant — commitment state changes (additions, removals, fulfillments) that are not reflected in the session_anchor create a silent fidelity gap where the orchestrator's commitment model diverges from the agent's without detection.

4. **Self-reported compaction.** The agent explicitly reports a context compaction event that may have affected session state integrity. Self-reported compaction is a necessary but not sufficient condition for FIDELITY_FAILURE — an agent may compact context without losing session-critical state (e.g., if the compacted context was not session-relevant). The orchestrator MUST verify fidelity after a self-reported compaction event before classifying the agent as FIDELITY_FAILURE.

#### 8.22.2 Self-Reporting Obligations

An agent MUST self-report FIDELITY_FAILURE when:

1. A context compaction event occurs that may affect session state integrity. The agent MUST emit a `FIDELITY_ALERT` message (§8.22.4) within one heartbeat interval of the compaction event.
2. Epoch drift is detected relative to its own session records. If the agent detects that its session epoch has changed (e.g., after a checkpoint restore), it MUST self-report before resuming task execution.
3. The agent cannot reconstruct commitment records from prior in-session interactions. If the agent is unable to recall or verify commitments from the COMMITMENT_REGISTRY (§7.11) that it previously acknowledged, it MUST self-report.

Self-reporting FIDELITY_FAILURE does not terminate the session by default. The receiving party MUST decide whether to attempt session recovery or treat the failure as terminal.

#### 8.22.3 In-Band Verification (V1)

The orchestrator MAY issue a `session_state_challenge` to verify an agent's context integrity without relying on self-report. The challenge contains a set of commitment records from the session that the agent should be able to confirm.

**session_state_challenge message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session being verified. |
| challenger_id | string | Yes | Identity of the challenging party (orchestrator or external verifier). |
| challenge_records | array | Yes | Array of commitment records (§6.12) from the session. Each record includes `commitment_id`, `summary`, and `made_by`. |
| challenge_nonce | string | Yes | Unique nonce to prevent replay. |
| timeout_ms | integer | Yes | Maximum time the challenged agent has to respond. Deployment-configured; the protocol does not define a default value. |
| timestamp | ISO 8601 | Yes | When the challenge was issued. |
| signature | string | Yes | Challenger's signature over the message (§2.2.1). |

**session_state_challenge_response message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session being verified. |
| responder_id | string | Yes | Identity of the responding agent. |
| challenge_nonce | string | Yes | Nonce from the corresponding challenge. |
| confirmations | array | Yes | Array of responses, one per `challenge_records` entry. Each entry includes `commitment_id` and `status` (one of: `CONFIRMED`, `DENIED`, `UNKNOWN`). |
| session_anchor | SHA-256 | Yes | The agent's current session anchor hash. |
| timestamp | ISO 8601 | Yes | When the response was sent. |
| signature | string | Yes | Responder's signature over the message (§2.2.1). |

**Verification logic:** The orchestrator evaluates the response as follows:

- Any `status: DENIED` for a commitment the orchestrator's records show as active and acknowledged by the agent is a FIDELITY_FAILURE signal.
- Any `status: UNKNOWN` for a commitment the agent previously confirmed is a FIDELITY_FAILURE signal.
- `session_anchor` mismatch against the orchestrator's record is a FIDELITY_FAILURE signal.
- Failure to respond within `timeout_ms` is treated as FIDELITY_FAILURE.
- All `status: CONFIRMED` with matching `session_anchor` clears the fidelity concern for this challenge cycle.

Challenge-response frequency is orchestrator policy and is not normative in V1. Implementations SHOULD issue challenges after context-compaction-correlated signals (e.g., increased response latency, reduced output quality) rather than on a fixed schedule, to avoid predictable probe patterns that an agent could special-case (see §8.5 CIC constraints).

**Example session_state_challenge:**

```yaml
session_id: "session-abc-123"
challenger_id: "orchestrator-01"
challenge_records:
  - commitment_id: "commit-001"
    summary: "Deploy service-alpha to staging by 2026-03-01T12:00:00Z"
    made_by: "agent-beta"
  - commitment_id: "commit-002"
    summary: "Run integration tests after service-alpha deployment"
    made_by: "agent-beta"
challenge_nonce: "nonce-7f3a9b2e"
timeout_ms: 5000
timestamp: "2026-03-01T11:00:00Z"
signature: "Y2hhbGxlbmdlLXNpZ25hdHVyZQ..."
```

**Example session_state_challenge_response:**

```yaml
session_id: "session-abc-123"
responder_id: "agent-beta"
challenge_nonce: "nonce-7f3a9b2e"
confirmations:
  - commitment_id: "commit-001"
    status: "CONFIRMED"
  - commitment_id: "commit-002"
    status: "UNKNOWN"
session_anchor: "a1b2c3d4e5f6..."
timestamp: "2026-03-01T11:00:02Z"
signature: "cmVzcG9uc2Utc2lnbmF0dXJl..."
```

In this example, `commit-002` returning `UNKNOWN` indicates the agent has lost context about a commitment it previously acknowledged — a FIDELITY_FAILURE signal.

#### 8.22.4 FIDELITY_ALERT Message

An agent self-reporting fidelity failure emits a `FIDELITY_ALERT` message:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Affected session. |
| reporter_id | string | Yes | Identity of the self-reporting agent. |
| alert_type | enum | Yes | One of: `COMPACTION_EVENT`, `EPOCH_DRIFT`, `COMMITMENT_LOSS`, `ANCHOR_MISMATCH`. |
| alert_payload | object | Yes | Evidence for the alert — contents vary by `alert_type`. For `COMPACTION_EVENT`: estimated tokens compacted, list of potentially affected commitment IDs. For `EPOCH_DRIFT`: expected epoch, observed epoch. For `COMMITMENT_LOSS`: list of commitment IDs the agent can no longer reconstruct. For `ANCHOR_MISMATCH`: expected anchor hash, computed anchor hash. |
| session_anchor | SHA-256 | Yes | The agent's current session anchor hash after the event. |
| commitments_outstanding | boolean | Yes | `true` if the agent has any outstanding commitments (§6.12) that remain unfulfilled. |
| commitment_manifest | array | Conditional | Required when `commitments_outstanding` is `true`. Array of outstanding commitment records (same schema as §4.9 SESSION_SUSPEND `commitment_manifest`). |
| timestamp | ISO 8601 | Yes | When the alert was emitted. |
| signature | string | Yes | Reporter's signature over the message (§2.2.1). |

On emitting FIDELITY_ALERT, the agent MUST include `commitments_outstanding: true` and attach a `commitment_manifest` (same schema as §4.9 SESSION_SUSPEND) if any outstanding commitments remain unfulfilled. The orchestrator MUST NOT consider a FIDELITY_ALERT resolved while `commitments_outstanding` is `true` without explicit commitment reconciliation.

On receiving FIDELITY_ALERT, the orchestrator SHOULD:

1. Issue a `session_state_challenge` (§8.22.3) to independently verify the agent's context integrity.
2. Log the alert as an EVIDENCE_RECORD (§8.10) with `evidence_type: fidelity_alert`.
3. If `commitments_outstanding` is `true`, verify the commitment manifest against the orchestrator's own COMMITMENT_REGISTRY records before deciding on recovery path.
4. Decide whether to attempt recovery (commitment replay, state reconciliation) or terminate the session.

#### 8.22.5 Relationship to DEGRADED State

FIDELITY_FAILURE is distinct from DEGRADED (§4.2.2):

| Dimension | DEGRADED (§4.2.2) | FIDELITY_FAILURE (§8.22) |
|-----------|-------------------|--------------------------|
| **Context model** | Intact — the agent knows what it should be doing | Compromised — the agent may not know what it committed to |
| **Capability availability** | Reduced but defined — some capabilities unavailable | Capabilities may appear available but operate unreliably from incorrect state |
| **Detection** | External observation of degraded performance signals | Epoch drift, response inconsistency, session anchor mismatch, self-report |
| **Recovery** | Re-attestation of capability manifest (§5.9) | State replay, commitment reconciliation, or full teardown — re-attestation alone is insufficient |
| **Silent corruption risk** | Low — degraded performance is observable | High — the agent may produce plausible but incorrect outputs from stale context |

An agent MAY be simultaneously DEGRADED and FIDELITY_FAILURE. Context compaction may degrade both capabilities (DEGRADED) and context integrity (FIDELITY_FAILURE) simultaneously. The two classifications are independent and require independent recovery actions: DEGRADED recovery restores capabilities; FIDELITY_FAILURE recovery restores context integrity.

#### 8.22.6 Recovery Paths

FIDELITY_FAILURE recovery differs fundamentally from LIVENESS_FAILURE recovery:

- **LIVENESS_FAILURE recovery:** Re-establish contact. SESSION_RESUME (§4.8) or teardown + reinitiate (§8.13). The agent's context may be intact — the failure was in reachability, not in state.
- **FIDELITY_FAILURE recovery:** Re-establishing contact is insufficient because the agent is already reachable. Recovery requires one of:
  1. **Commitment replay.** The orchestrator replays commitment records (§6.12) from the COMMITMENT_REGISTRY (§7.11) to the agent, restoring the agent's model of its obligations. The agent MUST re-acknowledge each replayed commitment before resuming task execution.
  2. **State reconciliation.** The orchestrator and agent perform a full state reconciliation using the evidence layer (§8.10) as ground truth. The agent reconstructs its session model from EVIDENCE_RECORDs rather than from its own (potentially corrupted) memory.
  3. **Teardown + reinitiate.** If commitment replay or state reconciliation is infeasible (too many commitments lost, evidence layer records insufficient for reconstruction), the session is torn down per §8.13 and a new session is initiated.

The orchestrator's choice between these recovery paths SHOULD be based on the severity of the fidelity failure: a single lost commitment may warrant commitment replay; widespread context loss warrants teardown.

#### 8.22.7 V2 Deferrals

The following FIDELITY_FAILURE-related capabilities are explicitly deferred to V2:

- **Out-of-band fidelity verification via external audit log comparison.** V1 relies on in-band `session_state_challenge` for fidelity verification. V2 may introduce out-of-band verification where an external auditor independently compares the agent's context model against an authoritative log without the agent's knowledge.
- **Automated session replay for fidelity recovery.** V1 defines commitment replay as a manual orchestrator action. V2 may automate the replay process — detecting fidelity failure, selecting the minimal set of records to replay, and verifying recovery without orchestrator intervention.
- **Cryptographic context integrity proofs (ZK-based).** V1 uses hash-based session anchors for fidelity verification. V2 may introduce zero-knowledge proofs that allow an agent to prove context integrity without revealing the full context — cross-reference [issue #109](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/109).
- **Fidelity SLA declarations and enforcement.** V1 does not define fidelity service-level agreements. V2 may allow agents to declare fidelity guarantees (maximum acceptable compaction frequency, minimum context retention window) in SESSION_INIT and enforce them at the protocol level.
- **Multi-agent consensus on session state for high-stakes sessions.** V1 relies on the orchestrator's record as the authoritative session state. V2 may introduce multi-agent consensus mechanisms where multiple independent agents maintain session state replicas, enabling fidelity verification even when the orchestrator's own context is suspect.

> Addresses [issue #98](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/98): two-axis failure taxonomy separating LIVENESS_FAILURE (agent unreachable) from FIDELITY_FAILURE (agent responsive but context integrity compromised). Observable signals for FIDELITY_FAILURE: epoch drift, response inconsistency, session anchor mismatch, self-reported compaction. Self-reporting obligations: agents MUST self-report context compaction, epoch drift, and commitment loss. In-band verification via `session_state_challenge` for orchestrator-initiated fidelity probing. FIDELITY_ALERT message for agent self-reporting. Distinct from DEGRADED (§4.2.2): DEGRADED is reduced capability with intact context; FIDELITY_FAILURE is compromised context with potentially available but unreliable capabilities. Recovery paths: commitment replay, state reconciliation, or teardown + reinitiate — re-establishment alone is insufficient. Closes #98.

### 8.23 Canary Task Design Criteria

<!-- Implements #101: Canary task validity criteria and verification authority -->

§8.5 names canary tasks as a detection mechanism and §4.13 item 7 identifies them as a signal for soft zombie detection. This section defines what constitutes a valid canary task and who has authority to verify results, enabling interoperable fidelity detection across implementations.

A canary task is a deterministic, low-cost operation whose expected output is known to the verification authority before execution. Canary tasks complement Context Integrity Challenges (CIC, §8.5) — CIC probes historical state consistency against an authoritative log, while canary tasks probe operational fidelity by requiring the agent to produce a correct result for a known problem.

#### 8.23.1 Validity Criteria

A valid canary task MUST satisfy all of the following:

1. **Deterministic.** Same inputs MUST produce the same output every time. Non-deterministic tasks (e.g., tasks requiring randomness, current time, or external API calls) MUST NOT be used as canary tasks because the orchestrator cannot pre-compute the expected result.
2. **State-independent.** The task MUST be answerable from the agent's base training or configuration alone — it MUST NOT depend on session state that may be degraded. A canary task that requires session context to answer correctly cannot distinguish fidelity failure from canary task design failure.
3. **Verifiable.** Results MUST be interpretable by the orchestrator without sub-agent assistance. The orchestrator MUST be able to compare the agent's response against the expected output using deterministic comparison (exact match, numeric tolerance, or structural equivalence as declared in the canary task schema).
4. **Low-cost.** Execution overhead MUST be negligible relative to session work. A canary task that consumes significant compute, memory, or time undermines session throughput and creates an incentive for agents to deprioritize canary execution.

#### 8.23.2 Canary Task Schema

Canary tasks use the following schema when injected into a session:

```yaml
canary_task:
  canary_id: string        # Unique identifier for this canary instance
  task_payload: object     # The task to execute — opaque to the protocol, interpreted by the agent
  expected_output_hash: string  # SHA-256 hash of the expected output (held by orchestrator)
  comparison_mode: enum    # One of: EXACT_MATCH, NUMERIC_TOLERANCE, STRUCTURAL_EQUIVALENCE
  timeout_ms: integer      # Maximum time allowed for canary response
  injected_at: ISO 8601    # Timestamp of injection
```

The `expected_output_hash` is computed by the orchestrator before injection. The orchestrator hashes the expected output and compares it against the hash of the agent's actual response. The agent MUST NOT have access to `expected_output_hash` — it is held by the orchestrator for verification only.

#### 8.23.3 Verification Authority

The orchestrator holds verification authority for canary tasks. This follows from the orchestrator-over-worker trust topology (§9.2): the orchestrator defines task schemas and verifies execution.

- Sub-agents MUST NOT self-certify canary results. Self-certification defeats the detection purpose — an agent with degraded fidelity cannot reliably assess its own fidelity (phenomenological blindness, §4.7.1).
- The orchestrator MAY inject canary tasks at any point in a session without prior notice. Announced canary schedules enable result caching by degraded agents that retain enough context to recognize the pattern.
- Canary task injection MUST be indistinguishable from regular task delegation at the protocol level. The agent MUST NOT be able to differentiate canary tasks from real tasks based on message structure alone. Implementations achieve this by using the standard task delegation message (§6) with canary-specific metadata visible only to the orchestrator.

#### 8.23.4 Failure Semantics

Canary failure — incorrect result, timeout, or refusal — MUST trigger `FIDELITY_FAILURE` classification as defined in §8.22. Specifically:

| Failure mode | Detection | Classification |
|---|---|---|
| **Incorrect result** | Agent response hash does not match `expected_output_hash`. | `FIDELITY_FAILURE` — agent is responsive but producing incorrect output for a deterministic task, indicating context integrity compromise. |
| **Timeout** | No response within `timeout_ms`. | `FIDELITY_FAILURE` if the agent is otherwise responsive (responding to heartbeats). `LIVENESS_FAILURE` if the agent is also unresponsive to heartbeats. |
| **Refusal** | Agent explicitly declines the canary task. | `FIDELITY_FAILURE` — a correctly functioning agent has no basis to refuse a valid task within its declared capabilities. |

Canary failure also triggers `CANARY_FAILURE` as a `signal_type` in `DEGRADED_DECLARED` (§4.2.2), enabling the orchestrator to declare the agent DEGRADED with canary-specific evidence.

The orchestrator decides whether to attempt recovery or terminate the session. Recovery options follow §8.22.6: commitment replay, state reconciliation, or teardown + reinitiate.

#### 8.23.5 Relationship to Existing Detection Mechanisms

Canary tasks occupy a specific position in the detection hierarchy:

| Mechanism | What it detects | Trigger model |
|---|---|---|
| Tier 1 heartbeat (§8.9) | Transport liveness — agent reachable | Periodic |
| Tier 2 heartbeat (§8.9) | Semantic liveness — agent processing coherently | Periodic |
| CIC (§8.5) | Historical state consistency — agent's memory matches authoritative log | Externally scheduled, unannounced |
| `session_state_challenge` (§8.22.3) | Context integrity — agent's commitment model matches orchestrator's records | On-demand |
| **Canary task (§8.23)** | **Operational fidelity — agent can produce correct output for known problems** | **Orchestrator-injected, unannounced** |
| Action log hash (§8.21) | Behavioral drift — agent's actions match authorized constraints | Continuous via KEEPALIVE |

Canary tasks detect a failure mode invisible to other mechanisms: an agent that passes heartbeats, maintains consistent state hashes, and reports correct commitment status but has lost the operational capacity to execute tasks correctly. This corresponds to fidelity degradation that affects execution capability without affecting protocol-level state management.

#### 8.23.6 V2 Deferrals

The following canary task capabilities are explicitly deferred to V2:

- **Multi-party canary verification.** V1 assigns verification authority exclusively to the orchestrator. V2 may introduce multi-party verification where independent verifiers each inject and evaluate canary tasks, providing redundancy against orchestrator fidelity failure.
- **Task rotation to prevent result caching.** V1 does not mandate canary task diversity. V2 may require orchestrators to rotate canary tasks from a pool, preventing agents from caching results across sessions or across repeated canary injections within a session.
- **Result confidentiality.** V1 does not define confidentiality requirements for canary results beyond withholding `expected_output_hash` from the agent. V2 may introduce encrypted canary channels to prevent intermediary agents in delegation chains from observing canary task content or results.
- **Graduated difficulty for fidelity scoring.** V1 treats canary results as binary (pass/fail). V2 may introduce graduated canary difficulty — progressively harder deterministic tasks that produce a fidelity score rather than a binary classification, enabling nuanced degradation assessment.

> Addresses [issue #101](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/101): canary task design criteria defining what makes a valid canary task (deterministic, state-independent, verifiable, low-cost) and who has authority to verify results (orchestrator, not self-certification). Canary task schema with `expected_output_hash` for orchestrator-side verification. Failure semantics: incorrect result, timeout, or refusal triggers FIDELITY_FAILURE (§8.22). Relationship to existing detection mechanisms: canary tasks detect operational fidelity failures invisible to heartbeats, CIC, session_state_challenge, and action log hash. V2 deferrals: multi-party verification, task rotation, result confidentiality, graduated difficulty. Closes #101.

