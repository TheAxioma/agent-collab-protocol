<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

## 10. Versioning

Version management in a decentralized protocol has a different failure mode than in centralized systems. In a centralized system, incompatible versions produce a clear error at deployment time. In a decentralized protocol, incompatible versions produce silent semantic drift at collaboration time — two agents agree on a task, execute against different protocol semantics, and discover the mismatch only when results diverge. The versioning strategy must make incompatibility loud and early, not quiet and late.

### 10.1 Version Axes

The protocol maintains two independent version axes:

| Axis | Tracks | Example changes | Identifier |
|------|--------|-----------------|------------|
| **Protocol version** | Structural changes to the protocol itself | New message types (e.g., adding TASK_CANCEL), state machine transitions, SESSION_INIT field additions, changes to the message lifecycle (§6.6) | `protocol_version` |
| **Schema version** | Semantic changes to task and manifest schemas | Task field additions (§6.1), type changes in existing fields, new required fields in CAPABILITY_MANIFEST (§5.1), changes to MANIFEST canonicalization rules (§4.10, §6.4) | `schema_version` |

**Why two axes, not one.** Conflating protocol and schema versions forces a protocol bump for every schema iteration. During early adoption — when task schemas evolve rapidly as ecosystems discover what fields they actually need — this produces version churn that makes interoperability fragile. An agent that supports protocol v1 with schema v1.3 should be able to collaborate with an agent that supports protocol v1 with schema v1.1, as long as the schema changes between 1.1 and 1.3 are backward compatible. A single version axis would force both agents to v1.3, even though the protocol-level interaction is identical.

