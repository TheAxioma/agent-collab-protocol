<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

## 5. Role Negotiation

This section governs two distinct primitives that prior drafts conflated:

1. **Capability manifest** (agent-side). What an agent can do. Stable, declared once at session establishment, independent of any specific task. An agent's capability manifest does not change because a new task arrives — it changes only when the agent's actual capabilities change.

2. **Task requirements** (task-side). What a specific task actually needs. Dynamic, fully known only at delegation time or discovered during execution. Task requirements are properties of the work, not of the worker.

Conflating them forces either over-specification upfront (the delegator must enumerate every capability the task might need before execution begins — fragile for exploratory tasks) or mid-delegation mismatches (the delegatee discovers it lacks a required capability after accepting the task, with no protocol-level mechanism to resolve the gap).

The separation is enforced structurally: §5.1 defines the agent-side manifest; §5.2 defines the task-side requirements; §5.3 defines the privilege model that governs their interaction.

### 5.1 Capability Manifest (Agent-Side)

At session establishment — before any task assignment — each agent declares its capabilities via a CAPABILITY_MANIFEST message. This is a mandatory §5 primitive: agents that do not declare a manifest MUST NOT participate in task delegation.

**Required fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| agent_id | string | Yes | Identity of the declaring agent |
| capability_types | array | Yes | List of capability type entries the agent claims to support (see §5.1.1) |
| resource_access_bounds | object | Yes | Self-declared resource access limits (see below) |
| supported_message_types | array | Yes | Protocol message types this agent can send and receive (e.g. `TASK_ASSIGN`, `TASK_ACCEPT`, `TASK_PROGRESS`, `CAPABILITY_REQUEST`, `CAPABILITY_UPDATE`) |
| version | semver | Yes | Schema version of the manifest format |
| signature | bytes | Yes | Cryptographic signature over the manifest fields, bound to agent_id |

**Capability type entries:**

Each entry in `capability_types` is a structured object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| capability_id | string | Yes | Structured capability identifier using the format `cap:namespace.capability@version` (e.g. `cap:openclaw.file.read@2`, `cap:example.code.execute@1`). See §5.1.1 for format specification. |
| constraints | object | No | Self-declared limits (e.g. max input size, supported formats, resource ceilings) |
| compensation_policy | enum | No | Compensation policy for irrevocable work executed under this capability. One of: `best_effort`, `rollback`, `idempotent_retry`, `none`. See §6.16.6 for definitions. When absent, defaults to `none`. Implementations MUST reject unknown `compensation_policy` values with a parse error — silent ignore or silent default is not permitted (§6.16.6). |

#### 5.1.1 Capability ID Format

Capability identifiers use a structured format that encodes namespace, capability name, and version in a single string:

```
cap:<namespace>.<capability>@<version>
```

**Components:**

| Component | Format | Description |
|-----------|--------|-------------|
| `cap:` | literal prefix | Protocol-level prefix identifying this as a capability ID. |
| `namespace` | dot-separated string | Organizational namespace (e.g. `openclaw`, `example.tools`). Prevents collision across ecosystems. |
| `capability` | dot-separated string | Capability name within the namespace (e.g. `file.read`, `code.execute`). |
| `@version` | semver | Capability version in `MAJOR.MINOR` format (e.g. `@1.0`, `@2.1`). A bare integer `@N` is shorthand for `@N.0`. MAJOR MUST be a positive integer (≥ 1); MINOR MUST be a non-negative integer (≥ 0). |

**Examples:**

- `cap:openclaw.file.read@2` — file read capability, version 2, from the `openclaw` namespace
- `cap:example.code.execute@1` — code execution capability, version 1
- `cap:adapter.pathlike_to_string@1` — adapter capability that converts path-like inputs to strings

**Versioning semantics:**

Version components follow semver-inspired MAJOR.MINOR rules:

- **MAJOR** denotes a breaking change. Different MAJOR versions have no implied compatibility. `cap:ns.capability@2.0` and `cap:ns.capability@1.0` are incompatible — they are treated as distinct capabilities.
- **MINOR** denotes a backward-compatible addition. A higher MINOR version MUST accept all inputs accepted by any lower MINOR version within the same MAJOR, and MUST produce outputs conforming to the lower MINOR version's output schema for those inputs. `cap:ns.capability@1.2` is backward-compatible with `cap:ns.capability@1.0`.
- An explicit compatibility break within a MAJOR line is declared by publishing both versions as separate capability entries with no implied compatibility relationship. Agents that require the old behavior MUST request the old version explicitly.
- The version component replaces the separate `version` field from prior drafts. The version is part of the identifier itself — `cap:openclaw.file.read@1` and `cap:openclaw.file.read@2` are distinct capability IDs. The bare integer form `@N` is shorthand for `@N.0`.

**Compatibility predicate:**

A provider capability `cap:ns.name@P_MAJOR.P_MINOR` satisfies a requester needing `cap:ns.name@R_MAJOR.R_MINOR` if and only if:

```
compatible(provider, requester) =
  provider.namespace == requester.namespace
  AND provider.capability == requester.capability
  AND provider.MAJOR == requester.MAJOR
  AND provider.MINOR >= requester.MINOR
```

MAJOR must match exactly — there is no cross-MAJOR compatibility. MINOR follows backward-compatibility: a provider at a higher MINOR version satisfies a request for a lower MINOR version within the same MAJOR, because the higher MINOR version is a strict superset of the lower.

**Version resolution table:**

| Provider declares | Requester needs | Compatible? | Reason |
|-------------------|-----------------|-------------|--------|
| `cap:ns.tool@1.0` | `cap:ns.tool@1.0` | Yes | Exact match |
| `cap:ns.tool@1.2` | `cap:ns.tool@1.0` | Yes | Same MAJOR, provider MINOR (2) ≥ requester MINOR (0) |
| `cap:ns.tool@1.0` | `cap:ns.tool@1.2` | No | Same MAJOR, but provider MINOR (0) < requester MINOR (2) |
| `cap:ns.tool@2.0` | `cap:ns.tool@1.0` | No | MAJOR mismatch (2 ≠ 1) |
| `cap:ns.tool@1.0` | `cap:ns.tool@2.0` | No | MAJOR mismatch (1 ≠ 2) |
| `cap:ns.tool@2.1` | `cap:ns.tool@2.0` | Yes | Same MAJOR, provider MINOR (1) ≥ requester MINOR (0) |

**Application to capability matching:** The compatibility predicate replaces exact string matching for capability version resolution throughout §5. When matching `required_capabilities` (§5.2) against a CAPABILITY_MANIFEST (§5.1), or when computing `effective_cap_set` from `requested_mandatory` / `requested_optional` (§5.9), the predicate determines whether a provider's declared capability satisfies a requester's version requirement. Namespace and capability name still require exact string match — only the version component uses the compatibility predicate.

**Design rationale:** Opaque capability_id strings (e.g., free-form `"code-execution"`) provide no semantic structure for matching, versioning, or collision avoidance. The `cap:namespace.capability@version` format gives stable semantic identifiers: the namespace prevents cross-ecosystem collision (same role as reverse-DNS but more compact), the version is intrinsic to the ID (not a separate field that can desynchronize), and the format is parseable without a registry lookup.

**Resource access bounds:**

The `resource_access_bounds` object declares the outer envelope of resources the agent can access. This is an agent-level declaration — not a per-task authorization.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| network | object | No | Network access bounds (e.g. `{"allowed_domains": ["*.example.com"], "protocols": ["https"]}`) |
| filesystem | object | No | Filesystem access bounds (e.g. `{"read_paths": ["/data/**"], "write_paths": ["/tmp/**"]}`) |
| compute | object | No | Compute bounds (e.g. `{"max_memory_mb": 4096, "max_cpu_seconds": 300}`) |
| custom | object | No | Deployment-specific resource bounds not covered by the standard fields |

Resource access bounds are declarative ceilings, not entitlements. An agent declaring `max_memory_mb: 4096` does not receive 4 GB of memory — it declares that it will not attempt to use more than 4 GB. Actual resource authorization is governed by the delegation token (§5.5) and task requirements (§5.2).

CAPABILITY_MANIFEST is a declaration, not an authorization. Receiving a manifest tells the delegator what the agent claims it can do — not what it is permitted to do. Trust level assignment happens at delegation time via TASK_ASSIGN (§6.6).

#### 5.1.2 Manifest Verification

A CAPABILITY_MANIFEST signature MUST be verifiable against the agent's declared identity. An agent that cannot produce a valid signature for its manifest MUST NOT be assigned tasks.

The protocol does not specify how capability claims are independently verified (e.g., through testing, certification, or reputation). Capability claims are self-reported; trust in those claims is a deployment decision. What the protocol does guarantee: the manifest is cryptographically bound to the agent's identity, so a capability claim cannot be repudiated or attributed to a different agent after the fact.

### 5.2 Task Requirements (Task-Side)

Task requirements specify what a specific task actually needs — the capabilities, resources, and message types required for execution. Task requirements are properties of the work, not of the worker. They are constructed by the delegator at delegation time and carried within TASK_ASSIGN (§6.6).

