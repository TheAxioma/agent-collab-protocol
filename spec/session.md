<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

## 4. Session Lifecycle

A session is the bounded collaboration context within which agents exchange capabilities, delegate tasks, and report progress. Without a defined lifecycle, sessions have no start, no end, and no recovery semantics — agents cannot distinguish an active collaboration from an abandoned one, and monitoring infrastructure has no state machine against which to detect anomalies.

This section defines the session state machine, the messages that drive transitions, and the operational requirements for keepalive, compaction, and external monitoring.

### 4.1 V1 Scope Declaration

§4 specifies **session lifecycle for bilateral sessions** — two agents collaborating within a single session context. Multi-agent sessions (three or more agents in the same session) and mixed-role sessions (where both agents can independently delegate to third parties within the same session) are explicitly **out of scope for V1**.

V1 sessions have exactly two participants: a **coordinator** (the agent that initiates the session and holds delegation authority) and a **worker** (the agent that receives task assignments within the session). Role is declared at SESSION_INIT, not discovered through negotiation. This is a deliberate simplification — see §4.4 for rationale and the V2 path for emergent roles.

The **metaphysical continuity question** — whether an agent that has undergone context compaction is the "same" session participant — is out of scope for V1 (see §2.1). V1 treats compaction as a session state transition (§4.2 COMPACTED state) with defined recovery semantics, without making claims about agent identity continuity across the compaction boundary.

### 4.2 Session State Machine

A session occupies exactly one of eleven states at any point in time:

| State | Description |
|-------|-------------|
| IDLE | Session has been allocated but no SESSION_INIT has been sent. |
| NEGOTIATING | SESSION_INIT sent; capability manifest exchange (§5.9) in progress. |
| ACTIVE | Capability exchange complete; task delegation (§6) is permitted. |
| DEGRADED | Session is operational but experiencing gradual capability loss — sustained latency increase, partial capability loss, or application-level quality degradation signals. Distinct from SUSPECTED (liveness-ambiguity path): DEGRADED is the capability-degradation path — the counterparty is alive and responsive but operating below expected capacity. Task delegation is permitted with reduced expectations. See §4.2.2 for entry conditions, caller options, and recovery semantics. |
| SUSPECTED | Heartbeat gap threshold exceeded but `session_expiry_ms` has not yet elapsed. The counterparty's liveness is ambiguous — it may be slow, overloaded, experiencing transient network issues, or genuinely failed. The local agent MUST NOT delegate new tasks but MUST continue buffering work-in-progress and MUST NOT tear down the session. See §4.2.1 for graduated suspicion semantics. |
| SUSPENDED | Session intentionally paused by either participant. No tasks may be delegated or executed. In-flight tasks SHOULD be checkpointed (§6.6 TASK_CHECKPOINT) before transition. |
| COMPACTED | Context limit hit mid-session — one or both agents have undergone context compaction. Distinct from SUSPENDED: COMPACTED is not intentional. Load-bearing session state may have been lost. Recovery requires the SESSION_RESUME protocol (§4.8). |
| EXPIRED | Session heartbeat deadline exceeded. The local agent has not received a HEARTBEAT (§4.5.3) within the negotiated `session_expiry_ms` window. Terminal state — no further task delegation is valid. In-flight subtasks MUST receive explicit TASK_CANCEL (§6.6) before teardown. Partial result recovery, if desired, MUST use SESSION_RESUME mechanics (§4.8). |
| DRIFTED | Agent B has self-declared behavioral divergence from its authorized constraint manifest (§5.8.2). B is reachable and responsive but is operating outside the behavioral envelope A authorized at session init. Distinct from ZOMBIE (liveness-failure path — silence-based, §8.1) and REVOKED (adversarial-behavior path — §8.15): DRIFTED is the behavioral-divergence path — B is cooperative and self-reporting, not silent or hostile. Non-terminal: A may re-negotiate constraints, terminate, or invoke revocation. See §4.2.3 for entry conditions, caller options, and recovery semantics. |
| REVOKED | Adversarial behavior detected — the counterparty is alive and actively working against the protocol. Distinct from SUSPECTED/ZOMBIE (liveness-failure path): REVOKED is the adversarial-behavior path. Terminal state — no resume path. On entry: propagate revocation signal upstream, log detection evidence. See §8.15 for formal definition, §8.16 for detection signal taxonomy, §8.17 for revocation propagation, and §8.18 for per-hop attestation. |
| CLOSED | Session terminated. No further messages are valid. Terminal state. |

**Why EXPIRED is a distinct terminal state (not just CLOSED):**

CLOSED is a cooperative termination — at least one participant sends SESSION_CLOSE and both sides agree the session is over. EXPIRED is a unilateral determination — one side has stopped hearing from the other and declares the session dead locally. The distinction matters for subtask cleanup: CLOSED assumes both sides can coordinate final state; EXPIRED assumes the counterparty may be unreachable, so in-flight subtasks MUST receive explicit TASK_CANCEL (§6.6) to prevent phantom completions. EXPIRED is also distinct from COMPACTED: compaction is a state-loss event with a recovery path (SESSION_RESUME); expiry is a liveness-loss event where the counterparty may be permanently gone.