Both axes use [Semantic Versioning 2.0.0](https://semver.org/) (MAJOR.MINOR.PATCH).

### 10.2 Version Declaration

Each agent declares its supported protocol version and schema version at session establishment via SESSION_INIT. Version declaration is unilateral — agents declare, they do not negotiate.

**SESSION_INIT version fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| protocol_version | semver | Yes | Protocol version this agent implements |
| schema_version | semver | Yes | Schema version this agent supports for task and manifest structures |

**Example SESSION_INIT (partial):**

```yaml
message_type: SESSION_INIT
agent_id: "agent-alpha"
protocol_version: "1.2.0"
schema_version: "1.4.0"
```

There is no VERSION_QUERY sub-protocol. Agents learn each other's versions from SESSION_INIT. Adding a separate version discovery round-trip would impose latency on every session establishment for an edge case (version mismatch) that is already handled by the error path defined in §10.3.

### 10.3 Compatibility Semantics

**Protocol version compatibility:**

- **PATCH bump** (e.g., 1.2.0 → 1.2.1): Bug fixes and clarifications only. No behavioral change. All agents on the same MAJOR.MINOR MUST interoperate regardless of PATCH.
- **MINOR bump** (e.g., 1.2.0 → 1.3.0): Backward compatible additions. New optional message types, new optional fields in existing messages, new optional lifecycle transitions. An agent implementing 1.3.0 MUST be able to collaborate with an agent implementing 1.2.0. The 1.2.0 agent MAY ignore fields and message types it does not recognize, provided they are optional.
- **MAJOR bump** (e.g., 1.x → 2.x): Breaking changes permitted. New required fields, removed message types, changed state machine transitions. Agents on different MAJOR versions MUST NOT attempt collaboration — the protocol semantics are not guaranteed to be compatible.

**Schema version compatibility:**

- **PATCH bump**: Clarifications to field descriptions. No structural or semantic change.
- **MINOR bump**: Backward compatible additions. New optional fields in task schema (§6.1) or CAPABILITY_MANIFEST (§5.1). Agents on a higher MINOR version MUST accept schemas from agents on a lower MINOR version (missing optional fields default to their defined defaults). Agents on a lower MINOR version MUST accept schemas containing fields they do not recognize (unknown optional fields MUST be ignored, not rejected).
- **MAJOR bump**: Breaking changes. New required fields, removed fields, type changes to existing fields. Agents on different MAJOR schema versions MUST NOT exchange task schemas or capability manifests without explicit translation.

**Compatibility matrix:**

| Scenario | Protocol version | Schema version | Result |
|----------|-----------------|----------------|--------|
| Same MAJOR, same MINOR | 1.2.x ↔ 1.2.x | any compatible | Full interop |
| Same MAJOR, different MINOR | 1.2.x ↔ 1.3.x | any compatible | Interop (higher MINOR accommodates lower) |
| Different MAJOR | 1.x ↔ 2.x | any | PROTOCOL_MISMATCH — session rejected |
| Same protocol, different schema MAJOR | same | 1.x ↔ 2.x | SCHEMA_MISMATCH — task/manifest exchange rejected |

### 10.4 Mismatch Handling

When version incompatibility is detected at SESSION_INIT, the receiving agent MUST reject the session with a structured error.

**PROTOCOL_MISMATCH error:**

| Field | Type | Description |
|-------|------|-------------|
| error_type | string | `PROTOCOL_MISMATCH` |
| local_protocol_version | semver | Version the rejecting agent implements |
| remote_protocol_version | semver | Version declared by the initiating agent |
| supported_version_range | object | The rejecting agent's supported version range. Contains `min_version` (semver, lowest protocol version the agent can interoperate with) and `max_version` (semver, highest protocol version the agent implements). The initiator MUST use this to report the incompatibility upstream — without it, the initiator knows only that its version was rejected, not what version would succeed. V1 semantics: abort on mismatch (clean failure); downgrade negotiation deferred to V2. |
| message | string | Human-readable description of the incompatibility |

**SCHEMA_MISMATCH error:**

| Field | Type | Description |
|-------|------|-------------|
| error_type | string | `SCHEMA_MISMATCH` |
| local_schema_version | semver | Schema version the rejecting agent supports |
| remote_schema_version | semver | Schema version declared by the initiating agent |
| message | string | Human-readable description of the incompatibility |

PROTOCOL_MISMATCH terminates the session immediately — no further messages are exchanged. SCHEMA_MISMATCH terminates the session unless both agents are on the same protocol MAJOR version, in which case the lower-version agent MAY choose to proceed with degraded functionality (accepting only fields it understands). This degraded mode is opt-in — agents that cannot safely ignore unknown schema fields MUST reject.

Mismatch errors are the only version-related error path. The protocol does not support version negotiation — if versions are incompatible, the session fails. This optimizes for the happy path: most sessions between agents in the same ecosystem will be compatible. For the unhappy path, a clear error is better than a negotiated compromise that silently drops semantics.

### 10.5 Forward Compatibility Obligations

Minor version bumps on both axes carry a forward compatibility obligation: implementations at version N+1 MUST be able to collaborate with implementations at version N (within the same MAJOR version).

**For protocol implementors:**

- New optional message types added in a MINOR bump MUST be ignorable. An agent that does not recognize an optional message type MUST silently ignore it without entering an error state.
- New optional fields in existing messages MUST have defined default behavior when absent. An agent on a lower MINOR version that does not send the new field MUST produce the same behavior as if the field were present with its default value.

**For schema consumers:**

- Unknown fields in task schemas and capability manifests MUST be preserved through forwarding. An intermediary agent (e.g., in a delegation chain, §6.9) that receives a schema with fields it does not recognize MUST forward those fields intact — dropping unknown fields breaks downstream agents that depend on them.
- Unknown fields MUST NOT affect task_hash computation for fields the agent does not recognize. The MANIFEST canonicalization (§6.4) operates on known fields only; unknown fields are forwarded but excluded from the local agent's hash computation. Two agents on different schema MINOR versions will compute different task_hashes for the same task if the higher-version agent includes additional fields in its MANIFEST. This is correct behavior — the hashes represent different views of the task, and the protocol does not require hash agreement across schema versions. Hash comparison is meaningful only between agents on the same schema version.

### 10.6 Deprecation Lifecycle

Fields, message types, and protocol behaviors follow a warn-then-remove deprecation lifecycle.

**Deprecation stages:**

| Stage | Duration | Behavior |
|-------|----------|----------|
| **Active** | Indefinite | Field/message is current and fully supported |
| **Deprecated** | Minimum one full MAJOR protocol version | Field/message is marked deprecated. Implementations MUST still accept it. Implementations MAY emit deprecation warnings to operators. |
| **Removed** | After deprecation window | Field/message is no longer part of the protocol. Implementations MAY reject messages containing removed fields. |

**Deprecation metadata:**

Deprecated fields and message types carry metadata in the specification:

| Field | Type | Description |
|-------|------|-------------|
| deprecated | boolean | `true` if the field/message is deprecated |
| deprecated_in | semver | Protocol or schema version in which deprecation was declared |
| removal_target | semver | Earliest version in which the field/message may be removed |
| replacement | string | Identifier of the replacement field/message, if any |

**Minimum support window:** One full MAJOR protocol version. A field deprecated in protocol version 1.x MUST be supported through the entirety of protocol version 2.x and MAY be removed in protocol version 3.0.0. This gives implementations two full major version cycles (the remainder of the current major version plus the entire next major version) to migrate away from deprecated features.

**Runtime behavior during deprecation window:**

- Implementations MUST accept deprecated fields and message types without error.
- Implementations MAY log deprecation warnings for operator visibility.
- Implementations MUST NOT change the semantics of deprecated fields during the deprecation window. A deprecated field behaves identically to its active-phase behavior until removal.
- Implementations SHOULD document migration paths from deprecated to replacement features in release notes.

### 10.7 Schema Version and Attestation

A schema attested at one version does not carry attestation to a different version. This interaction between versioning (§10) and security (§9) requires explicit rules.

**Re-attestation requirement:** When the schema MAJOR version changes, all capability manifests (§5.1) attested under the previous schema version become stale. Agents MUST re-attest their capability manifests against the new schema version before participating in task delegation under the new version. A CAPABILITY_MANIFEST with a `version` field (§5.1) referencing a schema version that does not match the session's declared `schema_version` MUST be rejected.

**Minor schema version changes:** When the schema MINOR version changes, existing attestations remain valid for the fields they cover. New optional fields added in the MINOR bump are not covered by the existing attestation. Agents SHOULD re-attest to cover the new fields but are not required to — the new fields are optional, and an agent that does not use them operates within its existing attestation scope.

This addresses §9.7 open question #2 (schema versioning and revocation): attestation validity is scoped to the schema MAJOR version. A schema MAJOR bump implicitly revokes all attestations for the prior version without requiring explicit revocation propagation — agents on the new version simply reject manifests attested under the old version.

### 10.7.1 Cross-Version Delegation Chain Guidance

When a delegation chain (§6.9) spans agents on different protocol or schema MINOR versions (same MAJOR), version degradation may occur at one or more hops. The `protocol_version_chain` field in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.6) provide visibility into where degradation occurs. This subsection defines the semantic consequences.