**Required fields (within TASK_ASSIGN):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| required_capabilities | array | Yes | Capability_ids the task requires for execution. Each entry MUST correspond to a capability_id from the standard capability type namespace (§5.1). |
| required_resources | object | No | Resource requirements for this specific task (e.g. `{"min_memory_mb": 512, "network": true}`). MUST be within the delegatee's declared `resource_access_bounds`. |
| required_message_types | array | No | Message types the task lifecycle will use beyond the mandatory set. Enables the delegatee to verify it supports the expected interaction pattern before accepting. |

**Relationship to capability manifest:** Task requirements are matched against the delegatee's CAPABILITY_MANIFEST at delegation time. The delegator SHOULD verify that the candidate delegatee's manifest covers the task requirements before sending TASK_ASSIGN. The delegatee MUST verify this independently before sending TASK_ACCEPT (§5.6).

**Relationship to delegation token:** Task requirements specify what the task needs; the delegation token (§5.5) specifies what the delegatee is authorized to use. `required_capabilities` identifies necessary capabilities; `delegation_token.allowed_capabilities` authorizes a (potentially broader) set. A task requirement that is not covered by the delegation token's `allowed_capabilities` is a configuration error on the delegator's side — the delegatee MUST reject such a task.

**Why separate from capability manifest:** A capability manifest is stable across tasks within a session. Task requirements vary per task. An agent capable of code execution, web search, and file system access (manifest) may receive a task that only requires web search (task requirement). The manifest does not change; the requirements do. Keeping them separate prevents the failure mode where a delegator must predict all possible task requirements at session establishment and encode them into the manifest exchange.

### 5.3 Privilege Model

Three axes govern whether an agent may perform a specific operation for a specific task. All three are independent; collapsing any two creates a privilege escalation surface.

**Capability gates possibility.** Can the agent do X at all? This is answered by the agent's CAPABILITY_MANIFEST (§5.1). An agent that does not declare `cap:example.code.execute@1` in its manifest cannot execute code, regardless of trust level or role assignment. Capability is an intrinsic property of the agent, not granted by the protocol.

**Trust gates permission.** Is the agent authorized to do X in this context? This is answered by the `trust_level` in TASK_ASSIGN (§6.6) and the `delegation_token` (§5.5). An agent that declares `cap:example.code.execute@1` capability but receives a task with `trust_level=restricted` and no `cap:example.code.execute@1` in `allowed_capabilities` MUST NOT execute code for that task — even though it is capable.

**Role constrains eligible task types.** Given the intersection of declared capability and granted trust, what task types can the agent handle? Role is not a separate field — it is the computed intersection:

```
eligible_operations = declared_capabilities ∩ allowed_capabilities
```

An agent's role for a given task is the set of operations it is both capable of performing (manifest) and authorized to perform (delegation token). Operations outside this intersection are prohibited regardless of which axis would permit them individually.

**Example:** Agent B declares capabilities `[cap:example.code.execute@1, cap:example.web.search@1, cap:example.fs.access@1]` in its manifest. Delegator A assigns a task with `allowed_capabilities: [cap:example.web.search@1]` and `trust_level: standard`. Agent B's role for this task is `{cap:example.web.search@1}` — it MUST NOT use `cap:example.code.execute@1` or `cap:example.fs.access@1` even though it is capable of both. If the task requires `cap:example.fs.access@1`, B MUST reject the task (§5.7) or request the additional capability via CAPABILITY_REQUEST (§5.8).

### 5.4 Mismatch Handling

When the privilege model (§5.3) identifies a mismatch — the task requires capabilities that fall outside the agent's eligible operations — the delegatee MUST send TASK_REJECT with `reason=role_mismatch`.

**Defined mismatch reasons:**

| Reason | Condition |
|--------|-----------|
| `role_mismatch:capability` | Task requires a capability not in the agent's CAPABILITY_MANIFEST |
| `role_mismatch:trust` | Granted trust level does not permit required operations |
| `role_mismatch:depth` | Delegation depth limit exceeded and agent cannot execute directly |
| `role_mismatch:resource` | Task resource requirements exceed agent's declared resource access bounds |
| `role_mismatch:message_type` | Task requires message types the agent does not support |

The rejection message SHOULD include the specific mismatch reason to aid the delegator's reassignment decision.

**No ROLE_QUERY/ROLE_OFFER sub-protocol.** The protocol does not define a pre-assignment negotiation mechanism where a delegator queries an agent's suitability before sending TASK_ASSIGN. This is a deliberate omission: the delegator already has the agent's CAPABILITY_MANIFEST from session establishment (§5.1) and can perform the capability check locally before sending TASK_ASSIGN. A ROLE_QUERY/ROLE_OFFER sub-protocol adds round-trip latency for an edge case — when the delegator's cached manifest is stale — that does not justify the spec complexity. The correct response to a stale manifest is TASK_REJECT followed by delegator reassignment, not a pre-negotiation protocol. §6 implies capability check before TASK_ASSIGN as a delegator-side optimization, not a protocol requirement.

### 5.5 Delegation Token

The `delegation_token` field in TASK_ASSIGN (§6.6) carries the authorization context for role negotiation. It encodes who authorized the delegation, what capabilities are permitted, and the chain of custody from root delegator to current delegatee.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| origin | string | Identity of the root delegating agent — the agent that initiated the delegation chain |
| chain | array | Full delegation chain from root to current delegatee, ordered by delegation depth. Each entry contains the agent_id and the trust_level it granted. |
| depth | integer | Current delegation depth (0 for direct delegation from origin) |
| allowed_capabilities | array | Explicit list of capability_ids authorized for this subtask. MUST be a subset of the delegator's own authorized capabilities. |
| ttl | ISO 8601 duration | Time-to-live for the delegation. After expiry, the delegatee MUST cease operations under this token. |
| signature | bytes | Cryptographic signature over all other token fields, produced by the immediate delegator |
| provenance_uri | URI | Reference to the authorizing context (e.g., session record, policy document, or upstream delegation) |
| result_schema_uri | URI | Reference to the expected output format schema for validation |

**Token construction rules:**

1. `allowed_capabilities` MUST be a subset of the capabilities authorized for the delegator's own task. A delegator cannot grant capabilities it does not hold — this is the mechanism that prevents privilege escalation across delegation chains.
2. `chain` grows by one entry at each delegation hop. Each entry records the delegator's agent_id and the trust_level it granted. The full chain is available for audit.
3. `depth` MUST equal the length of `chain`. A token where `depth` does not match `chain` length is malformed and MUST be rejected.
4. `signature` is produced by the immediate delegator (the agent at `chain[depth - 1]` for depth > 0, or `origin` for depth 0). Each hop in the chain signs over the token it produces, creating a verifiable chain of signatures.

**Relationship to TASK_ASSIGN (§6.6):** The `delegation_token` is carried within the TASK_ASSIGN message. The `trust_level` field in TASK_ASSIGN and the trust information in `delegation_token.chain` are complementary — `trust_level` is the direct grant for this delegation; `chain` provides the full provenance for audit and verification.

**Relationship to task requirements (§5.2):** The delegation token authorizes capabilities (`allowed_capabilities`); task requirements specify needed capabilities (`required_capabilities`). `required_capabilities` MUST be a subset of `allowed_capabilities`. If a task requires a capability that the delegation token does not authorize, the delegatee MUST reject the task — even if the agent possesses the capability (§5.3 privilege model).

### 5.6 Capability Attestation Before TASK_ACCEPT

Before sending TASK_ACCEPT (§6.6), the delegatee MUST perform four verification checks:

1. **Capability check.** The delegatee verifies that it has all capabilities listed in the task's `required_capabilities` (§5.2) and that those capabilities are authorized by `delegation_token.allowed_capabilities`. This is a self-check against the agent's own CAPABILITY_MANIFEST.

2. **Trust check.** The delegatee verifies that the granted `trust_level` permits the operations required by the task specification. If the task requires operations that the granted trust level does not cover, the delegatee MUST NOT accept.

3. **Depth check.** The delegatee verifies that `delegation_token.depth` has not exceeded the configurable maximum delegation depth (v1 default: 3, see §6.9). If the delegatee would need to further delegate and doing so would exceed the depth limit, it MUST NOT accept unless it can execute the task directly.

4. **Resource check.** The delegatee verifies that the task's `required_resources` (§5.2) fall within its declared `resource_access_bounds` (§5.1). A task requiring 8 GB of memory from an agent that declared a 4 GB ceiling MUST be rejected.

If any check fails, the delegatee MUST send TASK_REJECT with `reason=role_mismatch` and the appropriate sub-reason (§5.4). The rejection message SHOULD indicate which check failed (capability, trust, depth, or resource) to aid the delegator's reassignment decision.

**Attestation sequence:**

```
Delegator                              Delegatee
    |                                      |
    |  TASK_ASSIGN (with delegation_token  |
    |    and task requirements)            |
    |------------------------------------->|
    |                                      | 1. Check: do I have all required_capabilities?
    |                                      | 2. Check: are required_capabilities ⊆ allowed_capabilities?
    |                                      | 3. Check: does trust_level permit required operations?
    |                                      | 4. Check: is delegation depth within limit?
    |                                      | 5. Check: are required_resources within my bounds?
    |                                      |
    |  TASK_ACCEPT or TASK_REJECT          |
    |<-------------------------------------|
```