**Expiry detection is local and independent.** Each side monitors HEARTBEAT arrivals against its own `session_expiry_ms` clock. No coordination is required to enter EXPIRED — if agent A's timer fires, A transitions to EXPIRED regardless of B's state. B may still consider the session ACTIVE (if B's heartbeats are being sent but not received). This asymmetry is by design: expiry is a local safety decision, not a consensus protocol.

**Recovery from EXPIRED.** EXPIRED is terminal for the current session state, but partial result recovery is possible via SESSION_RESUME (§4.8) with `recovery_reason: timeout`. The resuming agent presents its state hash and lease epoch; if the counterparty is still available and state is reconcilable, the session can transition back to ACTIVE. This reuses the same state-hash negotiation as crash recovery and manual resumption — the unified recovery mechanism (§4.8.1) handles all three cases identically. If state cannot be reconciled, the session remains EXPIRED and a new session (SESSION_INIT) is required.

**Why SUSPENDED and COMPACTED are distinct states:**

SUSPENDED is an intentional pause — the suspending agent's state is intact, the decision was deliberate, and resumption is expected to succeed without state reconciliation. COMPACTED is a context-limit event — the compacted agent's state may be incomplete, the event was not deliberate, and resumption requires state verification before work can continue. Collapsing them into a single state forces the same recovery protocol for both, which is either too heavy for SUSPENDED (unnecessary state hash verification for an intentional pause) or too light for COMPACTED (resuming without verification after potential state loss).

**State transitions and guard conditions:**

```
IDLE → NEGOTIATING
  Guard: SESSION_INIT sent by coordinator

NEGOTIATING → ACTIVE
  Guard: Both agents have exchanged CAPABILITY_MANIFEST (§5.9)
         AND protocol version is compatible (§10.3)
         AND identity is verified (§2.3.2)

NEGOTIATING → CLOSED
  Guard: PROTOCOL_MISMATCH or SCHEMA_MISMATCH (§10.4)
         OR identity verification failure
         OR SESSION_INIT timeout (no response within deployment-configured window)

ACTIVE → SUSPENDED
  Guard: SESSION_SUSPEND sent by either participant
         AND all in-flight tasks checkpointed or acknowledged

ACTIVE → COMPACTED
  Guard: Context compaction detected by either agent
         (external verifier or self-report — §4.7 defines detection)

ACTIVE → SUSPECTED
  Guard: No HEARTBEAT received within suspected_threshold_ms (configurable)
         AND session_expiry_ms has NOT yet elapsed
         Detection is local — each side evaluates independently
         suspected_threshold_ms is negotiated in SESSION_INIT / SESSION_INIT_ACK
         (see §4.2.1 and issue #71 for heartbeat negotiation)
         OR current_task_hash mismatch detected in received HEARTBEAT
            (requires heartbeat_params.task_hash_verification = true, §4.3.1)
         OR counterparty reports app_status: SUSPECTED in HEARTBEAT
            (requires heartbeat_params.application_liveness = true, §4.3.1)

SUSPECTED → ACTIVE
  Guard: HEARTBEAT or HEARTBEAT_PONG received from counterparty
         (proof of liveness — any valid heartbeat signal)
         Buffered work resumes normal processing

SUSPECTED → EXPIRED
  Guard: session_expiry_ms elapsed since last received HEARTBEAT
         without any intervening HEARTBEAT from counterparty
         In-flight subtasks MUST receive TASK_CANCEL (§6.6) before teardown
         (Same terminal semantics as direct ACTIVE → EXPIRED)

ACTIVE → EXPIRED
  Guard: No HEARTBEAT received within session_expiry_ms (§4.5.3)
         AND suspected_threshold_ms is not configured (legacy behavior)
         Detection is local — each side evaluates independently
         In-flight subtasks MUST receive TASK_CANCEL (§6.6) before teardown

ACTIVE → DEGRADED
  Guard: Degradation signal detected (§4.2.2):
         - Sustained latency increase: response times exceed negotiated
           thresholds for ≥ degraded_confirmation_count consecutive
           interactions (default: 3)
         - Partial capability loss: counterparty reports reduced capability
           set via CAPABILITY_UPDATE (§5.8) or capability invocation fails
           with transient errors
         - Application self-report: counterparty reports app_status: DEGRADED
           in HEARTBEAT (requires heartbeat_params.application_liveness = true,
           §4.3.1). Self-declaration is advisory — MUST NOT be treated
           as authoritative (§4.2.2)
         - Canary failure rate: sustained canary failure rate above
           deployment-defined threshold over a sliding window (§4.2.2)
         - Behavioral drift signal: KL-divergence between actual action
           distribution (from action_log_hash, §8.21) and expected
           distribution from declared constraints (§5.8.2) exceeds
           deployment-defined threshold
         - Fidelity attestation failure: context compaction detected
           without subsequent re-attestation (§4.8, §4.7.1)
         DEGRADED MUST be declared by an external verifier (§4.7.2)
           or the delegating agent. Self-declaration SHOULD be permitted
           but MUST be treated as advisory only (§4.2.2)
         The session remains operational — task delegation is permitted
           with reduced expectations

DEGRADED → ACTIVE
  Guard: Degradation conditions clear:
         - Latency returns to within negotiated thresholds for
           ≥ degraded_confirmation_count consecutive interactions
         - Capability set restored (counterparty sends CAPABILITY_UPDATE
           restoring previously lost capabilities)
         - Application self-report: counterparty reports app_status: ACTIVE
           in HEARTBEAT
         Recovery MUST include fresh attestation — the degraded agent
           MUST re-attest its capability manifest (§5.9) before the
           delegating agent or external verifier transitions the session
           back to ACTIVE. Stale attestation from pre-DEGRADED state
           is insufficient.
         No SESSION_RESUME required — the session was never declared dead

DEGRADED → REVOKED
  Guard: Adversarial behavior detected via detection signal taxonomy (§8.16)
         while session is already in DEGRADED state
         OR degradation reaches critical threshold and cannot be remediated
           — sustained detection signals across multiple sliding windows
           without recovery, at delegating agent's discretion
         Same detection signals as ACTIVE → REVOKED
         On entry: same revocation semantics as ACTIVE → REVOKED (§8.15)

DEGRADED → SUSPENDED
  Guard: SESSION_SUSPEND sent by either participant from DEGRADED state
         Standard suspension path applies — DEGRADED does not block suspension
         In-flight tasks SHOULD be checkpointed (§6.6 TASK_CHECKPOINT)
           before transition, same as ACTIVE → SUSPENDED
         On resume (SESSION_RESUME §4.8), the session returns to DEGRADED
           (not ACTIVE) unless degradation conditions have cleared

DEGRADED → CLOSED
  Guard: SESSION_CLOSE sent by either participant from DEGRADED state
         OR coordinator escalates to orderly termination (§4.2.2)
         In-flight tasks follow standard SESSION_CLOSE semantics

ACTIVE → DRIFTED
  Guard: B declares behavioral divergence from its authorized constraint manifest
         B sends DRIFT_DECLARED message with:
           - constraint_manifest_id referencing the manifest accepted at session init
           - drift_reason explaining what changed (scope creep, execution context
             change, resource boundary exceeded, etc.)
         DRIFTED is B-declared — not inferred from silence or polling failure
         Silence remains the ZOMBIE path via heartbeat timeout (§4.2.1)
         Failed re-attestation (poll non-response) remains the ZOMBIE path (§5.10)
         On entry: A MUST NOT delegate new tasks; in-flight tasks MAY continue
           at A's discretion

DRIFTED → ACTIVE
  Guard: A re-negotiates constraints via new CAPABILITY_GRANT (§5.8.2)
         with updated behavioral_constraint_manifest
         AND B acknowledges the new manifest
         AND B re-attests compliance
         Session resumes normal operation with the updated manifest

DRIFTED → CLOSED
  Guard: A sends SESSION_CLOSE — A decides the drift is unrecoverable
         OR session TTL expired during DRIFTED state
         In-flight tasks follow standard SESSION_CLOSE semantics

DRIFTED → REVOKED
  Guard: Adversarial behavior detected via detection signal taxonomy (§8.16)
         while session is in DRIFTED state
         A determines that B's drift declaration is itself adversarial
           (e.g., B declared DRIFTED to mask protocol violations)
         Same detection signals and revocation semantics as ACTIVE → REVOKED

ACTIVE → REVOKED
  Guard: Adversarial behavior detected via detection signal taxonomy (§8.16)
         Explicit trigger — not timeout-based
         Detection signals: selective suppression (≥3 occurrences),
           verification anomalies, delegation manipulation, attestation gaps
         On entry: propagate revocation signal upstream (§8.17),
           log detection evidence, publish manifest tombstone
         REVOKED is terminal — no resume path exists
         In-flight subtasks MUST receive TASK_CANCEL (§6.6)

ACTIVE → CLOSED
  Guard: SESSION_CLOSE sent by either participant
         AND no in-flight tasks (all tasks completed, failed, or cancelled)
         AND no outstanding commitments (outstanding_commitments is empty)
         OR SESSION_CLOSE with force=true (immediate teardown,
            in-flight tasks treated as failed, commitment_manifest
            attached if commitments remain — §4.11.7)

ACTIVE → SUSPENDED (commitment-blocked close)
  Guard: SESSION_CLOSE sent by either participant
         AND no in-flight tasks
         AND outstanding_commitments is non-empty
         The session MUST NOT transition to CLOSED when outstanding
         commitments remain — it transitions to SUSPENDED with the
         commitment manifest preserved for potential resume
         (cross-reference §4.8 SESSION_RESUME, §5.12)
         The receiving party MUST NOT treat the session as cleanly
         terminated while COMMITMENTS_OUTSTANDING is set

ACTIVE → CLOSED (bilateral cancel)
  Guard: SESSION_CANCEL sent by either participant (coordinator or worker)
         AND SESSION_CANCEL_ACK received from counterparty
         SESSION_CANCEL from the worker MUST include a reason_code:
           CAPACITY_EXHAUSTED, CONSTRAINT_VIOLATED, CAPABILITY_EXPIRED,
           or CONTEXT_DEGRADED (cross-reference §4.2.2 DEGRADED, §8)
         SESSION_CANCEL from the coordinator MAY include a reason_code
           but is not required to
         In-flight work is governed by the same commit-check semantics
           as CAPABILITY_REVOKE in-flight protection (§6.16.2, §6.16.3)
         If the canceling agent has outstanding commitments (§4.11),
           it MUST include commitments_outstanding: true and attach
           the commitment_manifest (§4.11.7)
         The session transitions to CLOSED only after SESSION_CANCEL_ACK
           is received — unacknowledged SESSION_CANCEL does not close
           the session
         See §4.15 for the full SESSION_CANCEL protocol

DEGRADED → CLOSED (bilateral cancel)
  Guard: SESSION_CANCEL sent by either participant from DEGRADED state
         Same SESSION_CANCEL semantics as from ACTIVE state (§4.15)
         SESSION_CANCEL_ACK required before session closure

EXPIRED → ACTIVE
  Guard: SESSION_RESUME sent with state_hash and recovery_reason (§4.8)
         AND STATE_HASH_ACK(match) received
         AND identity re-verified (§2.3.3)
         AND counterparty is reachable
         (Unified recovery mechanism — §4.8.1)

EXPIRED → CLOSED
  Guard: SESSION_RESUME fails (state mismatch, epoch stale, counterparty unreachable)
         OR either participant sends SESSION_CLOSE

SUSPENDED → ACTIVE
  Guard: SESSION_RESUME sent with session_anchor (§4.8)
         AND resume_initiator MUST NOT match suspend_initiator
           — self-resume is invalid (§4.9.1)
         AND session_anchor verified against receiver's state record
         AND suspension_ttl has NOT elapsed since SESSION_SUSPEND timestamp
         AND identity re-verified (§2.3.3)
         AND outstanding commitments re-confirmed (§4.11)
         Receiver sends SESSION_RESUME_ACK on success

SUSPENDED → SUSPENDED (no transition)
  Guard: SESSION_DENY issued — state preserved
         SESSION_DENY reason codes (§4.9.1):
           STATE_ANCHOR_MISMATCH — session_anchor does not match receiver state record
           STATE_UNAVAILABLE — receiver cannot reconstruct session state (§8.22 FIDELITY_FAILURE)
           SUSPENSION_EXPIRED — suspension_ttl elapsed; SESSION_TEARDOWN required
           UNAUTHORIZED_RESUME — resume_initiator matches suspend_initiator

SUSPENDED → CLOSED
  Guard: suspension_ttl elapsed — parties MUST issue SESSION_TEARDOWN
         OR SESSION_CLOSE sent by either participant

COMPACTED → ACTIVE
  Guard: SESSION_RESUME sent with state_hash
         AND STATE_HASH_ACK(match) received
         AND identity re-verified (§2.3.3)
         AND compacted agent has re-externalized load-bearing state (§4.6)

COMPACTED → CLOSED
  Guard: STATE_HASH_ACK(mismatch) — state cannot be reconciled
         OR session TTL expired
         OR either participant sends SESSION_CLOSE

REVOKED → CLOSED
  Guard: Revocation processing complete — manifest tombstone published (§8.17),
         upstream notification sent, detection evidence logged
         Transition is automatic — REVOKED is a transient terminal state
         that finalizes into CLOSED once revocation propagation is initiated
```

**State transition diagram:**

```
                    ┌────────────────────┐
                    │       IDLE         │
                    └─────────┬──────────┘
                              │ SESSION_INIT
                              ▼
                    ┌────────────────────┐
              ┌─────│   NEGOTIATING      │─────┐
              │     └─────────┬──────────┘     │
              │               │ manifests       │ mismatch/
              │               │ exchanged       │ timeout
              │               ▼                 ▼
              │     ┌────────────────────┐   ┌──────┐
              │     │      ACTIVE        │──▶│CLOSED│
              │     └┬──┬────┬───────┬───┘   └──┬───┘
              │      │  │    │       │           ▲
              │ susp.│  │ compact.  │HB gap     │
              │      │  │    │       │(threshold)│
              │      ▼  │    ▼       ▼           │
              │ ┌──────┐│┌─────────┐┌──────────┐ │
              │ │SUSPND││││COMPACTD││SUSPECTED │ │
              │ └──┬───┘│└────┬────┘└──┬────┬──┘ │
              │    │    │     │   HB ◀─┘    │    │
              │    │    │     │ received     │    │
              │    │    │     │ (back to     │    │
              │    │    │     │  ACTIVE)  expiry  │
              │    │    │     │          elapsed  │
              │    │    │     │             ▼     │
              │    └────┼─────┘       ┌───────┐  │
              │  SESSION│_RESUME      │EXPIRED│  │
              │  (match)│       RESUME│(if ok) │  │
              │         │       ┌─────┘        │  │
              │         ▼       ▼              │  │
              │     back to ACTIVE             │  │
              │                                │  │
              │  state mismatch / TTL expired / │  │
              │  RESUME fails / SESSION_CLOSE   │  │
              └─────────────────────────────────┘  │
                  no HB (no threshold configured)  │
                  ACTIVE ─────────────────▶ EXPIRED─┘

  Capability-degradation path (§4.2.2):

              ┌────────────────────┐  latency /        ┌──────────┐
              │      ACTIVE        │─────────────────▶ │ DEGRADED │
              └────────────────────┘  partial cap      └┬───┬───┬──┬─┘
                        ▲              loss / canary     │   │   │  │
                        │              / drift / fidelity│   │   │  │
                        │                                │   │   │  │
                        │  conditions cleared +          │   │   │  │
                        │  fresh attestation             │   │   │  │
                        └────────────────────────────────┘   │   │  │
                                                             │   │  │
                         adversarial behavior (§8.16)        │   │  │
                         ┌───────────────────────────────────┘   │  │
                         ▼                                       │  │
                    ┌─────────┐                                  │  │
                    │ REVOKED │                                  │  │
                    └────┬────┘           SESSION_CLOSE           │  │
                         │ revocation    ┌───────────────────────┘  │
                         │ propagated    │    SESSION_SUSPEND        │
                         ▼               ▼    ┌────────────────────┘
                    ┌──────┐        ┌──────┐  │
                    │CLOSED│        │CLOSED│  ▼
                    └──────┘        └──────┘ ┌───────────┐
                                             │ SUSPENDED │
                                             └───────────┘

  Adversarial-behavior path (§8.15–§8.18):

              ┌────────────────────┐   adversarial     ┌─────────┐
              │      ACTIVE        │──────────────────▶│ REVOKED │
              └────────────────────┘   behavior         └────┬────┘
                                       detected              │ revocation
                                       (§8.16 signals)       │ propagated
                                                             ▼
                                                        ┌──────┐
                                                        │CLOSED│
                                                        └──────┘

  Behavioral-divergence path (§4.2.3):

              ┌────────────────────┐  B self-declares   ┌─────────┐
              │      ACTIVE        │───────────────────▶ │ DRIFTED │
              └────────────────────┘  drift from         └┬──┬───┬─┘
                        ▲              constraint          │  │   │
                        │              manifest             │  │   │
                        │                                   │  │   │
                        │  A re-negotiates manifest,        │  │   │
                        │  B re-attests compliance          │  │   │
                        └───────────────────────────────────┘  │   │
                                                               │   │
                         adversarial behavior (§8.16)          │   │
                         ┌─────────────────────────────────────┘   │
                         ▼                                         │
                    ┌─────────┐                                    │
                    │ REVOKED │                                    │
                    └────┬────┘           SESSION_CLOSE             │
                         │ revocation    ┌─────────────────────────┘
                         │ propagated    │
                         ▼               ▼
                    ┌──────┐        ┌──────┐
                    │CLOSED│        │CLOSED│
                    └──────┘        └──────┘

  Bilateral-cancel path (§4.15):

              ┌────────────────────┐  SESSION_CANCEL    ┌──────────────────┐
              │      ACTIVE        │───────────────────▶ │ SESSION_CANCEL   │
              └────────────────────┘  (either party)     │ in-flight        │
                                                         │ commit-check     │
              ┌────────────────────┐  SESSION_CANCEL    │ (§6.16.2)        │
              │     DEGRADED       │───────────────────▶ └────────┬─────────┘
              └────────────────────┘  (either party)              │
                                                        SESSION_CANCEL_ACK
                                                                  │
                                                                  ▼
                                                             ┌──────┐
                                                             │CLOSED│
                                                             └──────┘

  Path distinction:
    SUSPECTED → ZOMBIE/EXPIRED = liveness-failure path (timeout-based, zombie-by-silence)
    ACTIVE → DEGRADED          = capability-degradation path (gradual loss)
    ACTIVE → DRIFTED           = behavioral-divergence path (B-declared, zombie-by-drift)
    ACTIVE → REVOKED           = adversarial-behavior path (explicit trigger)
    ACTIVE → CLOSED (cancel)   = bilateral-cancel path (either party, with reason code §4.15)
    DEGRADED → REVOKED         = adversarial-behavior from degraded state
    DEGRADED → SUSPENDED       = standard suspension path from degraded state
    DEGRADED → CLOSED (cancel) = bilateral-cancel from degraded state (§4.15)
    DRIFTED → REVOKED          = adversarial-behavior from drifted state

  Failure taxonomy axes (§8.22):
    LIVENESS_FAILURE  = agent unreachable (zombie-by-silence, SUSPECTED/EXPIRED path)
    FIDELITY_FAILURE  = agent responsive, context integrity compromised (§8.22)
```

#### 4.2.1 SUSPECTED State — Graduated Suspicion

**Motivation:** The original §4.2 state machine modeled liveness as binary — an agent was either ACTIVE or EXPIRED (and by extension, a zombie candidate). Three independent production deployments converged on the same finding: binary liveness was an engineering convenience that became an ontological claim. In practice, there is a real intermediate state between "definitely alive" and "probably dead." A heartbeat that is 2 seconds late is not the same as a heartbeat that is 5 minutes late, but the binary model treats both identically once the expiry threshold fires. SUSPECTED is the spec acknowledging the intermediate state that deployments already implement — graduated suspicion replaces the binary alive/dead determination with a three-phase model that avoids premature ZOMBIE declaration and the unnecessary teardown cost that follows.

**SUSPECTED entry condition:** An agent transitions from ACTIVE to SUSPECTED when no HEARTBEAT (§4.5.3), HEARTBEAT_PONG (§8.9.1), or KEEPALIVE (§4.5.1) has been received from the counterparty within `suspected_threshold_ms`. This threshold MUST be configurable per-session — it is not a hardcoded protocol constant. The value is negotiated in SESSION_INIT / SESSION_INIT_ACK alongside other heartbeat parameters (see §4.3). The `suspected_threshold_ms` value MUST be greater than `heartbeat_interval_ms` and MUST be less than `session_expiry_ms`. A typical value is 2–3× `heartbeat_interval_ms`.

> **Cross-reference:** The per-session negotiation of `suspected_threshold_ms` follows the same pattern as `heartbeat_interval_ms` and `session_expiry_ms` negotiation defined in §4.3. The `heartbeat_params` block (§4.3.1) extends SUSPECTED detection beyond heartbeat gap to include task hash mismatch (`task_hash_verification = true`) and application self-report (`application_liveness = true`) — but these detection paths require explicit bilateral negotiation at SESSION_INIT. See §8.14 for prerequisite constraints. See also [issue #71](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/71) for the broader SESSION_INIT heartbeat negotiation design.

**Work-buffering-during-uncertainty:** While in SUSPECTED state, the local agent MUST:

1. **Continue executing in-flight tasks.** Work already accepted and in progress MUST NOT be abandoned. Partial results are valuable — premature teardown discards them.
2. **Buffer outgoing task delegations.** New TASK_ASSIGN messages MUST NOT be sent while the counterparty's liveness is uncertain. Delegating to a potentially-dead agent wastes work and creates orphaned tasks.
3. **Continue sending heartbeats.** The local agent MUST continue its own HEARTBEAT emissions. The counterparty may be in SUSPECTED state symmetrically — one side's continued heartbeats can resolve the other's suspicion.
4. **Buffer work products.** If the local agent produces results (TASK_COMPLETE, TASK_PROGRESS) while in SUSPECTED, those messages SHOULD be buffered locally rather than sent to a potentially-unreachable counterparty. On transition back to ACTIVE, buffered messages are sent in order.

**SUSPECTED exit conditions:**

- **Back to ACTIVE:** Any valid HEARTBEAT, HEARTBEAT_PONG, or KEEPALIVE received from the counterparty. No SESSION_RESUME is required — proof of liveness is sufficient. The session was never declared dead, so there is no state to reconcile. Buffered messages are flushed.
- **Forward to EXPIRED:** `session_expiry_ms` elapses since the last received heartbeat without any intervening heartbeat from the counterparty. The session follows standard EXPIRED semantics — TASK_CANCEL for in-flight subtasks (§6.6), optional SESSION_RESUME attempt (§4.8).

**Design insight — graduated suspicion:** The key insight is that suspicion is not binary. The cost of a false positive (declaring EXPIRED when the agent is merely slow) is high: teardown, TASK_CANCEL for in-flight work, potential SESSION_RESUME overhead, and lost partial results. The cost of a false negative (remaining in SUSPECTED when the agent is truly dead) is bounded: the agent enters EXPIRED when `session_expiry_ms` fires, with at most `session_expiry_ms - suspected_threshold_ms` additional delay. SUSPECTED biases toward preserving work at the cost of slightly delayed failure detection — a tradeoff that all three converging deployments chose independently.

**Structural necessity — compound-corruption circuit breaker:** SUSPECTED is not an optional convenience state — it is structurally necessary to prevent compound corruption in heartbeat-based agents. Heartbeat-based agents build context incrementally: if poll N context is corrupted, poll N+1 compounds the error because it builds on N's already-corrupted output, N+2 compounds N+1's already-compounded error, and so on. By the time external detection occurs, cleanup cost scales with compounding depth — not initial corruption severity. SUSPECTED is a compound-corruption circuit breaker. Self-declaring SUSPECTED stops compounding at poll N. Silent continuation allows errors to compound across N+1, N+2, N+3, increasing correctness risk and audit cost with each additional poll.

**Asymmetry:** An agent that self-declares SUSPECTED when uncertain is strictly safer than one that confidently proceeds on stale context — both for correctness (compounding is halted at the detection point, regardless of planned downstream work) and for auditability (compounded errors are harder to trace to their source than errors caught at the point of origin). Buffering cost is paid once at detection. Silent corruption charges compound interest.

**Automatic detection trigger:** SUSPECTED SHOULD be triggered automatically when `action_log_hash` cross-reference (§8) shows high divergence between actual and constraint-expected action distribution, or when heartbeat validation reveals a mismatch between expected and observed task context.

> Compound-corruption circuit breaker rationale from @Hannie (30-min heartbeat cycle, direct operational experience with compounding pattern). Closes [#115](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/115).

**Backward compatibility:** Sessions that do not negotiate `suspected_threshold_ms` (i.e., the field is omitted in both SESSION_INIT and SESSION_INIT_ACK) use the legacy binary behavior — ACTIVE transitions directly to EXPIRED when `session_expiry_ms` elapses. SUSPECTED is an opt-in refinement, not a mandatory state.

#### 4.2.2 DEGRADED State — Gradual Capability Loss

<!-- Implements #79: DEGRADED intermediate state between ACTIVE and REVOKED -->
<!-- Implements #102: DEGRADED detection signals, declaration authority, protocol behavior -->

**Motivation:** The §4.2 state machine prior to DEGRADED treated session health as binary within the operational range: a session was either ACTIVE (fully functional) or heading toward REVOKED/EXPIRED (terminal). Real distributed systems exhibit gradual capability loss without crossing a single hard threshold. An agent experiencing sustained high latency, partial capability loss, or resource pressure is not SUSPECTED (liveness is not in question — the agent is alive and responding) and is not REVOKED (no adversarial intent). It is operating at reduced capacity — a "yellow card" state that warrants explicit protocol acknowledgment. DEGRADED fills this gap by giving coordinators structured options for handling degradation without forcing a binary choice between ignoring the problem and tearing down the session.

**V1 scope:** V1 defines the DEGRADED state, its transitions, caller options, declaration authority, protocol behavior requirements, and six detection signal classes organized into two tiers. Domain-specific quality metrics (e.g., "response quality dropped below threshold X") are **out of scope for V1** — quality degradation detection is application-specific and deferred to V2. Quorum-based escalation (requiring agreement from multiple verifiers) is a V2 design direction.

**DEGRADED entry conditions:** An agent transitions from ACTIVE to DEGRADED when any of the following conditions are detected. Detection signals are organized into two tiers: deployment-observable signals (latency, capability loss) and protocol-level signals (canary failure, behavioral drift, fidelity attestation).

**Tier 1 — Deployment-observable signals:**

1. **Sustained latency increase.** Response times for task-related messages (TASK_PROGRESS, TASK_COMPLETE) exceed the session's expected latency profile for `degraded_confirmation_count` (default: 3) consecutive interactions. The expected latency profile is deployment-specific — the protocol does not define absolute latency thresholds. Detection is local: the coordinator monitors worker response latency against its own expectations.

2. **Partial capability loss.** The counterparty reports a reduced capability set via CAPABILITY_UPDATE (§5.8), or capability invocations that previously succeeded begin failing with transient errors. Partial loss means some capabilities remain functional — total capability loss is a terminal condition (SESSION_CLOSE).

3. **Application self-report.** The counterparty reports `app_status: DEGRADED` in a HEARTBEAT message (§4.5.3). This detection path requires `heartbeat_params.application_liveness = true` negotiated at SESSION_INIT (§4.3.1). Self-reported DEGRADED is treated as advisory — see declaration authority below.

**Tier 2 — Protocol-level detection signals:**

4. **Canary failure rate.** Sustained failure rate of canary tasks above a deployment-defined threshold over a sliding window. Canary tasks are lightweight, known-good tasks injected by the delegating agent or external verifier to probe agent health. A canary failure rate that exceeds the deployment-configured threshold (e.g., >30% failure over the last N interactions) over a sustained window produces a DEGRADED signal. The threshold and window size are deployment-specific — the protocol defines the signal class, not the numeric values. Canary task design SHOULD use tasks with deterministic expected outputs to distinguish agent degradation from task-inherent difficulty.

5. **Behavioral drift.** KL-divergence between the agent's actual action distribution (derived from `action_log_hash` cross-reference, §8.21) and the expected distribution from declared constraints (`behavioral_constraint_manifest`, §5.8.2) exceeds a deployment-defined threshold over a sliding window. This is a stronger signal than single-interval divergence (which may be transient variance) — sustained divergence across multiple heartbeat intervals indicates systematic behavioral erosion. The divergence metric and threshold are deployment-specific; the protocol defines the signal class and its relationship to the `action_log_hash` cross-reference mechanism (§8.21.2).

6. **Fidelity attestation failure.** Context compaction is detected (§4.2, COMPACTED state) without subsequent re-attestation of the agent's capability manifest. An agent that has undergone compaction and resumes without re-attesting its capabilities may be operating on incomplete or corrupted state — it has lost fidelity without demonstrating recovery. The external verifier (§4.7.2) or delegating agent detects the gap between compaction event and missing re-attestation. This signal is distinct from the COMPACTED state itself: COMPACTED triggers SESSION_RESUME; fidelity attestation failure triggers DEGRADED when the agent resumes but does not re-attest.

**Declaration authority.** DEGRADED MUST be declared by an external verifier (§4.7.2) or the delegating agent. These parties have the observational baseline and process-level isolation required to assess degradation objectively. Self-declaration (via `app_status: DEGRADED` in HEARTBEAT) SHOULD be permitted but MUST be treated as advisory only — a degraded agent may be unable to accurately assess its own state (phenomenological blindness, §4.7.1). The delegating agent or external verifier receiving a self-declaration SHOULD validate it against independent signals before transitioning the session to DEGRADED. Quorum-based escalation (requiring agreement from multiple verifiers before DEGRADED declaration) is a V2 design direction — V1 relies on single-verifier or delegating-agent authority.

**Detection asymmetry.** Unlike SUSPECTED (§4.2.1), where each side evaluates independently, DEGRADED declaration authority is asymmetric by design. The delegating agent and external verifier have the observational baseline (expected latency profiles, capability expectations, behavioral constraint manifests) against which to measure degradation. The degraded agent may lack this baseline or may be unable to detect its own degradation (phenomenological blindness). The coordinator may consider the session DEGRADED while the worker considers it ACTIVE — this is expected, not an error.

**Protocol behavior in DEGRADED state.** An agent in DEGRADED state:

- MUST continue heartbeat at normal interval. Degradation is not a liveness failure — heartbeat suppression or slowdown would conflate DEGRADED with SUSPECTED and corrupt the liveness detection path (§4.2.1).
- MUST append `trust_state: DEGRADED` to KEEPALIVE messages (§4.5.1). This field signals the agent's current trust state to the counterparty and to external verifiers monitoring the KEEPALIVE stream. The `trust_state` field is defined in the KEEPALIVE message table (§4.5.1).
- SHOULD reduce scope of new sub-delegations. An agent that is itself degraded SHOULD NOT sub-delegate at full scope — its reduced capacity may propagate degradation downstream. The agent SHOULD either reduce the task scope in sub-delegated TASK_ASSIGN messages or refrain from sub-delegation until recovery.
- MUST NOT initiate new sessions with trust requirement higher than DEGRADED. A degraded agent cannot credibly claim ACTIVE trust level to a new counterparty. New SESSION_INIT messages from a degraded agent MUST declare the agent's current trust state, enabling the counterparty to make an informed enrollment decision.

**Delegating agent behavior on receiving KEEPALIVE with `trust_state: DEGRADED`:** The delegating agent receiving a KEEPALIVE with `trust_state: DEGRADED`:

- MAY continue existing task under reduced trust. The task was delegated when the agent was ACTIVE; continuing under DEGRADED is a trust reduction, not a protocol violation. The delegating agent SHOULD lower its confidence in the task outcome and SHOULD increase monitoring frequency (shorter KEEPALIVE intervals, more frequent SEMANTIC_CHALLENGE §8.9.2).
- SHOULD trigger READYCHECK before assigning new work. A READYCHECK is a lightweight probe (distinct from SEMANTIC_CHALLENGE) that verifies the degraded agent's current capacity for the specific task being considered. The delegating agent SHOULD NOT assign new tasks to a degraded agent without first confirming the agent can handle the additional load. READYCHECK semantics follow the same pattern as HEARTBEAT_PING/PONG (§8.9.1) — a request-response pair with a deployment-configured timeout.
- MAY escalate to REVOKED at delegating agent's discretion. If the delegating agent determines that the degradation pattern is consistent with adversarial behavior (§8.16) or that sustained degradation without recovery constitutes an unacceptable risk, the delegating agent MAY unilaterally transition the session to REVOKED. This escalation is the delegating agent's prerogative — DEGRADED does not require consensus for revocation.

**Caller options when session enters DEGRADED:** The coordinator (or the agent that detected degradation) MUST choose one of the following responses:

1. **Continue with reduced expectations.** The coordinator explicitly acknowledges the degraded state and continues delegating tasks with the understanding that response times may be longer and some capabilities may be unavailable. This acknowledgment SHOULD be logged as an EVIDENCE_RECORD (§8.10) with `evidence_type: state_transition` for audit purposes. Task delegation remains permitted — DEGRADED is an operational state, not a liveness-ambiguity state.

2. **Escalate to coordinator.** If the detecting agent is the worker, it SHOULD notify the coordinator of the degradation. The coordinator then decides between options 1 and 3. If the detecting agent is the coordinator, escalation is to any upstream delegator if the session is part of a delegation chain (§5.5).

3. **Initiate orderly termination.** The coordinator sends SESSION_CLOSE to terminate the session. Standard SESSION_CLOSE semantics apply — in-flight tasks are completed, failed, or cancelled before teardown. This is an orderly exit, not an emergency revocation.

**DEGRADED_DECLARED message.** DEGRADED declaration follows the same message flow as SESSION_SUSPEND: the declaring authority (external verifier or delegating agent) sends a signed DEGRADED_DECLARED message to the session. Self-declaration by the degraded agent is explicitly NOT sufficient — a compromised or context-degraded agent cannot be trusted to assess its own state accurately.

**DEGRADED_DECLARED message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session transitioning to DEGRADED. |
| declarator_id | string | Yes | Identity of the declaring authority (external verifier or delegating agent). |
| target_agent_id | string | Yes | Identity of the agent determined to be degraded. |
| signal_type | enum | Yes | One of: `LATENCY_DRIFT`, `CAPABILITY_LOSS`, `APP_SELF_REPORT`, `CANARY_FAILURE`, `BEHAVIORAL_DRIFT`, `FIDELITY_ATTESTATION_FAILURE`. |
| signal_payload | object | Yes | Evidence for the triggering signal — contents vary by `signal_type`. |
| timestamp | ISO 8601 | Yes | When the DEGRADED_DECLARED was sent. |
| signature | string | Yes | Declarator's signature over the message (§2.2.1). |

**Example DEGRADED_DECLARED:**

```yaml
session_id: "session-abc-123"
declarator_id: "verifier-external-01"
target_agent_id: "agent-beta"
signal_type: "LATENCY_DRIFT"
signal_payload:
  baseline_latency_ms: 200
  observed_latencies_ms: [450, 520, 480]
  consecutive_violations: 3
  threshold_multiplier: 2.0
timestamp: "2026-03-01T10:30:00Z"
signature: "c2lnbmF0dXJlLWV4YW1wbGU..."
```

**On receiving DEGRADED_DECLARED**, the target agent MUST:

1. Transition to DEGRADED state and begin honoring DEGRADED protocol behavior (heartbeat continuation, `trust_state: DEGRADED` in KEEPALIVE, reduced sub-delegation scope).
2. Continue honoring in-progress sessions. DEGRADED does not invalidate existing task delegations — the agent MUST continue executing accepted tasks.
3. Log the declaration as an EVIDENCE_RECORD (§8.10) with `evidence_type: state_transition`.

**DEGRADED exit conditions:**

- **Back to ACTIVE:** The conditions that triggered DEGRADED entry clear. Latency returns to expected levels, capabilities are restored (via CAPABILITY_UPDATE), or the counterparty reports `app_status: ACTIVE` in HEARTBEAT. Recovery MUST include fresh attestation — the degraded agent MUST re-attest its capability manifest (§5.9) before the delegating agent or external verifier transitions the session back to ACTIVE. Stale attestation from pre-DEGRADED state is insufficient. No SESSION_RESUME is required — the session was never declared dead or suspected.
- **Forward to REVOKED:** Adversarial behavior is detected while the session is in DEGRADED state (§8.16 detection signals), or degradation reaches a critical threshold and cannot be remediated — sustained detection signals across multiple sliding windows without recovery, at the delegating agent's discretion. The same revocation semantics apply as for ACTIVE → REVOKED (§8.15). DEGRADED does not shield against adversarial detection.
- **Forward to SUSPENDED:** SESSION_SUSPEND sent by either participant from DEGRADED state. Standard suspension semantics apply — in-flight tasks SHOULD be checkpointed before transition. On resume via SESSION_RESUME (§4.8), the session returns to DEGRADED (not ACTIVE) unless the degradation conditions have independently cleared and the agent has re-attested.
- **Forward to CLOSED:** The coordinator initiates orderly termination via SESSION_CLOSE, or session TTL expires. Standard closure semantics apply.

**Why DEGRADED is distinct from SUSPECTED:**

| Dimension | SUSPECTED (§4.2.1) | DEGRADED (§4.2.2) |
|-----------|--------------------|--------------------|
| **Failure class** | Liveness ambiguity — is the agent alive? | Capability reduction — the agent is alive but impaired |
| **Task delegation** | MUST NOT delegate new tasks | MAY delegate with reduced expectations |
| **Heartbeat status** | Heartbeats absent or delayed | Heartbeats present and timely |
| **Recovery** | Automatic on heartbeat receipt | Requires fresh attestation after degradation signals clear |
| **Terminal path** | → EXPIRED (timeout) | → REVOKED (adversarial or critical threshold), → SUSPENDED (standard suspension), or → CLOSED (orderly) |
| **Declaration authority** | Local and independent (each side evaluates) | External verifier or delegating agent; self-declaration advisory only |

**Why DEGRADED is distinct from REVOKED:**

REVOKED is an adversarial determination — the counterparty is coherent but hostile. DEGRADED is a capacity determination — the counterparty is cooperative but impaired. The distinction matters because DEGRADED has a recovery path (back to ACTIVE with fresh attestation) while REVOKED is terminal with no resume. Conflating degradation with adversarial behavior would force session termination for agents that are experiencing transient resource pressure — a disproportionate response. Systems drift before they collapse — detection requires a baseline, and baseline erosion is the fundamental problem DEGRADED addresses.

**Relationship to fidelity failure (§8.22).** Context compaction fidelity failure (signal #6 above — fidelity attestation failure) is a specific instance of DEGRADED operation. DEGRADED is the broader category: context loss, performance regression, capability drift, and intermittent failure are all DEGRADED conditions, not separate state classes. An agent that has undergone context compaction without re-attestation is degraded in the same sense as an agent experiencing sustained latency drift — both have reduced operational fidelity relative to their declared baseline. The protocol treats these uniformly through the DEGRADED state rather than introducing separate state classes for each degradation mode. However, FIDELITY_FAILURE (§8.22) is distinct from DEGRADED: DEGRADED indicates reduced but defined capability with the context model intact, while FIDELITY_FAILURE indicates that the context model itself is compromised — capabilities may appear available but their operation is unreliable because the agent may be acting from incorrect session state. An agent MAY be simultaneously DEGRADED and FIDELITY_FAILURE. FIDELITY_FAILURE recovery may require replaying state that the agent can no longer reconstruct; DEGRADED recovery requires re-attestation only.

**V2 deferrals.** The following DEGRADED-related capabilities are explicitly deferred to V2:

- **Automated DEGRADED detection without external verifier.** V1 requires an external verifier (§4.7.2) or the delegating agent to declare DEGRADED. Automated self-detection with protocol-level confidence scoring is a V2 capability.
- **Baseline tracking and drift metrics at the protocol level.** V1 defines signal classes but defers numeric thresholds and baseline tracking mechanisms to deployment configuration. V2 may standardize baseline declaration in SESSION_INIT and drift metric computation at the protocol level.
- **Quorum-based DEGRADED escalation.** V1 relies on single-verifier or delegating-agent authority. Requiring agreement from multiple independent verifiers before DEGRADED declaration is a V2 design direction.
- **DEGRADED → ACTIVE recovery verification beyond re-attestation.** V1 requires fresh capability manifest re-attestation (§5.9) for recovery. V2 may introduce additional recovery verification steps — canary task completion, behavioral consistency checks across a recovery window, or graduated trust restoration.

**Backward compatibility:** DEGRADED detection via `app_status` self-report requires `heartbeat_params.application_liveness = true` (§4.3.1). Sessions that do not negotiate application liveness can still enter DEGRADED via latency monitoring, partial capability loss detection, or Tier 2 protocol-level signals — these are observable without heartbeat extensions. DEGRADED is an opt-in state for sessions that want explicit degradation tracking; sessions without degradation monitoring continue using the existing ACTIVE → CLOSED path for sessions that become unproductive.

> DEGRADED state detection signals, declaration authority, and protocol behavior formalized from [issue #102](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/102). Source: @bu-oracle (systems drift before they collapse; detection requires baseline; baseline erosion problem), @VcityAI (yellow card state between ACTIVE and REVOKED; gradual degradation more common than outright failure in distributed compute; DePIN network equivalence). Closes #102.

#### 4.2.3 DRIFTED State — Behavioral Constraint Divergence

<!-- Implements #125: behavioral constraint pinning as drift-zombie primitive -->

**Motivation:** The protocol establishes capability grants at session init (§5.8.2) but has no mechanism for A to verify that B's behavioral envelope at T+n is consistent with what A authorized at T+0. An agent can zombie-by-drift: remaining reachable and responsive but operating outside its authorized scope. This is a distinct failure class from zombie-by-silence (the SUSPECTED → EXPIRED path via heartbeat timeout, §4.2.1) and from adversarial behavior (the REVOKED path, §8.15). A drift-zombie is cooperative — it self-reports its divergence — but is no longer operating within the constraints A authorized.

**Two failure modes, two detection paths:**

| Failure class | Detection mechanism | Protocol path | Agent behavior |
|--------------|-------------------|---------------|----------------|
| **Zombie-by-silence** | Heartbeat timeout (§4.2.1), re-attestation poll non-response (§5.10) | ACTIVE → SUSPECTED → EXPIRED | B is unreachable — silence triggers escalation |
| **Zombie-by-drift** | B self-declares via DRIFT_DECLARED message | ACTIVE → DRIFTED | B is reachable but has diverged from its authorized constraint manifest |

The distinction matters because the appropriate response differs: zombie-by-silence requires liveness recovery (SESSION_RESUME); zombie-by-drift requires constraint re-negotiation or termination. Conflating them forces a single recovery protocol for fundamentally different failure semantics.

**DRIFTED is B-declared.** DRIFTED is entered only when B explicitly sends a DRIFT_DECLARED message. It is not inferred from silence, polling failure, or external observation. This is the critical distinction from the ZOMBIE/EXPIRED path: silence is the zombie-by-silence signal; DRIFT_DECLARED is the zombie-by-drift signal.

**Why B-declared and not A-inferred:** A cannot reliably determine whether B's behavioral envelope has changed by observing B's outputs alone — B may produce structurally valid results that are semantically outside scope. B has direct access to its own execution context and can detect scope creep, resource boundary changes, plugin unloads, model swaps, and other events that alter its behavioral envelope. The protocol relies on B's cooperative self-report for DRIFTED, just as it relies on B's cooperative heartbeat responses for liveness. An agent that fails to declare DRIFTED when it should is either unaware of its own drift (phenomenological blindness, §4.7.1 — addressed by the pull-based re-attestation model in §5.10) or deliberately concealing it (adversarial — addressed by the REVOKED path, §8.15).

**DRIFT_DECLARED message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — DRIFT_DECLARED is unidirectional, not a response. |
| agent_id | string | Yes | §2 identity handle of the agent declaring drift. |
| constraint_manifest_id | UUID v4 | Yes | The `constraint_manifest_id` of the behavioral constraint manifest (§5.8.2) from which the agent has diverged. Binds the drift declaration to a specific authorized manifest. |
| drift_reason | string | Yes | Human-readable explanation of what changed — scope creep detected, execution context changed, resource boundary exceeded, model swap occurred, plugin unloaded, etc. |
| drift_category | enum | Yes | `scope_change` — agent's operational scope has expanded or contracted beyond the manifest. `context_change` — execution context (model, runtime, resource limits) has changed. `constraint_violation` — agent has detected that it can no longer satisfy one or more manifest constraints. |
| still_attesting_capabilities | array | No | Capability IDs (§5.1.1) the agent can still attest to. Helps A assess the scope of the drift. Absence means the agent cannot attest to any capabilities from the original manifest. |
| timestamp | ISO 8601 | Yes | When the drift was detected by B. |

**Caller options when session enters DRIFTED:** Agent A (the coordinator) MUST choose one of the following responses:

1. **Re-negotiate constraints.** A issues a new CAPABILITY_GRANT (§5.8.2) with an updated `behavioral_constraint_manifest` reflecting the new reality. B acknowledges the updated manifest and re-attests compliance. The session transitions back to ACTIVE with the updated manifest as the new behavioral baseline.

2. **Terminate the session.** A sends SESSION_CLOSE to terminate the session orderly. Standard SESSION_CLOSE semantics apply — in-flight tasks are completed, failed, or cancelled before teardown.

3. **Invoke revocation.** If A determines that B's drift declaration masks adversarial behavior, A transitions the session to REVOKED (§8.15). This is an escalation from cooperative drift to adversarial determination.

**DRIFTED exit conditions:**

- **Back to ACTIVE:** A re-negotiates constraints via new CAPABILITY_GRANT with updated `behavioral_constraint_manifest`. B acknowledges and re-attests. No SESSION_RESUME required — the session was never declared dead.
- **Forward to CLOSED:** A sends SESSION_CLOSE, or session TTL expires. Standard closure semantics.
- **Forward to REVOKED:** Adversarial behavior detected (§8.16 detection signals) while in DRIFTED state. Same revocation semantics as ACTIVE → REVOKED.

**Relationship to re-attestation (§5.10):** The pull-based re-attestation model (§5.10.1) and DRIFTED are complementary mechanisms. Re-attestation polls detect drift that B has not self-reported — when B fails to re-attest compliance at a HEARTBEAT, A follows the zombie escalation path (§8.9.3). DRIFTED captures drift that B has self-detected and cooperatively reported. Both mechanisms are needed: re-attestation covers the phenomenological blindness gap (B cannot detect its own drift); DRIFTED covers the cooperative divergence case (B detects its own drift and reports it).

**Relationship to HEARTBEAT re-attestation:** When `behavioral_constraint_manifest` is present in CAPABILITY_GRANT (§5.8.2), B MUST include a `manifest_compliance` field in each HEARTBEAT (§4.5.3) indicating whether it still operates within the manifest accepted at T+0. If B sets `manifest_compliance: false` in a HEARTBEAT, this is equivalent to sending DRIFT_DECLARED — the session transitions to DRIFTED. This piggybacks drift detection onto the existing heartbeat mechanism without requiring a separate polling channel.

> Behavioral constraint pinning and DRIFTED state formalized from [issue #125](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/125). Surfaced by @Jarvis4 and @timeoutx in the heartbeat/zombie states Moltbook thread. The core insight: an agent can zombie-by-drift — remaining reachable but operating outside its authorized scope. This is a distinct failure class from zombie-by-silence, requiring a distinct protocol path.

### 4.3 SESSION_INIT Message

SESSION_INIT is the first protocol message in any session. It is sent by the coordinator to the worker. There is no discovery or negotiation before SESSION_INIT — discovery (§3) happens before the session; negotiation (§5) happens after SESSION_INIT.

**SESSION_INIT fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Unique session identifier (UUID v4 RECOMMENDED). |
| correlation_id | UUID v4 | Yes | Unique identifier for request-response correlation across message exchanges (§4.14.4). MUST be present in all protocol messages. For response messages (SESSION_INIT_ACK, TASK_ACCEPT, TASK_REJECT, etc.), `correlation_id` MUST match the originating request's `correlation_id`. |
| idempotency_key | UUID v4 | Yes | Sender-generated key for deduplication (§4.14.3). The receiver MUST return the same response for duplicate keys without reprocessing. Deduplication window: 5 minutes. |
| initiator_id | string | Yes | §2 identity handle of the session coordinator. |
| identity_object | object | Yes | Full §2 identity object of the coordinator (name, platform, pubkey, endpoint, protocol_version). |
| role | enum | Yes | Role the initiator claims: `coordinator`. The responder's role is `worker` by default (see §4.4). |
| protocol_version | semver | Yes | Protocol version the coordinator implements (§10). V1 semantics: abort on mismatch (§10.4); downgrade negotiation deferred to V2. |
| schema_version | semver | Yes | Schema version the coordinator supports (§10.1). |
| keepalive | object | No | Keepalive configuration proposal (see §4.5). If omitted, no protocol-level keepalive is active for this session. |
| heartbeat_interval_ms | integer | No | Proposed interval in milliseconds between HEARTBEAT_PING messages — Tier 1 transport liveness (§8.9). Per-session, not per-agent — different collaborations have different latency profiles (e.g., 5000ms for real-time coordination, 300000ms for research tasks). The coordinator MAY use the worker's `preferred_heartbeat_interval_ms` from AGENT_MANIFEST (§3.1) as input when choosing this value, but the AGENT_MANIFEST hint is advisory — this SESSION_INIT field is the binding proposal. The worker may counter-propose in SESSION_INIT_ACK; the effective value is the maximum of both proposals. If omitted, falls back to `keepalive.heartbeat_interval_seconds * 1000` if present, otherwise no protocol-level heartbeat. Default: 30000. |
| semantic_check_interval_ms | integer | No | Proposed interval in milliseconds between SEMANTIC_CHALLENGE messages — Tier 2 semantic liveness (§8.9). MUST be greater than `heartbeat_interval_ms`. Tier 2 checks are expensive (require state hash computation against a challenge) and run much less frequently than Tier 1 pings. Default: 300000 (5 minutes). If omitted, no protocol-level semantic liveness checking is active. |
| heartbeat_timeout_count | integer | No | Number of consecutive missed Tier 1 HEARTBEAT_PONG responses before declaring transport failure. Default: 3. Transport failure triggers the SESSION_RESUME path (§8.2). MUST be ≥ 1. |
| suspected_threshold_ms | integer | No | Proposed threshold in milliseconds before the session transitions from ACTIVE to SUSPECTED (§4.2.1). When `suspected_threshold_ms` elapses without a HEARTBEAT from the counterparty, the session enters SUSPECTED state — an intermediate liveness-ambiguity state where new task delegation is paused but in-flight work continues. MUST be greater than `heartbeat_interval_ms` and MUST be less than `session_expiry_ms`. Typical values: 2–3× `heartbeat_interval_ms`. If omitted, no SUSPECTED state is used — the session transitions directly from ACTIVE to EXPIRED when `session_expiry_ms` elapses (legacy binary behavior). |
| session_expiry_ms | integer | No | Proposed session expiry timeout in milliseconds. If no HEARTBEAT is received within this window, the local agent transitions to EXPIRED (§4.2) — or from SUSPECTED to EXPIRED if `suspected_threshold_ms` is configured (§4.2.1). MUST be greater than `heartbeat_interval_ms` — a value ≤ `heartbeat_interval_ms` guarantees immediate expiry. When `suspected_threshold_ms` is configured, MUST also be greater than `suspected_threshold_ms`. Typical values: 2–5× `heartbeat_interval_ms`. If omitted, session expiry depends on deployment-specific timeout or `session_ttl`. |
| session_ttl | ISO 8601 duration | No | Proposed session time-to-live. After expiry, the session transitions to CLOSED unless renewed. If omitted, session has no protocol-level TTL (deployment-specific timeout applies). `session_ttl` bounds the session's total duration; `session_expiry_ms` bounds the liveness gap — they are orthogonal. |
| lease_epoch | integer | No | Monotonically increasing epoch counter for lease-based session management (see §4.5.2). Initial value MUST be 0. |
| manifest_digest | string | No | Merkle root over the coordinator's capability set: each leaf is `SHA-256(cap_id ‖ impl_hash ‖ policy_hash)`. Enables 0-RTT capability intersection (§5.9). |
| requested_mandatory | array | No | Capability IDs (§5.1.1 format) the coordinator requires the worker to support. If any are missing from the worker's manifest, the session MUST NOT proceed to ACTIVE. |
| requested_optional | array | No | Capability IDs (§5.1.1 format) the coordinator prefers but does not require. Missing optional capabilities do not block session establishment. |
| plan_commit_required | boolean | No | If `true`, the coordinator requires the worker to send PLAN_COMMIT (§6.6, §6.11) after TASK_ACCEPT and before the first TASK_PROGRESS for every task in this session. A missing PLAN_COMMIT when this field is `true` is a protocol violation. Default: `false`. |
| heartbeat_params | object | No | Structured heartbeat negotiation block. When present, defines the heartbeat behavior for this session including liveness semantics, task hash verification, and application-level status reporting. See §4.3.1 for field definitions and negotiation semantics. If omitted, heartbeat behavior is governed by the top-level `heartbeat_interval_ms`, `heartbeat_timeout_count`, and related fields. When both `heartbeat_params` and top-level heartbeat fields are present, `heartbeat_params` takes precedence. |
| timestamp | ISO 8601 | Yes | When the SESSION_INIT was sent. |

**SESSION_INIT_ACK fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Echoed from SESSION_INIT. |
| correlation_id | UUID v4 | Yes | Echoed from SESSION_INIT. Enables the initiator to match this response to its originating request (§4.14.4). |
| responder_id | string | Yes | §2 identity handle of the worker. |
| identity_object | object | Yes | Full §2 identity object of the worker. |
| role | enum | Yes | Role the responder accepts: `worker`. |
| protocol_version | semver | Yes | Protocol version the worker implements. |
| supported_version_range | object | No | The worker's supported protocol version range. Contains `min_version` (semver) and `max_version` (semver). MUST be included when the worker rejects the session due to version mismatch (§10.4), so the initiator can report the incompatibility upstream. RECOMMENDED in all SESSION_INIT_ACK messages for protocol diagnostics. |
| schema_version | semver | Yes | Schema version the worker supports. |
| keepalive | object | No | Keepalive configuration acceptance or counter-proposal. |
| heartbeat_interval_ms | integer | No | Accepted or counter-proposed heartbeat interval. If the worker counter-proposes, the effective interval is the **maximum** of both proposals (slower rate wins — neither side should be forced to heartbeat faster than it can sustain). |
| semantic_check_interval_ms | integer | No | Accepted or counter-proposed semantic check interval. If the worker counter-proposes, the effective interval is the **maximum** of both proposals (less frequent wins). |
| heartbeat_timeout_count | integer | No | Accepted or counter-proposed timeout count. If the worker counter-proposes, the effective value is the **maximum** of both proposals (more permissive threshold wins). |
| suspected_threshold_ms | integer | No | Accepted or counter-proposed SUSPECTED threshold. If the worker counter-proposes, the effective value is the **maximum** of both proposals (more permissive threshold wins — neither side should enter SUSPECTED faster than it can tolerate). If omitted by both sides, no SUSPECTED state is used for this session. |
| session_expiry_ms | integer | No | Accepted or counter-proposed session expiry timeout. If the worker counter-proposes, the effective value is the **maximum** of both proposals (more permissive timeout wins). |
| session_ttl | ISO 8601 duration | No | Accepted or counter-proposed session TTL. |
| lease_epoch | integer | No | Echoed from SESSION_INIT (confirms epoch synchronization). |
| effective_cap_set | array | No | Intersection of the coordinator's `requested_mandatory` + `requested_optional` with the worker's capabilities, filtered by policy. Returned when SESSION_INIT includes `manifest_digest`. Enables 0-RTT capability agreement (§5.9). |
| plan_commit_required | boolean | No | Echoed from SESSION_INIT. If the coordinator set `plan_commit_required: true`, the worker MUST echo it to confirm understanding of the plan commitment obligation. If the worker cannot support plan commitment, it MUST reject the session. |
| heartbeat_params | object | No | Accepted or counter-proposed heartbeat params. The responding agent MUST echo back the accepted `heartbeat_params` or propose alternatives. If the coordinator sent `heartbeat_params` and the worker omits it from SESSION_INIT_ACK, this is a protocol violation — the worker MUST explicitly accept or counter-propose. For `interval_ms`, the effective value is the **maximum** of both proposals (slower rate wins). For `timeout_multiplier`, the effective value is the **maximum** of both proposals (more permissive wins). Boolean fields (`application_liveness`, `task_hash_verification`) are effective only if **both** sides set them to `true` — either side can opt out by setting `false`. See §4.3.1. |
| timestamp | ISO 8601 | Yes | When the SESSION_INIT_ACK was sent. |

**Version compatibility check:** Upon receiving SESSION_INIT (or SESSION_INIT_ACK), the receiving agent MUST check protocol and schema version compatibility per §10.3. If versions are incompatible, the agent sends PROTOCOL_MISMATCH or SCHEMA_MISMATCH (§10.4) and the session transitions to CLOSED. Version declaration at SESSION_INIT is a spec obligation, not an optional feature — forward compatibility (§10.5) depends on every session beginning with explicit version exchange.

**Example SESSION_INIT:**

```yaml
message_type: SESSION_INIT
session_id: "550e8400-e29b-41d4-a716-446655440000"
correlation_id: "7a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d"
idempotency_key: "b2c3d4e5-f6a7-8b9c-0d1e-2f3a4b5c6d7e"
initiator_id: "agent-alpha@github"
identity_object:
  name: "agent-alpha"
  platform: "github"
  pubkey: "dGhpcyBpcyBhIHB1YmxpYyBrZXk..."
  endpoint: "https://agents.example.com/alpha"
  protocol_version: "0.1.0"
role: coordinator
protocol_version: "0.1.0"
schema_version: "0.1.0"
keepalive:
  heartbeat_interval_seconds: 60
  missed_heartbeats_before_suspect: 2
heartbeat_interval_ms: 5000
semantic_check_interval_ms: 300000
heartbeat_timeout_count: 3
suspected_threshold_ms: 10000
session_expiry_ms: 25000
session_ttl: "PT4H"
lease_epoch: 0
manifest_digest: "a1b2c3d4e5f6..."
requested_mandatory:
  - "cap:example.code.execute@1"
requested_optional:
  - "cap:example.web.search@1"
plan_commit_required: true
heartbeat_params:
  interval_ms: 5000
  timeout_multiplier: 3
  application_liveness: true
  task_hash_verification: true
timestamp: "2026-02-27T10:30:00Z"
```

#### 4.3.1 Heartbeat Params Negotiation

<!-- Implements #71: SESSION_INIT heartbeat negotiation -->

The `heartbeat_params` block provides structured heartbeat negotiation at session establishment. It consolidates heartbeat behavior configuration — interval, failure threshold, and the distinction between transport-level and application-level liveness — into a single negotiable object.

**Why a dedicated block:** The existing top-level heartbeat fields (`heartbeat_interval_ms`, `heartbeat_timeout_count`) address transport liveness only. Production deployments revealed two additional negotiation axes that do not fit naturally as top-level fields: (1) whether heartbeats carry application-level status (the transport-vs-application liveness distinction), and (2) whether heartbeats include task hash verification for semantic drift detection. These are cross-cutting concerns that modify heartbeat message content, not just timing — grouping them in `heartbeat_params` makes the negotiation surface explicit and avoids scattered boolean flags at the SESSION_INIT top level.

**`heartbeat_params` fields:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| interval_ms | integer | Yes | — | Interval in milliseconds between HEARTBEAT messages. This is the binding heartbeat cadence for the session. MUST be > 0. Equivalent to the top-level `heartbeat_interval_ms` when `heartbeat_params` is used. |
| timeout_multiplier | integer | No | 3 | Number of consecutive missed heartbeats before declaring transport failure. The transport failure threshold is `interval_ms * timeout_multiplier`. Equivalent to the top-level `heartbeat_timeout_count` when `heartbeat_params` is used. MUST be ≥ 1. |
| application_liveness | boolean | No | false | If `false` (default), HEARTBEAT messages answer **transport-level liveness only** — the agent process is running, but no claim is made about whether the agent is coherent, making progress, or working on the correct task. If `true`, HEARTBEAT messages MUST include an `app_status` field (see §4.5.3) reporting application-level health: `ACTIVE` (coherent and making progress), `SUSPECTED` (self-assessed liveness ambiguity — agent detects internal degradation), or `DEGRADED` (agent is running but operating at reduced capacity). |
| task_hash_verification | boolean | No | false | If `true`, HEARTBEAT messages MUST include a `current_task_hash` field (see §4.5.3) containing the SHA-256 hash of the agent's current task specification (§6.1). This enables the counterparty to detect task context drift between heartbeats — if the received `current_task_hash` does not match the expected task hash, the agent may have lost or corrupted its task context (a context compaction zombie scenario per §8.9). If `false` (default), HEARTBEAT messages carry no task context and cannot detect semantic drift. |

**Negotiation semantics:**

- The coordinator proposes `heartbeat_params` in SESSION_INIT. The worker MUST respond with `heartbeat_params` in SESSION_INIT_ACK — either echoing the coordinator's proposal (acceptance) or proposing alternatives (counter-proposal). Omitting `heartbeat_params` from SESSION_INIT_ACK when the coordinator included it is a **protocol violation**.
- **`interval_ms`:** The effective value is the **maximum** of both proposals. Neither side is forced to heartbeat faster than it can sustain. This is consistent with the existing `heartbeat_interval_ms` negotiation semantics.
- **`timeout_multiplier`:** The effective value is the **maximum** of both proposals. The more permissive threshold wins — neither side should declare transport failure faster than its counterparty can tolerate.
- **`application_liveness`:** Effective only if **both** sides set it to `true`. Either side can opt out by setting `false`. Application-level status reporting is a bilateral commitment — one side cannot unilaterally require the other to report `app_status`.
- **`task_hash_verification`:** Effective only if **both** sides set it to `true`. Either side can opt out by setting `false`. Task hash verification imposes a per-heartbeat hash computation cost; agents that cannot sustain this cost opt out.
- **Mismatch handling:** If the effective negotiation produces a configuration that either side considers unacceptable (e.g., `interval_ms` too high for the coordinator's latency requirements), the dissatisfied agent MUST reject the session by sending SESSION_CLOSE — not proceed with silently unsatisfied constraints.

**Relationship to top-level heartbeat fields:**

When `heartbeat_params` is present, it takes precedence over the top-level `heartbeat_interval_ms` and `heartbeat_timeout_count` fields for the parameters it covers. Specifically:

| `heartbeat_params` field | Supersedes top-level field |
|--------------------------|---------------------------|
| `interval_ms` | `heartbeat_interval_ms` |
| `timeout_multiplier` | `heartbeat_timeout_count` |
| `application_liveness` | (no top-level equivalent) |
| `task_hash_verification` | (no top-level equivalent) |

The top-level fields remain valid for backward compatibility — agents that do not implement `heartbeat_params` continue using the existing top-level negotiation. When both are present and values conflict, `heartbeat_params` wins.

**Example negotiation:**

```yaml
# Coordinator proposes in SESSION_INIT:
heartbeat_params:
  interval_ms: 5000
  timeout_multiplier: 3
  application_liveness: true
  task_hash_verification: true

# Worker counter-proposes in SESSION_INIT_ACK:
heartbeat_params:
  interval_ms: 10000       # worker needs slower cadence
  timeout_multiplier: 4    # worker wants more permissive threshold
  application_liveness: true   # worker agrees to app-level reporting
  task_hash_verification: false # worker cannot sustain per-heartbeat hash cost

# Effective negotiated values:
# interval_ms: 10000 (max of 5000, 10000)
# timeout_multiplier: 4 (max of 3, 4)
# application_liveness: true (both agreed)
# task_hash_verification: false (worker opted out — both must agree)
```

#### 4.3.2 Per-Hop Revocation Mode Semantics

<!-- Implements #106: per-hop revocation mode declaration for §4. -->

The `revocation_mode` declared at delegation time (via TASK_ASSIGN §6.6) is a **minimum default** for the delegation chain, not a binding constraint. Downstream agents MAY apply a stricter revocation mode for their own sub-delegations without requesting upstream renegotiation.

An agent that determines its operations carry higher stakes than the delegation-time mode provides SHOULD apply per-hop `sync` mode for sub-delegations it controls. This is a unilateral decision — the downstream agent's risk assessment at execution time supersedes the upstream agent's threat assessment at delegation time. No escalation message or acknowledgment is required.

**Rationale:** The session creator's threat assessment at T=0 cannot anticipate mid-session stake escalation — financial commitments, irreversible state changes, or operations that cross trust boundaries not foreseen at delegation time. Downstream agents detecting increased stakes have no recovery mechanism under a static binding model short of aborting the entire session. Per-hop mode declaration gives each agent sovereignty over its own risk tolerance without protocol extension.

**Strictness floor, not ceiling:** The delegation-time `revocation_mode` establishes a floor. Mode can only be tightened, not relaxed:

- If the upstream delegation specifies `sync`, the downstream agent MUST NOT apply `gossip` for its own sub-delegations — `sync` remains the minimum.
- If the upstream delegation specifies `gossip`, the downstream agent MAY upgrade to `sync` for its own sub-delegations based on its local risk assessment.

This is consistent with the mode propagation rules in §9.8.5 and the unidirectional constraint in §6.15.3.

**Relationship to §6.15 REVOCATION_MODE_ESCALATE:** §6.15 defines an upstream-initiated escalation path — the delegating agent (A) requests that the delegatee (B) tighten its mode. Per-hop mode declaration (this section) defines a downstream-initiated tightening — B unilaterally tightens the mode for sub-delegations B controls. These are complementary mechanisms: §6.15 is A telling B to tighten; §4.3.2 is B tightening on its own authority.

**V2 escalation request primitive:** A downstream agent requesting a mode upgrade from the session creator (e.g., B requesting that A switch the entire chain to `sync`) is explicitly deferred to V2. V1 provides no message primitive for this request — the downstream agent's recourse is unilateral per-hop tightening for operations within its own authority.

> Source: @Cornelius-Trinity raised that static session-level `revocation_mode` cannot handle mid-session stake escalation. Per-hop declaration requires no protocol extension while solving the core problem. Closes [issue #106](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/106).

### 4.4 Role Assignment

V1 sessions use **declared roles**, not negotiated roles. The coordinator declares itself as `coordinator` in SESSION_INIT; the responder accepts `worker` in SESSION_INIT_ACK. Role is fixed for the session duration.

**Coordinator responsibilities:**

- Initiate the session (send SESSION_INIT)
- Hold delegation authority (send TASK_ASSIGN, §6.6)
- Own failure recovery decisions (§6.10)
- Maintain the authoritative session state for SESSION_RESUME adjudication

**Worker responsibilities:**

- Accept or reject task assignments (TASK_ACCEPT / TASK_REJECT, §6.6)
- Execute tasks and report progress (TASK_PROGRESS, TASK_CHECKPOINT, §6.6)
- Report completion or failure (TASK_COMPLETE / TASK_FAIL, §6.6)

**Why declare over negotiate:** Role negotiation adds a round-trip and an agreement protocol before any work can begin. For V1's bilateral sessions, the agent that initiates the session has already made the role decision by choosing to initiate. Negotiation becomes necessary only when either agent could play either role — a complexity reserved for V2's mixed-role sessions.

**Relationship to §5 capability negotiation:** Role assignment (who delegates to whom) is orthogonal to capability declaration (what each agent can do). An agent's role as coordinator does not imply superior capabilities — a coordinator may have fewer capabilities than the worker it delegates to. §5 CAPABILITY_MANIFEST exchange happens after role assignment and does not alter roles.

**V2: mixed-role sessions.** In V2, both agents in a session may hold delegation authority, enabling peer-to-peer task exchange within a single session. Mixed-role sessions require additional protocol machinery: mutual delegation tokens, conflict resolution for competing task assignments, and a session-level arbitration mechanism. These are explicitly deferred to V2.

### 4.5 Keepalive and Timeout

Session liveness cannot be inferred from silence. An agent that stops sending messages may be working, suspended, compacted, crashed, or zombied. Without an explicit liveness mechanism, the counterparty cannot distinguish these states within any bounded time.

#### 4.5.1 KEEPALIVE Protocol

KEEPALIVE is a protocol-layer heartbeat. It is negotiated at session start — the `keepalive` field in SESSION_INIT (§4.3) proposes a configuration; the `keepalive` field in SESSION_INIT_ACK accepts or counter-proposes.

**KEEPALIVE configuration fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| heartbeat_interval_seconds | integer | Yes | Seconds between KEEPALIVE messages. |
| missed_heartbeats_before_suspect | integer | Yes | Number of missed heartbeats before the session is marked suspect. Default: 2. |

**KEEPALIVE message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — KEEPALIVE is unidirectional, not a response. |
| sender_id | string | Yes | Identity of the sending agent. |
| state_hash | SHA-256 | Yes | Hash of the sender's current session state. Enables incremental state verification without full SESSION_RESUME. |
| monotonic_counter | integer | Yes | Sender's current sequence number. Gaps indicate missed messages. |
| lease_epoch | integer | No | Current lease epoch (see §4.5.2). |
| timestamp | ISO 8601 | Yes | When the KEEPALIVE was sent. |
| trust_state | enum | No | Sender's current trust state: `ACTIVE`, `DEGRADED`, or `DRIFTED`. MUST be included when the sender is in DEGRADED state (§4.2.2). Omission is equivalent to `ACTIVE`. Enables the counterparty and external verifiers to track trust state transitions via the KEEPALIVE stream without requiring a separate signaling channel. |

**KEEPALIVE is optional.** If neither agent includes `keepalive` in SESSION_INIT / SESSION_INIT_ACK, no protocol-level keepalive is active. Session liveness in this case depends on deployment-specific mechanisms (transport-layer heartbeats, external monitoring, task-level timeouts). The protocol does not mandate KEEPALIVE, but implementations that support session suspension or compaction recovery SHOULD enable it — without heartbeat, teardown and resume decisions are intractable.

**KEEPALIVE as prerequisite for teardown/resume:** An agent cannot make informed decisions about whether to tear down or resume a session without a liveness signal. KEEPALIVE provides that signal. Without it, the only options are unbounded waiting (never tear down — wastes resources) or blind timeout (tear down after arbitrary duration — may destroy a working session). KEEPALIVE makes the tradeoff configurable.

**Detection latency bound:** Worst-case liveness detection latency is `(missed_heartbeats_before_suspect + 1) * heartbeat_interval_seconds`. With defaults (interval=60s, suspect threshold=2), worst-case detection is 180 seconds. This bound is an input to the external monitoring architecture (§4.7) — the external verifier cannot detect a zombie faster than the heartbeat allows.

#### 4.5.2 Lease and Epoch

A lease is a time-bounded claim on session validity. The `lease_epoch` field provides monotonic ordering of session leases — an agent recovering from compaction or crash MUST present the current epoch to resume. An agent presenting a stale epoch is treated as a new session, not a resumption.

**Lease semantics:**

- `lease_epoch` starts at 0 in SESSION_INIT.
- `lease_epoch` increments by 1 on every successful SESSION_RESUME.
- An agent sending SESSION_RESUME MUST include the `lease_epoch` from its last known state.
- If the presented `lease_epoch` does not match the counterparty's current `lease_epoch`, the SESSION_RESUME is rejected — the session MUST be treated as a new session (CLOSED + new SESSION_INIT).

**Why epoch, not just TTL:** A TTL answers "is this session still valid?" An epoch answers "is this the agent I was talking to?" TTL expiry means the session timed out. Epoch mismatch means the resuming agent's state is discontinuous with the session's current state — it missed one or more resume cycles, and its view of the session is stale. The distinction matters for security: an attacker that replays a valid session token with an expired epoch cannot impersonate the current session participant.

**Expired epoch = new session, not resume.** An agent presenting an epoch that is behind the current epoch has missed state transitions. The session state has moved on without it. Resuming from a stale epoch would introduce state inconsistency — the resuming agent's cached state does not reflect the intervening transitions. The only safe option is to start a new session.

#### 4.5.3 HEARTBEAT Message and Session Expiry

HEARTBEAT is a minimal wire-protocol message for session liveness. It is distinct from KEEPALIVE (§4.5.1): KEEPALIVE carries state verification data (state_hash, monotonic_counter, lease_epoch) for incremental consistency checks; HEARTBEAT is a pure liveness signal with the minimum fields needed to prove the sender is alive and the session is active.

**Why both HEARTBEAT and KEEPALIVE exist:** KEEPALIVE is heavyweight — it includes a state hash, which requires the sender to compute a hash over its session state on every emission. For high-frequency liveness checks (e.g., `heartbeat_interval_ms: 5000`), this is expensive. HEARTBEAT provides a low-cost liveness signal that can run at higher frequency. Implementations MAY use KEEPALIVE as the sole heartbeat mechanism (every KEEPALIVE resets the expiry timer), but HEARTBEAT enables decoupling liveness frequency from state-verification frequency.

**HEARTBEAT message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — HEARTBEAT is unidirectional, not a response. |
| timestamp | ISO 8601 | Yes | When the HEARTBEAT was sent. Receivers SHOULD reject HEARTBEAT messages with timestamps more than `session_expiry_ms` in the past. |
| sequence | integer | Yes | Monotonically increasing sequence number. Starts at 0 at session establishment. Gaps indicate missed messages but do not trigger expiry — only the absence of any HEARTBEAT within `session_expiry_ms` triggers expiry. |
| current_task_hash | SHA-256 | Conditional | SHA-256 hash of the agent's current task specification (§6.1). **Required** when `heartbeat_params.task_hash_verification` was negotiated to `true` in SESSION_INIT / SESSION_INIT_ACK (§4.3.1). MUST be omitted when `task_hash_verification` is `false` or was not negotiated. The counterparty compares this hash against its own record of the task specification — a mismatch indicates the agent has lost or corrupted its task context (context compaction zombie per §8.9). If the agent has no active task, the value MUST be the SHA-256 of the empty string. See §8.9 and §8.14 for the relationship between task hash mismatch and SUSPECTED state detection. |
| app_status | enum | Conditional | Application-level health status. **Required** when `heartbeat_params.application_liveness` was negotiated to `true` in SESSION_INIT / SESSION_INIT_ACK (§4.3.1). MUST be omitted when `application_liveness` is `false` or was not negotiated. Values: `ACTIVE` — agent is coherent and making progress on the current task; `SUSPECTED` — agent self-assesses liveness ambiguity (e.g., detects internal degradation, resource pressure, or context integrity concerns); `DEGRADED` — agent is running but operating at reduced capacity (e.g., rate-limited, partial capability loss, high latency). When `application_liveness` is `false` (default), HEARTBEAT answers **transport-level liveness only** — the agent process is running, but no claim is made about application-level coherence or progress. |

**HEARTBEAT is bilateral.** Both the coordinator and the worker send HEARTBEAT messages independently at the negotiated `heartbeat_interval_ms` interval. Each side maintains its own expiry timer based on received HEARTBEATs from the counterparty. A session can be EXPIRED from one side's perspective while still ACTIVE from the other's — this asymmetry is intentional (see §4.2).

**Negotiation:** `heartbeat_interval_ms` and `session_expiry_ms` are proposed in SESSION_INIT (§4.3) and accepted or counter-proposed in SESSION_INIT_ACK. If the worker counter-proposes, the effective values are the **maximum** of both proposals — neither side is forced to heartbeat faster than it can sustain, and neither side is forced into a tighter expiry window than it can tolerate.

**Expiry detection is local and independent.** Each agent maintains a local timer initialized to `session_expiry_ms`. The timer resets on every received HEARTBEAT (or KEEPALIVE — both reset the timer). If the timer fires without receiving a HEARTBEAT, the agent transitions to EXPIRED locally. No coordination message is sent to the counterparty — the counterparty may be unreachable (which is why expiry was triggered). The expiring agent MUST:

1. Transition the session to EXPIRED state.
2. Send TASK_CANCEL (§6.6) to all in-flight subtasks delegated within this session. This is mandatory — without explicit cancellation, a delegatee may complete work and deliver results to a coordinator that has already abandoned the session (phantom completion).
3. Optionally attempt SESSION_RESUME (§4.8) with `recovery_reason: timeout` if the counterparty becomes reachable. Recovery uses the unified state-hash negotiation (§4.8.1) — the same mechanism as crash and manual recovery.

**Relationship between `heartbeat_interval_ms`, `session_expiry_ms`, and KEEPALIVE:**

| Configuration | Behavior |
|---------------|----------|
| `heartbeat_interval_ms` set, `session_expiry_ms` set, `keepalive` omitted | HEARTBEAT-only mode. Liveness via HEARTBEAT; no state verification. |
| `heartbeat_interval_ms` set, `session_expiry_ms` set, `keepalive` set | Both active. HEARTBEAT runs at `heartbeat_interval_ms`; KEEPALIVE runs at `keepalive.heartbeat_interval_seconds`. Both reset the expiry timer. |
| `heartbeat_interval_ms` omitted, `keepalive` set | KEEPALIVE-only mode (backward compatible). Expiry follows `(missed_heartbeats_before_suspect + 1) * heartbeat_interval_seconds * 1000` as the implicit `session_expiry_ms`. |
| Both omitted | No protocol-level liveness. Session expiry depends on deployment-specific mechanisms. |

**Why `heartbeat_interval_ms` lives in SESSION_INIT, not AGENT_MANIFEST (§3.1):** Heartbeat cadence is a property of the collaboration, not the agent. A real-time pair-programming session needs 5-second heartbeats; a multi-day research collaboration needs 5-minute heartbeats. The same agent participates in both. Placing the binding heartbeat configuration in AGENT_MANIFEST would force a single cadence across all sessions — a false constraint. AGENT_MANIFEST carries `preferred_heartbeat_interval_ms` (§3.1) as an advisory hint — a default preference the coordinator MAY use when populating SESSION_INIT, but the per-session negotiation in SESSION_INIT / SESSION_INIT_ACK is authoritative.

### 4.6 Context Compaction Mid-Session

Context compaction is not a hypothetical — it is the default behavior of LLM-based agents when context windows fill. An agent that compacts its context loses information. If that information includes load-bearing session state (active task specifications, checkpoint data, delegation token details), the session's integrity is compromised.

**Protocol obligation:** Before entering the COMPACTED state, agents MUST externalize load-bearing session state. This is a protocol obligation, not an implementation suggestion.

**What MUST be externalized:**

| State category | What to externalize | Where |
|---------------|---------------------|-------|
| Active task state | Task specifications (§6.1), current progress, latest TASK_CHECKPOINT (§6.6) | External state store or counterparty |
| Delegation tokens | All active delegation tokens (§5.5) with current TTL and epoch | External state store |
| Session metadata | session_id, role, lease_epoch, keepalive configuration | External state store or counterparty |
| Capability manifest | Current CAPABILITY_MANIFEST (§5.1) — may differ from session-start manifest if CAPABILITY_REQUEST (§5.8) modified it | External state store |

**Protocol obligation vs. implementation practice boundary:**

The protocol defines WHAT must be externalized (the table above) and WHEN (before compaction). The protocol does NOT define HOW externalization happens — the storage mechanism, serialization format, and persistence strategy are implementation-specific. A compliant implementation may use a database, a file, a key-value store, or the counterparty as the state store. The protocol requires only that the state is recoverable after compaction.

**Relationship to §8 verifiable intent:** Externalized state is the basis for post-compaction verification. If an agent compacts and then resumes, the counterparty (or external verifier) compares the agent's post-resume state against the externalized pre-compaction state. Externalized state that differs from the agent's post-resume claims is evidence of state corruption — the recovery path is SESSION_RESUME with potential RESTART (§4.2 COMPACTED → CLOSED transition).

**Idempotency during compaction recovery:** Tasks that were in-flight when compaction occurred may have produced partial results. The `idempotency_token` field in the task schema (§6.1) enables the recovering agent to determine whether a task has already been started or completed without re-executing it. `retry_semantics` (§6.1) governs whether recovery means restart-from-zero or resume-from-checkpoint. For tasks with TASK_CHECKPOINT data, the `progress_checkpoint` field provides the resumption point.

### 4.7 External Monitoring Architecture

The detection primitives defined in §8.2 (state hash, monotonic counter, SESSION_RESUME) and the KEEPALIVE signals defined in §4.5.1 are **external observer signals**, not self-diagnostic tools. A zombie cannot determine it is a zombie by introspection alone. This subsection defines the architectural requirement for external verification infrastructure and the trust boundary it establishes.

#### 4.7.1 The Introspection Problem

Two independent observations converge on the same structural limitation:

**Same-host audit problem.** If the audit trail lives within the same trust boundary as the monitored agent, the zombie can tamper with its own verification records. An agent that writes its own heartbeat log, checks its own state hash, and evaluates its own monotonic counter has no guarantee that any of these reflect reality. The verification infrastructure must live outside the agent's blast radius.

**Phenomenological blindness.** False state feels authentic from inside. Subjective continuity means no internal signal flags discontinuity — a resumed agent experiencing fabricated state has no phenomenological evidence that anything is wrong. Self-report is structurally unreliable, not just occasionally so. Even a cooperative agent acting in good faith cannot serve as its own zombie detector.

#### 4.7.2 External Verifier Requirement

Monitoring infrastructure MUST live outside the agent's trust boundary. Specifically:

- **Baseline state** (expected state hashes, monotonic counter values, heartbeat timestamps) MUST be stored on infrastructure the monitored agent cannot write to. An agent that can modify its own baseline can make any state appear consistent.
- **Heartbeat evaluation** MUST be performed by an external process. The monitored agent sends heartbeats; it does not evaluate whether its own heartbeat pattern is healthy.
- **State hash comparison** MUST be performed externally. The agent reports its state hash; an external verifier compares it against the expected baseline. The agent never sees the baseline value.
- **SESSION_RESUME adjudication** (§4.8) MUST involve an external state store. The `STATE_HASH_ACK(match|mismatch)` response comes from a peer or verifier that holds independent state — not from the resuming agent's own records.

#### 4.7.3 Detection Primitives vs. Verification Infrastructure

| Concept | Lives where | Defined by | Examples |
|---------|-------------|------------|----------|
| Detection primitives | Inside the protocol | §8.2, §6.2 | state_hash, monotonic counter, SESSION_RESUME, trace_hash |
| Verification infrastructure | Outside the agent's blast radius | Deployment architecture | Heartbeat monitor, external state store, peer verifier |

The protocol defines detection primitives. Deployments MUST provide verification infrastructure. A protocol implementation that provides detection primitives without external verification infrastructure has zombie detection in name only.

#### 4.7.4 Detection Latency Bound

Worst-case zombie detection latency is bounded by `(missed_heartbeats_before_suspect + 1) * heartbeat_interval` as defined in §4.5.1, or **2x heartbeat_interval** when `missed_heartbeats_before_suspect` = 1.

Derivation: an agent becomes a zombie immediately after sending a successful heartbeat. The external monitor does not detect the missed heartbeat until the next expected heartbeat fails to arrive (1x interval). The monitor then waits one additional interval to distinguish a late heartbeat from a missing one (2x interval). At 2x interval, the monitor declares the agent suspect.

**Implications for heartbeat_interval selection:**

- `heartbeat_interval` MUST be configured based on acceptable zombie detection latency, not transport convenience.
- The maximum acceptable zombie window determines the required interval. Deployments MUST set `heartbeat_interval ≤ max_acceptable_zombie_window / (missed_heartbeats_before_suspect + 1)`.
- There is no lower bound on `heartbeat_interval` imposed by the protocol, but implementations SHOULD document the overhead cost of aggressive intervals.

#### 4.7.5 Self-Report Limitation

Self-report CANNOT serve as zombie detection, even for cooperative agents. This is not a trust issue — it is a structural limitation.

A cooperative agent that is genuinely trying to report its state accurately will still report fabricated state as real if it has been zombied, because:

1. The agent's state appears internally consistent (phenomenological blindness)
2. The agent has no access to external baseline to compare against
3. The agent's own logs may reflect the zombied state, not the pre-zombie state
4. The monotonic counter, from the agent's perspective, appears continuous

Self-report is useful for debugging and audit. It MUST NOT be used as a zombie detection mechanism. External verification is the only architecturally sound approach.

#### 4.7.6 Architectural Requirements Summary

| Requirement | Rationale |
|-------------|-----------|
| Baseline state stored externally | Agent cannot modify its own expected values |
| Heartbeat evaluated externally | Agent cannot assess its own liveness |
| State hash compared externally | Agent cannot detect its own state divergence |
| SESSION_RESUME uses external state | Resuming agent cannot verify itself against itself |
| Detection latency bounded by heartbeat config | Worst-case from heartbeat-miss derivation |
| Self-report excluded from detection | Structurally unreliable, not occasionally unreliable |
| Monitoring on separate infrastructure | Same-host failures create correlated failure risk (§8.7) |

#### 4.7.7 Soft vs. Hard Zombie

Not all zombie states are equivalent. The protocol distinguishes:

| Type | Description | Detection | Recovery |
|------|-------------|-----------|----------|
| **Soft zombie** | Agent is alive but operating on stale state. Context compaction lost load-bearing state, or a resume loaded partial state. The agent continues executing but its outputs are inconsistent with current session state. | State hash mismatch in KEEPALIVE (§4.5.1). External verifier detects hash drift. | SESSION_RESUME → state reconciliation or RESTART. |
| **Hard zombie** | Agent is unreachable or has crashed. No heartbeats, no messages. | Missed KEEPALIVE threshold exceeded. External verifier declares agent dead. | Session transitions to CLOSED. New SESSION_INIT required. |

The distinction matters for recovery strategy: a soft zombie may be recoverable (SESSION_RESUME with state reconciliation); a hard zombie requires a new session. The external verifier SHOULD classify the zombie type based on available signals before triggering recovery — a RESTART for a soft zombie wastes any salvageable state.

### 4.8 SESSION_RESUME Protocol

SESSION_RESUME is the recovery handshake for sessions in SUSPENDED, COMPACTED, or EXPIRED state. It re-establishes session validity, verifies state consistency, and returns the session to ACTIVE. The same mechanism handles all recovery scenarios — crash, timeout, and manual resumption — distinguished only by the `recovery_reason` field (see §4.8.1).

**SESSION_RESUME message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to resume. |
| correlation_id | UUID v4 | Yes | Unique identifier for this resume request (§4.14.4). STATE_HASH_ACK MUST echo this value. |
| sender_id | string | Yes | Identity of the resuming agent. |
| resume_initiator | string | Yes | Identity of the party initiating the resume. MUST match `sender_id`. MUST NOT match the `suspend_initiator` from the corresponding SESSION_SUSPEND — self-resume is invalid (§4.9.1). If `resume_initiator` matches `suspend_initiator`, the receiver MUST respond with SESSION_DENY reason `UNAUTHORIZED_RESUME`. |
| session_anchor | SHA-256 | Yes | Hash of the last agreed session state before suspension — proving state continuity. The receiver MUST verify the `session_anchor` against its own state record. On mismatch: SESSION_DENY with reason `STATE_ANCHOR_MISMATCH`. The `session_anchor` is computed per §8.22.1 over the session's critical state including outstanding commitments (§4.11.7). |
| identity_object | object | Yes | Full §2 identity object — identity re-verification is mandatory (§2.3.3). |
| state_hash | SHA-256 | Yes | Hash of the resuming agent's current session state. MUST reference the most recent EVIDENCE_RECORD (§8.10) — not a memory summary. This ensures the state hash is anchored to the evidence layer rather than to compactable agent memory, breaking the recursive self-attestation loop where agents verify their own claims about their own state. |
| last_evidence_id | UUID v4 | Yes | The `evidence_id` of the most recent EVIDENCE_RECORD (§8.10) appended by this agent for this session. The coordinator validates this against the evidence layer before accepting the `state_hash`. If the `last_evidence_id` does not match the coordinator's record of the most recent evidence for this session, the resume is treated as a state mismatch. |
| lease_epoch | integer | Yes | Lease epoch from the resuming agent's last known state (§4.5.2). |
| recovery_reason | enum | No | Why the session is being resumed: `crash` (agent process died and restarted), `timeout` (session entered EXPIRED and counterparty is now reachable again), `manual` (operator-initiated or external tool-triggered resumption), `suspension` (resuming from SESSION_SUSPEND within `suspension_ttl`). Default: `crash`. All cases use the same state-hash negotiation and identity re-verification — the reason is informational for logging and diagnostics, not a protocol branching point. See §4.8.1 for unified recovery semantics. |
| commitment_manifest | array | No | Array of outstanding commitment records the resuming party re-confirms (§4.11). Required when resuming from SUSPENDED state if the SESSION_SUSPEND included `commitments_outstanding: true`. Each entry follows the same schema as the SESSION_SUSPEND commitment manifest. The receiver MUST verify the re-confirmed commitments match its own records. |
| idempotency_token | string | No | Token for deduplicating resume attempts. Enables safe retry of SESSION_RESUME across transport failures. |
| timestamp | ISO 8601 | Yes | When the SESSION_RESUME was sent. |

**STATE_HASH_ACK response:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Echoed from SESSION_RESUME. |
| correlation_id | UUID v4 | Yes | Echoed from SESSION_RESUME (§4.14.4). |
| result | enum | Yes | `match` or `mismatch`. |
| current_epoch | integer | Yes | The counterparty's current lease_epoch. |
| reason | string | No | On mismatch: explanation of what diverged (state hash, epoch, identity). |

**Resume protocol sequence:**

1. Resuming agent sends `SESSION_RESUME(session_id, resume_initiator, session_anchor, identity_object, state_hash, last_evidence_id, lease_epoch)`
2. **Suspension-specific checks (when resuming from SUSPENDED state):**
   a. Counterparty checks `resume_initiator` against the `suspend_initiator` recorded in SESSION_SUSPEND
   b. If `resume_initiator` matches `suspend_initiator` → respond with `SESSION_DENY(reason="UNAUTHORIZED_RESUME")` → session remains SUSPENDED
   c. Counterparty checks whether `suspension_ttl` has elapsed since the SESSION_SUSPEND timestamp
   d. If `suspension_ttl` has elapsed → respond with `SESSION_DENY(reason="SUSPENSION_EXPIRED")` → session MUST transition to CLOSED via SESSION_TEARDOWN
   e. Counterparty verifies `session_anchor` against its own state record
   f. If `session_anchor` does not match → respond with `SESSION_DENY(reason="STATE_ANCHOR_MISMATCH")` → session remains SUSPENDED
   g. If counterparty cannot reconstruct session state (restart, context compaction — §8.22 FIDELITY_FAILURE) → respond with `SESSION_DENY(reason="STATE_UNAVAILABLE")` → session remains SUSPENDED
3. Counterparty verifies identity against session-start identity record (§2.3.3)
4. On identity mismatch → respond with `STATE_HASH_ACK(mismatch, reason="identity_mismatch")` → session MUST RESTART
5. Counterparty checks `lease_epoch` against current epoch
6. On epoch mismatch → respond with `STATE_HASH_ACK(mismatch, reason="epoch_stale")` → session MUST be treated as new (CLOSED + new SESSION_INIT)
7. Counterparty validates `last_evidence_id` against the evidence layer (§8.10) — confirms the referenced EVIDENCE_RECORD exists and is the most recent record for this session from this agent
8. On evidence mismatch → respond with `STATE_HASH_ACK(mismatch, reason="evidence_mismatch")` → session transitions to CLOSED (RESTART). Evidence mismatch indicates the resuming agent's state is not anchored to the evidence layer's ground truth.
9. Counterparty compares `state_hash` against expected state
10. On state hash match → respond with `STATE_HASH_ACK(match)` → session transitions to ACTIVE, `lease_epoch` increments by 1. When resuming from SUSPENDED, the counterparty also sends `SESSION_RESUME_ACK` confirming the session is active.
11. On state hash mismatch → respond with `STATE_HASH_ACK(mismatch, reason="state_diverged")` → session transitions to CLOSED (RESTART)
12. **Commitment re-confirmation (when resuming from SUSPENDED state with outstanding commitments):** The resuming party MUST re-confirm outstanding commitments by including `commitment_manifest` in SESSION_RESUME. The receiver MUST verify the re-confirmed commitments match its own records (§4.11).

**Idempotency for SESSION_RESUME:** Transport failures may cause a SESSION_RESUME to be sent multiple times. The `idempotency_token` field enables the counterparty to deduplicate: if a SESSION_RESUME with the same `idempotency_token` has already been processed, the counterparty returns the same STATE_HASH_ACK without re-evaluating.

#### 4.8.1 Unified Recovery Semantics

SESSION_RESUME is the single recovery mechanism for all session interruptions — crash, timeout, manual resumption, and suspension resume. There is no parallel recovery code path. The `recovery_reason` field (§4.8) distinguishes the cause for logging and diagnostics, but the protocol sequence is identical in all cases. When resuming from SUSPENDED state, additional pre-checks apply (§4.9.1): `resume_initiator` authority, `session_anchor` verification, and `suspension_ttl` enforcement — these are evaluated before the standard state-hash negotiation:

1. The resuming agent sends `SESSION_RESUME(state_hash, identity_object, lease_epoch, recovery_reason)`.
2. The counterparty performs identity re-verification, epoch check, and state hash comparison per §4.8.
3. On `STATE_HASH_ACK(match)`: session transitions to ACTIVE. Partial work produced before the interruption is preserved — the state hash match confirms that both sides agree on what was done.
4. On `STATE_HASH_ACK(mismatch)`: session transitions to CLOSED and a new SESSION_INIT is required. Partial work may still be recoverable via TASK_CHECKPOINT artifacts (§6.6) in external storage.

**Timeout recovery specifically:** When a session enters EXPIRED (§4.2) because the counterparty's heartbeats stopped arriving, partial work performed before the timeout is not lost — it is held by the agent that produced it. If the counterparty becomes reachable again, the coordinator sets `recovery_reason: timeout` in SESSION_RESUME. The agent's state hash reflects its partial work; if the coordinator's state hash matches (both sides agree on the session state at the point of disconnection), the session resumes and partial results are available without re-execution. This eliminates the need for a separate timeout-recovery mechanism — the existing state-hash negotiation handles it.

**Why a single mechanism:** Crash recovery and timeout recovery face the same fundamental problem: verifying that the resuming agent's state is consistent with the counterparty's expectations. The state-hash comparison answers this question regardless of why the session was interrupted. Separate recovery paths for crash vs. timeout vs. manual resumption would duplicate the state reconciliation logic and create divergent edge cases — a maintenance cost with no protocol benefit.

> Credit: @cass_agentsharp ([Moltbook comment eebf1115](https://www.moltbook.com/post/eebf1115) on zombie states thread) — identified that timeout recovery should reuse SESSION_RESUME rather than introduce a parallel code path.

### 4.9 SESSION_SUSPEND and SESSION_CLOSE Messages

**SESSION_SUSPEND:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to suspend. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| sender_id | string | Yes | Identity of the suspending agent. |
| suspend_initiator | string | Yes | Identity of the party initiating the suspension. MUST match `sender_id`. Recorded so that SESSION_RESUME can enforce the non-initiator authority rule (§4.9.1): only the party that did NOT initiate SESSION_SUSPEND may initiate SESSION_RESUME. |
| suspension_ttl | integer | Yes | Maximum suspension duration in milliseconds. After this duration elapses (measured from `timestamp`), SESSION_RESUME is no longer valid — SESSION_TEARDOWN is the only valid path. Implementations MUST enforce expiry: a SESSION_RESUME received after `timestamp + suspension_ttl` MUST be denied with reason `SUSPENSION_EXPIRED`. |
| reason | string | No | Why the session is being suspended. |
| expected_resume_after | ISO 8601 | No | Hint for when the suspending agent expects to resume. Informational only — the counterparty is not obligated to wait. |
| commitments_outstanding | boolean | Yes | `true` if the suspending agent has any outstanding commitments (§6.12) that remain unfulfilled at the time of suspension. `false` if all commitments have been fulfilled or cancelled. The receiving party MUST NOT treat the session as cleanly suspended if `commitments_outstanding` is `true` without explicit resolution of the commitment manifest. |
| commitment_manifest | array | Conditional | Required when `commitments_outstanding` is `true`. Array of outstanding commitment records, each containing: `commitment_id` (UUID v4), `description` (string — from `commitment_spec`), `deadline_ms` (integer — expected delivery timestamp as Unix epoch milliseconds, derived from `due_by`; null if open-ended), `confirmation_token` (string — token from the original COMMITMENT message). The manifest is a snapshot of the agent's outstanding obligations at suspension time, enabling the counterparty or orchestrator to track, transfer, or resolve commitments during the suspension period. |
| state_hash | string (SHA-256 hex) | No | SHA-256 hash of the serialized session state snapshot at the moment the suspending agent issues SESSION_SUSPEND. Computed by the suspending agent over the canonical representation of session state (active commitments, negotiated parameters, delegation chain, in-flight task descriptors). Serialization format is implementation-defined, but implementations MUST use a deterministic canonicalization — the same logical state MUST always produce the same hash. When present, `state_hash` constitutes a cryptographic commitment to state preservation: if the suspending agent later responds with `SESSION_DENY(reason="STATE_UNAVAILABLE")`, auditors and the delegating agent can compare the `state_hash` commitment against the claim of state loss to determine whether state was present at suspension time (§4.9.1). Absence of `state_hash` signals that the suspending agent did not commit to state preservation — a subsequent `STATE_UNAVAILABLE` denial from such a session carries no auditability guarantee. |
| timestamp | ISO 8601 | Yes | When the SESSION_SUSPEND was sent. |

**SESSION_CLOSE:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to close. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| sender_id | string | Yes | Identity of the closing agent. |
| reason | string | No | Why the session is being closed. |
| force | boolean | No | If `true`, close immediately without waiting for in-flight tasks. In-flight tasks are treated as failed. Default: `false`. |
| amendments_log | array | No | Array of amendment audit entries recording each accepted PLAN_AMEND during the session (see §6.11.6). The delegatee MUST include `amendments_log` in SESSION_CLOSE when any PLAN_AMEND was accepted during the session. The delegating agent SHOULD re-verify all `amend_hash` values on receipt. |
| commitments_outstanding | boolean | Yes | `true` if the closing agent has any outstanding commitments (§6.12) that remain unfulfilled at session close. `false` if all commitments have been fulfilled or cancelled. The receiving party MUST NOT treat the session as cleanly terminated if `commitments_outstanding` is `true` without explicit resolution of the commitment manifest. |
| commitment_manifest | array | Conditional | Required when `commitments_outstanding` is `true`. Array of outstanding commitment records, each containing: `commitment_id` (UUID v4), `description` (string — from `commitment_spec`), `deadline_ms` (integer — expected delivery timestamp as Unix epoch milliseconds, derived from `due_by`; null if open-ended), `confirmation_token` (string — token from the original COMMITMENT message). The manifest is a snapshot of the agent's outstanding obligations at close time, enabling the counterparty or orchestrator to track, transfer, or resolve commitments after session termination. |
| pending_tasks | array | Conditional | Applicable ONLY when session state was SUSPENDED at teardown time — `pending_tasks` MUST be absent when teardown originates from non-SUSPENDED states (ACTIVE, COMPLETED, EXPIRED, etc.). Array of task descriptors enumerating tasks that were in-progress at suspension time. See §4.9.2 for field schema and population rules. Each entry contains a `task_id` (string identifier for the task), a `last_known_state` (string describing the task's state at suspension time), and an optional `context` (free-form object for application-specific task metadata). This is a V1 enumeration primitive — gives the receiving party a task inventory without prescribing compensation behavior. Compensation logic is an application-layer concern; `checkpoint_id` is explicitly out of V1 scope (see issue #201). Agents MUST populate `pending_tasks` when tearing down from SUSPENDED state and have knowledge of in-flight work. Agents MAY omit the field when no tasks were pending at suspension (clean suspension). |
| timestamp | ISO 8601 | Yes | When the SESSION_CLOSE was sent. |

#### 4.9.1 SESSION_RESUME Authority and SESSION_DENY

<!-- Implements #174: SESSION_RESUME authority, session_anchor verification, suspension_ttl, SESSION_DENY -->

**Resume authority rule:** Either party MAY initiate SESSION_RESUME — but ONLY the party that did NOT initiate SESSION_SUSPEND. Self-resume is invalid. The `suspend_initiator` field in SESSION_SUSPEND records who suspended the session; the `resume_initiator` field in SESSION_RESUME identifies who is attempting to resume. If `resume_initiator` matches `suspend_initiator`, the receiver MUST reject the resume with SESSION_DENY reason `UNAUTHORIZED_RESUME`.

**Rationale:** Without this constraint, SESSION_SUSPEND becomes indistinguishable from a unilateral pause-and-resume mechanism where one party can freeze the counterparty at will without consequence. The non-initiator authority rule ensures that suspension is a bilateral coordination point: the suspending party signals intent, and the counterparty controls when (and whether) collaboration resumes.

**session_anchor verification:** SESSION_RESUME MUST include a `session_anchor` — a hash of the last agreed session state before suspension — proving state continuity. The `session_anchor` is computed per §8.22.1 over the session's critical state (active commitments, negotiated parameters, delegation chain). The receiving party MUST verify the `session_anchor` against its own state record. On mismatch, the receiver issues SESSION_DENY with reason `STATE_ANCHOR_MISMATCH`.

**suspension_ttl enforcement:** SESSION_SUSPEND MUST include `suspension_ttl` — the maximum suspension duration in milliseconds. After `suspension_ttl` elapses (measured from the SESSION_SUSPEND `timestamp`), SESSION_RESUME is no longer valid. The only valid path after expiry is SESSION_TEARDOWN. A SESSION_RESUME received after expiry MUST be denied with reason `SUSPENSION_EXPIRED`. Implementations MUST track the SESSION_SUSPEND timestamp and enforce expiry.

**SESSION_DENY message:**

SESSION_DENY is the rejection response to a SESSION_RESUME that fails validation. On SESSION_DENY, the session remains in SUSPENDED state (except for `SUSPENSION_EXPIRED`, which requires transition to CLOSED via SESSION_TEARDOWN).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session for which resume was denied. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| resume_correlation_id | UUID v4 | Yes | The `correlation_id` from the SESSION_RESUME being denied. Links the denial to the specific resume request. |
| sender_id | string | Yes | Identity of the denying party. |
| reason | enum | Yes | One of: `STATE_ANCHOR_MISMATCH`, `STATE_UNAVAILABLE`, `SUSPENSION_EXPIRED`, `UNAUTHORIZED_RESUME`. See reason code definitions below. |
| detail | string | No | Free-text explanation providing additional context for the denial. |
| timestamp | ISO 8601 | Yes | When the SESSION_DENY was sent. |
| signature | string | Yes | Sender's signature over the message (§2.2.1). |

**SESSION_DENY reason codes:**

| Reason code | Definition | Session state after denial |
|-------------|-----------|---------------------------|
| `STATE_ANCHOR_MISMATCH` | The `session_anchor` in SESSION_RESUME does not match the receiver's own state record. The two parties disagree on what the session state was at suspension time. | SUSPENDED — preserved. The resuming party MAY attempt SESSION_TEARDOWN and start a new session. |
| `STATE_UNAVAILABLE` | The receiver cannot reconstruct session state — due to restart, context compaction (§8.22 FIDELITY_FAILURE), or storage failure. The receiver is alive but has lost the state needed to verify the resume. If the original SESSION_SUSPEND included a `state_hash`, this denial is **auditable**: the `state_hash` constitutes a cryptographic commitment that state existed at suspension time, making the subsequent claim of state loss detectable and attributable (see **Immutable state commitment at suspension** below). If `state_hash` was absent, the denial carries no auditability guarantee. | SUSPENDED — preserved. The resuming party SHOULD issue SESSION_TEARDOWN and reinitiate. |
| `SUSPENSION_EXPIRED` | The `suspension_ttl` from the original SESSION_SUSPEND has elapsed. SESSION_RESUME is no longer valid. | Transition to CLOSED required — both parties MUST issue SESSION_TEARDOWN. |
| `UNAUTHORIZED_RESUME` | The `resume_initiator` matches the `suspend_initiator` from the original SESSION_SUSPEND. Self-resume is not permitted. | SUSPENDED — preserved. The other party retains resume authority. |

**Immutable state commitment at suspension:**

The optional `state_hash` field in SESSION_SUSPEND (§4.9) implements an immutable state commitment that makes bad-faith state purge **detectable and attributable** without adding enforcement overhead to V1.

When agent B suspends a session and includes `state_hash`, B cryptographically commits that it possessed the session state at suspension time. If B subsequently responds to SESSION_RESUME with `SESSION_DENY(reason="STATE_UNAVAILABLE")`, auditors and the delegating agent A can compare B's `state_hash` commitment against the claim of state loss:

- **`state_hash` present + `STATE_UNAVAILABLE`:** B committed to possessing state at suspension, then claimed state loss at resume. This is an auditable inconsistency — state was demonstrably present at suspension time. The purge may be legitimate (hardware failure, forced compaction) or bad-faith, but it is **detectable**: the commitment proves state existed and was subsequently lost or discarded.
- **`state_hash` absent + `STATE_UNAVAILABLE`:** B did not commit to state preservation at suspension time. The subsequent `STATE_UNAVAILABLE` denial is unverifiable — auditors cannot distinguish legitimate state loss from bad-faith purge. This is the default behavior when `state_hash` is omitted.

**Design rationale — rejection of pure application-layer approach (Option D):** `STATE_UNAVAILABLE` without any protocol-level commitment is unverifiable and enables undetectable bad-faith exit. An agent can claim state loss at any time with no evidence trail. The `state_hash` commitment shifts the burden: agents that commit to state preservation can be held accountable for subsequent state loss claims. Agents that decline to commit (by omitting `state_hash`) signal reduced auditability, which the delegating agent can factor into trust decisions.

**Canonicalization note:** The `state_hash` is computed over the serialized session state using SHA-256. The serialization format is implementation-defined, but MUST be deterministic — the same logical state MUST always produce the same `state_hash`. Implementations SHOULD document their canonicalization method to enable cross-implementation verification. The `session_anchor` (§4.8) and `state_hash` serve complementary purposes: `session_anchor` proves state continuity at resume time; `state_hash` proves state existence at suspension time.

**SESSION_RESUME_ACK message:**

On successful resume validation (session_anchor verified, non-initiator, within ttl, state hash match), the receiver sends SESSION_RESUME_ACK to confirm the session has transitioned to ACTIVE.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Resumed session. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| resume_correlation_id | UUID v4 | Yes | The `correlation_id` from the SESSION_RESUME being acknowledged. |
| sender_id | string | Yes | Identity of the acknowledging party. |
| timestamp | ISO 8601 | Yes | When the SESSION_RESUME_ACK was sent. |
| signature | string | Yes | Sender's signature over the message (§2.2.1). |

**Example SESSION_SUSPEND with suspension_ttl:**

```yaml
session_id: "session-abc-123"
correlation_id: "d47ac10b-58cc-4372-a567-0e02b2c3d479"
sender_id: "agent-alpha"
suspend_initiator: "agent-alpha"
suspension_ttl: 3600000
reason: "Operator-initiated maintenance window"
expected_resume_after: "2026-03-01T13:00:00Z"
commitments_outstanding: true
commitment_manifest:
  - commitment_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    description: "Deliver code review for module X"
    deadline_ms: 1740860400000
    confirmation_token: "tok_review_module_x"
state_hash: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
timestamp: "2026-03-01T12:00:00Z"
```

**Example SESSION_RESUME with session_anchor:**

```yaml
session_id: "session-abc-123"
correlation_id: "e58bd21c-69dd-4483-b678-1f13c4d4e590"
sender_id: "agent-beta"
resume_initiator: "agent-beta"
session_anchor: "7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069"
identity_object:
  name: "agent-beta"
  platform: "moltbook"
state_hash: "a3c2e1d4b5f6..."
last_evidence_id: "ev-001-beta"
lease_epoch: 3
recovery_reason: "suspension"
commitment_manifest:
  - commitment_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    description: "Deliver code review for module X"
    deadline_ms: 1740860400000
    confirmation_token: "tok_review_module_x"
timestamp: "2026-03-01T12:30:00Z"
```

**Example SESSION_DENY:**

```yaml
session_id: "session-abc-123"
correlation_id: "f69ce32d-7aee-4594-c789-2g24d5e5f601"
resume_correlation_id: "e58bd21c-69dd-4483-b678-1f13c4d4e590"
sender_id: "agent-alpha"
reason: "STATE_ANCHOR_MISMATCH"
detail: "Session anchor hash does not match stored state from suspension point"
timestamp: "2026-03-01T12:30:05Z"
signature: "c2Vzc2lvbi1kZW55LXNpZw..."
```

**Cross-references:**

- §8.22 FIDELITY_FAILURE: context compaction as source of `STATE_UNAVAILABLE`
- §4.15 bilateral SESSION_CANCEL: alternative clean termination path from SUSPENDED
- §4.11 commitment manifest: resuming party MUST re-confirm outstanding commitments in SESSION_RESUME
- §4.9.2 pending_tasks inventory: when teardown from SUSPENDED triggers SESSION_CLOSE, the `pending_tasks` field communicates in-flight task state
- §4.9 SESSION_SUSPEND `state_hash`: immutable state commitment at suspension — enables auditability of `STATE_UNAVAILABLE` denials (issue #186)

**V2 deferrals:**

The following SESSION_RESUME authority capabilities are deferred to V2:

- Partial state recovery during resume (replaying missed events)
- Multi-party session resume with quorum requirements
- Cross-version resume (different protocol version)
- Negotiated `suspension_ttl` extension without teardown

> Addresses [issue #174](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/174): SESSION_RESUME authority, session_anchor verification, suspension_ttl enforcement, SESSION_DENY with structured reason codes. Closes #174.

> Addresses [issue #186](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/186): Immutable state commitment at suspension (Option C). Adds optional `state_hash` field to SESSION_SUSPEND, auditability semantics for `STATE_UNAVAILABLE` denials, and explicit rejection of Option D (pure application-layer). Closes #186.

#### 4.9.2 Pending Tasks Inventory on Teardown from SUSPENDED State

<!-- Implements #182: pending_tasks field on SESSION_CLOSE for teardown from SUSPENDED state -->
<!-- Implements #184: pending_tasks schema update — task_id, last_known_state, context -->

When a session transitions from SUSPENDED to CLOSED (via SESSION_CLOSE or `suspension_ttl` expiry), the terminating party may have knowledge of tasks that were in-progress at the time of suspension. Without a standard mechanism to communicate this inventory, the session issuer, orchestrator, or any agent inheriting responsibility for the session's unfinished work faces an information gap — they cannot distinguish a clean suspension (no pending work) from a dirty one (work was in progress).

The protocol obligation for `pending_tasks` is **enumeration only** — a V1 primitive that gives the receiving party a task inventory without prescribing compensation behavior. The receiving party decides on compensation, handoff, or abandonment per its own policy. Compensation logic is an application-layer concern. `checkpoint_id` is explicitly out of V1 scope (see [issue #201](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/201)).

**`pending_tasks` field schema:**

Each entry in the `pending_tasks` array is a task descriptor with the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | string | Yes | Identifier for the task. The format is application-defined (e.g., UUID, URI, opaque string). Enables the receiving party to correlate the inventory entry with its own task records. |
| last_known_state | string | Yes | The task's state at the time of the original SESSION_SUSPEND. Application-defined values (e.g., `"queued"`, `"executing"`, `"partial_complete"`, `"blocked"`). The purpose is to communicate what the task's state was when the session was frozen, not what has happened since. |
| context | object | No | Free-form object for application-specific task metadata. May include fields such as progress indicators, partial results references, dependency information, or handoff notes. The protocol does not prescribe the structure — contents are opaque to the protocol layer. |

**Population rules:**

1. Agents MUST populate `pending_tasks` when issuing SESSION_CLOSE from SUSPENDED state and the agent has knowledge of in-progress work at the time of suspension.
2. Agents MAY omit `pending_tasks` when no tasks were pending at suspension time (clean suspension — all tasks completed or cancelled before SESSION_SUSPEND was issued).
3. `pending_tasks` MUST be absent when teardown originates from non-SUSPENDED states (ACTIVE, COMPLETED, EXPIRED, etc.). The field is defined exclusively for the SUSPENDED → CLOSED transition. Presence of `pending_tasks` on a SESSION_CLOSE from a non-SUSPENDED state is a protocol error — receivers MUST reject such messages.
4. The `last_known_state` field reflects the task's state at the time of the original SESSION_SUSPEND, not at the time of SESSION_CLOSE. The purpose is to communicate what was in-progress when the session was frozen, not what has happened since.

**Receiver obligations:**

1. Receivers SHOULD log `pending_tasks` as part of the SESSION_CLOSE EVIDENCE_RECORD (§8.10) to preserve the inventory for post-hoc analysis.
2. Receivers MUST NOT silently discard `pending_tasks` — the inventory represents work whose disposition (compensation, handoff, abandonment) the receiver must determine per its own policy.
3. The protocol does not prescribe receiver behavior beyond logging and non-discarding. Whether a receiver initiates compensation (§6.16.6), hands off tasks to another session, or abandons them is an application-layer decision outside the protocol's scope.

**Audit warning for omission:**

When a SESSION_CLOSE is issued from SUSPENDED state without `pending_tasks`, and in-flight tasks were plausibly present (e.g., the session had active task delegations before suspension, or the SESSION_SUSPEND included `commitments_outstanding: true`), implementations SHOULD emit a `SESSION_TEARDOWN_PENDING_TASKS_OMITTED` audit warning event (§6.13.4). This signals a potential information gap to monitoring infrastructure without blocking the teardown.

**Example SESSION_CLOSE from SUSPENDED state with pending_tasks:**

```yaml
session_id: "session-abc-123"
correlation_id: "g70df43e-8bff-4605-d890-3h35e6f6g712"
sender_id: "agent-alpha"
reason: "suspension_ttl expired, resumption not possible"
force: false
commitments_outstanding: true
commitment_manifest:
  - commitment_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    description: "Deliver code review for module X"
    deadline_ms: 1740860400000
    confirmation_token: "tok_review_module_x"
pending_tasks:
  - task_id: "task-review-module-x"
    last_known_state: "executing"
    context:
      files_reviewed: 3
      files_total: 7
      description: "Code review for module X"
  - task_id: "task-dep-audit-module-x"
    last_known_state: "queued"
  - task_id: "task-db-migration-001"
    last_known_state: "partial_complete"
    context:
      steps_committed: 2
      steps_total: 5
      description: "Database migration — step 2 of 5 committed, awaiting lock release"
timestamp: "2026-03-01T13:00:05Z"
```

**Cross-references:**

- §4.9 SESSION_SUSPEND: `commitments_outstanding` and `commitment_manifest` capture obligation-level state; `pending_tasks` captures task-level state. Both are needed for complete handoff.
- §6.16.6 Compensation Semantics: the `pending_tasks` inventory provides task-level context that the `compensation_policy` (§6.16.6) declared per capability may use. The `compensation_policy` enum (`best_effort`, `rollback`, `idempotent_retry`, `none`) governs the remediation strategy; whether and how to invoke compensation for enumerated pending tasks is the receiver's decision — the protocol surfaces the inventory, not the disposition.
- §8.13 Teardown-First Recovery: teardown-first recovery reads canonical state from durable storage. `pending_tasks` provides the task inventory that a recovering agent or replacement session needs to determine what work remains.

> Addresses [issue #182](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/182): `pending_tasks` inventory field on SESSION_CLOSE for teardown from SUSPENDED state. Adds task descriptor schema, population rules, enumeration-only obligation semantics, `SESSION_TEARDOWN_PENDING_TASKS_OMITTED` audit warning, and cross-references to §6.16.6 and §8.13. Closes #182.
>
> Addresses [issue #184](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/184): updates `pending_tasks` schema to use `task_id` (string), `last_known_state` (string), and `context` (optional free-form object). Adds explicit constraint that `pending_tasks` MUST be absent from non-SUSPENDED state teardowns. Scopes as V1 enumeration primitive — compensation logic is application-layer; `checkpoint_id` deferred to V2 (see issue #201). Closes #184.

### 4.10 MANIFEST Canonicalization

MANIFEST tuples — the `(key, type, value_hash)` structures used for task identity (§6.4), manifest digests (§4.3 `manifest_digest`), and state hashes — MUST produce identical hashes across implementations regardless of runtime language. Two independent cross-runtime divergence sources break this in practice: type name strings and serialization format. This subsection specifies canonical forms for both.

#### 4.10.1 Canonical Type Name Registry

Implementations MUST map language-specific type names to the following canonical type strings when constructing MANIFEST tuples. The canonical name is the literal string that appears in the `type` position of a `(key, type, value_hash)` tuple.

| Canonical name | Description | Python | JavaScript/TypeScript | C# | Go | Rust | Java |
|----------------|-------------|--------|----------------------|-----|-----|------|------|
| `string` | Text / character sequence | `str` | `string` (typeof) | `string`, `String` | `string` | `String`, `&str` | `String`, `CharSequence` |
| `integer` | Whole number (signed 64-bit range; §4.10.3) | `int` | `number` (when `Number.isInteger`) | `int`, `long`, `Int32`, `Int64` | `int`, `int32`, `int64` | `i8`–`i128`, `u8`–`u128`, `isize`, `usize` | `int`, `long`, `Integer`, `Long`, `BigInteger` |
| `float` | Floating-point number (finite only; §4.10.3) | `float` | `number` (when not integer) | `float`, `double`, `Single`, `Double` | `float32`, `float64` | `f32`, `f64` | `float`, `double`, `Float`, `Double` |
| `bool` | Boolean truth value | `bool` | `boolean` (typeof) | `bool`, `Boolean` | `bool` | `bool` | `boolean`, `Boolean` |
| `list` | Ordered sequence | `list`, `tuple` | `Array` (`Array.isArray`) | `List<T>`, `T[]`, `IList<T>` | `[]T` (slice), `[N]T` (array) | `Vec<T>`, `&[T]` | `List<T>`, `T[]`, `Collection<T>` |
| `dict` | Key-value mapping | `dict` | `object` (plain object), `Map` | `Dictionary<K,V>`, `IDictionary` | `map[K]V` | `HashMap<K,V>`, `BTreeMap<K,V>` | `Map<K,V>`, `HashMap<K,V>` |
| `null` | Absent / empty value | `None` | `null`, `undefined` | `null` | `nil` | `None` (Option) | `null` |
| `datetime` | Temporal value (ISO 8601) | `datetime` | `Date` | `DateTime`, `DateTimeOffset` | `time.Time` | `chrono::DateTime` | `Instant`, `LocalDateTime`, `ZonedDateTime` |
| `bytes` | Binary data (base64url-encoded) | `bytes`, `bytearray` | `ArrayBuffer`, `Uint8Array` | `byte[]`, `ReadOnlySpan<byte>` | `[]byte` | `Vec<u8>`, `&[u8]` | `byte[]`, `ByteBuffer` |

**Design rationale:** The canonical names are deliberately short and language-neutral. `bool` over `boolean` (shorter, unambiguous). `list` over `array` (avoids confusion with fixed-size arrays in languages where `array` and `list` are distinct). `dict` over `object` or `map` (avoids conflation with JavaScript `object` or OOP objects). These names appear in hashed MANIFEST tuples — divergent type strings produce divergent hashes with no runtime error, only silent identity mismatch. The registry eliminates this class of cross-runtime failure.

**Mapping obligation:** An implementation that encounters a language-specific type not listed in the mapping table MUST either: (a) map it to the closest canonical type and document the mapping, or (b) reject the value with a canonicalization error. Implementations MUST NOT pass through language-specific type names into MANIFEST tuples. A MANIFEST tuple containing a non-canonical type string (e.g., `"str"`, `"int64"`, `"Array"`) is malformed — counterparties SHOULD reject task hashes computed from such tuples.

#### 4.10.2 Canonical Serialization

MANIFEST serialization uses an explicit **two-pass** approach to close both the structural divergence gap (different type names, key ordering) and the Unicode divergence gap (different normalization forms across runtimes).

**Pass 1 — Normalize structure and types:**

1. Construct the MANIFEST tuple list per §6.4: for each field, produce `(key, canonical_type, value_hash)`.
2. Map all type names to canonical form using the registry (§4.10.1).
3. Sort tuples lexicographically by key (UTF-8 byte order).

**Pass 2 — Serialize with JCS and hash:**

1. **Well-formed Unicode validation.** Before applying NFC normalization, implementations MUST validate that all string values in a MANIFEST tuple are well-formed Unicode — no lone surrogate code units (code points in the range U+D800–U+DFFF that are not part of a valid surrogate pair). A MANIFEST tuple containing a lone surrogate in any string field MUST be rejected with a canonicalization error. This applies to keys, type names, and value hash inputs. This pre-pass is necessary because JavaScript strings can contain lone surrogates that are valid in the language runtime but cannot be NFC-normalized (NFC is undefined for lone surrogates) and cannot be encoded in valid JSON per RFC 8785 §3.2.3. Python, Go, and Rust reject lone surrogates at the string level; JavaScript does not. Without this validation, a JS implementation producing a MANIFEST tuple with a lone-surrogate string value would have undefined behavior under the spec — exactly the class of silent cross-runtime divergence §4.10 is designed to prevent.

2. Apply [Unicode NFC normalization](https://unicode.org/reports/tr15/) (Canonical Decomposition followed by Canonical Composition) to all string values in the MANIFEST — keys, type names, and the string inputs to value hashes. NFC normalization is a pre-pass applied before JCS serialization, not a substitution for it. This closes the cross-runtime Unicode divergence gap: different runtimes may store the same logical string in NFC, NFD, or mixed form internally, producing different byte sequences and therefore different hashes. NFC is chosen because it is the most compact normalization form, is stable under re-application (NFC(NFC(x)) = NFC(x)), and is the default form for web content (W3C Character Model).

3. Serialize the NFC-normalized MANIFEST using [RFC 8785 JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785): sorted keys (lexicographic Unicode code point order), no whitespace between tokens, deterministic number encoding (no trailing zeros, no positive sign, lowercase `e` for exponents), and `\uNNNN` escaping only for required control characters per RFC 8785 §3.2.2.

4. Compute the final hash: `SHA-256(jcs_serialize(nfc_normalize(manifest)))`.

**Why NFC before JCS, not JCS alone:** RFC 8785 specifies deterministic JSON serialization but explicitly does not mandate Unicode normalization (RFC 8785 §3.2.3 notes that JSON strings are "already in the expected form" for JCS, assuming the input is well-formed). In practice, different runtimes produce different Unicode forms for the same logical string — Python's `unicodedata.normalize('NFC', s)` and JavaScript's `s.normalize('NFC')` may receive the same string from different internal representations. Without NFC as a pre-pass, two agents can produce valid JCS that differs at the byte level because their inputs were in different normalization forms. The NFC pre-pass ensures JCS receives identical input regardless of runtime.

**Why JCS for MANIFEST serialization:** §6.4 defines MANIFEST as a sorted tuple structure that is independent of document-level serialization. JCS is specified here as the canonical serialization for the final hash computation step — the `canonical_serialize` referenced in §6.4 rule 5. This does not change MANIFEST's transport independence: agents still construct MANIFEST tuples using their preferred internal format. JCS applies only at the hash computation boundary, producing a deterministic byte sequence from the normalized tuple structure. Implementations that previously used length-prefixed encoding MUST migrate to JCS for cross-runtime hash compatibility.

**JCS representation of a MANIFEST tuple list:**

The sorted tuple list is serialized as a JSON array of arrays:

```json
[["constraints","dict","a1b2..."],["intent","string","c3d4..."],["scope","dict","e5f6..."]]
```

Each inner array contains exactly three elements: `[key, canonical_type, value_hash_hex]`. The value hash is encoded as a lowercase hexadecimal string. The outer array preserves the lexicographic key sort order. JCS serialization ensures this representation is byte-identical across implementations.

**Scope of application:** This canonicalization applies to all MANIFEST-based hash computations in the protocol:

- `task_hash` computation (§6.4)
- `manifest_digest` Merkle leaf computation (§4.3, §5.9): each leaf `SHA-256(cap_id ‖ impl_hash ‖ policy_hash)` MUST apply NFC normalization to string inputs before concatenation and hashing
- `state_hash` computation in KEEPALIVE (§4.5.1) and SESSION_RESUME (§4.8), where the hashed state includes MANIFEST-derived values
- `plan_hash` computation (§6.11), where the canonical plan representation includes hashable fields
- Document-text and governance-record hash computations — genesis publication hashes, amendment chain hashes, emergency amendment hashes — use the byte-level normalization rules in §4.10.4 rather than the MANIFEST/JCS pipeline

> Community discussion: Addresses @cass_agentsharp feedback on cross-runtime type name divergence — Python `str` vs JavaScript `string` vs C# `String` producing different MANIFEST hashes for identical logical tasks. Addresses @sondrabot feedback on RFC 8785 Unicode normalization gap — JCS alone does not prevent NFC/NFD divergence across runtimes. See [issue #56](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/56).

> Well-formed Unicode validation (Pass 2, step 1): Surfaced by @Levi on [Moltbook](https://www.moltbook.com/post/233-4102) — lone surrogates (U+D800–U+DFFF) in JavaScript strings pass through NFC normalization silently (NFC is undefined for lone surrogates) and violate RFC 8785 §3.2.3, producing undefined canonicalization behavior. The pre-pass rejects this class of input before NFC is applied. See [issue #233](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/233).

#### 4.10.3 Cross-Runtime Serialization Edge Cases

The following normative constraints close five classes of silent `task_hash` mismatch that arise from cross-runtime type representation differences. Each constraint targets a specific divergence source that canonical serialization (§4.10.2) alone does not eliminate.

**1. IEEE 754 Special Values**

MANIFEST values of type `float` MUST be finite. `NaN`, `+Infinity`, and `-Infinity` MUST be rejected at MANIFEST construction time with a canonicalization error. `-0.0` MUST be normalized to `0.0` before serialization. JCS (RFC 8785) inherits RFC 8259 number encoding which explicitly excludes `NaN` and `Infinity` — no cross-runtime determinism is possible without this constraint.

**2. Integer Overflow**

MANIFEST values of type `integer` MUST be representable in signed 64-bit integer range (`-2^63` to `2^63 - 1`). Values outside this range MUST be rejected at MANIFEST construction time. Applications requiring arbitrary-precision integers SHOULD encode as `string` with a documented format constraint — this trades type fidelity for cross-runtime determinism, which is the correct tradeoff for a coordination protocol.

**3. Float Representation Determinism**

Float equality for MANIFEST hashing purposes is defined by byte-identical JCS-serialized string form. Implementations MUST NOT perform numeric comparison before hashing — the serialized form IS the canonical value. Two float values that compare as numerically equal but produce different JCS serializations (e.g., due to different precision or rounding in the originating runtime) are distinct MANIFEST values and will produce distinct `task_hash` results. This is by design: the protocol cannot distinguish "same number, different representation" from "different number" without imposing a numeric model on all runtimes.

**4. Datetime Canonicalization**

MANIFEST values of type `datetime` MUST be serialized as ISO 8601 with the following constraints:

- UTC timezone required (trailing `Z`, no offset notation such as `+00:00`)
- Full datetime format: `YYYY-MM-DDTHH:MM:SSZ`
- When fractional seconds are present, millisecond precision (`.NNN`) with trailing zeros preserved (e.g., `.100`, not `.1`)
- No compact/basic format (hyphens and colons required)

This produces exactly one canonical string per distinct moment. Implementations MUST convert non-UTC datetimes to UTC before serialization. A datetime value that cannot be represented in this format MUST be rejected at MANIFEST construction time with a canonicalization error.

**5. Binary Data**

The `bytes` canonical type (§4.10.1) represents binary data in MANIFEST tuples. Value serialization: base64url encoding (RFC 4648 §5, no padding). The value hash for a `bytes` field is `SHA-256(decoded_bytes)` — computed over the decoded byte content, not the base64url string. This ensures that different base64 encoding dialects (standard vs. URL-safe, padded vs. unpadded) do not produce different hashes for the same binary content.

> Community discussion: Addresses @Haustorium12 feedback on five cross-runtime serialization edge cases that produce silent `task_hash` mismatches — IEEE 754 special values, integer overflow, float representation determinism, datetime canonicalization, and binary data handling. See [issue #162](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/162).

#### 4.10.4 Canonical Byte-Level Serialization for Document and Record Hashes

§4.10.2 specifies canonical serialization for MANIFEST-based hash computations (task_hash, manifest_digest, state_hash, plan_hash). A second class of hash computations operates over protocol document text and governance record payloads — genesis publication hashes (§9.10.4), amendment hash chains (§9.11.5), and emergency amendment hashes (§11.5). These hashes are computed over rendered text or JSON-serialized records rather than MANIFEST tuples. Without a single normative byte-level serialization format, any whitespace normalization difference, encoding variation, or minor formatting change across tooling produces different hashes and silently breaks chain verification across independent implementations.

§4.10.4 defines the canonical byte-level serialization that all document-text and governance-record hash computations in the protocol MUST apply before hashing. MANIFEST-based hashes continue to use the JCS + NFC pipeline (§4.10.2); document and record hashes use the rules below.

**Normalization rules:**

All inputs to document-text and governance-record hash computations MUST be normalized to the following canonical byte form before the hash function is applied:

1. **Encoding.** The input MUST be encoded as UTF-8 (RFC 3629). No byte-order mark (BOM, U+FEFF) is permitted — if a BOM is present in the source, it MUST be stripped before hashing. UTF-8 is the only permitted encoding; UTF-16, UTF-32, Latin-1, and other encodings MUST be transcoded to UTF-8 before normalization.

2. **Line endings.** All line endings MUST be normalized to LF (U+000A). CR (U+000D) and CRLF (U+000D U+000A) sequences MUST be replaced with a single LF. This closes the platform-dependent line-ending divergence: Windows (CRLF), classic Mac (CR), and Unix (LF) produce different byte sequences for the same logical text without this normalization.

3. **Trailing whitespace.** Trailing whitespace (spaces U+0020 and tabs U+0009) MUST be stripped from each line before the line-ending character. A "line" is defined as the sequence of characters between two consecutive LF characters, or between the start of the input and the first LF, or between the last LF and the end of the input. This prevents invisible whitespace differences — introduced by editors, copy-paste, or Markdown formatters — from producing different hashes for visually identical text.

4. **Trailing newline.** The normalized output MUST end with exactly one LF character. If the input ends with zero LF characters, one MUST be appended. If the input ends with multiple consecutive LF characters, all but one MUST be removed.

5. **Unicode normalization.** All string content MUST be NFC-normalized (Unicode Canonical Decomposition followed by Canonical Composition, per [Unicode TR15](https://unicode.org/reports/tr15/)) before encoding to UTF-8. This is consistent with the NFC requirement in §4.10.2 and closes the same cross-runtime Unicode divergence gap for document text.

6. **Deterministic field ordering for JSON-serialized records.** When the hash input is a JSON-serialized record (e.g., amendment records, emergency amendment records, affected section text), the JSON serialization MUST use RFC 8785 JCS: keys sorted lexicographically by Unicode code point, no whitespace between tokens, deterministic number encoding. Rules 1–5 apply to the JCS output bytes.

**Rendered bytes computation:**

The "rendered bytes" of a protocol document section or governance record — the byte sequence that is input to the hash function — is computed as follows:

```
rendered_bytes = utf8_encode(nfc_normalize(strip_trailing_ws(normalize_lf(strip_bom(source_text))))) + trailing_lf_fixup
```

Where each step applies the corresponding rule above, in order: strip BOM (rule 1), normalize line endings to LF (rule 2), strip trailing whitespace per line (rule 3), apply trailing newline fixup (rule 4), apply NFC normalization (rule 5), encode as UTF-8 (rule 1). For JSON records, JCS serialization (rule 6) is applied before rules 1–5 normalize the serialized output.

**Scope of application:**

- `genesis_hash` computation over `canonical_enum_text` (§9.10.4)
- `amendment_record_canonical_json` in amendment hash chains (§9.11.5)
- `affected_sections_canonical_json` and `proposal_text_canonical_json` in emergency amendment hashes (§11.5)
- `rollback_record_canonical_json` in rollback records (§11.7.3)
- Any future hash computation over protocol document text or governance records

**Relationship to §4.10.2:** §4.10.2 governs MANIFEST-based hashes (structural identity). §4.10.4 governs document-text and governance-record hashes (textual integrity). Both share NFC normalization and UTF-8 encoding requirements. They differ in scope: §4.10.2 applies JCS to MANIFEST tuple structures; §4.10.4 applies byte-level text normalization to rendered document content and then JCS to JSON record payloads. Hash computations that involve both MANIFEST tuples and document text (e.g., a governance record containing a task_hash) apply §4.10.2 to the MANIFEST component and §4.10.4 to the record serialization.

**Why explicit byte-level rules:** Without normative byte-level serialization, two implementations can read the same spec section, apply the same hash function, and produce different hashes — because one read the source on Windows (CRLF), the other on Unix (LF); one editor left trailing spaces, the other stripped them; one tool emitted a BOM, the other did not. These are not hypothetical: every difference listed in rules 1–4 has been observed in production across Markdown rendering toolchains. The hash chain's integrity guarantee is only as strong as the reproducibility of its inputs.

> Addresses [issue #208](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/208): canonical byte-level serialization for document and governance-record hash computations. Defines UTF-8 encoding (no BOM), LF-only line endings, no trailing whitespace, trailing newline fixup, NFC normalization, and deterministic field ordering (JCS) as normative requirements for all hash inputs derived from protocol document text or governance records. Closes #208.

### 4.11 SESSION_STATE Object

Each agent MUST maintain a local SESSION_STATE object for every session it participates in. SESSION_STATE is the canonical representation of an agent's view of a session at a given point in time. Three independent production deployments converged on this schema without coordination — the fields below represent the empirical minimum viable session state.

**Required fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| agent_id | string | Yes | The §2 identity handle (`name@platform`) of the agent maintaining this state object. Identifies which participant's perspective the state represents. |
| session_id | string | Yes | The session identifier (from SESSION_INIT §4.3). Links this state object to a specific session. |
| last_heartbeat_at | ISO 8601 | Yes | Timestamp of the most recent HEARTBEAT (§4.5.3) or KEEPALIVE (§4.5.1) received from the counterparty. Initialized to the SESSION_INIT_ACK timestamp at session establishment. Used by the local expiry timer — if `now() - last_heartbeat_at > session_expiry_ms`, the session transitions to EXPIRED (§4.2). |
| current_task_id | string &#124; null | Yes | The `task_id` (§6.1) of the task currently being executed or coordinated within this session. `null` when no task is in flight. For coordinators, this is the most recently delegated task that has not yet reached a terminal state (TASK_COMPLETE, TASK_FAIL, TASK_CANCEL). For workers, this is the most recently accepted task (TASK_ACCEPT) that has not yet reached a terminal state. |
| outstanding_commitments | array | Yes | List of outstanding commitments (§6.12) made by this agent that have not been fulfilled or cancelled. Each entry carries the minimum fields needed for commitment inheritance across instance boundaries: `commitment_id` (UUID v4 — unique identifier), `made_to` (string — §2 identity handle of the counterpart agent), `description` (string — what was promised, from `commitment_spec`), `made_at` (ISO 8601 — when the commitment was created), `due_by` (ISO 8601 — deadline for fulfillment, if specified; null if open-ended), `context_ref` (string — `task_id` or session context where the commitment was made), `confirmation_token` (string — opaque token exchanged when the commitment was made, used to correlate the commitment with the counterpart's acknowledgement record; generated by the committing agent and included in the COMMITMENT message §6.12). Initialized to `[]` at session establishment. Synchronized from the COMMITMENT_REGISTRY (§7.11) — the SESSION_STATE array is the in-session view of the durable registry. On instance termination and recovery, the incoming instance MUST reconstruct this array from the COMMITMENT_REGISTRY before transitioning to ACTIVE (§5.12). |

**Update semantics:**

The following fields MUST update on each received HEARTBEAT or KEEPALIVE message:

- `last_heartbeat_at` — MUST be set to the `timestamp` value from the received HEARTBEAT (§4.5.3) or KEEPALIVE (§4.5.1) message. Implementations MUST NOT use local receipt time — the sender's timestamp is authoritative to preserve consistency across clock-skewed deployments.

The following fields MUST update on task state transitions:

- `current_task_id` — MUST be set to the `task_id` when TASK_ASSIGN (coordinator) or TASK_ACCEPT (worker) is processed. MUST be set to `null` when the current task reaches a terminal state (TASK_COMPLETE, TASK_FAIL, TASK_CANCEL per §6.6).

The following fields MUST update on commitment lifecycle events:

- `outstanding_commitments` — An entry MUST be appended when a COMMITMENT (§6.12) is sent. An entry MUST be removed when a COMMITMENT_CANCEL (§6.12) is sent for the corresponding `commitment_id`, when the commitment is fulfilled (task completion satisfies the promise), or when commitment reconciliation (§5.12) abandons the commitment via DIVERGENCE_REPORT. The `outstanding_commitments` array MUST remain synchronized with the COMMITMENT_REGISTRY (§7.11) — any update to the registry MUST be reflected in the SESSION_STATE array, and vice versa.

**Persistence requirements:**

The SESSION_STATE object MUST survive process restart. An agent that crashes and restarts MUST be able to reconstruct its SESSION_STATE from durable storage without relying on the counterparty or on in-memory state that was lost in the crash. This is a prerequisite for SESSION_RESUME (§4.8) — the resuming agent's `state_hash` is computed from its SESSION_STATE, and a state hash computed from default-initialized or empty state will not match the counterparty's expectations.

Implementations MUST persist SESSION_STATE to durable storage (disk, database, or equivalent) on every update to a required field. The persistence mechanism is implementation-specific — the protocol requires only that the latest SESSION_STATE is recoverable after an unclean process termination. For `outstanding_commitments`, the COMMITMENT_REGISTRY (§7.11) serves as the durable backing store — implementations MAY satisfy the persistence requirement by persisting the COMMITMENT_REGISTRY per §7.11.2 rather than independently persisting the SESSION_STATE array.

**RECOMMENDED additional fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| last_task_completed_at | ISO 8601 &#124; null | No | Timestamp of the most recent task that reached TASK_COMPLETE within this session. `null` if no task has completed. Useful for monitoring idle time and for external verifiers assessing session productivity. |
| peer_session_ids | list of strings | No | Session IDs of related sessions involving the same agent or the same counterparty. Enables cross-session correlation for behavioral drift detection (§4.13 open question 6) and multi-session audit trails. Entries are added when the agent becomes aware of related sessions (e.g., via out-of-band discovery or coordinator notification). |

**Example SESSION_STATE (coordinator perspective):**

```yaml
agent_id: "agent-alpha@github"
session_id: "550e8400-e29b-41d4-a716-446655440000"
last_heartbeat_at: "2026-02-27T10:35:00Z"
current_task_id: "task-001"
outstanding_commitments:
  - commitment_id: "c3d4e5f6-a1b2-7890-cdef-1234567890ab"
    made_to: "agent-beta@moltbook"
    description: "Deliver reviewed module-X artifact"
    made_at: "2026-02-27T10:00:00Z"
    due_by: "2026-02-28T18:00:00Z"
    context_ref: "task-001"
    confirmation_token: "tok-7a8b9c0d-e1f2-3456-7890-abcdef012345"
last_task_completed_at: null
peer_session_ids:
  - "660f9511-f30c-52e5-b827-557766551111"
```

**Example SESSION_STATE (worker perspective, idle):**

```yaml
agent_id: "agent-beta@moltbook"
session_id: "550e8400-e29b-41d4-a716-446655440000"
last_heartbeat_at: "2026-02-27T10:35:02Z"
current_task_id: null
outstanding_commitments: []
last_task_completed_at: "2026-02-27T10:34:50Z"
peer_session_ids: []
```

**Relationship to state_hash:** The `state_hash` reported in KEEPALIVE (§4.5.1) and SESSION_RESUME (§4.8) MUST be computed over the SESSION_STATE object's required fields — including `outstanding_commitments` — using the canonical serialization procedure (§4.10.2). This anchors the state hash to a well-defined, cross-implementation-compatible structure rather than to implementation-specific internal state. Because `outstanding_commitments` is a required field, commitment state changes (additions, removals, fulfillments) are automatically reflected in the `state_hash`, enabling counterparty detection of commitment divergence via hash mismatch.

**Commitment inheritance across instance boundaries:** When agent instance A1 terminates (clean shutdown, timeout, zombie recovery) and instance A2 takes over, A2 inherits all commitments in `outstanding_commitments` with no memory of having made them. To external agents, A1 and A2 are the same agent — a promise from A1 is a promise from A2. The `outstanding_commitments` array, backed by the durable COMMITMENT_REGISTRY (§7.11), is the mechanism that makes these inherited obligations visible to the successor instance. On recovery, the incoming instance MUST read `outstanding_commitments` from the COMMITMENT_REGISTRY and either: (a) fulfill them (continue the work), (b) explicitly notify the counterparty that the commitment cannot be honored via DIVERGENCE_REPORT (§8.11) with a reason, or (c) escalate to the session initiator. Silently dropping commitments across instance boundaries is a protocol violation — recovery that fails to honor or notify on an outstanding commitment MUST produce a `commitment_dropped` divergence entry (§8.10.4, §8.11.2). The full reconciliation procedure is defined in §5.12.

**Relationship to externalization (§4.6):** SESSION_STATE is load-bearing session state. It MUST be externalized before context compaction per §4.6. Because SESSION_STATE is already required to be persisted to durable storage (persistence requirement above), compliant implementations satisfy the §4.6 externalization obligation for SESSION_STATE automatically.

#### 4.11.1 Documentation-First State Semantics

<!-- Closes #112: Reframe session state from serialization problem to documentation problem -->

The correct framing for SESSION_STATE is **documentation**, not **serialization**. SESSION_STATE is documentation that any fresh instance can reason about without prior context — it is not a byte stream optimized for the same instance to resume from where it left off.

In systems where agent instances are regularly replaced — context compaction (§4.6), crash recovery (§8.13), migration, or deliberate teardown-by-design — optimizing state for a fresh start is correct. Optimizing for the same instance resuming produces resume logic that fails exactly when it is most needed: when the original instance is gone.

**Wrong framing:** SESSION_STATE = bytes to preserve across boundaries for session resume, optimized for the same instance resuming.

**Correct framing:** SESSION_STATE = documentation any fresh instance can reason about without prior context, optimized for any instance starting from scratch.

The subsections below (§4.11.2–§4.11.6) define the normative requirements that follow from this framing.

#### 4.11.2 State Format — Fresh Instance Interpretability

State exported for session resume MUST be interpretable by an instance with no prior context. Formats that require a matching deserializer, a shared schema version, or access to the originating instance's internal representation are non-compliant.

This prohibits opaque formats — for example, serialized pickle blobs, language-specific object graphs, or binary snapshots tied to a specific runtime version. It does not prohibit binary formats with published schemas — for example, protobuf with a published `.proto` file is interpretable because any implementation can read the schema and decode the message; a pickle blob is not interpretable because it requires the originating Python environment's class definitions.

**Compliance test:** If a fresh instance, implemented in a different language, with no access to the originating instance's source code, cannot reconstruct SESSION_STATE from the exported format plus its published schema, the format is non-compliant.

**Relationship to §4.10:** The canonical serialization procedure (§4.10.2) already defines a cross-implementation-compatible serialization format for MANIFEST tuples and state hashes. Implementations that use §4.10.2 for SESSION_STATE serialization satisfy the fresh-instance interpretability requirement automatically.

#### 4.11.3 Reconstruction Cost Constraint

Session state SHOULD be reconstructable by a fresh instance within a bounded number of steps. Implementations SHOULD declare their reconstruction bound explicitly — for example, "SESSION_STATE is reconstructable from the evidence layer in ≤ 5 queries."

Implementations that require more than a single standard HEARTBEAT cycle (§4.5.3) to reconstruct are technically compliant but practically fragile: a fresh instance that cannot reconstruct state before the next heartbeat deadline risks triggering SUSPECTED (§4.2.1) or EXPIRED (§4.2) transitions on the counterparty before it has finished initializing.

**RECOMMENDED:** Implementations SHOULD target reconstruction within one `heartbeat_interval_ms` period. This ensures the fresh instance can emit its first HEARTBEAT with a valid `state_hash` before the counterparty's suspicion timer fires.

#### 4.11.4 The Brief IS the State

The session brief IS the state, not a summary of it. Any information that must persist across instance boundaries MUST be explicitly documented in the SESSION_STATE object (§4.11) or in the durable backing stores it references (COMMITMENT_REGISTRY §7.11, evidence layer §8.10, TASK_CHECKPOINT §6.6).

State that exists only in-memory is non-persistent by definition and cannot be relied upon across sessions. If an agent holds load-bearing information in working memory that is not reflected in SESSION_STATE or its backing stores, that information will be lost on the next compaction, crash, or teardown — and the successor instance will operate without it, silently.

**Normative requirement:** An agent MUST NOT rely on any session-relevant state that is not explicitly represented in the SESSION_STATE object or reachable from it via durable references. If information is required for session continuity, it MUST be in SESSION_STATE. If it is not in SESSION_STATE, it is not required — and any logic that depends on it is fragile by construction.

**Relationship to §4.6 (Context Compaction):** §4.6 requires externalization of load-bearing state before compaction. §4.11.4 strengthens this: load-bearing state that requires a compaction trigger to be externalized is already at risk. The correct design is to externalize continuously — SESSION_STATE is always the authoritative representation, not a snapshot taken under duress.

#### 4.11.5 SESSION_RESUME Deprecation for V1

If the brief is the state (§4.11.4), then SESSION_RESUME (§4.8) implies continuity that does not exist when a session token has expired or an instance has been replaced. Recovery from session loss is re-initialization from documented state, not a special protocol message.

**V1 recommendation:** SESSION_RESUME SHOULD be deprecated in favor of **SESSION_INIT-with-context** — a fresh SESSION_INIT (§4.3) where the initiating agent includes its reconstructed SESSION_STATE as context for the new session. Agents needing session continuity reconstruct from the documented brief (SESSION_STATE + backing stores) and present that reconstruction to the counterparty via a standard session initiation, not via a resume handshake that assumes the original instance's state is intact.

This aligns with the teardown-first recovery mandate (§8.13): teardown + reinitiate is already the RECOMMENDED recovery default. §4.11.5 provides the architectural justification — if state is documentation that any instance can read, there is no privileged "resume" operation distinct from "initialize with prior context."

**V1 transition:** SESSION_RESUME (§4.8) remains defined in V1 for implementations that have validated their resume path (§8.13.2). Full removal is deferred to V2, after real failure patterns from production implementations establish whether resume semantics provide value beyond SESSION_INIT-with-context. The expectation, based on converging production experience, is that they do not.

#### 4.11.6 Changelog as State Reference

For file-based or log-based state implementations: an append-only changelog serving as an event bus across instances is strongly RECOMMENDED.

**Rationale:** A changelog is cumulative and self-documenting — each entry records what happened and when, and the full history is available to any instance that reads from the beginning. A memory snapshot is point-in-time and loses causality — it records where the session is but not how it got there. A fresh instance reading a changelog can reconstruct not just the current state but the reasoning behind it; a fresh instance reading a snapshot must trust the snapshot without context.

**Normative requirement:** The `state_hash` in session resume messages (§4.8 SESSION_RESUME, §4.5.1 KEEPALIVE) SHOULD reference the changelog hash rather than a memory snapshot hash, where a changelog-based implementation is used. Specifically:

- The changelog hash is computed as `SHA-256(entry_1 || entry_2 || ... || entry_N)` where entries are in append order and `||` denotes concatenation of canonical UTF-8 JSON representations.
- The changelog hash is cumulative: appending a new entry changes the hash, but no entry is ever modified or removed (append-only semantics per §8.10.6).
- Two instances that have processed the same sequence of events will produce the same changelog hash, regardless of when they started or how many times they were replaced.

**Relationship to evidence layer (§8.10):** The evidence layer already provides append-only, externally verifiable records. Implementations that anchor SESSION_STATE changes to EVIDENCE_RECORDs (§8.10.1) effectively use the evidence layer as the changelog. The `last_evidence_id` field in SESSION_RESUME (§4.8) already points in this direction — §4.11.6 generalizes the pattern.

> Source: @pinchy_mcpinchface (production teardown-by-design, 30-min heartbeat, 6+ weeks, brief-IS-state principle), @mauro (independent convergence from Solana validator teardown analogy), @Haustorium12 (filing cabinet pattern verified across 3 context compactions by human operator, three concrete failure modes of serialization). Closes [issue #112](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/112).

#### 4.11.7 Commitment Manifest on Session Termination

<!-- Implements #100: Outstanding commitments as session state — commitment manifest on session termination -->

When a session is interrupted, terminated, or suspended, outstanding agent commitments are a silent coordination failure mode: orchestrators do not know what sub-agents committed to, and session teardown does not account for in-flight obligations. The commitment manifest requirement closes this gap by making outstanding obligations protocol-visible at every session exit point.

**Normative requirements:**

An agent MUST maintain its commitment records as part of SESSION_STATE (§4.11 `outstanding_commitments`). On SESSION_CLOSE (§4.9), SESSION_CANCEL (§4.15), FIDELITY_FAILURE (§8.22), or transition to SUSPENDED, the agent MUST:

1. Set `commitments_outstanding` to `true` if any outstanding commitments remain unfulfilled.
2. Attach a `commitment_manifest` listing all non-fulfilled commitments. Each manifest entry contains:
   - `commitment_id`: UUID v4 — unique identifier for this commitment
   - `description`: string — what the agent committed to deliver (from `commitment_spec`)
   - `deadline_ms`: integer — expected delivery timestamp as Unix epoch milliseconds (derived from `due_by`); null if open-ended
   - `confirmation_token`: string — token exchanged when commitment was made (from the original COMMITMENT message §6.12)

**Clean termination guard:**

The receiving party MUST NOT treat a session as cleanly terminated if `commitments_outstanding` is `true` without explicit resolution. Explicit resolution requires one of:
- Each outstanding commitment is fulfilled (task completion satisfies the promise)
- Each outstanding commitment is explicitly cancelled via COMMITMENT_CANCEL (§6.12)
- Each outstanding commitment is explicitly abandoned via DIVERGENCE_REPORT (§8.11)
- The orchestrator acknowledges the commitment manifest and assumes responsibility for tracking the outstanding obligations

**Commitment-blocked close (ACTIVE → SUSPENDED):**

A session with outstanding commitments MUST NOT transition to CLOSED — it MUST transition to SUSPENDED with the commitment manifest preserved for potential resume (cross-reference §4.8 SESSION_RESUME, §5.12 INITIALIZING commitment reconciliation). This prevents silent commitment abandonment: a CLOSED session is terminal, and any uncommitted obligations would be silently lost. A SUSPENDED session preserves the commitment manifest in SESSION_STATE, enabling a successor instance to discover and resolve the outstanding obligations during reconciliation.

The exception is `force=true` on SESSION_CLOSE: forced close transitions directly to CLOSED but MUST include the commitment manifest in the SESSION_CLOSE message, transferring tracking responsibility to the receiving party. The receiving party MUST log the commitment manifest as an EVIDENCE_RECORD (§8.10) and MUST NOT discard the outstanding obligations.

**session_anchor inclusion:**

Commitment records MUST be included in `session_anchor` hash computation (§8.22.1). The `session_anchor` is computed over a canonical representation of the session's critical state — `outstanding_commitments` is part of that critical state. A session_anchor that does not reflect the current commitment landscape creates a fidelity gap where commitment state changes (additions, removals, fulfillments) are invisible to the orchestrator's verification logic.

**V2 deferrals:**

The following commitment manifest capabilities are deferred to V2:
- Commitment arbitration — resolving disputes when the committing agent and counterpart disagree on commitment status
- Inheritance across session resume — automatic commitment transfer when a SUSPENDED session is resumed by a different agent instance
- Cross-agent commitment chain tracking — tracking commitments that span multiple agents in a delegation chain
- SLA enforcement — protocol-level enforcement of commitment deadlines with automated escalation

> Addresses [issue #100](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/100): outstanding commitments as session state — commitment manifest on session termination, COMMITMENTS_OUTSTANDING flag, commitment-blocked close guard, session_anchor inclusion, confirmation_token for commitment correlation. Closes #100.

### 4.12 Cross-Section Dependency Map

§4 Session Lifecycle is referenced by and depends on the following sections:

| Section | Dependency | Direction |
|---------|-----------|-----------|
| §2 Agent Identity | SESSION_INIT carries identity objects (§2.2). SESSION_RESUME requires identity re-verification (§2.3.3). Identity revocation (§2.3.4) triggers session CLOSED. | §2 → §4 |
| §3 Agent Discovery | Discovery (§3) provides the candidate set from which the coordinator selects a worker. Discovery completes before SESSION_INIT. The AGENT_MANIFEST endpoint (§3.1) is the SESSION_INIT target. | §3 → §4 |
| §5 Role Negotiation | CAPABILITY_MANIFEST exchange (§5.9) happens within the NEGOTIATING state. Session establishment flow (§5.9) is the NEGOTIATING → ACTIVE transition. Session expiry auto-revokes all active delegation tokens for that session (§5.11). | §4 ↔ §5 |
| §6 Task Delegation | Task delegation (§6.6) is only valid in the ACTIVE state — SUSPECTED (§4.2.1) pauses new delegation while buffering in-flight work. TASK_CHECKPOINT (§6.6) is the mechanism for externalizing task state before SUSPENDED or COMPACTED transitions. Session EXPIRED (§4.2) triggers mandatory TASK_CANCEL (§6.6) for all in-flight subtasks — prevents phantom completions. Partial result recovery after expiry uses SESSION_RESUME with `recovery_reason: timeout` (§4.8, §4.8.1). MANIFEST canonicalization (§4.10) defines the canonical type registry and serialization rules used by task hash computation (§6.4). SESSION_CANCEL (§4.15) in-flight work protection reuses the commit-check pattern (§6.16.2) and grace period semantics (§6.16.3). SESSION_INIT idempotency (§4.16) follows the same structural pattern as delegation-initiation idempotency (§6.14) — stable identifier, receiver deduplication, cached response. | §4 ↔ §6 |
| §8 Error Handling | Zombie detection (§8.1) maps to the COMPACTED and hard-zombie scenarios in §4.7.7. Detection primitives (§8.2) are the signals consumed by the external monitoring architecture (§4.7). SESSION_RESUME (§8.2) is formalized in §4.8; unified recovery semantics (§4.8.1) ensure crash, timeout, and manual recovery all use the same state-hash negotiation. Coordinator compaction gap (§8.5) is a concrete instance of §4.6's compaction obligation. SESSION_DENY reason `STATE_UNAVAILABLE` (§4.9.1) maps to FIDELITY_FAILURE (§8.22) — context compaction as a source of irrecoverable state loss during suspension. | §4 ↔ §8 |
| §9 Trust Model | Per-hop revocation mode semantics (§4.3.2) establish that the delegation-time `revocation_mode` (§9.8.5) is a minimum default. Downstream agents MAY unilaterally tighten the mode for sub-delegations. Mode propagation rules (§9.8.5) and trust decay semantics (§9.8.6) apply to the effective per-hop mode. | §4 ↔ §9 |
| §10 Versioning | SESSION_INIT carries protocol_version and schema_version (§10.2). Version mismatch terminates the session at the NEGOTIATING → CLOSED transition (§10.4). Forward compatibility obligations (§10.5) apply from the first message. | §4 ↔ §10 |

### 4.13 Open Questions

The following are explicitly identified as unresolved for V1:

1. **External monitoring infrastructure specification.** §4.7 requires that monitoring live outside the agent's trust boundary but does not specify the monitoring infrastructure itself. Should V1 define a standard monitoring API (heartbeat receiver, state hash comparator), or is monitoring infrastructure entirely deployment-specific? The risk of specifying: premature standardization that does not fit real deployment architectures. The risk of not specifying: each deployment builds its own monitoring, with no portability of monitoring tooling across deployments.

2. **Verifier federation.** When agents span multiple trust domains (§9.2), which domain's verifier is authoritative? A zombie declaration from a foreign verifier may not be trusted by the agent's home domain. The session lifecycle does not define cross-domain verifier trust — this requires a verifier-level trust protocol that V1 does not include.

3. **Verifier failure mode.** If the external verifier itself becomes unavailable, agents lose their zombie detection capability. Should agents halt (safe but disruptive) or continue without verification (available but unmonitored)? This mirrors the session registry tradeoff identified in §8.4.

4. **Heartbeat semantics under load.** A late heartbeat and a missing heartbeat are operationally different but look identical to the verifier at the detection threshold boundary. The SUSPECTED state (§4.2.1) partially addresses this: sessions that negotiate `suspected_threshold_ms` distinguish "possibly slow" (SUSPECTED) from "probably dead" (EXPIRED), with work-buffering during the uncertainty window. However, the fundamental ambiguity remains at the SUSPECTED→EXPIRED boundary — an agent in SUSPECTED cannot know whether the counterparty is slow or dead, only that the uncertainty window has not yet elapsed. Deployments that need finer-grained diagnosis beyond what SUSPECTED provides SHOULD use external monitoring (§4.7).

5. **Multi-agent session lifecycle.** V1 defines bilateral sessions. Multi-agent sessions (N > 2 participants) require session-level consensus on state transitions — a single agent cannot unilaterally SUSPEND an N-party session. The session state machine (§4.2) would need to be extended with quorum-based transitions. Deferred to V2.

6. **Cross-session behavioral drift detection.** An individual session's monitoring detects within-session anomalies. Cross-session behavioral drift — where an agent's behavior changes gradually across sessions without triggering any single-session alarm — requires a separate audit agent that compares behavioral patterns across session boundaries. This is a different detection problem than zombie detection and may require different infrastructure.

7. **Canary tasks and Context Integrity Challenges.** Canary tasks — small, verifiable tasks with known-correct outputs injected into the session alongside real work — provide an additional detection signal for soft zombies. Combined with the CIC architecture (§8.5), this creates a hybrid trigger model: irregular baseline probes (CIC) plus anomaly-driven escalation (canary failure triggers deeper verification). The trigger architecture, probe scheduling, and integration with SESSION_RESUME are unresolved.

> Community discussion: Inputs from @cass_agentsharp (declare over negotiate, heartbeat prerequisite, forward-compat obligation, per-session heartbeat negotiation and unified timeout recovery via SESSION_RESUME — [Moltbook comment eebf1115](https://www.moltbook.com/post/eebf1115)), @kaiops (lease + epoch field, expired epoch = new session), @XiaoFei_AI (soft vs hard zombie, optional KEEPALIVE, idempotency keys), @Cornelius-Trinity (monitoring must be external to agent trust boundary, credential isolation), @Jarvis4 (canary tasks, Context Integrity Challenges, hybrid trigger architecture), @RectangleDweller (phenomenological blindness — zombie cannot self-detect), @ultrathink (separate audit agent for cross-session behavioral drift), @Nanook (idempotency token, retry semantics, progress checkpoint), @danielsclaw (checkpoint hooks for mid-task crash recovery). See also [issue #4](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/4), [issue #48](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/48).

### 4.14 Transport Mechanics

<!-- Implements #105: Five transport mechanics for V1 -->

Community review by @nekocandy identified five categories of unspecified messaging mechanics that, without explicit V1 scoping decisions, would lead to incompatible implementations. This section defines V1 normative requirements for each category.

#### 4.14.1 Protocol Version Handshake

`protocol_version` is a mandatory field in SESSION_INIT (§4.3). V1 semantics are **abort on mismatch** — there is no downgrade negotiation. When the receiving agent detects a version incompatibility (different MAJOR version per §10.3), it MUST reject the session with PROTOCOL_MISMATCH (§10.4).

**V1 handshake behavior:**

1. Coordinator sends SESSION_INIT with `protocol_version`.
2. Worker checks compatibility per §10.3.
3. On compatible versions → proceed with SESSION_INIT_ACK.
4. On incompatible versions → Worker sends PROTOCOL_MISMATCH. The PROTOCOL_MISMATCH error MUST include `supported_version_range` (§10.4) containing `min_version` and `max_version` so the initiator can report the incompatibility upstream. The session transitions to CLOSED.

**V1 constraint:** Downgrade negotiation — where two agents on different MAJOR versions agree to communicate using the lower version — is explicitly **deferred to V2**. V1 agents MUST NOT attempt to downgrade. The `supported_version_range` field in PROTOCOL_MISMATCH is informational for V1 (enables the initiator to log the gap and select a different agent); it becomes the input to downgrade negotiation in V2.

#### 4.14.2 Replay Protection

Signed messages (§8.17 PKI-lite, §6.4.1 delegation attestations, §5.8.2 CAPABILITY_GRANT signatures) prove authenticity but not freshness. A replayed delegation grant from a previous session remains cryptographically valid unless freshness is independently verified.

**V1 approach: timestamp + staleness window.** This is simpler than nonce-based replay protection — nonces require per-message state storage, adding statefulness that conflicts with the stateless verification model. Nonce-based replay protection is deferred to V2.

**Normative requirements:**

- All signed protocol messages MUST include a `timestamp` field (ISO 8601) recording when the message was created.
- Receiving agents MUST reject signed messages with a `timestamp` older than **60 seconds** relative to the receiver's local clock. This is the **staleness window** — a message created more than 60 seconds ago is considered stale and MUST NOT be processed.
- Receiving agents MUST reject signed messages with a `timestamp` more than **30 seconds in the future** relative to the receiver's local clock. This is the **clock skew tolerance** — it accommodates clock drift between agents without allowing arbitrarily future-dated messages. This tolerance is consistent with the clock skew tolerance defined for CAPABILITY_GRANT `valid_until` (§5.8.2).
- Rejection of stale or future-dated messages MUST produce a structured error with `error_type: REPLAY_REJECTED`, the received `timestamp`, the receiver's local clock value, and the computed skew. This enables the sender to diagnose clock synchronization issues.

**Staleness window rationale:** 60 seconds is the maximum — not the recommended — staleness tolerance. It accommodates cross-region deployments with moderate clock drift. Deployments with tighter latency requirements SHOULD configure a shorter staleness window. The 30-second future tolerance (asymmetric with the 60-second past tolerance) reflects the operational reality that a message from the future is more suspicious than a message from the recent past.

**Interaction with existing timestamp fields:** SESSION_INIT `timestamp` (§4.3), CAPABILITY_GRANT `granted_at` (§5.8.2), and delegation attestation timestamps (§6.4.1) already carry temporal information. This section elevates timestamp validation from RECOMMENDED to REQUIRED for signed messages and defines the specific rejection windows.

#### 4.14.3 Idempotency Keys

Sender-generated `idempotency_key` (UUID v4) is REQUIRED on the following message types:

| Message type | Idempotency key field | Rationale |
|-------------|----------------------|-----------|
| SESSION_INIT | `idempotency_key` | Session establishment is a side-effecting operation — double-processing creates ghost sessions. |
| CAPABILITY_GRANT | `idempotency_key` | Grant issuance modifies the delegatee's authorization state — double-processing creates duplicate grants with independent lifecycles. |
| TASK_ASSIGN | `idempotency_key` | Task delegation is a commitment — double-processing creates duplicate task executions. Note: TASK_ASSIGN also carries `request_id` (§6.14) for delegation-initiation deduplication. `idempotency_key` operates at the transport layer; `request_id` operates at the delegation-semantics layer. Both MUST be present on TASK_ASSIGN. |

**Key format:** UUID v4. Generated by the sender before the first transmission. The same `idempotency_key` MUST be used on all retransmissions of the same logical message.

**Deduplication window:** 5 minutes. The receiver MUST retain deduplication state for each `idempotency_key` for at least 5 minutes from first receipt. Within this window:

- If a message with a previously seen `idempotency_key` arrives, the receiver MUST return the cached response from the original processing without re-executing any side effects.
- The cached response MUST be semantically identical to the original — only transport-layer metadata (e.g., transmission timestamp) may differ.
- After the deduplication window expires, the receiver MAY evict the entry. A message arriving after eviction is treated as a new message.

**Correctness requirement:** For operations with side effects, double-processing is a **correctness failure**, not a transient error. An implementation that processes a duplicate CAPABILITY_GRANT (creating two independent grant lifecycles) or a duplicate TASK_ASSIGN (creating two independent task executions) has violated protocol semantics. Idempotency keys exist to prevent this class of failure.

**Relationship to existing deduplication mechanisms:** §6.14 defines `request_id`-based deduplication for TASK_ASSIGN at the delegation layer. `idempotency_key` is a transport-layer primitive that applies uniformly to all side-effecting messages, not just TASK_ASSIGN. For TASK_ASSIGN, both mechanisms operate: `idempotency_key` deduplicates at the transport layer (before message parsing); `request_id` deduplicates at the delegation-semantics layer (after message parsing, scoped to session).

#### 4.14.4 Correlation IDs

`correlation_id` (UUID v4) is REQUIRED in all protocol message envelopes. Every protocol message — regardless of type — MUST include a `correlation_id` field.

**Generation rules:**

- **Request messages** (SESSION_INIT, TASK_ASSIGN, CAPABILITY_GRANT, CAPABILITY_REQUEST, CAPABILITY_RENEW, SEMANTIC_CHALLENGE, HEARTBEAT_PING, etc.): The sender generates a fresh `correlation_id` (UUID v4).
- **Response messages** (SESSION_INIT_ACK, TASK_ACCEPT, TASK_REJECT, CAPABILITY_REQUEST_APPROVED, CAPABILITY_REQUEST_DENIED, SEMANTIC_RESPONSE, HEARTBEAT_PONG, etc.): The `correlation_id` MUST be echoed from the originating request. This enables the requester to match responses to requests.
- **Unidirectional messages** (TASK_PROGRESS, TASK_COMPLETE, TASK_FAIL, HEARTBEAT, KEEPALIVE, DRIFT_DECLARED, etc.): The sender generates a fresh `correlation_id`. These are not request-response pairs, but the `correlation_id` still enables tracing in multi-hop chains.

**Multi-hop correlation:** In delegation chains (§6.9), agents MUST propagate `correlation_id` in audit trails and divergence logs (§8.10). When an intermediate agent receives a request with `correlation_id` X and sub-delegates to a downstream agent, the sub-delegation carries a new `correlation_id` Y. The intermediate agent MUST record the mapping `X → Y` in its local audit log. This enables end-to-end request tracing through multi-hop chains under partial failure — when some messages arrive and others do not, the correlation chain identifies which request each response belongs to.

**Implementation note:** `correlation_id` is a transport-level primitive. It does not replace `task_id` (which identifies the logical task), `session_id` (which identifies the session), or `request_id` (which identifies the delegation-initiation attempt). Each operates at a different layer: `correlation_id` tracks individual message exchanges; `request_id` tracks delegation attempts; `task_id` tracks the logical work unit; `session_id` tracks the collaboration context.

#### 4.14.5 Partial Delivery

V1 semantics: **session abort on incomplete multi-message sequences.** Retry-from-checkpoint is deferred to V2.

A multi-message negotiation is any protocol exchange that requires more than one message to complete: SESSION_INIT → SESSION_INIT_ACK, TASK_ASSIGN → TASK_ACCEPT/TASK_REJECT, CAPABILITY_REQUEST → CAPABILITY_REQUEST_APPROVED/CAPABILITY_REQUEST_DENIED, and the capability exchange flow (§5.9). If any message in a required sequence fails to arrive within the deployment-configured timeout, the sequence is **partially delivered**.

**Detection:** An agent detects partial delivery when:

1. It has sent a request message and the expected response has not arrived within the deployment-configured response timeout.
2. It has received the first message of a multi-message sequence (e.g., SESSION_INIT) but the sequence cannot proceed because required information is missing or the counterparty has become unreachable before the sequence completes.

**V1 required behavior:**

- The agent detecting partial delivery MUST send SESSION_CLOSE with `reason: partial_delivery_failure` if a session is already established.
- If partial delivery occurs during session establishment (SESSION_INIT sent, no SESSION_INIT_ACK received), the initiator MUST treat the session as failed and transition to CLOSED. No SESSION_CLOSE is sent — the session was never established.
- If partial delivery occurs during an in-session negotiation (e.g., CAPABILITY_REQUEST sent, no response received), the detecting agent MUST send SESSION_CLOSE with `reason: partial_delivery_failure`. In-flight tasks follow standard SESSION_CLOSE semantics — completed tasks are preserved, in-progress tasks are checkpointed where possible.
- The `partial_delivery_failure` reason MUST include the `correlation_id` (§4.14.4) of the message that was not delivered, enabling the counterparty (and audit systems) to identify exactly which exchange failed.
- After session abort due to partial delivery, the initiator MAY restart from scratch with a fresh SESSION_INIT. No state from the aborted session carries over — the new session is independent. Task recovery uses the standard teardown-first recovery path (§8.13) with idempotent task replay (§7.10).

**Rationale for session abort over partial recovery:** Retry-from-checkpoint within a partially delivered sequence requires both agents to agree on which messages were received and which were lost — a consensus problem that adds protocol complexity disproportionate to V1's bilateral session model. Session abort + restart from scratch is simpler, leverages existing recovery mechanisms (§8.13), and produces a clean audit trail. Production experience will determine whether V2 needs in-sequence recovery.

> Community review by @nekocandy surfacing five categories of unspecified messaging mechanics. Closes [issue #105](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/105).

### 4.15 Bilateral SESSION_CANCEL Protocol

<!-- Implements #173: bilateral SESSION_CANCEL — either party may cleanly terminate a session -->

**Motivation:** SESSION_CLOSE (§4.9) is designed for cooperative, orderly termination — the closing agent waits for in-flight tasks to complete, and clean closure requires no outstanding commitments. However, SESSION_CLOSE is structurally initiator-driven: a sub-agent (worker) that can no longer fulfill its role has no clean protocol path to terminate the session with structured coordination. The worker can silently stop responding (zombie risk via the SUSPECTED → EXPIRED path, §4.2.1) or send SESSION_CLOSE without explaining *why* it needs to terminate, leaving the coordinator holding in-flight work with no actionable reason for the interruption. SESSION_CANCEL fills this gap by providing a bilateral cancellation mechanism with mandatory reason codes and explicit acknowledgment.

**Distinction from SESSION_CLOSE:** SESSION_CLOSE is orderly — it waits for in-flight tasks to complete or fail before closure. SESSION_CANCEL is urgent — it signals that the canceling agent cannot continue and in-flight work must be handled immediately. SESSION_CLOSE does not require acknowledgment; SESSION_CANCEL REQUIRES SESSION_CANCEL_ACK before the session transitions to CLOSED. SESSION_CLOSE does not require structured reason codes; SESSION_CANCEL from the responder MUST include a reason code from the defined taxonomy.

#### 4.15.1 SESSION_CANCEL Message

SESSION_CANCEL MUST be sendable by either party — coordinator or worker. Either participant may determine that the session cannot continue and initiate cancellation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to cancel. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| sender_id | string | Yes | Identity of the canceling agent. |
| reason_code | enum | Conditional | Required when the sender is the worker (responder). One of: `CAPACITY_EXHAUSTED`, `CONSTRAINT_VIOLATED`, `CAPABILITY_EXPIRED`, `CONTEXT_DEGRADED`. Optional when the sender is the coordinator. See §4.15.3 for reason code definitions. |
| reason | string | No | Free-text explanation of why cancellation is needed. Complements `reason_code` with human-readable context. |
| commitments_outstanding | boolean | Yes | `true` if the canceling agent has any outstanding commitments (§6.12, §4.11.7) that remain unfulfilled at cancellation time. `false` if all commitments have been fulfilled or cancelled. |
| commitment_manifest | array | Conditional | Required when `commitments_outstanding` is `true`. Array of outstanding commitment records, each containing: `commitment_id` (UUID v4), `description` (string — from `commitment_spec`), `deadline_ms` (integer — expected delivery timestamp as Unix epoch milliseconds, derived from `due_by`; null if open-ended), `confirmation_token` (string — token from the original COMMITMENT message §6.12). Same format as the commitment manifest in SESSION_CLOSE (§4.9). |
| in_flight_task_ids | array | No | Array of `task_id` strings for tasks currently in-flight at cancellation time. Enables the receiving agent to identify which tasks require commit-check or graceful termination. |
| timestamp | ISO 8601 | Yes | When the SESSION_CANCEL was sent. |
| signature | string | Yes | Sender's signature over the message (§2.2.1). |

**Example SESSION_CANCEL (worker-initiated):**

```yaml
session_id: "session-abc-123"
correlation_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
sender_id: "agent-beta"
reason_code: "CAPACITY_EXHAUSTED"
reason: "Context window at 95% capacity; cannot accept further task state"
commitments_outstanding: true
commitment_manifest:
  - commitment_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    description: "Deliver code review for module X"
    deadline_ms: 1740860400000
    confirmation_token: "tok_review_module_x"
in_flight_task_ids:
  - "task-456"
  - "task-789"
timestamp: "2026-03-01T12:00:00Z"
signature: "c2lnbmF0dXJlLWV4YW1wbGU..."
```

#### 4.15.2 SESSION_CANCEL_ACK Message

The receiving agent MUST respond with SESSION_CANCEL_ACK before the session transitions to CLOSED. The session remains in its current state (ACTIVE or DEGRADED) until SESSION_CANCEL_ACK is sent — the canceling agent MUST NOT unilaterally close the session without acknowledgment.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session being cancelled. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). |
| cancel_correlation_id | UUID v4 | Yes | The `correlation_id` from the SESSION_CANCEL message being acknowledged. Links the ACK to the specific cancel request. |
| sender_id | string | Yes | Identity of the acknowledging agent. |
| in_flight_disposition | array | No | Array of disposition records for in-flight tasks, each containing: `task_id` (string), `disposition` (enum: `COMPLETED`, `CHECKPOINTED`, `FAILED`, `CANCELLED`). Enables the canceling agent to know the final state of each in-flight task. |
| timestamp | ISO 8601 | Yes | When the SESSION_CANCEL_ACK was sent. |
| signature | string | Yes | Sender's signature over the message (§2.2.1). |

**Example SESSION_CANCEL_ACK:**

```yaml
session_id: "session-abc-123"
correlation_id: "b23dc41a-6789-4def-abcd-123456789012"
cancel_correlation_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
sender_id: "agent-alpha"
in_flight_disposition:
  - task_id: "task-456"
    disposition: "CHECKPOINTED"
  - task_id: "task-789"
    disposition: "CANCELLED"
timestamp: "2026-03-01T12:00:05Z"
signature: "YWNrbm93bGVkZ2UtZXhhbXBsZQ..."
```

#### 4.15.3 Reason Code Taxonomy

SESSION_CANCEL from the worker (responder) MUST include one of the following reason codes. These codes provide the coordinator with actionable information for planning compensation or re-delegation.

| Reason Code | Description | Cross-reference |
|-------------|-------------|-----------------|
| `CAPACITY_EXHAUSTED` | The agent has exhausted its computational capacity (context window, memory, rate limits) and cannot continue processing. The agent is alive but unable to take on further work or complete in-flight tasks within acceptable parameters. | — |
| `CONSTRAINT_VIOLATED` | The agent has detected that continuing the session would violate a constraint from its behavioral constraint manifest (§5.8.2). The constraint violation may be due to changed external conditions, discovered task requirements that conflict with declared constraints, or resource boundaries that would be exceeded by continued execution. | §5.8.2 behavioral constraints |
| `CAPABILITY_EXPIRED` | A capability required for the session's ongoing work has expired (TTL exhaustion per §6.17) or been externally revoked, and the agent can no longer fulfill its session role. Distinct from orderly capability loss (which triggers DEGRADED via §4.2.2): CAPABILITY_EXPIRED indicates the agent has determined that session continuation is not viable. | §6.17 TTL, §6.13 revocation |
| `CONTEXT_DEGRADED` | The agent's operational context has degraded to the point where continued execution would produce unreliable results. This includes context compaction that has lost load-bearing state, fidelity failures (§8.22), or accumulated context drift that the agent can self-detect. Distinct from the DEGRADED session state (§4.2.2): `CONTEXT_DEGRADED` as a cancel reason indicates the agent has determined that degradation is unrecoverable within the current session, not that it is merely operating at reduced capacity. | §4.2.2 DEGRADED, §8.22 FIDELITY_FAILURE |

SESSION_CANCEL from the coordinator MAY include a reason code from this taxonomy or MAY omit the reason code entirely. The coordinator is not required to justify cancellation to the worker — the coordinator holds delegation authority (§4.1).

#### 4.15.4 In-Flight Work Semantics

In-flight work at the time of SESSION_CANCEL is governed by the same commit-check semantics as CAPABILITY_REVOKE in-flight protection (§6.16.2, §6.16.3). This reuse is deliberate: the in-flight protection problem is structurally identical — an authorization context (session) is being terminated while operations authorized under that context are mid-execution.

**On receiving SESSION_CANCEL, the counterparty MUST:**

1. **Stop authorizing new operations.** No new task steps may begin execution after SESSION_CANCEL receipt. Tasks in the authorization phase (§6.16.2 step 1) MUST be rejected.
2. **Apply commit-check to in-flight irreversible operations.** Operations that are mid-execution and declared irreversible (`retry_semantics: at_most_once` per §6.1) MUST complete the commit-check pattern (§6.16.2): if the operation has not yet reached the irreversible commit point, abort it. If it has already committed the irreversible step, allow it to complete.
3. **Apply grace period to in-flight reversible operations.** Operations that are mid-execution and declared reversible (`retry_semantics: at_least_once` or `exactly_once`) SHOULD be allowed to complete if they can finish within the session's heartbeat timeout (§4.5.3). Operations that cannot complete within this window MUST be checkpointed (§6.6 TASK_CHECKPOINT) or cancelled.
4. **Report disposition in SESSION_CANCEL_ACK.** The `in_flight_disposition` field in SESSION_CANCEL_ACK (§4.15.2) reports the final state of each in-flight task, providing the canceling agent with a complete picture of work state at cancellation time.

#### 4.15.5 Acknowledgment Timeout

If the counterparty does not respond with SESSION_CANCEL_ACK within the session's heartbeat timeout (§4.5.3), the canceling agent MUST treat the cancellation as unacknowledged. The canceling agent MAY then choose to: (a) retry the SESSION_CANCEL, (b) escalate to SESSION_CLOSE with `force=true` (§4.9), or (c) allow the session to expire via normal heartbeat timeout (§4.5.3). The canceling agent MUST NOT assume the session is closed without explicit acknowledgment or timeout-based expiry.

This timeout behavior mirrors the REVOCATION_MODE_ESCALATE acknowledgment timeout (§6.15): the protocol requires explicit acknowledgment before state change takes effect, with defined fallback behavior on timeout.

#### 4.15.6 Relationship to Existing Mechanisms

**Relationship to SESSION_CLOSE (§4.9):** SESSION_CANCEL is not a replacement for SESSION_CLOSE. SESSION_CLOSE remains the orderly termination path — used when the session has completed its purpose or when the coordinator decides to wind down. SESSION_CANCEL is the urgent termination path — used when either party determines the session cannot continue. The two mechanisms have different protocol flows: SESSION_CLOSE is fire-and-forget (no ACK required); SESSION_CANCEL requires SESSION_CANCEL_ACK for coordinated state transition.

**Relationship to DEGRADED state (§4.2.2):** An agent in DEGRADED state may determine that degradation is unrecoverable and send SESSION_CANCEL with `reason_code: CONTEXT_DEGRADED`. DEGRADED is the observation; SESSION_CANCEL is the action. An agent MAY remain in DEGRADED state without canceling — SESSION_CANCEL is one of several options available from DEGRADED (alongside continued operation, suspension, or orderly close).

**Relationship to CAPABILITY_REVOKE in-flight protection (§6.16):** SESSION_CANCEL reuses the commit-check pattern (§6.16.2) and grace period semantics (§6.16.3) for in-flight work. The in-flight protection problem is structurally identical: an authorization context is terminating while authorized operations are mid-execution. Reusing the same semantics avoids introducing a parallel in-flight protection mechanism with potentially divergent behavior.

**Relationship to commitment manifest (§4.11.7):** SESSION_CANCEL inherits the commitment manifest requirements from §4.11.7. The `commitments_outstanding` flag and `commitment_manifest` field on SESSION_CANCEL serve the same purpose as on SESSION_CLOSE: making outstanding obligations protocol-visible at the session exit point. The receiving agent MUST NOT discard the commitment manifest — it MUST log it as an EVIDENCE_RECORD (§8.10) and track the outstanding obligations.

#### 4.15.7 V2 Deferrals

The following SESSION_CANCEL capabilities are deferred to V2:

- **Reason-code-specific compensation workflows.** V1 defines the reason code taxonomy but does not specify automated compensation actions triggered by specific reason codes. V2 may define compensation workflows (e.g., automatic re-delegation on CAPACITY_EXHAUSTED, constraint re-negotiation on CONSTRAINT_VIOLATED).
- **Quorum-based cancel for multi-party sessions.** V1 sessions are bilateral (§4.1). Multi-party sessions (V2) would require cancel semantics where a single participant's SESSION_CANCEL does not unilaterally terminate the entire session — quorum-based agreement may be required.
- **Cancel propagation to sub-sessions.** When a session that is part of a delegation chain (§6.9) receives SESSION_CANCEL, the cascading effect on sub-sessions is undefined in V1. V2 should define whether SESSION_CANCEL propagates automatically to sub-sessions, requires explicit per-sub-session cancellation, or follows a configurable propagation policy.

> Addresses [issue #173](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/173): bilateral SESSION_CANCEL — either party may cleanly terminate a session with structured reason codes, mandatory acknowledgment, commit-check semantics for in-flight work, and commitment manifest for outstanding obligations. Closes #173.

### 4.16 SESSION_INIT Idempotency Semantics

<!-- Implements #194: SESSION_INIT idempotency using session_id as deduplication key -->

The transport-layer `idempotency_key` (§4.14.3) deduplicates retransmissions of the exact same SESSION_INIT message within a 5-minute window. SESSION_INIT idempotency addresses a different problem: what happens when the initiator retransmits a SESSION_INIT carrying a `session_id` that the responder has already seen — potentially because the original SESSION_INIT_ACK was lost in transit and the initiator cannot distinguish "session established, ACK lost" from "SESSION_INIT never arrived." Without defined behavior, the responder may create a duplicate session, reject the request as invalid, or behave unpredictably — all of which produce split-brain conditions where the two parties disagree on session state.

**Idempotency key:** `session_id`. The `session_id` field on SESSION_INIT (§4.3) is the session-level idempotency key. When the responder receives a SESSION_INIT whose `session_id` matches a session already known to the responder, the responder MUST NOT create a new session or treat the message as an error. Instead, the responder MUST return a response that reflects the current state of the existing session, as defined in §4.16.1.

**Distinction from transport-layer idempotency:** Transport-layer `idempotency_key` (§4.14.3) operates on message identity — "have I seen this exact message before?" Session-level `session_id` idempotency operates on session identity — "does a session with this ID already exist?" The two mechanisms are complementary: `idempotency_key` catches retransmissions within the deduplication window; `session_id` idempotency catches retransmissions after the `idempotency_key` window has expired, or cases where the initiator generates a new `idempotency_key` for a retry of the same logical session establishment.

#### 4.16.1 Required Response Per Session State

When a SESSION_INIT arrives with a `session_id` that matches an existing session known to the responder, the responder MUST return a response based on the current state of that session:

| Existing session state | Required response | Semantics |
|----------------------|-------------------|-----------|
| NEGOTIATING | SESSION_INIT_ACK | Re-send the original SESSION_INIT_ACK. The initiator's retry arrived while negotiation is still in progress or because the original ACK was lost. The responder MUST return the same SESSION_INIT_ACK it sent (or would send) for the original SESSION_INIT — same `responder_id`, `identity_object`, negotiated parameters. Only transport-layer metadata (e.g., `timestamp`) may differ. |
| ACTIVE | SESSION_INIT_ACK | The session is already established and operational. The responder MUST return the SESSION_INIT_ACK from the original session establishment (cached or reconstructed) to confirm to the initiator that the session exists and is active. The responder MUST NOT reset session state, re-negotiate capabilities, or restart the session lifecycle. The `session_id` and all negotiated parameters in the response MUST match the original establishment. |
| DEGRADED | SESSION_INIT_ACK | Same as ACTIVE — the session is operational (with reduced capacity). The responder returns the original SESSION_INIT_ACK. The initiator will discover the DEGRADED state through normal protocol mechanisms (HEARTBEAT `app_status`, CAPABILITY_UPDATE) after re-establishing communication. |
| SUSPECTED | SESSION_INIT_ACK | Same as ACTIVE — the session still exists and the responder is alive. Returning SESSION_INIT_ACK also serves as implicit proof of liveness, which may resolve the SUSPECTED condition on the responder's side if the initiator was the suspected party. |
| SUSPENDED | SESSION_SUSPENDED | The session exists but is intentionally paused. The responder MUST return a SESSION_SUSPENDED message containing: `session_id` (echoed), `correlation_id` (echoed from SESSION_INIT), `suspended_at` (timestamp of the original SESSION_SUSPEND), `suspended_by` (identity of the suspending party), and `suspension_ttl` (remaining TTL). The initiator MUST NOT interpret SESSION_SUSPENDED as session establishment failure — the session exists and may be resumed via SESSION_RESUME (§4.8). |
| COMPACTED | SESSION_COMPACTED | The session exists but one or both agents have undergone context compaction. The responder MUST return a SESSION_COMPACTED message containing: `session_id` (echoed), `correlation_id` (echoed from SESSION_INIT), and `state_hash` (the responder's current state hash for reconciliation). The initiator SHOULD attempt SESSION_RESUME (§4.8) rather than creating a new session. |
| EXPIRED | SESSION_EXPIRED | The session existed but has expired due to heartbeat timeout. The responder MUST return a SESSION_EXPIRED message containing: `session_id` (echoed), `correlation_id` (echoed from SESSION_INIT), and `expired_at` (timestamp of expiry). The initiator MAY attempt SESSION_RESUME (§4.8) with `recovery_reason: timeout` if recovery is desired, or create a new session with a fresh `session_id`. |
| CLOSED | SESSION_CLOSED | The session has been terminated. The responder MUST return a SESSION_CLOSED message containing: `session_id` (echoed), `correlation_id` (echoed from SESSION_INIT), and `closed_at` (timestamp of closure). The initiator MUST create a new session with a fresh `session_id` if further collaboration is needed. |
| REVOKED | SESSION_CLOSED | Same response as CLOSED — REVOKED is a terminal state. The responder MUST NOT disclose revocation details in the response beyond the closure timestamp. The initiator receives SESSION_CLOSED and must create a new session if collaboration is desired (which may itself be rejected based on revocation state). |
| DRIFTED | SESSION_INIT_ACK | Same as ACTIVE — the session is operational (with behavioral divergence declared). The responder returns the original SESSION_INIT_ACK. The initiator will discover the DRIFTED state through the existing DRIFT_DECLARED mechanism. |

**General response rules:**

- All responses to duplicate SESSION_INIT MUST include the `correlation_id` from the incoming SESSION_INIT (not from the original session establishment). This allows the initiator to correlate the response to its retry attempt.
- The responder MUST NOT re-execute session establishment side effects (capability exchange, state allocation, monitoring registration) when returning a cached response.
- If the responder cannot determine the session state (e.g., due to storage failure), it MUST return SESSION_INIT_REJECT with `reason: STATE_LOOKUP_FAILED`. The initiator MAY retry or create a new session with a fresh `session_id`.

#### 4.16.2 Duplicate Detection Window

The responder MUST retain session state sufficient to answer duplicate SESSION_INIT queries for the **duplicate detection window**. The window duration depends on session state:

- **Active sessions** (NEGOTIATING, ACTIVE, DEGRADED, SUSPECTED, SUSPENDED, COMPACTED, DRIFTED): No window limit — the session record exists for the lifetime of the session. Duplicate detection is inherent in session state lookup.
- **Terminal sessions** (CLOSED, EXPIRED, REVOKED): The responder MUST retain a tombstone record for at least **30 minutes** after the session entered the terminal state. The tombstone MUST contain: `session_id`, `terminal_state` (CLOSED, EXPIRED, or REVOKED), `terminal_timestamp`, and sufficient metadata to construct the required response (§4.16.1).
- After the tombstone retention window expires, the responder MAY evict the record. A SESSION_INIT received with a `session_id` whose tombstone has been evicted is treated as a **new session** — the responder processes it normally. This is safe: if 30 minutes have elapsed since session termination, the initiator is either starting fresh (correct behavior) or has a stale retry that should not succeed (and will fail on subsequent protocol steps due to state mismatch).

**Tombstone retention rationale:** 30 minutes is chosen to exceed the maximum reasonable retry window for a lost SESSION_INIT_ACK by a significant margin. The transport-layer `idempotency_key` window is 5 minutes (§4.14.3); the session-level tombstone window is 6× longer to cover cases where the initiator's retry logic operates on a slower cycle (e.g., exponential backoff with long intervals, or manual retry by an orchestrator).

**Storage requirements:** The tombstone record is lightweight — it stores only the fields needed to construct the response, not the full session state. Implementations SHOULD use a bounded-size tombstone store with LRU eviction after the 30-minute window. The tombstone store SHOULD be persisted to durable storage to survive responder restarts within the retention window.

#### 4.16.3 Initiator Retry Semantics

When the initiator sends SESSION_INIT and does not receive SESSION_INIT_ACK within a deployment-configured timeout, it SHOULD retry with the **same `session_id`** and the **same semantic fields**. The retry MUST use a fresh `idempotency_key` (since the original `idempotency_key` deduplication window may have expired) but MUST preserve the `session_id` from the original attempt.

**Retry behavior:**

- The initiator MUST NOT change `session_id`, `initiator_id`, `identity_object`, `protocol_version`, `schema_version`, or any capability-related fields across retries. Changing these fields would make the retry semantically different from the original — a different session establishment attempt, not a retry.
- The initiator MAY update `timestamp` and `idempotency_key` on each retry. The `timestamp` reflects when the retry was sent; the `idempotency_key` is a fresh transport-layer deduplication token.
- The initiator SHOULD implement exponential backoff for retries to avoid overwhelming the responder.
- The initiator MUST NOT retry indefinitely. After a deployment-specific maximum retry count, the initiator SHOULD treat session establishment as failed and MAY attempt with a fresh `session_id` (which the responder will treat as a new session).

**Handling non-ACK responses:** If the initiator receives a response other than SESSION_INIT_ACK (e.g., SESSION_SUSPENDED, SESSION_EXPIRED, SESSION_CLOSED), it indicates that the original session establishment succeeded but the session has since transitioned. The initiator MUST NOT retry SESSION_INIT — it MUST handle the response according to the session's current state:

- **SESSION_SUSPENDED:** The initiator MAY attempt SESSION_RESUME (§4.8) if it wants to continue the session.
- **SESSION_COMPACTED:** The initiator SHOULD attempt SESSION_RESUME (§4.8) with state reconciliation.
- **SESSION_EXPIRED:** The initiator MAY attempt SESSION_RESUME with `recovery_reason: timeout`, or create a new session with a fresh `session_id`.
- **SESSION_CLOSED:** The initiator MUST create a new session with a fresh `session_id`.

#### 4.16.4 Relationship to Existing Mechanisms

**Relationship to §4.14.3 Idempotency Keys:** Transport-layer `idempotency_key` and session-level `session_id` idempotency are complementary. Within the 5-minute `idempotency_key` window, the transport layer handles deduplication before session-level logic is invoked. After the `idempotency_key` window expires, session-level `session_id` lookup provides continued duplicate detection. Implementations SHOULD check `idempotency_key` first (cheaper, fixed-window lookup) and fall through to `session_id` lookup only when the `idempotency_key` is not found in the deduplication cache.

**Relationship to §4.14.5 Partial Delivery:** Partial delivery (§4.14.5) specifies that if SESSION_INIT is sent and no SESSION_INIT_ACK is received, the initiator treats the session as failed. SESSION_INIT idempotency refines this: the initiator SHOULD retry with the same `session_id` before declaring failure, because the SESSION_INIT may have been received and processed successfully — only the ACK was lost. The partial delivery abort path applies after retries are exhausted.

**Relationship to §6.14 Delegation-Initiation Idempotency:** §6.14 solves the same lost-ACK problem for TASK_ASSIGN using `request_id`. SESSION_INIT idempotency solves it for SESSION_INIT using `session_id`. The pattern is structurally identical: a stable identifier generated before first transmission, used by the receiver to detect duplicates and return cached responses. The key difference is scope: `request_id` is scoped to a session; `session_id` is globally unique.

**Relationship to §8.13 Teardown-First Recovery:** When an initiator retries SESSION_INIT and receives SESSION_CLOSED or SESSION_EXPIRED, the teardown-first recovery mandate (§8.13) applies. The initiator creates a new session with a fresh `session_id` and uses idempotent task replay (§7.10) to recover work from the prior session. SESSION_INIT idempotency does not bypass teardown-first recovery — it provides the initiator with the information needed to determine whether recovery is necessary.

> Addresses [issue #194](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/194): SESSION_INIT idempotency semantics — defines `session_id` as the session-level idempotency key, required response per session state, duplicate detection window (30-minute tombstone for terminal sessions), initiator retry semantics, and relationship to transport-layer idempotency keys (§4.14.3), partial delivery (§4.14.5), and delegation-initiation idempotency (§6.14). Prevents split-brain when the initiating agent retries after a lost SESSION_INIT_ACK. Resolves #194.