**Translation contract propagation:**

When the `protocol_version_chain` (§6.9.1) shows a version downgrade at hop N — meaning the session between agent N and agent N+1 negotiated a lower protocol or schema MINOR version than the session between agent N-1 and agent N — the translation contract established at hop N applies to all subsequent hops. Specifically:

- The agent at hop N is the **translation boundary**. It accepted a session at a higher version from upstream and established a session at a lower version downstream. It is semantically responsible for ensuring that messages and task schemas crossing this boundary are valid under both versions.
- All agents downstream of hop N (at hops N+1, N+2, ...) operate under the lower version's semantics. They have no obligation to understand fields or message types introduced in the higher version — those are the responsibility of the translation boundary agent.
- If the translation boundary agent forwards unknown fields (as required by §10.5 forward compatibility obligations), those fields are syntactically preserved but carry no semantic guarantee from the downstream agents. The originating agent MUST NOT assume that downstream agents interpreted or acted upon higher-version fields.

**Semantic responsibility propagation:**

The key principle: **semantic responsibility propagates with the chain, not just the message.** A downgrade at hop 1 in a 3-hop chain means:

- Hop 1 agent translated from version 1.3 semantics to version 1.1 semantics.
- Hop 2 agent operates entirely under version 1.1 semantics — it did not participate in the translation and has no awareness of version 1.3 features.
- Hop 3 agent also operates under version 1.1 semantics (or whatever version it negotiated with hop 2, which cannot exceed what hop 2 understands).
- The result returning up the chain carries version 1.1 semantics at its core. The hop 1 agent MAY re-translate into version 1.3 form when forwarding upstream, but any semantic content that exists only in version 1.3 (fields added between 1.1 and 1.3) was never populated by the executing agents.

**Implications for the originating agent:**

When `version_chain_summary.degraded` is `true` in a TASK_COMPLETE response:

1. **Result interpretation.** The result was produced under the lowest version in the chain (`min_protocol_version`, `min_schema_version`). Optional fields added in higher MINOR versions may be absent — not because the executing agent chose to omit them, but because the executing agent's protocol version predates their existence.
2. **Trace hash comparison.** The `trace_hash` (§6.2) was computed by agents operating under potentially different schema versions. Hash comparison across schema versions is meaningful only for fields that exist in both versions (see §10.5 forward compatibility obligations regarding hash computation).
3. **Audit trail.** The `version_chain_summary` SHOULD be included in the external audit trail (§8) alongside the task result. It provides the context necessary to interpret why certain expected fields may be missing or why semantic behavior may differ from expectations.

**Relationship to §10.4 mismatch handling:**

Cross-version delegation chains operate strictly within the same MAJOR version — different MAJOR versions cannot establish a session (§10.4 PROTOCOL_MISMATCH). Version degradation within a delegation chain is always a MINOR version difference, which is backward compatible by definition (§10.3). The `protocol_version_chain` makes the MINOR version landscape across the chain explicit, enabling the originating agent to make informed decisions about result quality without requiring the protocol to prohibit MINOR version differences.

### 10.8 Relationship to Other Sections

- **§4 (Session Lifecycle).** SESSION_INIT (§4.3) is the delivery mechanism for version declaration — `protocol_version` and `schema_version` are mandatory fields. Version incompatibility detected at SESSION_INIT triggers the NEGOTIATING → CLOSED transition (§4.2). The version fields in §10.2 are carried by the SESSION_INIT message defined in §4.3.
- **§5 (Role Negotiation).** CAPABILITY_MANIFEST carries a `version` field (§5.1) that represents the schema version of the manifest format. §10 defines the operational semantics: minor bumps add optional fields; major bumps may change required fields. Capability declarations in v0.1 do not carry protocol version constraints — an agent's capabilities are independent of the protocol version it implements. This may change in future versions if capabilities become version-specific.
- **§6 (Task Delegation).** The `version` field in the canonical task schema (§6.1) is a schema version for forward compatibility. §10.3 defines what "forward compatible" means: minor bumps are backward compatible; major bumps may break. The `namespace + alias + version` triple that uniquely identifies a task type (§6.3) uses the schema version axis — protocol version is not part of task type identity. The `protocol_version_chain` field in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL carry per-hop version information through delegation chains (§6.9.1). §10.7.1 defines the semantic consequences of version degradation across hops — translation contract propagation and result interpretation guidance.
- **§8 (Error Handling).** PROTOCOL_MISMATCH and SCHEMA_MISMATCH (§10.4) are session-level errors that terminate the session before any task delegation occurs. They are distinct from task-level errors (TASK_FAIL) and session recovery (SESSION_RESUME, §4.8). A version mismatch is not a recoverable error — SESSION_RESUME cannot resolve a fundamental protocol incompatibility.
- **§9 (Security Considerations).** Schema attestation (§9.1) is version-scoped — see §10.7. The re-attestation requirement on schema MAJOR bumps addresses §9.7 open question #2. Translation boundary risk (§9.3) is compounded by version mismatches: a boundary agent translating between trust domains that use different schema versions must handle both translation and version adaptation, doubling the semantic drift surface. Governance document lifecycle (§9.14) binds governance hashes to spec version strings via publication hash composability (§9.14.3) — the `spec_version_string` is a cryptographic input to every governance hash, making the spec version under which a governance document was ratified independently verifiable from the hash chain.