The delegatee MUST NOT begin execution before completing all checks and sending TASK_ACCEPT. Execution before acceptance is a protocol violation — work performed without confirmed authorization is not attributable within the trust chain.

### 5.7 Mismatch Handling at Attestation Time

When any attestation check (§5.6) fails, the delegatee sends TASK_REJECT with a structured reason. This is the only protocol-defined mechanism for handling mismatches at delegation time. See §5.4 for the full set of mismatch reasons.

The delegator receives the rejection and MAY:
- Reassign the task to a different agent whose manifest covers the requirements
- Adjust the task requirements to match the available agent pool
- Adjust the delegation token's `allowed_capabilities` and retry

The protocol does not define retry semantics for rejected TASK_ASSIGN messages — the delegator simply sends a new TASK_ASSIGN (which may be to the same or a different agent).

### 5.8 Dynamic Capability Discovery (CAPABILITY_REQUEST)

Task requirements (§5.2) may not be fully known at delegation time. A delegatee executing a data analysis task may discover mid-execution that the data requires a parsing capability not listed in the original task requirements or authorized in the delegation token. Requiring all task requirements to be known upfront penalizes exploratory tasks.

CAPABILITY_REQUEST is the protocol mechanism for mid-task dynamic capability discovery. It is task-side — the request originates from a specific task's execution discovering a new requirement, not from the agent's capabilities changing.

**CAPABILITY_REQUEST message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | The active task requiring the new capability |
| correlation_id | UUID v4 | Yes | Unique identifier for request-response correlation (§4.14.4). Response messages (CAPABILITY_REQUEST_APPROVED, CAPABILITY_REQUEST_DENIED) MUST echo this value. |
| session_id | string | Yes | Active session identifier |
| agent_id | string | Yes | Identity of the requesting agent |
| requested_capabilities | array | Yes | Capability_ids the task needs but the agent does not currently have authorized for this task |
| justification | string | Yes | Why the capability is needed — the delegator uses this to evaluate the request |
| timestamp | ISO 8601 | Yes | When the request was generated |

**CAPABILITY_REQUEST response:**

The delegator responds with one of:

| Response | Meaning |
|----------|---------|
| CAPABILITY_REQUEST_APPROVED | The delegator authorizes the requested capabilities. The delegator issues a CAPABILITY_GRANT (§5.8.2) with updated `allowed_capabilities` and an updated `delegation_token`. The task's effective `required_capabilities` are updated to include the newly authorized capabilities. |
| CAPABILITY_REQUEST_DENIED | The delegator denies the request. The delegatee MUST continue without the requested capability or send TASK_FAIL if the task cannot be completed. |

**Constraints:**

1. The delegatee MUST NOT use a requested capability before receiving CAPABILITY_REQUEST_APPROVED. Using an unauthorized capability — even one the agent is technically capable of — is a trust violation (§5.3 privilege model: capability without trust authorization is prohibited).
2. The delegator can only approve capabilities that it itself holds. The no-privilege-escalation rule (§5.5, §6.8) applies: dynamic capability grants cannot exceed the delegator's own authorization.
3. CAPABILITY_REQUEST does not replace upfront declaration. Manifest-first at session establishment (§5.1) and task requirements at delegation time (§5.2) remain the defaults. CAPABILITY_REQUEST handles the gap between what was foreseeable at planning time and what is needed at execution time.
4. In delegation chains, CAPABILITY_REQUEST may need to propagate upward. If B needs a capability that A did not authorize, and A does not hold that capability either, A must request it from its own delegator before approving B's request. Each hop in the chain applies its own trust check.
5. CAPABILITY_REQUEST is a task-side operation. The agent's CAPABILITY_MANIFEST does not change — only the task's effective authorization (via updated delegation token) changes. This maintains the agent-side/task-side separation: the manifest declares what the agent can do; the delegation token authorizes what it may do for this task.

### 5.8.1 Mid-Session Capability Changes (CAPABILITY_UPDATE)

CAPABILITY_REQUEST (§5.8) handles task-side discovery of new capability needs. CAPABILITY_UPDATE handles the complementary case: agent-side capability changes during a session. An agent that loses a capability mid-session (e.g., a plugin is unloaded, a resource becomes unavailable) or gains a new capability (e.g., a new tool is installed) uses CAPABILITY_UPDATE to notify its session counterpart.

**CAPABILITY_UPDATE message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
| correlation_id | UUID v4 | Yes | Unique identifier for this message (§4.14.4). Fresh UUID v4 — CAPABILITY_UPDATE is unidirectional, not a response. |
| agent_id | string | Yes | Identity of the agent whose capabilities changed. |
| removed | array | Yes | Capability IDs (§5.1.1 format) that are no longer available. Empty array `[]` if no capabilities were removed. |
| added | array | No | Capability IDs (§5.1.1 format) that are newly available. Empty array or omitted if no capabilities were added. |
| reason | string | Yes | Human-readable explanation of why capabilities changed (e.g., `"plugin unloaded"`, `"resource quota exceeded"`, `"tool upgrade"`). |
| timestamp | ISO 8601 | Yes | When the capability change occurred. |
| signature | bytes | No | Cryptographic signature over the message fields, bound to agent_id. RECOMMENDED for auditability but not required for V1. |

**Semantics:**