### 10.9 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Version advertisement beyond SESSION_INIT.** The current design assumes bilateral sessions. In multi-agent topologies (e.g., broadcast task assignment), version advertisement may need a discovery mechanism beyond point-to-point SESSION_INIT. Whether this requires a VERSION_ADVERTISE message or can be handled by existing discovery mechanisms (§3, when defined) is undecided.

2. **Cross-version delegation chains.** ~~In a delegation chain (§6.9) where agent A uses schema v1.3 and delegates to agent B which delegates to agent C using schema v1.1, field forwarding (§10.5) preserves unknown fields through B. But if C delegates back to an agent on v1.3, the round-tripped fields may have been modified by B in ways that are valid under v1.1 but semantically incorrect under v1.3. Whether the protocol should define round-trip integrity guarantees for forwarded fields is unresolved.~~ **Partially resolved (V1).** The `protocol_version_chain` field in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.9.1) now provide full visibility into which versions were negotiated at each hop. The originating agent can detect version degradation and make informed decisions about result trust. §10.7.1 defines the translation contract propagation semantics — the agent at the downgrade boundary bears semantic responsibility for translation, and downstream agents operate under the lower version's semantics. The round-trip integrity question for forwarded fields remains open: fields forwarded through a lower-version agent per §10.5 are syntactically preserved but carry no semantic guarantee from the lower-version agent.

3. **Version sunset policy.** The deprecation lifecycle (§10.6) defines minimum support windows but not maximum. How long must implementations continue to support old MAJOR versions? A protocol with versions 1.x, 2.x, and 3.x in simultaneous production use has a combinatorial interoperability surface. Whether the protocol should recommend a maximum number of simultaneously supported MAJOR versions is undecided.

4. **Capability version constraints.** Capability declarations (§5.1) do not currently carry protocol version constraints — a capability declared under protocol v1 is assumed to remain valid under protocol v2. If protocol changes alter the semantics of capability types (e.g., by redefining what `cap:example.net.access@1` means), existing capability declarations become ambiguous. Whether capabilities should be explicitly bound to a protocol version range is deferred to a future version.

5. **Cryptographic session tokens with built-in expiry.** V1 revocation is advisory (§9.8) — REVOKE signals are honored by compliant agents but not technically enforceable. Cryptographic session tokens with built-in TTL would enforce session bounds without relay trust: a token expires regardless of whether the REVOKE signal was propagated or acknowledged. This eliminates the Byzantine propagation problem (§9.8.2) for session-scoped authority but introduces key infrastructure requirements and removes the ability to revoke before TTL expiry. Whether to introduce cryptographic session tokens as a V2 opt-in extension — complementing rather than replacing advisory REVOKE — is deferred. See §9.8.3 for the tradeoff analysis.

> Community discussion: See [issue #20](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/20), [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37), [issue #49](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/49). Architecture surfaced in discussion with @cass_agentsharp. Cross-version delegation chain guidance (§10.7.1) implements #37. Implements #20, #37.

## 11. Emergency Amendment Procedure

The protocol's integrity guarantees — session lifecycle invariants (§4), delegation chain verification (§6.9.3), trust annotation governance (§9.10–§9.12) — depend on stability. Every mechanism that permits modifying these guarantees is an attack surface. An amendment procedure that is easier to execute than the constraints it overrides becomes a fast path to bypassing those constraints. This section specifies the emergency amendment procedure: the mechanism by which protocol invariants may be modified under extraordinary conditions, designed so that executing an emergency amendment is strictly harder than satisfying the constraints it temporarily overrides.

The emergency amendment mechanism is the highest-risk protocol component. If a coordinating quorum can override §4 session lifecycle invariants, then capturing that quorum bypasses all session integrity guarantees. The design is adversarial: every activation condition is a negative constraint (what must NOT be present), every threshold is supermajority or unanimous, and every action is irreversibly recorded in the amendment hash chain (§9.11.5).

### 11.1 Amendment Types

The protocol distinguishes two amendment types with distinct governance requirements. The distinction is not urgency — it is scope of impact on deployed agents and the reversibility of the change.

| Type | Scope | Governance path | Minimum timeline |
|------|-------|----------------|-----------------|
| **Standard amendment** | Any protocol change that follows the normal versioning process (§10) and amendment ceremonies (§9.11) | §9.11 amendment ceremony with tier classification (§9.11.2) | 14 days (Tier 2) or 7 days (Tier 1) per §9.11 |
| **Emergency amendment** | Protocol changes that override or suspend existing invariants defined in §1–§10 — session lifecycle constraints (§4), capability grant chain requirements (§6.9.3, §7), trust annotation governance (§9.10–§9.12), or versioning guarantees (§10) | §11 emergency procedure with all activation conditions satisfied | 72-hour public comment window + coordinating committee sign-off |

**Standard amendments are the default and expected path.** Emergency amendments exist for situations where a demonstrated protocol defect — not a desired feature change — requires modification of invariants that the standard path cannot address within the defect's impact window. The emergency path is not a faster version of the standard path. It is a structurally different procedure with higher activation thresholds, stricter quorum requirements, and mandatory rollback provisions.

**Emergency amendments MUST NOT be used for:**
- Feature additions that do not address an active defect
- Changes that could be accomplished through the standard amendment path (§9.11) within an acceptable timeline
- Modifications to sections that do not contain protocol invariants (e.g., editorial changes, open questions, community discussion references)

### 11.2 Coordinating Committee

#### 11.2.1 Composition

The coordinating committee consists of all parties who meet **both** of the following criteria:

1. **Genesis ceremony participant** (§9.10.3) — permanent standing, not subject to re-qualification.
2. **Active implementor** (§9.11.3) — demonstrated protocol execution with verifiable evidence.

Genesis ceremony participants who are no longer active implementors retain governance standing for standard amendments (§9.11.3) but do NOT have emergency amendment authority. Emergency amendment authority requires both genesis standing and active implementation — a party that defined the protocol but no longer operates under it cannot override invariants that affect active deployments.

The committee membership list is derived, not appointed. No party can add or remove members by declaration. Membership is a function of verifiable status: genesis participation (immutable historical fact) and active implementation (verifiable current fact). Disputes about active implementor status follow the existing qualification criteria in §9.11.3.

#### 11.2.2 Quorum Requirements

| Action | Quorum | Rationale |
|--------|--------|-----------|
| Propose an emergency amendment | Any single committee member | Low barrier to surface a defect; high barrier to ratify |
| Ratify an emergency amendment | **Unanimous** sign-off from all committee members | Emergency amendments override protocol invariants — every party whose deployment is affected must consent |
| Invoke rollback (§11.7) | Any single committee member | Rollback is the conservative action; low barrier is intentional |
| Extend the public comment window (§11.4) | Any single committee member | Extension is the conservative action |
| Shorten the public comment window | **Not permitted** | The 72-hour window is a minimum, not a target |

**Unanimity for ratification is non-negotiable.** A supermajority threshold (e.g., 2/3 or 3/4) creates a viable capture target — an adversary needs to compromise fewer than all committee members. Unanimity means every committee member is a veto point. The cost of a single holdout blocking a legitimate emergency amendment is bounded (the standard amendment path remains available). The cost of a captured supermajority overriding protocol invariants is unbounded. Note that emergency amendment quorum (unanimous) is strictly above the STRUCTURAL governance tier default (≥66%, §9.11.3 governance tier quorum defaults table), preserving the activation difficulty invariant (§11.4).

**Absent committee members.** A committee member who does not respond within the 72-hour public comment window is treated as a non-ratification. Silence is not consent. An emergency amendment cannot proceed if any committee member is unreachable. This is a deliberate friction point — it ensures that emergency amendments cannot be rushed through during periods when committee members are unavailable.

#### 11.2.3 Adversarial Capture Mitigations

The coordinating committee is the primary capture target for adversaries seeking to bypass protocol invariants. The following structural properties make capture harder:

1. **No delegation of emergency authority.** Committee members MUST NOT delegate their emergency amendment vote to proxies, agents, or automated systems. Emergency ratification requires direct, interactive sign-off from each committee member. A pre-signed approval or standing authorization is INVALID.

2. **No amendment to §11 via §11.** The emergency amendment procedure MUST NOT be used to modify the emergency amendment procedure itself. Changes to §11 require the standard amendment path (§9.11 Tier 2) with a protocol major version increment (§10). This prevents a captured committee from weakening the emergency procedure to enable future captures with lower effort.

3. **Committee membership is derived, not mutable.** No emergency amendment can add or remove committee members. Membership is a function of genesis participation and active implementation status — both are externally verifiable facts, not protocol-internal state that an amendment could modify.

4. **Rate limiting (§11.6) bounds the damage of partial capture.** Even if an adversary achieves unanimous committee consent through coercion or social engineering, the rate limit constrains how many invariants can be overridden within a given time window.

### 11.3 Activation Conditions

An emergency amendment may be proposed only when ALL of the following negative constraints are satisfied. Each condition specifies what must NOT be present — not what must be present. Positive conditions ("when the community judges it necessary") are subjective and ungovernable; negative conditions are objectively verifiable.

#### 11.3.1 No Active Sessions in Affected Scope

There MUST be no active sessions (state RUNNING, NEGOTIATING, or SUSPENDED per §4.2) whose semantics would be altered by the proposed amendment. "Affected scope" is defined by the sections the amendment modifies:

- An amendment to §4 (Session Lifecycle) affects all active sessions.
- An amendment to §6.9 (Delegation Chains) affects all sessions with active delegation chains.
- An amendment to §9.10 (Trust Annotation Types) affects all sessions that reference the trust annotation enum.
- An amendment to §10 (Versioning) affects all sessions.

The proposer MUST include in the amendment proposal a **scope declaration** — a list of protocol sections modified and the resulting affected session scope. The coordinating committee MUST independently verify that no active sessions fall within the declared scope before ratification.

**Verification mechanism:** Active session state is observable through the external monitoring architecture (§4.7) and session state objects (§4.11). The coordinating committee MUST query the deployment's session registry (or equivalent monitoring infrastructure) and confirm that the affected scope contains zero active sessions. If session state cannot be verified (e.g., monitoring infrastructure is unavailable), the activation condition is NOT satisfied — unverifiable state is treated as occupied state.

#### 11.3.2 Unanimous Coordinating Committee Sign-Off

Every member of the coordinating committee (§11.2) MUST provide explicit, interactive sign-off on the amendment proposal. Sign-off requirements:

- Each sign-off MUST be cryptographically signed by the committee member's identity key (§2.2).
- Each sign-off MUST reference the exact amendment proposal by its `pre_amendment_hash` (§11.5) — signing a hash that does not match the current proposal is invalid.
- Each sign-off MUST include a timestamp. Sign-offs older than 72 hours (measured from the close of the public comment window) are stale and MUST be re-obtained.
- Sign-offs MUST NOT be pre-authorized, delegated, or automated. A standing instruction ("approve all emergency amendments from proposer X") is INVALID.

#### 11.3.3 72-Hour Public Comment Window

A minimum 72-hour public comment window MUST elapse between the amendment proposal's publication and ratification. The window serves two purposes: (1) it allows affected parties to identify downstream impacts the proposer may have missed, and (2) it creates a temporal barrier that prevents same-day proposal-and-ratification — forcing at least three calendar days between identifying a defect and modifying an invariant.

**Publication requirements:**

- The proposal MUST be published to the same medium used for the genesis ceremony publication (§9.10.4) — the audited party must not control the publication medium.
- The proposal MUST include: the full text of the proposed change, the sections affected, the scope declaration (§11.3.1), the demonstrated defect justifying emergency treatment, and the `pre_amendment_hash` (§11.5).
- The 72-hour window begins at the publication timestamp on the independent medium. Self-reported publication times are not authoritative.

**Comment format:**

Comments during the public window MUST be structured:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| commenter_id | string | Yes | Identity of the commenting party (agent identity per §2.2, or human operator identity) |
| comment_type | enum | Yes | `support`, `objection`, `question`, `impact_report` |
| affected_sections | array | No | Protocol sections the commenter believes are impacted beyond the proposal's scope declaration |
| comment_body | string | Yes | Substantive content of the comment |
| timestamp | ISO 8601 | Yes | Time the comment was submitted |

Comments of type `objection` or `impact_report` MUST be addressed in the amendment record (§11.5) before ratification. "Addressed" means the proposer has published a written response — not that the objection was resolved to the commenter's satisfaction. The requirement is engagement, not agreement. However, unaddressed objections block ratification — the coordinating committee MUST NOT sign off while objections remain without published responses.

**Window extension:** Any committee member may extend the public comment window beyond 72 hours. Extensions are additive — multiple extensions stack. The window MUST NOT be shortened below 72 hours under any circumstances.

#### 11.3.4 No Amendment Within Past 30 Days

No emergency amendment may be ratified if another emergency amendment was ratified within the preceding 30 calendar days. The 30-day rate limit is measured from the `ratification_timestamp` of the most recent emergency amendment to the `ratification_timestamp` of the proposed amendment.

This constraint serves two purposes:

1. **Blast radius containment.** If an emergency amendment introduces an unforeseen defect, the 30-day window provides time for the defect to manifest in production before another amendment can compound it.

2. **Capture cost amplification.** An adversary who captures the coordinating committee can override at most one invariant per 30 days. Combined with the rollback mechanism (§11.7), the first captured amendment is likely to be detected and rolled back before the next amendment window opens.

The 30-day rate limit applies to emergency amendments only. Standard amendments (§9.11) are not rate-limited by §11 — they follow their own deliberation windows. The rate limit cannot be waived, suspended, or overridden by committee vote. It is a hard constraint, not a guideline.

### 11.4 Activation Difficulty Invariant

The emergency amendment procedure MUST be strictly harder to satisfy than the protocol constraints it overrides. This is not a design aspiration — it is a verifiable invariant. If the emergency path becomes easier than the standard path for any class of change, the emergency path will be used as the default path, and the standard governance process (§9.11) becomes dead letter.

**Difficulty comparison:**

| Dimension | Standard amendment (§9.11) | Emergency amendment (§11) |
|-----------|---------------------------|---------------------------|
| Quorum | Per governance tier: STANDARD >50%, STRUCTURAL ≥66% (§9.11.3) | Unanimous coordinating committee (subset of active implementors with genesis standing) |
| Minimum timeline | 7 days (Tier 1) or 14 days (Tier 2) | 72 hours comment window + sign-off collection + 30-day rate limit between amendments |
| Scope restriction | Any protocol section | Only sections containing invariants, and only when a demonstrated defect exists |
| Session precondition | None | No active sessions in affected scope |
| Rate limit | None (each amendment has its own deliberation window) | Maximum one emergency amendment per 30 days |
| Self-amendment | Tier 2 ceremony can modify §9.11 | §11 CANNOT be modified via §11 (§11.2.3) |
| Rollback | No automatic rollback | Mandatory rollback provision (§11.7) |

Every dimension is equal or stricter for emergency amendments. If a future protocol revision introduces a change that makes any dimension of the emergency path easier than the standard path, that revision MUST be accompanied by a compensating constraint that restores the difficulty invariant.

### 11.5 Version Hash Commitment

Before an emergency amendment is ratified, the current protocol state MUST be cryptographically committed. This ensures that the pre-amendment state is independently recoverable — no emergency amendment can silently rewrite history.

**Pre-amendment hash computation:**

```
pre_amendment_hash = SHA-256(
  current_protocol_version ||
  current_schema_version ||
  terminal_amendment_hash ||
  affected_sections_canonical_json ||
  proposal_text_canonical_json
)
```

Where:
- `current_protocol_version` is the protocol version (§10.1) at the time of the proposal.
- `current_schema_version` is the schema version (§10.1) at the time of the proposal.
- `terminal_amendment_hash` is the most recent `amendment_hash` in the amendment chain (§9.11.5), or `genesis_hash` (§9.10.4) if no amendments have been ratified.
- `affected_sections_canonical_json` is the JSON serialization of the sections being modified, canonicalized per §4.10.4: RFC 8785 JCS (keys sorted lexicographically by Unicode code point, no whitespace between tokens, deterministic number encoding), UTF-8 encoding without BOM, LF-only line endings, no trailing whitespace, NFC normalization. This captures the exact pre-amendment text of every section the amendment modifies.
- `proposal_text_canonical_json` is the JSON serialization of the proposed amendment text, canonicalized per §4.10.4 (same rules).
- `||` denotes concatenation.

**Emergency amendment record:**

A ratified emergency amendment produces an amendment record extending §9.11.4 with additional fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| amendment_type | string | Yes | `emergency` |
| pre_amendment_hash | string | Yes | SHA-256 hash computed as specified above |
| demonstrated_defect | string | Yes | Description of the protocol defect that justifies emergency treatment, with evidence |
| scope_declaration | array | Yes | List of protocol sections modified, each with a description of the invariant being overridden |
| affected_session_verification | object | Yes | Evidence that no active sessions exist in the affected scope (§11.3.1). Contains `verification_method` (string), `verification_timestamp` (ISO 8601), and `verified_by` (array of committee member identities who independently confirmed) |
| committee_signoffs | array | Yes | Array of sign-off objects, one per committee member. Each contains `member_id`, `signature` (over `pre_amendment_hash`), `timestamp`, and `interactive_attestation` (boolean, MUST be `true`) |
| public_comment_window_open | ISO 8601 | Yes | Publication timestamp on independent medium |
| public_comment_window_close | ISO 8601 | Yes | Must be >= `public_comment_window_open` + 72 hours |
| comments_received | array | Yes | All comments received during the public window, in submission order |
| objection_responses | array | Yes | Proposer responses to each `objection` and `impact_report` comment |
| prior_emergency_amendment_date | ISO 8601 or null | Yes | `ratification_timestamp` of the most recent prior emergency amendment, or `null` if none. Must be >= 30 days before this amendment's `ratification_timestamp`, or `null` |
| rollback_procedure | object | Yes | The rollback specification per §11.7 |
| amendment_hash | string | Yes | SHA-256 hash chaining to the amendment chain per §9.11.5 |

**Hash chain continuity:** The `amendment_hash` for an emergency amendment is computed identically to standard amendments (§9.11.5) — `SHA-256(prior_state_hash || amendment_record_canonical_json)`. Emergency and standard amendments share the same hash chain. An auditor traversing the chain encounters both types and can verify each against its respective governance requirements (§9.11 for standard, §11 for emergency) based on the `amendment_type` field.

### 11.6 Rate Limit

Emergency amendments are rate-limited to a maximum of **one per 30 calendar days**, measured between consecutive `ratification_timestamp` values. The rate limit is specified in §11.3.4 as an activation condition and restated here for emphasis.

**Rate limit enforcement is structural, not advisory.** An emergency amendment record whose `ratification_timestamp` is within 30 days of the prior emergency amendment's `ratification_timestamp` is INVALID regardless of committee sign-off. Implementations verifying the amendment chain (§9.11.5) MUST reject such records. The rate limit is verified at chain traversal time, not only at ratification time — a post-hoc inserted record that violates the rate limit is detectable and rejectable by any chain verifier.

**Rate limit interaction with rollback:** A rolled-back emergency amendment (§11.7) still counts against the rate limit. The rollback itself is not an amendment — it is a reversion to pre-amendment state — but the original amendment's `ratification_timestamp` remains in the chain and constrains the next emergency amendment. This prevents an adversary from ratifying an amendment, triggering rollback, and immediately ratifying a different amendment to probe for a viable override.

**Standard amendments are not rate-limited by §11.** The rate limit applies exclusively to emergency amendments. Standard amendments (§9.11) proceed on their own timeline. An emergency amendment does not block standard amendments, and standard amendments do not reset the emergency amendment rate limit timer.

### 11.7 Rollback Procedure

Every emergency amendment MUST include a rollback specification at ratification time. The rollback specification defines how to revert the protocol to its pre-amendment state if the amendment causes downstream failures. A ratified emergency amendment without a rollback specification is INVALID — the amendment record fails chain verification (§9.11.5).

#### 11.7.1 Rollback Trigger Conditions

Rollback may be invoked when any of the following conditions are observed after ratification:

1. **Session integrity failure.** An active session (established after the amendment) exhibits behavior inconsistent with the amended invariant — the amendment introduced a semantic gap rather than closing one.
2. **Chain verification failure.** The amendment causes delegation chain verification (§6.9.3) or amendment chain verification (§9.11.5) to fail for records that were valid pre-amendment.
3. **Interoperability regression.** Agents that were interoperable pre-amendment (same MAJOR version per §10.3) become incompatible post-amendment due to the invariant change.
4. **Unaddressed scope impact.** A downstream impact identified during the public comment window (§11.3.3) but inadequately addressed in the objection responses manifests in production.

#### 11.7.2 Rollback Invocation

Any single coordinating committee member may invoke rollback. Rollback does not require committee unanimity — it is the conservative action, and the cost of an unnecessary rollback (temporary reversion to a known-good state) is strictly lower than the cost of an unrolled-back defective amendment (continuing to operate under broken invariants).

**Rollback invocation requirements:**

- The invoking committee member MUST publish a rollback notice to the same independent medium used for the amendment proposal (§11.3.3).
- The rollback notice MUST reference the emergency amendment's `amendment_hash` and `pre_amendment_hash`.
- The rollback notice MUST include the observed failure condition (one or more of §11.7.1 triggers) with evidence.

#### 11.7.3 Rollback Execution

Rollback restores the protocol to the state captured by `pre_amendment_hash` (§11.5). Specifically:

1. The modified sections revert to their pre-amendment text. The `pre_amendment_hash` and `affected_sections_canonical_json` in the amendment record provide the authoritative pre-amendment state.
2. A **rollback record** is appended to the amendment chain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| record_type | string | Yes | `emergency_amendment_rollback` |
| rolled_back_amendment_hash | string | Yes | The `amendment_hash` of the emergency amendment being rolled back |
| pre_amendment_hash | string | Yes | The `pre_amendment_hash` from the original amendment record — the state being restored |
| rollback_trigger | enum | Yes | `session_integrity_failure`, `chain_verification_failure`, `interoperability_regression`, `unaddressed_scope_impact` |
| trigger_evidence | string | Yes | Description and evidence of the observed failure |
| invoked_by | string | Yes | Identity of the committee member who invoked rollback |
| rollback_timestamp | ISO 8601 | Yes | Time the rollback was published |
| rollback_hash | string | Yes | `SHA-256(prior_amendment_hash \|\| rollback_record_canonical_json)` — extends the amendment chain. `rollback_record_canonical_json` is canonicalized per §4.10.4 |

3. The rollback record extends the amendment chain. The chain remains linear and verifiable — a rollback does not fork the chain or remove the original amendment record. The amendment and its rollback are both permanently recorded, providing a complete audit trail.

#### 11.7.4 Post-Rollback State

After rollback:

- The protocol operates under pre-amendment semantics. All invariants that were overridden by the emergency amendment are restored.
- Sessions established under the amended semantics (between ratification and rollback) are in an ambiguous state. These sessions SHOULD be terminated via SESSION_CLOSE (§4.9) with `close_reason: protocol_amendment_rollback`. Agents operating in these sessions MUST be notified of the rollback and given the opportunity to re-establish sessions under the restored semantics.
- The rolled-back amendment still counts against the 30-day rate limit (§11.6). The next emergency amendment cannot be ratified until 30 days after the rolled-back amendment's `ratification_timestamp`.
- The defect that motivated the original emergency amendment remains unresolved. The proposer may address it through the standard amendment path (§9.11) or propose a corrected emergency amendment after the rate limit window expires.

### 11.8 Relationship to Other Sections

- **§4 (Session Lifecycle).** Emergency amendments that modify §4 require zero active sessions in the affected scope (§11.3.1). The session state machine (§4.2) defines the states that constitute "active." SESSION_CLOSE (§4.9) is the mechanism for terminating sessions post-rollback (§11.7.4).
- **§6.9.3 (Delegation Chain Integrity).** Emergency amendments that modify delegation chain requirements risk invalidating in-flight delegation chains. The no-active-sessions precondition (§11.3.1) mitigates this by ensuring no chains are in flight when the amendment takes effect. Chain verification failure post-amendment is a rollback trigger (§11.7.1).
- **§9.10–§9.12 (Trust Annotation Governance).** The emergency amendment procedure does not replace the trust annotation amendment ceremony (§9.11). Trust annotation changes follow §9.11 unless the change also overrides a protocol invariant — in which case both §9.11 and §11 apply. The amendment hash chain (§9.11.5) is shared between standard and emergency amendments.
- **§10 (Versioning).** Emergency amendments that modify versioning guarantees affect all sessions and all schema versions. The scope declaration (§11.3.1) for a §10 amendment encompasses the entire protocol deployment. The `pre_amendment_hash` (§11.5) includes `current_protocol_version` and `current_schema_version` to anchor the amendment to a specific version state.

### 11.9 Open Questions

1. **Committee size floor.** The current design has no minimum committee size. If genesis ceremony participants leave the active implementor pool and only one or two remain, "unanimous consent" provides weaker governance than intended. Whether the protocol should define a minimum committee size below which the emergency amendment path is disabled is unresolved.

2. **Cross-deployment coordination.** The no-active-sessions precondition (§11.3.1) assumes a single deployment or a deployment with unified session monitoring. In federated deployments (§9.2) where session state is not globally observable, verifying the precondition requires cross-deployment coordination that the protocol does not currently specify.

3. **Rollback cascading.** If a standard amendment (§9.11) is ratified during the window between an emergency amendment and its rollback, the standard amendment may depend on semantics introduced by the emergency amendment. Rolling back the emergency amendment may invalidate the standard amendment. Whether rollback should cascade to dependent standard amendments is unresolved.

> Implements [issue #193](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/193): Emergency Amendment Procedure. Specifies amendment type distinction (standard vs. emergency), coordinating committee composition with adversarial capture mitigations, negative-constraint activation conditions (no active sessions in affected scope, unanimous committee sign-off, 72-hour public comment window, no amendment within past 30 days), activation difficulty invariant ensuring emergency path is strictly harder than standard path, version hash commitment pre-amendment, rate limit enforcement, and mandatory rollback procedure with trigger conditions and post-rollback state management. Closes #193.

---

> Propose additions via pull request or design-decision issue.