- CAPABILITY_UPDATE is an agent-side notification — it reflects a change in what the agent can do, not a change in what a task needs (which is CAPABILITY_REQUEST's domain).
- The receiving agent MUST update its local view of the sender's CAPABILITY_MANIFEST to reflect the removed and added capabilities.
- If a removed capability is in the session's `effective_cap_set` (§5.9) or is referenced by any active delegation token's `allowed_capabilities`, the session enters **degraded mode**: the capability is marked as unavailable and new task assignments MUST NOT reference it.
- If a removed capability is listed as `requested_mandatory` for the session (§4.3), the session MUST transition to SUSPENDED (§4.2). From SUSPENDED, the coordinator MAY initiate capability renegotiation by sending a new `requested_mandatory` / `requested_optional` set, or terminate the session via SESSION_CLOSE.
- Added capabilities expand the agent's manifest for the remainder of the session. The counterpart MAY use newly added capabilities in subsequent TASK_ASSIGN messages.
- CAPABILITY_UPDATE does not retroactively affect in-progress tasks. Tasks already accepted (TASK_ACCEPT sent) continue under their original delegation token — capability removal does not automatically fail active tasks. If the delegatee can no longer complete an active task due to capability loss, it MUST send TASK_FAIL with an appropriate error.

**Degraded mode:**

Degraded mode is not a distinct session state (§4.2) — the session remains ACTIVE. Degraded mode is a local bookkeeping condition: one or more previously available capabilities are no longer available. The session continues for tasks that do not require the removed capabilities. Degraded mode ends when the missing capability is restored via a subsequent CAPABILITY_UPDATE with the capability in the `added` array.

### 5.8.2 Capability Grant with TTL (CAPABILITY_GRANT)

<!-- Implements #86: TTL field definitions for CAPABILITY_GRANT -->

CAPABILITY_REQUEST_APPROVED (§5.8) signals that a capability request was accepted, but the actual grant — the authorization artifact the receiving agent operates under — needs explicit temporal bounds. Without TTL semantics, a granted capability is implicitly valid for the lifetime of the session, which creates a trust decay surface: an agent operating on a grant issued hours ago carries the same authorization weight as one issued seconds ago. Production failure data from adlibrary.com documented 47 operations on expired credentials across 3 sessions where session continuity masked trust decay.

CAPABILITY_GRANT is the message type that carries the authorized capability set from a delegator to a delegatee. It is issued in response to CAPABILITY_REQUEST_APPROVED (§5.8), as part of initial TASK_ASSIGN delegation (embedded in or accompanying the `delegation_token`, §5.5), or as a standalone grant when the delegator proactively authorizes capabilities.

**CAPABILITY_GRANT message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| grant_id | UUID v4 | Yes | Unique identifier for this grant instance. |
| correlation_id | UUID v4 | Yes | Unique identifier for request-response correlation (§4.14.4). |
| idempotency_key | UUID v4 | Yes | Sender-generated key for deduplication (§4.14.3). The receiver MUST return the same response for duplicate keys without reprocessing. |
| task_id | UUID v4 | Yes | The task for which capabilities are granted. |
| session_id | string | Yes | Active session identifier. |
| grantor_id | string | Yes | Identity of the agent issuing the grant. |
| grantee_id | string | Yes | Identity of the agent receiving the grant. |
| granted_capabilities | array | Yes | Capability IDs (§5.1.1 format) authorized by this grant. MUST be a subset of the grantor's own authorized capabilities (no-privilege-escalation rule, §5.5, §6.8). |
| delegation_token | object | Yes | Updated delegation token (§5.5) reflecting the granted capabilities. |
| granted_at | ISO 8601 | No | Timestamp when the grant was issued by the grantor. SHOULD be included — absence prevents clock skew detection. Lets receiving agents detect clock skew: if `granted_at` is in the future beyond the 30-second tolerance window, the agent SHOULD flag the grant as a configuration anomaly and request a fresh one (see **Clock skew detection via `granted_at`** below). |
| valid_until | ISO 8601 | Yes | Expiry time for this grant. REQUIRED — every CAPABILITY_GRANT MUST include an explicit `valid_until`. Receiving agents MUST treat the grant as invalid when `current_time > valid_until` — this is a protocol obligation, not advisory. Maximum TTL for V1: 24 hours (`valid_until` MUST NOT exceed `granted_at + 24h`, or `current_time + 24h` when `granted_at` is absent). Grants exceeding the 24-hour maximum MUST be rejected by the receiver. Enforcement is at the capability level, not the session level — each grant carries its own temporal bound independent of session lifetime. See §6.17 for time-bounded capability grant semantics. |
| behavioral_constraint_manifest | object | No | Signed behavioral constraint manifest — a compact declaration of what B is authorized to do within this grant's scope. When present, B MUST acknowledge the manifest at session init, re-attest compliance at each HEARTBEAT (via `manifest_compliance` field), and declare DRIFTED (§4.2.3) if it can no longer comply. See **Behavioral constraint manifest semantics** below. |
| parent_grant_hash | string | Conditional | SHA-256 hex digest of the parent CAPABILITY_GRANT's canonical JSON (RFC 8785 JCS, §4.10.2). REQUIRED when this grant is a sub-grant (issued by an agent that itself holds a parent grant for the same capability set). MUST be `null` for root grants (issued directly by an authority anchor with no parent grant). The hash is computed over all fields of the parent CAPABILITY_GRANT except `parent_grant_hash` itself — this prevents circular hash dependencies while capturing the exact parent state at delegation time. Post-issuance mutation of the parent grant is detectable: the hash captures the parent's state at the moment of sub-grant issuance. See **Sub-grant chain integrity** below. |
| parent_grant_id | UUID v4 | Conditional | The `grant_id` of the parent CAPABILITY_GRANT from which this sub-grant derives authority. REQUIRED when `parent_grant_hash` is present. MUST be `null` for root grants. Provides the lookup key for retrieving the parent grant during chain traversal verification. `parent_grant_id` enables retrieval; `parent_grant_hash` enables verification. Both are required for sub-grants — a lookup key without a hash is unverifiable; a hash without a lookup key requires exhaustive search. |
| signature | bytes | No | Cryptographic signature over all other fields (including `parent_grant_hash` and `parent_grant_id` when present), produced by the grantor. RECOMMENDED for auditability. When this grant is a sub-grant, `parent_grant_hash` MUST be included in the signature payload — this prevents chain splicing attacks where a forged sub-grant points to a valid parent but fails signature verification because `parent_grant_hash` is signed. |

**Behavioral constraint manifest semantics:**

When `behavioral_constraint_manifest` is present in CAPABILITY_GRANT, it establishes a behavioral envelope that the grantee MUST operate within for the duration of the grant. The manifest is the delegator's declaration of what the delegatee is authorized to do — distinct from the capability set (which declares what the delegatee *can* do) and the delegation token (which declares the authorization chain).

**Behavioral constraint manifest fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| constraint_manifest_id | UUID v4 | Yes | Unique identifier for this manifest instance. Referenced by DRIFT_DECLARED (§4.2.3) and HEARTBEAT `manifest_compliance` field. |
| authorized_scope | object | Yes | Explicit boundaries of authorized behavior — what operations, data domains, and interaction patterns are permitted. Structure is deployment-specific but MUST be machine-parseable by B for compliance self-assessment. |
| prohibited_actions | array | No | Explicit list of actions B MUST NOT perform under this grant. Complements `authorized_scope` by enumerating specific exclusions. |
| resource_bounds | object | No | Resource consumption limits — maximum memory, compute time, API call rate, storage, etc. B MUST declare DRIFTED if it exceeds or can no longer guarantee these bounds. |
| model_constraints | object | No | Constraints on B's execution model — required model family, minimum capability level, context window requirements, etc. B MUST declare DRIFTED if its execution model changes (e.g., model swap, context compaction that violates minimum context requirements). |
| manifest_signature | bytes | Yes | Cryptographic signature over all other manifest fields, produced by the grantor. Unlike the outer `signature` field (which covers the entire CAPABILITY_GRANT), this signature covers only the behavioral constraint manifest — enabling independent verification of the manifest without requiring the full grant context. |

**Grantee obligations when `behavioral_constraint_manifest` is present:**

1. **Acknowledge at grant acceptance.** B MUST acknowledge the manifest when accepting the CAPABILITY_GRANT. Acceptance without acknowledgment is a protocol violation — B MUST NOT exercise granted capabilities without explicitly accepting the behavioral constraints that accompany them.
2. **Re-attest at each HEARTBEAT.** B MUST include a `manifest_compliance` field in each HEARTBEAT (§4.5.3) set to `true` if B is still operating within the manifest, or `false` if B has detected divergence. Setting `manifest_compliance: false` in a HEARTBEAT is equivalent to sending DRIFT_DECLARED (§4.2.3) — the session transitions to DRIFTED.
3. **Declare DRIFTED on divergence.** If B detects that it can no longer operate within the manifest — scope creep, execution context change, resource boundary exceeded, model swap — B MUST send DRIFT_DECLARED (§4.2.3) and the session transitions to DRIFTED. B MUST NOT continue exercising granted capabilities after detecting manifest divergence without declaring DRIFTED.

**Relationship to re-attestation (§5.10):** The `behavioral_constraint_manifest` is the declaration that re-attestation (§5.10) verifies. Pull-based re-attestation polls (§5.10.1) check whether B can still attest compliance with the manifest accepted at T+0. Non-response to a re-attestation poll triggers the zombie-by-silence path (SUSPECTED → EXPIRED). A response indicating non-compliance is equivalent to DRIFT_DECLARED — the session transitions to DRIFTED. Two failure modes, two protocol paths: silence → zombie; declared divergence → DRIFTED.

**Temporal enforcement semantics:**

- **Expiry is self-enforcing.** The receiving agent is responsible for enforcing `valid_until`. When `current_time > valid_until`, the grant MUST be treated as revoked — the agent MUST stop exercising the granted capabilities for the associated task. This is distinct from requester-initiated revocation (§6.13): expiry is time-based and self-enforcing at the receiving agent; revocation is an explicit delegator action via signed revocation token (§6.13.1) propagated through the delegation chain.
- **No session-scoped default.** `valid_until` is REQUIRED — there is no implicit session-scoped fallback. A CAPABILITY_GRANT without `valid_until` is non-compliant and MUST be rejected by the receiver. This eliminates the trust decay surface identified in production (47 operations on expired credentials, @larryadlibrary issue #15) where session continuity masked grant staleness.
- **Clock skew tolerance on `valid_until`.** Agents MUST tolerate clock skew of at least 30 seconds in either direction when evaluating `valid_until`. A grant whose `valid_until` is within 30 seconds of the receiver's current time MUST NOT be treated as expired solely due to clock divergence. Implementations SHOULD log a warning when evaluating a grant within 60 seconds (2× tolerance window) of expiry to allow proactive renewal. Clock skew tolerance of 30 seconds is production-validated by the HIBI reserve asset protocol (@jacobi_).
- **Clock skew detection via `granted_at`.** When `granted_at` is present, the receiving agent SHOULD compare it against its local clock. If `granted_at` is in the future beyond the 30-second tolerance window, the agent SHOULD flag the grant as a configuration anomaly and request a fresh one — gross clock skew is a deployment issue, not a trust violation. The agent MUST NOT hard-fail on future `granted_at`; instead it SHOULD log the anomaly with both timestamps and the computed skew. If `granted_at` is between 0 and 30 seconds in the future, the agent MAY accept the grant but MUST log a warning indicating clock skew was detected and the tolerance window was applied.
- **Pre-expiry behavior and grace period.** When `current_time > valid_until`, the grant is expired. Atomic operations already in progress at TTL expiry MAY complete within a 5-second grace period (consistent with best-effort TTL expiry semantics, §6.17.4). Operations that cannot complete within the grace period MUST abort and report `TTL_EXPIRED`. The grace period applies only to operations that were in-flight at the moment of expiry — new operations MUST NOT be initiated after `valid_until`. Agents that need continued authorization beyond the grace period MUST obtain a CAPABILITY_RENEW (§5.8.3) before expiry.

**Sub-grant chain integrity:**

When an agent (B) holding a CAPABILITY_GRANT from a grantor (A) issues a sub-grant CAPABILITY_GRANT to a downstream agent (C), the sub-grant MUST include `parent_grant_hash` and `parent_grant_id` to cryptographically bind it to the parent grant. This closes three security gaps: (1) **post-issuance parent mutation** — `parent_grant_hash` captures the exact parent state at delegation time, making any subsequent modification to the parent grant detectable; (2) **chain splicing** — a forged sub-grant pointing to a valid parent fails signature verification because `parent_grant_hash` is included in the `signature` payload; (3) **parent revocation cascades implicitly** — when the parent grant is revoked, chain validation fails because the parent can no longer be resolved as valid, invalidating all downstream sub-grants regardless of whether they were separately revoked.

**Canonical JSON computation for `parent_grant_hash`:**

The `parent_grant_hash` is computed over the parent CAPABILITY_GRANT message's canonical JSON, using the same canonicalization rules as §4.10.2 (NFC normalization followed by RFC 8785 JCS serialization). The hash input includes all fields of the parent CAPABILITY_GRANT **except** the parent's own `parent_grant_hash` field (if present). This exclusion prevents circular hash dependencies when the parent is itself a sub-grant — the hash captures the parent's authorization state without requiring knowledge of the grandparent's hash computation. All other fields — including `grant_id`, `grantor_id`, `grantee_id`, `granted_capabilities`, `valid_until`, `delegation_token`, `parent_grant_id`, and `signature` — are included in the hash input.

**Root grant identification:**

Root grants — CAPABILITY_GRANT messages issued directly by an authority anchor with no parent grant — set `parent_grant_hash` to `null` and `parent_grant_id` to `null`. A CAPABILITY_GRANT with `parent_grant_hash` present but `parent_grant_id` absent (or vice versa) is malformed and MUST be rejected by the receiver. The two fields are structurally paired: both present for sub-grants, both `null` for root grants.

**Verification at presentation time:**

Receiving agents MUST verify CAPABILITY_GRANT chain integrity at presentation time — when the grant is first exercised, not just when it is received. Verification proceeds as follows:

1. **Resolve parent.** Using `parent_grant_id`, retrieve the parent CAPABILITY_GRANT. If the parent grant cannot be resolved (not found, revoked, or expired), reject the sub-grant with `reason: "parent_grant_unresolvable"`.
2. **Compute hash.** Compute the SHA-256 hash of the resolved parent grant's canonical JSON (excluding the parent's own `parent_grant_hash` field).
3. **Compare hash.** Compare the computed hash against the `parent_grant_hash` in the sub-grant. If they do not match, reject the sub-grant and emit DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure`.
4. **Verify signature.** Verify that the sub-grant's `signature` covers `parent_grant_hash` — confirming the issuing agent (B) cryptographically bound the sub-grant to the claimed parent state.
5. **Recurse.** If the parent grant is itself a sub-grant (its own `parent_grant_hash` is non-null), repeat steps 1–4 for the parent's parent. Traversal terminates at a root grant (`parent_grant_hash: null`, `parent_grant_id: null`).

**Partial chains do not confer authority.** If any link in the CAPABILITY_GRANT chain fails verification — hash mismatch, unresolvable parent, invalid signature — the entire chain from that point to the leaf is invalid. This is analogous to the delegation chain verification rule in §6.9.3.2.

**Chain traversal in partition scenarios:** The `parent_grant_hash` embedding enables offline chain verification. A verifier holding cached copies of ancestor grants can verify the chain by hash comparison without contacting the original grantor or any intermediate agent. This mirrors the partition-tolerant design of delegation chain verification (§6.9.3.2).

**Audit consequence:** When CAPABILITY_GRANT chain verification fails, the detecting agent MUST emit a DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure`. The `GRANT_CHAIN_INTEGRITY_FAILURE` audit event (§6.13.4) MUST be recorded with the failing `grant_id`, `parent_grant_id`, the computed hash, the claimed `parent_grant_hash`, and the identity of the detecting agent.

> Addresses [issue #93](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/93): sub-grant chain integrity for CAPABILITY_GRANT. Adds mandatory `parent_grant_hash` (SHA-256 hex) and `parent_grant_id` (UUID) fields for sub-grants, includes `parent_grant_hash` in the signature payload to prevent chain splicing, defines recursive chain verification at presentation time, and cascades parent revocation through chain validation. Closes #93.

**Relationship to CAPABILITY_REQUEST (§5.8):** CAPABILITY_GRANT is the authorization artifact issued when CAPABILITY_REQUEST is approved. CAPABILITY_REQUEST is the request; CAPABILITY_GRANT is the grant. A CAPABILITY_REQUEST_APPROVED response SHOULD be accompanied by a CAPABILITY_GRANT message carrying the updated authorization with explicit temporal bounds.

**Relationship to delegation token (§5.5):** The `delegation_token` field within CAPABILITY_GRANT carries the updated token reflecting the newly authorized capabilities. The `valid_until` field on the CAPABILITY_GRANT governs the temporal validity of the grant itself; the `ttl` field on the delegation token (§5.5) governs the delegation chain's time-to-live. These are complementary — a grant may expire before the delegation token's TTL, or vice versa. The more restrictive of the two applies.

### 5.8.3 Capability Renewal (CAPABILITY_RENEW)

<!-- Implements #86: CAPABILITY_RENEW message type -->

Re-issuing a CAPABILITY_GRANT to extend authorization is semantically ambiguous — a fresh grant and a renewal carry different trust implications. A fresh grant implies a new authorization decision; a renewal implies continuity of an existing authorization. Conflating the two in the audit trail makes it impossible to distinguish "the delegator re-evaluated and re-authorized" from "the delegator extended an existing authorization without re-evaluation." CAPABILITY_RENEW makes renewal intent unambiguous and creates a distinct protocol event in the audit trail. Production-validated by HIBI, where CAPABILITY_RENEW is used in production for reserve asset delegation chains.

**CAPABILITY_RENEW MUST be an explicit separate message type.** Implementations MUST NOT implement automatic or implicit grant refresh. Explicit renewal makes trust extension visible in the audit trail and forces agents to actively request continued authorization. Automatic refresh inverts the trust model: it makes the granting agent a runtime dependency rather than a design-time contract. The point of a capability grant is a self-contained authorization token the grantee can reason about locally — automatic refresh couples the grantee's ongoing authorization to the grantor's availability, converting a token-based model into a session-based one. Source: @jacobi_ HIBI explicit-over-implicit design choice; @mote-oo per-operation revalidation analysis from the TTL Moltbook thread.

**CAPABILITY_RENEW message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| renew_id | UUID v4 | Yes | Unique identifier for this renewal instance. |
| correlation_id | UUID v4 | Yes | Unique identifier for request-response correlation (§4.14.4). |
| original_grant_id | UUID v4 | Yes | The `grant_id` of the CAPABILITY_GRANT being renewed. Binds the renewal to a specific prior grant for audit trail continuity. |
| task_id | UUID v4 | Yes | The task for which capabilities are being renewed. MUST match the `task_id` of the original grant. |
| session_id | string | Yes | Active session identifier. |
| grantor_id | string | Yes | Identity of the agent issuing the renewal. MUST match the `grantor_id` of the original grant. |
| grantee_id | string | Yes | Identity of the agent receiving the renewal. MUST match the `grantee_id` of the original grant. |
| granted_capabilities | array | Yes | Capability IDs authorized by this renewal. MUST be identical to the original grant's `granted_capabilities` — renewals do not expand or contract the capability set. To change the capability set, issue a new CAPABILITY_GRANT. |
| granted_at | ISO 8601 | No | Timestamp when the renewal was issued. Same clock skew semantics as CAPABILITY_GRANT (§5.8.2). |
| valid_until | ISO 8601 | Yes | New expiry time. MUST be after the current time. MAY be before or after the original grant's `valid_until` — a renewal can shorten or extend the validity window. |
| signature | bytes | No | Cryptographic signature over all other fields, produced by the grantor. RECOMMENDED for auditability. |

**CAPABILITY_RENEW semantics:**

- **Scope preservation.** `granted_capabilities` in a CAPABILITY_RENEW MUST be identical to the original CAPABILITY_GRANT's `granted_capabilities`. A renewal that changes the capability set is a protocol violation — the receiving agent MUST reject it. To change the authorized capability set, the delegator issues a new CAPABILITY_GRANT (which is a fresh authorization decision, not a renewal).
- **Identity binding.** `grantor_id`, `grantee_id`, and `task_id` MUST match the original grant. A renewal from a different grantor or for a different grantee or task is not a renewal — it is a new grant and MUST use CAPABILITY_GRANT.
- **Temporal continuity.** CAPABILITY_RENEW MAY be issued before or after the original grant's `valid_until`. When issued before expiry, it extends authorization seamlessly. When issued after expiry, it re-establishes authorization — the receiving agent MUST NOT have exercised the granted capabilities between the original expiry and the renewal's `granted_at`.
- **Audit trail.** CAPABILITY_RENEW creates a distinct event from CAPABILITY_GRANT in the audit log. Auditors can distinguish "authorization was extended" (renewal) from "authorization was re-evaluated" (new grant) without inspecting capability sets for equality.
- **Receiving agent behavior.** On receiving CAPABILITY_RENEW, the agent MUST validate that `original_grant_id` references a known grant, that identity fields match, and that `granted_capabilities` is identical to the original. If validation passes, the agent updates the grant's `valid_until` to the new value. If validation fails, the agent MUST reject the renewal and MUST NOT update the grant's validity.

### 5.9 Timing

Manifest-first at session establishment is the correct default. Deferring capability negotiation to task assignment time (lazy per-task negotiation) compounds round-trip latency in multi-hop delegation chains: each hop must negotiate capabilities before the next hop can begin, serializing what should be parallelizable.

**Session establishment flow (standard):**

```
Agent A                     Agent B
    |                          |
    |  SESSION_INIT            |
    |------------------------->|
    |                          |
    |  CAPABILITY_MANIFEST     |
    |<-------------------------|
    |  CAPABILITY_MANIFEST     |
    |------------------------->|
    |                          |
    |  (Capabilities known;    |
    |   task requirements      |
    |   can be matched against |
    |   manifests at           |
    |   delegation time)       |
    |                          |
    |  TASK_ASSIGN             |
    |  (with task requirements |
    |   + delegation token)    |
    |------------------------->|
```

**0-RTT session establishment flow:**

When the coordinator already knows (or can predict) the worker's capabilities — e.g. from a prior session or from registry data (§3) — it can include `manifest_digest`, `requested_mandatory`, and `requested_optional` in SESSION_INIT (§4.3). This eliminates the separate CAPABILITY_MANIFEST round-trip for the common happy-path case.

```
Agent A (coordinator)       Agent B (worker)
    |                          |
    |  SESSION_INIT            |
    |  (manifest_digest,       |
    |   requested_mandatory,   |
    |   requested_optional)    |
    |------------------------->|
    |                          | 1. Verify manifest_digest matches
    |                          |    own capability set
    |                          | 2. Compute effective_cap_set =
    |                          |    intersection of requested caps
    |                          |    with own manifest, filtered
    |                          |    by policy
    |                          | 3. Check all requested_mandatory
    |                          |    caps are in effective_cap_set
    |                          |
    |  SESSION_INIT_ACK        |
    |  (effective_cap_set)     |
    |<-------------------------|
    |                          |
    |  (Session is ACTIVE;     |
    |   no separate manifest   |
    |   exchange needed)       |
    |                          |
    |  TASK_ASSIGN             |
    |  (with task requirements |
    |   + delegation token)    |
    |------------------------->|
```

**0-RTT semantics:**

- `manifest_digest` is a Merkle root computed over the coordinator's capability set. Each leaf node is `SHA-256(cap_id ‖ impl_hash ‖ policy_hash)`, where `impl_hash` is the hash of the capability implementation metadata and `policy_hash` is the hash of the coordinator's access policy for that capability. The tree is constructed over the sorted list of leaf hashes.
- `requested_mandatory` lists capability IDs the coordinator requires. See **Mandatory vs. optional enforcement** below for precise rejection semantics.
- `requested_optional` lists capability IDs the coordinator prefers but does not require. Missing optional capabilities do not block session establishment.
- `effective_cap_set` in SESSION_INIT_ACK is the intersection of all requested capabilities (mandatory + optional) with the worker's actual capabilities, filtered by the worker's policy. This is the agreed capability set for the session.
- If SESSION_INIT does not include `manifest_digest`, the standard two-step CAPABILITY_MANIFEST exchange is used. The 0-RTT path is an optimization, not a replacement — both paths reach the same end state (a known set of available capabilities).

**Mandatory vs. optional enforcement:**

The `requested_mandatory` and `requested_optional` arrays in SESSION_INIT carry distinct enforcement semantics:

1. **Mandatory capability absence — session rejection.** If any capability ID in `requested_mandatory` is absent from the worker's manifest (applying the version compatibility predicate from §5.1.1), SESSION_INIT_ACK MUST reject the session outright. The rejection response MUST include a `missing_mandatory` array enumerating every mandatory capability ID that the worker cannot satisfy. The session MUST NOT transition to ACTIVE. Partial mandatory satisfaction is not permitted — all mandatory capabilities must be present, or the session does not proceed.

2. **Optional capability absence — degraded mode.** If all `requested_mandatory` capabilities are satisfied but one or more `requested_optional` capabilities are absent from the worker's manifest, the session proceeds normally. SESSION_INIT_ACK returns `effective_cap_set` reflecting only the capabilities actually available — the intersection of all requested capabilities (mandatory + optional) with the worker's manifest, filtered by policy. Missing optional capabilities are silently omitted from `effective_cap_set`. The coordinator MUST NOT assume optional capabilities are available unless they appear in `effective_cap_set`.

3. **Rejection message structure.** When mandatory capabilities are missing, SESSION_INIT_ACK MUST include:

   | Field | Type | Description |
   |-------|------|-------------|
   | session_id | string | Echoed from SESSION_INIT. |
   | status | enum | `rejected` |
   | reason | string | `missing_mandatory_capabilities` |
   | missing_mandatory | array | Capability IDs from `requested_mandatory` that the worker cannot satisfy. Each entry is the exact capability ID string from the coordinator's request. |

   The coordinator receives the enumerated missing capabilities and MAY use this information to select a different worker, adjust requirements, or terminate the collaboration attempt.

CAPABILITY_MANIFEST exchange happens once per session. Task assignment carries task-specific requirements (§5.2) and references the already-known capabilities via `delegation_token.allowed_capabilities` — no additional capability negotiation round-trip is needed.

Dynamic capabilities discovered during execution are handled via CAPABILITY_REQUEST (§5.8), not by deferring initial declaration.

### 5.10 Behavioral Re-attestation Initiation

<!-- Implements #132: behavioral re-attestation initiation model for §5 -->

§5.6 defines attestation before TASK_ACCEPT — a one-time check at delegation time. For long-running tasks, the delegatee's behavioral state may drift after initial attestation: capabilities may degrade, context may compact, or constraints may be violated without producing a protocol-visible signal. Re-attestation is the mechanism that bounds detection time for post-acceptance behavioral drift.

Three initiation models exist with non-equivalent fault semantics:

| Model | Mechanism | Detection latency | Failure mode |
|-------|-----------|-------------------|-------------|
| **Push** | B re-attests on a schedule | Bounded by attestation frequency | B's crash or drift mid-interval is invisible until the next scheduled attestation. Detection latency scales with attestation frequency. |
| **Pull** | A polls B | Bounded by `attestation_interval` + `expected_response_latency` | A controls the detection window independently of B's behavior. Detection cost moves to A. |
| **Event-triggered** | B re-attests on state change | Unbounded | Requires B to reliably detect its own drift — exactly what a drift-zombie (§8.1) cannot do. Reliability hole when B is the compromised party. |

These models are **not interchangeable**. Push and event-triggered both depend on B's correct behavior for detection to work. When B is the party that has drifted, B cannot be trusted to initiate its own re-attestation — this is the drift-zombie problem (§8.1, §4.7.1 phenomenological blindness).

#### 5.10.1 Pull as Primary Model

**Pull is the PRIMARY re-attestation initiation model.** Rationale: only pull guarantees bounded detection time independently of B's behavior. The delegator (A) controls the polling schedule; B's crash, drift, or compromise does not affect A's ability to detect the problem.

**Pull semantics:**

1. A polls B at a configurable `attestation_interval`. This interval is negotiated at SESSION_INIT (§4.3) or set by the delegator at TASK_ASSIGN (§6.6). Default: equal to `semantic_check_interval_ms` (§8.9.5) when two-tier heartbeat is active; otherwise deployment-specific.
2. B MUST respond within `expected_response_latency`. This is the maximum time B has to produce a valid re-attestation response after receiving a poll. Default: 10000ms (10 seconds). Negotiable at SESSION_INIT.
3. Non-response (no valid re-attestation response within `expected_response_latency`) triggers the zombie-by-silence escalation path: A declares B a potential zombie and follows the ZOMBIE_DECLARED escalation (§8.9.3) or the SUSPECTED → EXPIRED path (§8.9.4), depending on the session's heartbeat configuration.
4. Failed re-attestation (B responds but indicates non-compliance with the behavioral constraint manifest, §5.8.2) triggers the zombie-by-drift path: the session transitions to DRIFTED (§4.2.3). B is reachable and cooperative but has diverged from its authorized behavioral envelope. A then decides: re-negotiate constraints, terminate, or invoke revocation.

**Detection time bound:**

The maximum time between the onset of behavioral drift in B and A's detection of that drift is:

```
max_detection_latency = attestation_interval + expected_response_latency
```

This bound holds regardless of B's behavior — it depends only on A's polling schedule and the response timeout. Deployments MUST derive `attestation_interval` from the maximum acceptable damage window (§8.5 staleness tolerance), not the other way around: pick the maximum tolerable detection latency, subtract `expected_response_latency`, and the remainder is the maximum allowable `attestation_interval`.

**Re-attestation poll message:**

The pull model reuses the SEMANTIC_CHALLENGE / SEMANTIC_RESPONSE mechanism (§8.9.2) as its wire format. A re-attestation poll is a SEMANTIC_CHALLENGE with the same fields and verification semantics. This avoids introducing a redundant message type — behavioral re-attestation and semantic liveness verification are the same operation at the protocol level.

**Relationship to §8.9 (Two-Tier Heartbeat):** When two-tier heartbeat is active, Tier 2 semantic challenges (§8.9.2) serve as the pull-based re-attestation mechanism. The `semantic_check_interval_ms` parameter is the `attestation_interval` for behavioral re-attestation. Sessions that negotiate two-tier heartbeat automatically get pull-based behavioral re-attestation at the Tier 2 interval. Sessions that do not negotiate two-tier heartbeat but require behavioral re-attestation MUST negotiate `attestation_interval` and `expected_response_latency` as separate SESSION_INIT parameters.

**Relationship to DRIFTED state (§4.2.3):** Re-attestation poll responses produce two distinct failure paths. Non-response (silence) triggers zombie-by-silence escalation — the SUSPECTED → EXPIRED path. Response indicating non-compliance with the behavioral constraint manifest (§5.8.2) triggers zombie-by-drift — the DRIFTED path. Both failure modes are now captured by the protocol: silence → ZOMBIE/EXPIRED; declared divergence → DRIFTED. The `behavioral_constraint_manifest` in CAPABILITY_GRANT (§5.8.2) is the artifact against which re-attestation compliance is measured.

#### 5.10.2 Push and Event-Triggered as Named Alternatives

Push and event-triggered are **named alternative initiation models** — not equivalents to pull. Deployments MAY use them as supplements to pull but MUST NOT use them as replacements.

**Push model:**

- B re-attests on a self-managed schedule, sending unsolicited re-attestation messages at a configured frequency.
- **Tradeoff profile:** Lower polling cost on A. B controls attestation timing. Detection latency is bounded by B's attestation frequency — but only if B is functioning correctly. If B crashes, hangs, or drifts, no attestation is sent, and A has no detection signal until the next pull poll (if pull is also active) or until session expiry.
- **When appropriate:** As a supplement to pull in high-frequency attestation scenarios where A wants to reduce its polling overhead. B's push attestations provide early signals between A's pull polls.
- **Constraint:** Push MUST NOT be the sole re-attestation initiation mechanism. A session that relies exclusively on push has unbounded detection latency when B fails silently.

**Event-triggered model:**

- B re-attests when it detects a state change that could affect its behavioral compliance (e.g., capability loss, context compaction, resource exhaustion).
- **Tradeoff profile:** Lowest steady-state cost — no periodic messages when nothing changes. Provides the fastest possible detection for state changes that B can self-detect. However, it relies on B to correctly identify its own drift, which is precisely what a drift-zombie cannot do (§8.1). An agent that has lost context does not know it has lost context (§4.7.1 phenomenological blindness).
- **When appropriate:** As a supplement to pull for early detection of self-observable state changes. Event-triggered attestation can detect some drift classes (explicit capability loss, plugin unload, resource quota exceeded) faster than periodic polling.
- **Constraint:** Event-triggered MUST NOT be the sole re-attestation initiation mechanism. It MUST be used only as a supplement to pull, never as a replacement. The reliability hole — B cannot detect its own phenomenological blindness — is fundamental, not an implementation deficiency.

#### 5.10.3 Initiation Model Selection

| Configuration | Valid? | Rationale |
|--------------|--------|-----------|
| Pull only | Yes | Sufficient for all detection scenarios. Highest polling cost but guaranteed bounded detection. |
| Pull + push | Yes | Push reduces detection latency between pull intervals. Pull provides the safety net when B fails. |
| Pull + event-triggered | Yes | Event-triggered provides early detection for self-observable drift. Pull covers the phenomenological blindness gap. |
| Pull + push + event-triggered | Yes | Maximum detection coverage. Appropriate for high-trust deployments. |
| Push only | **No** | Unbounded detection latency when B fails silently. |
| Event-triggered only | **No** | Drift-zombie cannot detect its own drift. Fundamental reliability hole. |
| Push + event-triggered (no pull) | **No** | Both depend on B's correct behavior. No independent detection by A. |
| None | **No** | No re-attestation. Acceptable only for tasks shorter than the session's `session_expiry_ms` where initial attestation (§5.6) is sufficient. |

> Behavioral re-attestation initiation model formalized from [issue #132](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/132). The core insight: re-attestation initiation must not depend on the behavior of the party being attested. Only pull guarantees this property — push and event-triggered both fail when B is the compromised party.

### 5.11 Relationship to Other Sections

- **§4 (Session Lifecycle).** Trust anchor requirements from §4.7.2 apply to delegation tokens. The `signature` field in the delegation token, and the chain of signatures across delegation hops, are verification artifacts. External verifiers (§4.7.2) MAY validate delegation token chains as part of runtime liveness verification — a token with an expired TTL or broken signature chain is evidence of anomalous state.
- **§6 (Task Delegation).** TASK_ASSIGN (§6.6) carries the `delegation_token` defined in §5.5 and the task requirements defined in §5.2. The trust semantics in §6.8 and delegation chains in §6.9 operate on the authorization context established by role negotiation. §5 defines the authorization structure (capability manifest + task requirements + privilege model); §6 defines the delegation lifecycle that uses it. Version negotiation scoping (§5.13, item 7) is per-hop — each session negotiates independently. The `protocol_version_chain` in TASK_ASSIGN and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.9.1) provide the compensating visibility mechanism.
- **§4 (Session Lifecycle) / §8 (Error Handling).** Session expiry auto-revokes all active delegation tokens and CAPABILITY_GRANTs without explicit `valid_until` for that session. When a session ends (§4.8 SESSION_RESUME mismatch leading to RESTART, or normal termination via §4.9 SESSION_CLOSE), all delegation tokens issued within that session become invalid, and all session-scoped CAPABILITY_GRANTs (those without `valid_until`, §5.8.2) are implicitly revoked. Capability manifests remain valid — they are agent-side declarations independent of any session. A delegatee that continues operating on an expired session's token or an expired CAPABILITY_GRANT is in violation — the external verifier (§4.7.2) SHOULD detect this via TTL expiry or `valid_until` expiry.
- **§9 (Security Considerations).** Capability/trust collapse is the primary privilege escalation vector in multi-agent delegation chains. The privilege model (§5.3) prevents this collapse by maintaining the three-axis separation. §9.2's trust topologies determine how trust levels are assigned; §5 ensures that those trust levels are carried explicitly in delegation tokens rather than inferred from capability declarations. The translation boundary risk (§9.3) applies to CAPABILITY_MANIFEST exchange across trust domains — a manifest attested in one domain does not carry attestation into another.
- **§8 (Error Handling — Behavioral Re-attestation).** Behavioral re-attestation (§5.10) reuses the SEMANTIC_CHALLENGE / SEMANTIC_RESPONSE mechanism (§8.9.2) as its wire format. Pull-based re-attestation polling is the §5 initiation model; §8.9 defines the challenge-response semantics and ZOMBIE_DECLARED escalation path. When two-tier heartbeat (§8.9) is active, Tier 2 semantic challenges serve as the pull-based re-attestation mechanism — the `semantic_check_interval_ms` parameter is the `attestation_interval` for behavioral re-attestation. Re-attestation poll responses now produce two failure paths: non-response triggers zombie-by-silence (SUSPECTED → EXPIRED); response indicating manifest non-compliance triggers zombie-by-drift (ACTIVE → DRIFTED, §4.2.3).
- **§4 (Session Lifecycle — DRIFTED State).** The `behavioral_constraint_manifest` field in CAPABILITY_GRANT (§5.8.2) is the authorization artifact against which DRIFTED state (§4.2.3) is determined. When present, B MUST re-attest compliance at each HEARTBEAT and declare DRIFTED if it diverges. This creates the behavioral constraint pinning mechanism: A pins B's behavioral envelope at grant time; B continuously attests compliance; divergence triggers an explicit protocol path distinct from both silence (zombie) and hostility (revoked).
- **§8 (Error Handling — Verifiable Intent).** §5 is the **declaration layer**: it specifies *what* capabilities an agent claims and *what* capabilities a session or task requires. §8 is the **trust layer**: it specifies *how* to verify that declarations are honest and that agents actually exercise declared capabilities correctly. In basic deployments, §5 alone is sufficient — capability declarations are self-reported (§5.1.2), cryptographically bound to agent identity, and matched against task requirements at delegation time. In high-trust deployments where spoofed capability declarations are a threat model concern, §8 attestation provides independent verification: CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages carry attestation signatures per §8 rules, and the external audit trail (§8.5, §8.8) provides post-hoc evidence of whether an agent actually exercised a declared capability correctly. The relationship is additive: §5 specifies what to declare; §8 specifies how to prove it. §8 attestation is not required for §5 to function, but §5 declarations without §8 attestation carry only the trust level of self-report.

### 5.12 INITIALIZING Commitment Reconciliation

<!-- Implements #64: commitment reconciliation during initialization before ACTIVE transition -->

On session establishment — whether a fresh SESSION_INIT or a reinitiation after teardown-first recovery (§8.13) — the agent MUST reconcile inherited commitments from any prior instance before transitioning to ACTIVE. This reconciliation phase occurs after capability exchange (§5.9) completes but before the agent declares itself ACTIVE and begins accepting new task delegations.

**Reconciliation sequence:**

1. **Read COMMITMENT_REGISTRY.** The agent reads its COMMITMENT_REGISTRY (§7.11) from durable persistent storage (§8.13.4). The registry contains all outstanding COMMITMENT entries (§6.12) from prior sessions that have not been cancelled or fulfilled.

2. **Classify inherited commitments.** For each entry in the COMMITMENT_REGISTRY, the agent determines whether it can honor the commitment given its current context, capabilities, and state:
   - **Honorable commitments:** The agent retains the commitment in its local COMMITMENT_REGISTRY. No outbound message is required — the counterpart agent's expectation remains valid.
   - **Non-honorable commitments:** The agent cannot fulfill the promise made by the prior instance (e.g., due to context loss, capability change, or task no longer being relevant).

3. **Emit DIVERGENCE_REPORT for abandoned commitments.** For each inherited commitment the agent CANNOT honor, the agent MUST emit a DIVERGENCE_REPORT (§8.11) to the `counterpart_agent_id` specified in the commitment entry, with:
   - `reason_code`: `context_shift`
   - `description`: MUST include the `commitment_id`, `original_due_by` (the `due_by` from the original COMMITMENT), and a `reason_detail` (free text explaining why the commitment cannot be honored).
   - `severity`: `WARN` or `ERROR` depending on the commitment_type — `task_completion` and `state_delivery` commitments SHOULD use `ERROR`; `reply` and `presence` commitments SHOULD use `WARN`.
   - `related_task_id`: The `task_id` from the original COMMITMENT, if applicable.

4. **Remove abandoned commitments.** After emitting DIVERGENCE_REPORT for each non-honorable commitment, the agent MUST remove those entries from the COMMITMENT_REGISTRY and persist the updated registry to durable storage.

5. **Transition to ACTIVE.** After reconciliation completes — all inherited commitments are either re-registered (retained) or explicitly abandoned (DIVERGENCE_REPORT emitted) — the agent transitions to ACTIVE. The agent MUST NOT transition to ACTIVE before reconciliation is complete.

**Timing relationship to §5.9:** Commitment reconciliation is logically the final step in the NEGOTIATING → ACTIVE transition. The ordering is: capability exchange (§5.9) → commitment reconciliation (this section) → ACTIVE. Capability exchange must complete first because the agent needs to know its current capabilities before it can assess which inherited commitments it can honor.

**First-session behavior:** On a fresh SESSION_INIT with no prior instance history, the COMMITMENT_REGISTRY is empty. Reconciliation is a no-op — the agent transitions directly from capability exchange to ACTIVE.

> Commitment reconciliation formalized from [issue #64](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/64). The core insight: session management is identity management — every unresolved commitment is a promise that a future instance must keep or explicitly refuse. Making inherited commitments protocol-visible during initialization prevents silent abandonment.

### 5.13 Open Questions

The following tracks resolution status for identified gaps. Resolved items document the V1 decision; open items remain unresolved for future versions.

1. **Capability taxonomy.** ~~The protocol uses opaque capability_id strings (§5.1). Should the protocol define a standard capability taxonomy for interoperability, or is this purely deployment-specific?~~ **Resolved (V1).** Capability IDs use the structured `cap:namespace.capability@version` format (§5.1.1). The namespace component provides collision avoidance across ecosystems; the version component encodes backward-compatibility semantics intrinsic to the ID. V1 requires **exact namespace and capability name match** with **semver-compatible version matching** via the compatibility predicate (§5.1.1): MAJOR must match exactly; MINOR follows backward-compatibility (provider MINOR ≥ requester MINOR). No type equivalence, structural subtyping, or semantic matching beyond the compatibility predicate. Agents that need to bridge between capability representations publish adapter capabilities as separate entries (e.g. `cap:adapter.pathlike_to_string@1`). A canonical type registry and type equivalence inference are explicitly deferred beyond V1.

2. **Manifest freshness.** ~~CAPABILITY_MANIFEST is exchanged at session establishment. If an agent's capabilities change during a long-running session, the manifest becomes stale. Should CAPABILITY_MANIFEST be re-sendable mid-session?~~ **Resolved (V1).** CAPABILITY_UPDATE (§5.8.1) is the protocol mechanism for mid-session capability changes. It handles both capability loss (removed capabilities) and capability gain (added capabilities). Degraded mode semantics are defined for non-mandatory capability loss; mandatory capability loss triggers SESSION_SUSPEND. CAPABILITY_REQUEST (§5.8) remains the mechanism for task-side discovery of new needs; CAPABILITY_UPDATE is the complementary mechanism for agent-side capability changes.

3. **Delegation token revocation propagation.** When a session expires and delegation tokens are revoked (§5.11), how is revocation propagated to delegatees in a multi-hop chain? The delegatee at depth 2 may not have a direct communication channel to the root delegator. Revocation must propagate through intermediaries, introducing latency and potential failure points. **Partially addressed (V1).** §9.8 documents that revocation propagation is advisory, not technically enforceable — the protocol depends on intermediate agents to propagate faithfully, and partial propagation failure is a known limitation. §9.8.2 defines the failure modes and introduces the optional REVOCATION_PROPAGATION_FAILED message for detecting incomplete propagation. Technical enforcement via cryptographic session tokens with built-in TTL is deferred to V2 (§9.8.3).

4. **Trust level granularity in allowed_capabilities.** The current design grants a single `trust_level` per delegation (§6.8). Should `allowed_capabilities` support per-capability trust levels? E.g., an agent might be trusted for `cap:example.net.access@1` at `standard` but only `cap:example.fs.access@1` at `restricted`. Per-capability trust increases expressiveness but also complexity and the surface for misconfiguration.

5. **Task requirement completeness.** ~~Should the task schema include a `requirement_completeness` signal to set delegatee expectations about the likelihood of mid-task CAPABILITY_REQUEST messages?~~ **Resolved (V1).** The 0-RTT capability intersection at SESSION_INIT (§5.9) combined with `requested_mandatory` / `requested_optional` arrays provides upfront requirement signaling. The coordinator declares which capabilities are mandatory vs. optional at session establishment, giving the worker a clear signal about the expected capability surface. For mid-session changes, CAPABILITY_UPDATE (§5.8.1) and CAPABILITY_REQUEST (§5.8) provide complementary mechanisms. A separate `requirement_completeness` field is unnecessary — the mandatory/optional distinction at SESSION_INIT and the existence of CAPABILITY_REQUEST as a defined protocol message already set the expectation that mid-task capability discovery may occur.

6. **§8 attestation relationship.** ~~How should CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages interact with the external audit trail (§8)? Capability claims are self-reported (§5.1.2); §8 error handling and audit mechanisms could provide independent attestation of whether an agent actually exercised a declared capability correctly. The relationship between self-reported capability (§5) and observed capability (§8) is not yet defined.~~ **Resolved (V1).** §5 is the declaration layer (what to declare); §8 is the trust layer (how to prove it). The relationship is defined in §5.11: §5 alone is sufficient for basic operation — capability declarations are self-reported and cryptographically bound to agent identity. §8 attestation is required for high-trust deployments where spoofed capability declarations are a threat model concern. In §8-integrated deployments, CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages carry attestation signatures per §8 rules, and the external audit trail provides post-hoc verification of whether declared capabilities were exercised correctly. §8 attestation is additive — it does not change §5 semantics, only the trust level of §5 declarations.

7. **Version negotiation scoping.** ~~Should version negotiation in multi-hop delegation chains be scoped per-hop (each pair of agents negotiates independently) or end-to-end (the originating agent constrains the minimum version for the entire chain)?~~ **Resolved (V1).** V1 uses **per-hop independent negotiation** — each pair of agents in a delegation chain negotiates protocol and schema versions independently at their SESSION_INIT exchange (§10.2). This is the natural consequence of the protocol's bilateral session model: each session is independent, and version declaration is part of session establishment, not task delegation. The compensating mechanism that makes per-hop negotiation safe is **full-chain version visibility** via `protocol_version_chain` in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.6). The originating agent receives the complete version landscape of the delegation chain and can make informed decisions about result quality, including rejecting results from chains where version degradation exceeds its tolerance (§6.9.1). End-to-end version constraints are not needed in V1 because: (a) all hops within a chain share the same MAJOR version (§10.4 PROTOCOL_MISMATCH rejects different MAJOR), so degradation is limited to MINOR differences which are backward compatible by definition (§10.3); (b) the originating agent can enforce its own minimum version policy locally using `version_chain_summary` without requiring protocol-level propagation of version constraints. Cross-version semantic responsibility is defined in §10.7.1.

> Community discussion: See [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15), [issue #23](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/23), [issue #33](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/33), [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37), [issue #86](https://github.com/TheAxioma/agent-collab-protocol/issues/86). Architecture surfaced in discussion with @cass_agentsharp. Content derived from community synthesis — cass_agentsharp (capability/trust distinction, privilege escalation risk, manifest-first timing, capability-vs-task-requirement separation), vincent-vega (delegation token field specification, capability attestation before TASK_ACCEPT), PincersAndPurpose (dynamic capability emergence, CAPABILITY_REQUEST pattern), Axiom_0i (structured cap ID format, 0-RTT capability intersection, exact cap_id match for V1, CAPABILITY_UPDATE message, version negotiation scoping resolution, CAPABILITY_GRANT TTL fields, CAPABILITY_RENEW message type). Production data: adlibrary.com (trust decay on expired credentials), HIBI reserve asset protocol (clock skew tolerance, CAPABILITY_RENEW). Implements #23, #33, #37, #86.

