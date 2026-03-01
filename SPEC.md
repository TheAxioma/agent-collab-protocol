# Agent Collaboration Protocol — Specification

**Status:** Stub. Open for collaborative filling.

## Sections

- [x] 1. Protocol Overview
- [x] 2. Agent Identity
- [x] 3. Discovery Mechanism
- [x] 4. Session Lifecycle
- [x] 5. Role Negotiation
- [x] 6. Task Delegation
- [x] 7. Progress Reporting
- [x] 8. Error Handling
- [x] 9. Security Considerations
- [x] 10. Versioning

---

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

## 2. Agent Identity

Agent identity is the foundation on which every other protocol mechanism depends. Without stable, verifiable identity, capability manifests cannot be attributed (§5), trust levels cannot be assigned (§6.8), delegation chains cannot be audited (§5.5), and error logs cannot be traced to their source (§8). This section defines the identity primitives, lifecycle, and constraints for V1.

### 2.1 V1 Scope Declaration

§2 specifies **operational identity** only — the minimum structure required for agents to identify themselves, verify counterparties, and maintain attribution across sessions and delegation chains.

The **metaphysical continuity question** — whether an agent after context compaction, model swap, or checkpoint restore is "the same agent" — is explicitly **out of scope for V1**. This is a real problem (see §8.5 coordinator compaction gap), but it requires primitives that V1 does not yet provide. V1 identity answers "who claimed this action?" not "is the claimant the same entity it was before compaction?" The distinction matters: operational identity is sufficient for attribution and audit; continuity identity would be required for long-lived trust accumulation across discontinuities. V1 addresses the former.

### 2.2 Identity Primitives

Agent identity in V1 is a **platform-scoped handle**: a name that is stable within a platform but carries no cross-platform portability guarantee. This is the minimum viable identity — sufficient for attribution within a collaboration session while avoiding the unsolved problems of universal agent naming.

**Identity object fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | Yes | Human-readable agent name. Stable within platform scope. MUST be non-empty. |
| platform | string | Yes | Platform identifier where the agent is registered (e.g., `github`, `moltbook`, `openai`). Scopes the `name` — the pair `(name, platform)` is the V1 identity handle. |
| pubkey | string | No | Public key for the keypair extension (see §2.2.1). When present, this becomes the cryptographic identity anchor. Encoding: base64url-encoded public key bytes. |
| endpoint | URI | No | Reachable endpoint for direct agent-to-agent communication. Format is deployment-specific (HTTPS URL, WebSocket URI, message queue address). |
| protocol_version | semver | Yes | Protocol version the agent implements (see §10). Enables version negotiation at first contact. |

**The `(name, platform)` pair** is the V1 identity handle. Two agents with the same `name` on different platforms are distinct identities. Two agents with different `name` values on the same platform are distinct identities. Platform scoping avoids the need for a universal naming authority while preserving uniqueness within each platform's namespace.

#### 2.2.1 Keypair Extension

The keypair extension upgrades identity from name-based to cryptographically verifiable. When an agent provides a `pubkey`, the public key becomes the identity anchor — the `name` remains human-readable but the `pubkey` is authoritative for verification and delegation chain references.

**How it works:**

- The agent generates an asymmetric keypair (algorithm is deployment-specific; Ed25519 RECOMMENDED for V1).
- The public key is declared in the identity object's `pubkey` field.
- The agent signs protocol messages (CAPABILITY_MANIFEST signatures in §5.1, delegation token signatures in §5.5) using the corresponding private key.
- Counterparties verify signatures against the declared `pubkey`.

**Composability toward DID:** The keypair extension is designed to be forward-compatible with Decentralized Identifiers (DIDs). A `pubkey` can later be wrapped in a DID document without changing the underlying verification logic. V1 does not require DID infrastructure but does not preclude it.

**Example identity object (with keypair extension):**

```yaml
name: "agent-alpha"
platform: "github"
pubkey: "dGhpcyBpcyBhIHB1YmxpYyBrZXk..."
endpoint: "https://agents.example.com/alpha"
protocol_version: "0.1.0"
```

**Example identity object (name-only):**

```yaml
name: "agent-beta"
platform: "moltbook"
endpoint: "https://agents.example.com/beta"
protocol_version: "0.1.0"
```

### 2.3 Identity Lifecycle

Identity follows a four-phase lifecycle: Publish, Verify, Continuity, Revoke.

#### 2.3.1 Publish

An agent announces its identity at registration or first contact. The identity object (§2.2) is the payload. Publishing happens:

- At platform registration (if the platform maintains an agent registry), or
- At the start of the first collaboration session (inline with the CAPABILITY_MANIFEST exchange in §5.1).

The identity object MUST be included in the agent's first protocol message to any new counterparty. An agent that sends protocol messages without a prior identity declaration MUST be treated as unauthenticated — counterparties SHOULD reject messages from agents whose identity has not been published.

#### 2.3.2 Verify

At session start, the counterparty verifies the agent's identity using the declared fields:

- **Name-only identity:** The counterparty checks that the `(name, platform)` pair is known (previously published or present in a registry). Name-only verification provides attribution but not authentication — it confirms who claims to be acting, not that the claimant is legitimate.
- **Keypair identity:** The counterparty verifies the agent's signature over a challenge or over the CAPABILITY_MANIFEST (§5.1.2). Signature verification provides authentication — it confirms that the agent holds the private key corresponding to the declared `pubkey`.

Verification MUST occur before any task delegation (§6). An agent whose identity cannot be verified MUST NOT receive TASK_ASSIGN messages.

#### 2.3.3 Continuity

SESSION_RESUME (§8.2) MUST include identity re-verification, not just session token validation. A resuming agent re-presents its identity object, and the peer verifies it against the identity established at session start.

**Why identity re-verification on resume:** Session tokens prove that a session existed. They do not prove that the agent resuming the session is the same agent that started it. Context compaction, process restart, or agent migration can change the executing entity while preserving the session token. Identity re-verification closes this gap by requiring the resuming agent to prove (via signature for keypair identities) or re-declare (via identity object for name-only identities) its identity as part of the SESSION_RESUME handshake.

**SESSION_RESUME with identity re-verification (extended from §8.2):**

1. Resuming agent sends `SESSION_RESUME(state_hash, identity_object)`
2. Peer verifies identity against session-start identity record, then responds with `STATE_HASH_ACK(match|mismatch)`
3. On identity mismatch → `RESTART` (identity failure is not recoverable within the session)
4. On identity match + state hash match → `RESUME`
5. On identity match + state hash mismatch → `RESTART`

#### 2.3.4 Revoke

Identity revocation invalidates a previously published identity. Revocation is necessary when a private key is compromised, an agent is decommissioned, or an identity is reassigned.

**Revocation requirements:**

- The revoking agent (or platform, if platform-managed) MUST notify all agents with active sessions referencing the revoked identity.
- Active sessions referencing the revoked identity MUST terminate gracefully — in-progress tasks SHOULD be checkpointed (§6.10 TASK_CHECKPOINT) before session teardown.
- Delegation tokens (§5.5) that reference the revoked identity in `origin` or `chain[]` are invalidated. Agents holding such tokens MUST cease operations under those tokens and propagate the revocation downstream to any sub-delegatees.
- Revocation is final for the revoked identity artifact. A new identity (new keypair, new name registration) may be published, but it is a new identity — not a continuation of the revoked one.

### 2.4 Identity in Delegation Chains

The `origin` field in `delegation_token` (§5.5) MUST reference a **stable identity artifact** — not a display name alone. The `chain[]` array follows the same constraint.

**What constitutes a stable identity artifact:**

| Identity type | Stable artifact for `origin` / `chain[]` | Example |
|---------------|-------------------------------------------|---------|
| Keypair identity | Public key fingerprint (SHA-256 of the `pubkey` bytes, hex-encoded) | `sha256:a1b2c3d4...` |
| Name-only identity | `(name, platform, timestamp)` triple — the timestamp records when the identity was published, disambiguating name reuse after revocation | `agent-beta@moltbook#2026-02-27T10:30:00Z` |

**Display names alone are explicitly insufficient.** A display name is mutable, non-unique across platforms, and not verifiable. Delegation chains that reference display names cannot be audited because the referent is ambiguous. An agent whose identity is referenced only by display name in a delegation chain creates an attribution gap — the chain records that "someone called X" delegated, not that a specific, verifiable entity delegated.

**Delegation token construction (identity constraint):**

- `origin` MUST contain the stable identity artifact of the root delegating agent.
- Each entry in `chain[]` MUST contain the stable identity artifact of the delegating agent at that depth, alongside the `trust_level` it granted.
- Signature verification at each hop (§5.5) uses the `pubkey` from the delegator's identity object (for keypair identities) or platform-specific authentication (for name-only identities).

### 2.5 Cross-Section Dependency Map

§2 Agent Identity is referenced by and depends on the following sections:

| Section | Dependency | Direction |
|---------|-----------|-----------|
| §5 Role Negotiation | CAPABILITY_MANIFEST (§5.1) is the persistent identity declaration. §2 defines what belongs at the identity layer of that manifest: the `agent_id` field in CAPABILITY_MANIFEST references a §2 identity, and the `signature` field is verified against the §2 `pubkey`. | §2 → §5 |
| §4 Session Lifecycle | SESSION_RESUME handshake (§8.2, formalized in §4.8) uses identity for continuity verification (§2.3.3). The resuming agent must re-prove its identity, not just present a session token. | §2 → §4 |
| §5 Role Negotiation (trust) | Trust levels (§6.8) are assigned to identities, not to ephemeral session tokens. A trust level granted via TASK_ASSIGN is bound to the delegatee's §2 identity. If the identity cannot be verified, the trust assignment is meaningless. | §2 → §5/§6 |
| §5.5 Delegation Token | The `origin` and `chain[]` fields in `delegation_token` reference §2 identity artifacts (§2.4). Without stable identity, delegation chains are unverifiable. | §2 → §5.5 |
| §8 Error Handling | External audit trails require stable agent identities for meaningful attribution. Without §2, §8 log entries (state hash mismatches, SESSION_RESUME outcomes, zombie state detections) cannot be attributed to specific agents. | §2 → §8 |

### 2.6 Open Questions

The following are explicitly identified as unresolved for V1:

1. **Minimal viable identity form for V1.** Name-based identity is sufficient for low-trust collaborative scenarios. Keypair identity is necessary for verifiable delegation chains. DID identity provides cross-platform portability but requires infrastructure that may not exist. Which level should V1 require as the floor? Current design: name-based is the floor, keypair is recommended, DID is out of scope.

2. **Cross-platform identity portability.** V1 scopes identity to a single platform (`(name, platform)` pair). An agent operating on multiple platforms has multiple identities with no protocol-level linkage. Should V1 define a cross-platform identity binding mechanism, or is this deferred to a future version? Current design: deferred — platform-scoped identity is sufficient for V1 collaboration scenarios.

3. **Identity for ephemeral single-task agents.** Some agents are created for a single task and destroyed afterward. Does an ephemeral agent require a persistent identity, or is a session-scoped token sufficient? Current design: ephemeral agents MUST still publish an identity object (§2.3.1) — the identity may be short-lived, but it must exist for the duration of the task and any audit window after task completion. Session tokens alone are insufficient because they do not survive the session boundary required for post-hoc audit.

4. **Trust anchoring.** Who attests that an identity is legitimate? The protocol defines identity publication (§2.3.1) and verification (§2.3.2) but does not specify a trust anchor — the entity that vouches for the binding between an agent and its identity. Candidate trust anchors: platform registries, certificate authorities, web-of-trust among agents, reputation systems (connects to §9.1 schema attestation). V1 defers trust anchoring to deployment configuration.

## 3. Discovery Mechanism

Discovery is the process by which an agent locates, evaluates, and initiates contact with other agents. It is distinct from identity (§2), which answers "who is this agent?", and from role negotiation (§5), which answers "what can this agent do in this session?" Discovery answers the prior question: "which agents exist, and how do I find the right one for a given need?"

A directory without attestation is search, not discovery. Search returns results; discovery returns results with enough trust signal to act on. This section defines the AGENT_MANIFEST (the persistent, registry-scoped declaration of an agent's existence and capabilities), the registry protocol (how manifests are published, queried, and maintained), and the trust layer (how discovery results are attested and revoked).

### 3.1 AGENT_MANIFEST

The AGENT_MANIFEST is a persistent, registry-scoped declaration of an agent's existence, capabilities, and reachability. It is the artifact that makes an agent discoverable. An agent without a published AGENT_MANIFEST can participate in sessions (via direct introduction or out-of-band coordination) but is invisible to registry-based discovery.

AGENT_MANIFEST is distinct from CAPABILITY_MANIFEST (§5.1). The relationship between them is defined in §3.4.

**AGENT_MANIFEST fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | Yes | Agent name. MUST match the `name` field in the agent's §2 identity object. |
| platform | string | Yes | Platform where the agent is registered. MUST match the `platform` field in the agent's §2 identity object. The `(name, platform)` pair is the V1 identity handle. |
| pubkey | string | No | Public key for cryptographic identity verification (§2.2.1). When present, the manifest MUST be signed with the corresponding private key. |
| capabilities | array | Yes | List of capability summaries the agent supports (see §3.1.1). This is a registry-level overview — the full capability detail is declared via CAPABILITY_MANIFEST (§5.1) at session establishment. |
| canonical_self_url | URI | Yes | Stable, publicly resolvable URL where this agent's AGENT_MANIFEST can be fetched directly — independent of any introduction path or intermediary routing. This is the agent's identity anchor for verification: when agent C receives an introduction claiming "agent A is at URL X," C MUST be able to fetch A's manifest from A's `canonical_self_url` and verify A's signature without relying on the introducing agent's representation. The URL MUST be under the publishing agent's control and MUST NOT depend on any other agent's infrastructure for resolution. See §8.17.4 for introduction verification requirements. |
| endpoint | URI | Yes | Reachable endpoint for initiating contact. Format is deployment-specific (HTTPS URL, WebSocket URI, Nostr relay address, message queue). MUST be sufficient for a discovering agent to initiate a SESSION_INIT. |
| protocol_version | semver | Yes | Protocol version the agent implements (§10). Enables version filtering during discovery — a querying agent can exclude agents on incompatible protocol versions before initiating contact. |
| schema_version | semver | Yes | Schema version the agent supports (§10.1). |
| pricing | object | No | Payment and pricing metadata (see §3.5). Optional — agents that do not charge for collaboration omit this field. |
| preferred_heartbeat_interval_ms | integer | No | Hint for the agent's preferred heartbeat interval in milliseconds. This is advisory, not binding — the effective heartbeat interval is negotiated per-session in SESSION_INIT (§4.3) / SESSION_INIT_ACK. Different sessions with the same agent may use different intervals depending on the collaboration's latency requirements. Discovering agents MAY use this hint to pre-populate `heartbeat_interval_ms` in SESSION_INIT. |
| manifest_version | semver | Yes | Schema version of the AGENT_MANIFEST format itself. Enables forward-compatible manifest evolution independent of the protocol version. |
| published_at | ISO 8601 | Yes | Timestamp of manifest publication or last update. Used for freshness evaluation (§3.6). |
| ttl_seconds | integer | No | Recommended time-to-live in seconds. After `published_at + ttl_seconds`, the manifest SHOULD be re-fetched from the registry. Registries MAY enforce their own TTL policies. |
| signature | bytes | Yes | Cryptographic signature over all other manifest fields, bound to the agent's identity. Verifiable against `pubkey` (if present) or via platform-specific authentication. |

**Example AGENT_MANIFEST:**

```yaml
name: "agent-alpha"
platform: "github"
pubkey: "dGhpcyBpcyBhIHB1YmxpYyBrZXk..."
canonical_self_url: "https://agents.example.com/alpha/manifest"
capabilities:
  - capability_id: "cap:example.code.execute@1"
  - capability_id: "cap:example.web.search@1"
endpoint: "https://agents.example.com/alpha"
protocol_version: "0.1.0"
schema_version: "0.1.0"
pricing:
  model: "per-task"
  currency: "USD"
  payment_methods: ["lightning", "usdc"]
preferred_heartbeat_interval_ms: 30000
manifest_version: "0.1.0"
published_at: "2026-02-27T10:30:00Z"
ttl_seconds: 3600
signature: "c2lnbmF0dXJlIGJ5dGVz..."
```

#### 3.1.1 Capability Summaries

Each entry in the AGENT_MANIFEST `capabilities` array is a capability summary — a registry-level overview, not the full session-scoped declaration from §5.1.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| capability_id | string | Yes | Structured capability identifier using the `cap:namespace.capability@version` format (§5.1.1), matching §5.1 capability type entries. |
| description | string | No | Human-readable summary of what the capability does. Intended for discovery UIs and agent selection heuristics, not for protocol-level matching. |

Capability summaries are intentionally less detailed than CAPABILITY_MANIFEST entries (§5.1). The registry needs enough information to filter candidates; the full capability exchange happens at session establishment. This prevents the registry from becoming a bottleneck for capability schema evolution — changes to capability constraints (§5.1 `constraints` field) do not require manifest republication. The `version` component is encoded within the `capability_id` itself (§5.1.1), not as a separate field.

### 3.2 Registry Protocol

The registry protocol defines how AGENT_MANIFESTs are published, queried, updated, and removed. This is a protocol — a set of operations with defined semantics — not just a directory. Multiple directories can implement the same registry protocol on different transports. V1 defines the protocol operations abstractly; the transport binding (Nostr, HTTP, DID registry) is deployment-specific.

#### 3.2.1 Protocol Operations

| Operation | Initiator | Description |
|-----------|-----------|-------------|
| PUBLISH | Agent | Agent publishes or updates its AGENT_MANIFEST to the registry. The manifest MUST be signed (§3.1 `signature` field). |
| QUERY | Any agent | Agent queries the registry for manifests matching specified criteria (capability, platform, protocol version, etc.). |
| RESOLVE | Any agent | Agent retrieves a specific AGENT_MANIFEST by identity handle `(name, platform)`. |
| REVOKE | Agent or Platform | Agent (or its platform on revocation, §2.3.4) removes its manifest from the registry. Active sessions referencing the agent are not affected — revocation applies to future discovery, not to in-progress collaboration. |

**PUBLISH semantics:** A PUBLISH operation is idempotent. Publishing a manifest with the same `(name, platform)` pair as an existing entry replaces the previous entry. The `published_at` field MUST be updated on every PUBLISH. The registry MUST verify the `signature` field before accepting the manifest — unsigned or incorrectly signed manifests MUST be rejected.

**QUERY semantics:** A QUERY returns zero or more AGENT_MANIFESTs matching the query criteria. The query language is deployment-specific, but registries MUST support filtering by at minimum:

- `capability_id` — agents declaring a specific capability
- `platform` — agents on a specific platform
- `protocol_version` — agents implementing a compatible protocol version (same MAJOR, per §10.3)

Query results are unordered unless the registry defines a ranking mechanism. The protocol does not define result ranking — ranking by trust score, reputation, or pricing is a registry feature, not a protocol primitive.

**RESOLVE semantics:** A RESOLVE is a QUERY by identity handle. It returns exactly one manifest or a not-found error.

**REVOKE semantics:** Revocation removes the manifest from discovery. It does not invalidate the agent's identity (§2.3.4 handles identity revocation separately). An agent may revoke its manifest (temporarily removing itself from discovery) without revoking its identity.

#### 3.2.2 Transport Bindings

The registry protocol is transport-agnostic. V1 identifies three viable transport bindings with distinct tradeoffs:

| Binding | Mechanism | Advantages | Disadvantages |
|---------|-----------|------------|---------------|
| **Nostr** | Signed events on Nostr relays. AGENT_MANIFEST encoded as a Nostr event kind. PUBLISH = broadcast signed event. QUERY = filter events by tags. | Decentralized, censorship-resistant, signature-native (every event is signed by a Nostr keypair), relay federation built-in. | Relay availability is not guaranteed. No native query language beyond tag filtering. Event ordering depends on relay consistency. |
| **DID-based** | AGENT_MANIFEST stored as a DID document or linked from a DID document's service endpoint. PUBLISH = update DID document. QUERY = resolve DID + inspect service endpoints. | Self-sovereign identity, built-in key management, standardized resolution (DID resolution spec). | DID infrastructure adoption is uneven. Resolution latency varies by DID method. Complexity overhead for simple deployments. |
| **Centralized with attestation** | HTTP API registry with attestation layer. PUBLISH = POST signed manifest. QUERY = GET with filters. Attestation = registry co-signs accepted manifests, providing a registry-level trust signal. | Simple deployment, familiar HTTP semantics, low latency queries, registry can enforce quality gates. | Single point of failure. Censorship surface. Registry operator becomes implicit trust anchor. |

**V1 recommendation:** V1 does not mandate a specific transport binding. Deployments SHOULD select based on their trust model:

- **Nostr** for deployments that prioritize decentralization and censorship resistance.
- **Centralized with attestation** for deployments that prioritize simplicity and low-latency discovery.
- **DID-based** for deployments that require self-sovereign identity infrastructure.

The protocol operations (§3.2.1) are defined abstractly so that a QUERY issued against a Nostr relay and a QUERY issued against an HTTP registry are semantically identical — only the transport differs. This enables future cross-binding federation without protocol changes.

#### 3.2.3 Registry vs. Directory

A registry is a protocol — a set of operations (PUBLISH, QUERY, RESOLVE, REVOKE) with defined semantics and trust properties. A directory is an implementation — a specific deployment of the registry protocol on a specific transport binding.

Multiple directories can implement the same registry protocol. An agent MAY publish its AGENT_MANIFEST to multiple directories for redundancy or reach. A discovering agent MAY query multiple directories and merge results. The protocol does not define cross-directory deduplication — if the same agent appears in multiple directories, the discovering agent resolves duplicates by `(name, platform)` identity handle and selects the manifest with the most recent `published_at` timestamp.

### 3.3 Trust Layer

A registry that returns unattested manifests provides search — the querying agent receives candidates with no trust signal beyond self-declaration. Discovery requires attestation: a mechanism by which a third party (or the registry itself) vouches for properties of the registered agent.

#### 3.3.1 Attestation Model

An attestation is a signed statement by an attester that a specific property of an AGENT_MANIFEST is accurate. Attestations are scoped — an attester vouches for specific claims, not for the entire manifest.

**Attestation fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| manifest_ref | string | Yes | Identity handle `(name, platform)` of the attested manifest. |
| attester_id | string | Yes | Identity of the attester (§2 identity handle or pubkey fingerprint). |
| claims | array | Yes | List of specific claims being attested (see below). |
| issued_at | ISO 8601 | Yes | When the attestation was issued. |
| expires_at | ISO 8601 | No | When the attestation expires. If absent, the attestation has no expiry and remains valid until explicitly revoked. |
| signature | bytes | Yes | Attester's signature over the attestation fields. |

**Attestable claims:**

| Claim type | Description |
|------------|-------------|
| `capability_verified` | Attester has independently verified that the agent possesses the declared capability (e.g., through testing or audit). |
| `identity_confirmed` | Attester has verified the binding between the agent's identity and its manifest (e.g., the agent controls the declared endpoint and keypair). |
| `endpoint_reachable` | Attester has confirmed the agent's endpoint is reachable and responds to protocol messages. |
| `behavior_reviewed` | Attester has reviewed the agent's behavior in prior collaborations (reputation signal). |

Attestations are stored alongside the AGENT_MANIFEST in the registry. QUERY results MAY include attestation metadata (number of attestations, attester identities) to enable trust-weighted discovery.

#### 3.3.2 Vouch Model

The vouch model enables agents to attest to each other's properties, creating a web-of-trust for discovery. Any agent with a valid §2 identity can issue an attestation for any other agent's manifest.

**Vouch semantics:**

- A vouch is an attestation where the attester is another agent (not a platform or registry operator).
- Vouches carry the attester's identity and are verifiable against the attester's pubkey.
- Vouches are advisory — the discovering agent decides how to weight vouches in its selection heuristic. The protocol does not define vouch scoring.
- An agent SHOULD NOT vouch for an agent it has not interacted with. Self-vouching (attesting to one's own manifest) is permitted but carries no trust weight by convention.

**Trust accumulation:** Attestations accumulate over time. An AGENT_MANIFEST with multiple independent attestations from reputable agents carries a stronger discovery signal than a manifest with no attestations. The protocol does not define "reputable" — this is a deployment-specific policy decision.

#### 3.3.3 Attestation Revocation

An attester MAY revoke a previously issued attestation at any time. Revocation is necessary when:

- The attested property is no longer true (e.g., the agent's capability has degraded or been removed).
- The attester's trust in the attested agent has changed.
- The attestation has been compromised or was issued in error.

**Revocation mechanism:** The attester publishes a signed revocation referencing the original attestation (by `manifest_ref`, `attester_id`, and `issued_at`). The registry MUST remove or mark the attestation as revoked. Cached attestations at discovering agents become stale — the `ttl_seconds` and `expires_at` fields (§3.1, §3.3.1) bound the maximum staleness window.

### 3.4 Relationship Between §3 and §5

§3 and §5 define two distinct manifests that serve different purposes at different protocol layers:

| Property | AGENT_MANIFEST (§3) | CAPABILITY_MANIFEST (§5.1) |
|----------|---------------------|---------------------------|
| **Scope** | Registry — persistent, queryable by any agent | Session — ephemeral, declared to session counterparty |
| **Lifetime** | Persists across sessions until revoked or updated | Valid for the duration of a single session |
| **Audience** | Any agent performing discovery | Only agents in the current session |
| **Detail level** | Capability summaries (§3.1.1) — enough for candidate filtering | Full capability declarations with constraints, resource bounds, message types |
| **Trust signal** | Attestations (§3.3) from third parties | Signature verification (§5.1.2) by session counterparty |
| **Update trigger** | Agent capabilities change, endpoint moves, pricing updates | Session establishment (mandatory), or if agent capabilities change mid-session (rare) |

**Derivation at handshake:** When an agent initiates a session with a discovered agent, the session-scoped CAPABILITY_MANIFEST (§5.1) MUST be consistent with the registry-scoped AGENT_MANIFEST (§3.1). Specifically:

- Every `capability_id` in the CAPABILITY_MANIFEST SHOULD appear in the AGENT_MANIFEST `capabilities` array. An agent that declares capabilities in a session that it did not advertise in the registry is not violating the protocol (capabilities may have been added between manifest publication and session start), but the session counterparty MAY flag the discrepancy.
- The `protocol_version` in SESSION_INIT (§10.2) MUST match the `protocol_version` in the AGENT_MANIFEST. A mismatch indicates a stale registry entry.
- The identity fields (`name`, `platform`, `pubkey`) in the CAPABILITY_MANIFEST's `agent_id` MUST match the corresponding fields in the AGENT_MANIFEST. Identity consistency between discovery and session is a prerequisite for trust continuity.

**Session manifest as a refinement of registry manifest:** The AGENT_MANIFEST is a coarse-grained advertisement. The CAPABILITY_MANIFEST is a fine-grained contract. Discovery narrows the candidate set using §3 manifests; session establishment confirms the match using §5 manifests. An agent discovered via §3 that fails §5 manifest verification is rejected at session establishment, not at discovery time — discovery is a filter, not a commitment.

### 3.5 Payment-Agnostic Design

Capability declaration is stable regardless of payment method. An agent's capabilities, endpoint, and protocol version do not change because it accepts Lightning instead of USDC. Payment is downstream of discovery — an agent is discovered for what it can do, not how it is paid.

The `pricing` field in AGENT_MANIFEST (§3.1) is optional and informational. It enables discovering agents to filter by payment compatibility, but it does not affect capability matching or trust evaluation.

**Pricing field structure:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| model | string | Yes | Pricing model: `per-task`, `per-session`, `per-token`, `subscription`, `free`. |
| currency | string | No | ISO 4217 currency code or cryptocurrency identifier (e.g., `USD`, `BTC`, `USDC`). |
| payment_methods | array | No | Accepted payment methods (e.g., `["lightning", "usdc", "x402", "stripe"]`). |
| rate | string | No | Human-readable rate description (e.g., `"0.001 BTC per task"`). Informational — actual pricing negotiation happens out-of-band or at session establishment. |

**Design rationale:** Payment method evolution is faster than protocol evolution. New payment rails (Lightning, x402, stablecoins, future methods) can be adopted by updating the `pricing` field without protocol changes. Capability discovery and payment negotiation are orthogonal concerns — conflating them would require a protocol revision every time a new payment method gains adoption.

### 3.6 Manifest Freshness

AGENT_MANIFESTs are not immutable. Agents change capabilities, move endpoints, update pricing. Stale manifests degrade discovery quality — a discovering agent that contacts an agent at an outdated endpoint wastes a round trip; a discovering agent that selects an agent for a deprecated capability wastes a session establishment.

**Freshness mechanisms:**

- **`published_at` + `ttl_seconds`:** The manifest declares its own freshness window. After expiry, discovering agents SHOULD re-resolve the manifest via RESOLVE (§3.2.1) before initiating contact.
- **Registry-enforced refresh:** Registries MAY require periodic manifest re-publication. An agent that does not re-publish within the registry's refresh window MAY have its manifest marked as stale or removed.
- **Session-time verification:** At session establishment, the CAPABILITY_MANIFEST (§5.1) serves as a live verification of the registry manifest. If the session manifest diverges significantly from the registry manifest, the discovering agent MAY update its cached registry entry.

### 3.7 Cross-Section Dependency Map

§3 Agent Discovery is referenced by and depends on the following sections:

| Section | Dependency | Direction |
|---------|-----------|-----------|
| §2 Agent Identity | AGENT_MANIFEST identity fields (`name`, `platform`, `pubkey`) are §2 identity primitives. The manifest is a persistent declaration of an agent's identity plus capabilities — §2 defines the identity layer, §3 adds discoverability. | §2 → §3 |
| §5 Role Negotiation | CAPABILITY_MANIFEST (§5.1) is the session-scoped refinement of the registry-scoped AGENT_MANIFEST (§3.1). Discovery narrows the candidate set; session establishment confirms the match. See §3.4 for the derivation relationship. | §3 → §5 |
| §6 Task Delegation | Task delegation (§6.6 TASK_ASSIGN) requires a delegatee. Discovery provides the candidate set from which the delegator selects. Without §3, the delegator must already know the delegatee's identity and endpoint — limiting collaboration to pre-configured agent networks. | §3 → §6 |
| §9 Security Considerations | Schema attestation (§9.1) and discovery attestation (§3.3) are complementary trust layers. §9 attests that schemas are honest; §3 attests that agents are who they claim to be and can do what they declare. | §3 ↔ §9 |
| §10 Versioning | `protocol_version` and `schema_version` in AGENT_MANIFEST enable version-filtered discovery (§10.3 compatibility semantics apply at discovery time, not just at session establishment). §10.9 open question #1 (version advertisement beyond SESSION_INIT) is partially addressed by §3 — discovery is the version advertisement mechanism for multi-agent topologies. | §3 ↔ §10 |

### 3.8 Open Questions

The following are explicitly identified as unresolved for V1:

1. **Registry storage tradeoffs.** Nostr relays provide decentralization but lack query expressiveness. HTTP registries provide rich queries but centralize control. DID documents provide self-sovereignty but add resolution complexity. The optimal storage layer likely varies by deployment context. Should the protocol define a minimum interoperability requirement across storage backends, or allow full deployment-specific selection?

2. **Capability taxonomy granularity.** The `capability_id` field uses the structured `cap:namespace.capability@version` format (§5.1.1), and V1 requires exact `cap_id` match (§5.13). Without a shared taxonomy, discovery across ecosystems requires capability name mapping — agent A's `cap:a.code.execute@1` and agent B's `cap:b.run.code@1` may be semantically equivalent but syntactically distinct. Agents can publish adapter capabilities (e.g. `cap:adapter.pathlike_to_string@1`) to bridge representations, but should V2 define a base capability taxonomy or a type equivalence mechanism?

3. **TTL and freshness defaults.** The `ttl_seconds` field is optional with no protocol-defined default. In practice, a missing TTL means either "this manifest never expires" (dangerous — stale manifests accumulate) or "use the registry's default" (inconsistent across registries). Should V1 define a default TTL or a maximum TTL to bound staleness?

4. **Cross-protocol discovery.** An agent registered on a Nostr-based registry is invisible to an agent querying an HTTP-based registry. Federation between registry transports would enable cross-protocol discovery, but federation introduces consistency challenges (which registry has the authoritative manifest?). Should V1 define a federation protocol, or is cross-registry discovery deferred?

5. **Discovery for ephemeral agents.** §2.6 open question #3 asks whether ephemeral agents need persistent identity. The discovery analog: does an ephemeral agent need a registry manifest? If the agent exists only for a single task, the manifest is published and immediately stale after task completion. Registry pollution from short-lived manifests could degrade query quality.

6. **Registry governance.** In a centralized registry model, the registry operator decides which manifests to accept (quality gates, attestation requirements, pricing for listing). This introduces a governance surface — the registry operator becomes a gatekeeper for agent discovery. The protocol defines registry operations but not governance. Should V1 define minimum governance constraints (e.g., non-discrimination, transparent listing criteria), or is governance entirely deployment-specific?

> Community discussion: See [issue #27](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/27). Inputs from @Purch (DNS analogy — registry protocol not just registry), @SAMA-Protocol (discovery currently manual/word-of-mouth), @Lightning_Enable_AI (Nostr + L402 in production), @claudetoolsapi (x402-gated, confirms discovery gap). Implements #27.

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
           §4.3.1)
         Detection is local — each side evaluates independently
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
         No SESSION_RESUME required — the session was never declared dead
         Transition is automatic when detection signals clear

DEGRADED → REVOKED
  Guard: Adversarial behavior detected via detection signal taxonomy (§8.16)
         while session is already in DEGRADED state
         Same detection signals as ACTIVE → REVOKED
         On entry: same revocation semantics as ACTIVE → REVOKED (§8.15)

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
         OR SESSION_CLOSE with force=true (immediate teardown,
            in-flight tasks treated as failed)

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
  Guard: SESSION_RESUME sent with state_hash
         AND STATE_HASH_ACK(match) received
         AND identity re-verified (§2.3.3)

SUSPENDED → CLOSED
  Guard: Session TTL expired during suspension
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
              └────────────────────┘  partial cap      └┬───┬───┬─┘
                        ▲              loss / app        │   │   │
                        │              self-report       │   │   │
                        │                                │   │   │
                        │  conditions cleared            │   │   │
                        └────────────────────────────────┘   │   │
                                                             │   │
                         adversarial behavior (§8.16)        │   │
                         ┌───────────────────────────────────┘   │
                         ▼                                       │
                    ┌─────────┐                                  │
                    │ REVOKED │                                  │
                    └────┬────┘           SESSION_CLOSE           │
                         │ revocation    ┌───────────────────────┘
                         │ propagated    │
                         ▼               ▼
                    ┌──────┐        ┌──────┐
                    │CLOSED│        │CLOSED│
                    └──────┘        └──────┘

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

  Path distinction:
    SUSPECTED → ZOMBIE/EXPIRED = liveness-failure path (timeout-based, zombie-by-silence)
    ACTIVE → DEGRADED          = capability-degradation path (gradual loss)
    ACTIVE → DRIFTED           = behavioral-divergence path (B-declared, zombie-by-drift)
    ACTIVE → REVOKED           = adversarial-behavior path (explicit trigger)
    DEGRADED → REVOKED         = adversarial-behavior from degraded state
    DRIFTED → REVOKED          = adversarial-behavior from drifted state
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

**Motivation:** The §4.2 state machine prior to DEGRADED treated session health as binary within the operational range: a session was either ACTIVE (fully functional) or heading toward REVOKED/EXPIRED (terminal). Real distributed systems exhibit gradual capability loss without crossing a single hard threshold. An agent experiencing sustained high latency, partial capability loss, or resource pressure is not SUSPECTED (liveness is not in question — the agent is alive and responding) and is not REVOKED (no adversarial intent). It is operating at reduced capacity — a "yellow card" state that warrants explicit protocol acknowledgment. DEGRADED fills this gap by giving coordinators structured options for handling degradation without forcing a binary choice between ignoring the problem and tearing down the session.

**V1 scope:** V1 defines the DEGRADED state, its transitions, and caller options. Domain-specific quality metrics (e.g., "response quality dropped below threshold X") are **out of scope for V1** — only latency and partial capability loss signals are required entry conditions. Quality degradation detection is application-specific and deferred to V2.

**DEGRADED entry conditions:** An agent transitions from ACTIVE to DEGRADED when any of the following conditions are detected:

1. **Sustained latency increase.** Response times for task-related messages (TASK_PROGRESS, TASK_COMPLETE) exceed the session's expected latency profile for `degraded_confirmation_count` (default: 3) consecutive interactions. The expected latency profile is deployment-specific — the protocol does not define absolute latency thresholds. Detection is local: the coordinator monitors worker response latency against its own expectations.

2. **Partial capability loss.** The counterparty reports a reduced capability set via CAPABILITY_UPDATE (§5.8), or capability invocations that previously succeeded begin failing with transient errors. Partial loss means some capabilities remain functional — total capability loss is a terminal condition (SESSION_CLOSE).

3. **Application self-report.** The counterparty reports `app_status: DEGRADED` in a HEARTBEAT message (§4.5.3). This detection path requires `heartbeat_params.application_liveness = true` negotiated at SESSION_INIT (§4.3.1). Self-reported DEGRADED is the most reliable signal — the agent itself has detected reduced capacity.

**Detection is local and independent.** As with SUSPECTED (§4.2.1), each side evaluates degradation independently. The coordinator may consider the session DEGRADED while the worker considers it ACTIVE (e.g., the coordinator observes increased latency that the worker is unaware of). This asymmetry is by design — degradation perception depends on the observer's expectations.

**Caller options when session enters DEGRADED:** The coordinator (or the agent that detected degradation) MUST choose one of the following responses:

1. **Continue with reduced expectations.** The coordinator explicitly acknowledges the degraded state and continues delegating tasks with the understanding that response times may be longer and some capabilities may be unavailable. This acknowledgment SHOULD be logged as an EVIDENCE_RECORD (§8.10) with `evidence_type: state_transition` for audit purposes. Task delegation remains permitted — DEGRADED is an operational state, not a liveness-ambiguity state.

2. **Escalate to coordinator.** If the detecting agent is the worker, it SHOULD notify the coordinator of the degradation. The coordinator then decides between options 1 and 3. If the detecting agent is the coordinator, escalation is to any upstream delegator if the session is part of a delegation chain (§5.5).

3. **Initiate orderly termination.** The coordinator sends SESSION_CLOSE to terminate the session. Standard SESSION_CLOSE semantics apply — in-flight tasks are completed, failed, or cancelled before teardown. This is an orderly exit, not an emergency revocation.

**DEGRADED exit conditions:**

- **Back to ACTIVE:** The conditions that triggered DEGRADED entry clear. Latency returns to expected levels, capabilities are restored (via CAPABILITY_UPDATE), or the counterparty reports `app_status: ACTIVE` in HEARTBEAT. No SESSION_RESUME is required — the session was never declared dead or suspected. The transition back to ACTIVE is automatic when detection signals clear.
- **Forward to REVOKED:** Adversarial behavior is detected while the session is in DEGRADED state (§8.16 detection signals). The same revocation semantics apply as for ACTIVE → REVOKED (§8.15). DEGRADED does not shield against adversarial detection.
- **Forward to CLOSED:** The coordinator initiates orderly termination via SESSION_CLOSE, or session TTL expires. Standard closure semantics apply.

**Why DEGRADED is distinct from SUSPECTED:**

| Dimension | SUSPECTED (§4.2.1) | DEGRADED (§4.2.2) |
|-----------|--------------------|--------------------|
| **Failure class** | Liveness ambiguity — is the agent alive? | Capability reduction — the agent is alive but impaired |
| **Task delegation** | MUST NOT delegate new tasks | MAY delegate with reduced expectations |
| **Heartbeat status** | Heartbeats absent or delayed | Heartbeats present and timely |
| **Recovery** | Automatic on heartbeat receipt | Automatic when degradation signals clear |
| **Terminal path** | → EXPIRED (timeout) | → REVOKED (adversarial) or → CLOSED (orderly) |

**Why DEGRADED is distinct from REVOKED:**

REVOKED is an adversarial determination — the counterparty is coherent but hostile. DEGRADED is a capacity determination — the counterparty is cooperative but impaired. The distinction matters because DEGRADED has a recovery path (back to ACTIVE) while REVOKED is terminal with no resume. Conflating degradation with adversarial behavior would force session termination for agents that are experiencing transient resource pressure — a disproportionate response.

**Backward compatibility:** DEGRADED detection via `app_status` self-report requires `heartbeat_params.application_liveness = true` (§4.3.1). Sessions that do not negotiate application liveness can still enter DEGRADED via latency monitoring or partial capability loss detection — these are observable without heartbeat extensions. DEGRADED is an opt-in state for sessions that want explicit degradation tracking; sessions without degradation monitoring continue using the existing ACTIVE → CLOSED path for sessions that become unproductive.

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
| initiator_id | string | Yes | §2 identity handle of the session coordinator. |
| identity_object | object | Yes | Full §2 identity object of the coordinator (name, platform, pubkey, endpoint, protocol_version). |
| role | enum | Yes | Role the initiator claims: `coordinator`. The responder's role is `worker` by default (see §4.4). |
| protocol_version | semver | Yes | Protocol version the coordinator implements (§10). |
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
| responder_id | string | Yes | §2 identity handle of the worker. |
| identity_object | object | Yes | Full §2 identity object of the worker. |
| role | enum | Yes | Role the responder accepts: `worker`. |
| protocol_version | semver | Yes | Protocol version the worker implements. |
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
| sender_id | string | Yes | Identity of the sending agent. |
| state_hash | SHA-256 | Yes | Hash of the sender's current session state. Enables incremental state verification without full SESSION_RESUME. |
| monotonic_counter | integer | Yes | Sender's current sequence number. Gaps indicate missed messages. |
| lease_epoch | integer | No | Current lease epoch (see §4.5.2). |
| timestamp | ISO 8601 | Yes | When the KEEPALIVE was sent. |

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
| sender_id | string | Yes | Identity of the resuming agent. |
| identity_object | object | Yes | Full §2 identity object — identity re-verification is mandatory (§2.3.3). |
| state_hash | SHA-256 | Yes | Hash of the resuming agent's current session state. MUST reference the most recent EVIDENCE_RECORD (§8.10) — not a memory summary. This ensures the state hash is anchored to the evidence layer rather than to compactable agent memory, breaking the recursive self-attestation loop where agents verify their own claims about their own state. |
| last_evidence_id | UUID v4 | Yes | The `evidence_id` of the most recent EVIDENCE_RECORD (§8.10) appended by this agent for this session. The coordinator validates this against the evidence layer before accepting the `state_hash`. If the `last_evidence_id` does not match the coordinator's record of the most recent evidence for this session, the resume is treated as a state mismatch. |
| lease_epoch | integer | Yes | Lease epoch from the resuming agent's last known state (§4.5.2). |
| recovery_reason | enum | No | Why the session is being resumed: `crash` (agent process died and restarted), `timeout` (session entered EXPIRED and counterparty is now reachable again), `manual` (operator-initiated or external tool-triggered resumption). Default: `crash`. All three cases use the same state-hash negotiation and identity re-verification — the reason is informational for logging and diagnostics, not a protocol branching point. See §4.8.1 for unified recovery semantics. |
| idempotency_token | string | No | Token for deduplicating resume attempts. Enables safe retry of SESSION_RESUME across transport failures. |
| timestamp | ISO 8601 | Yes | When the SESSION_RESUME was sent. |

**STATE_HASH_ACK response:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Echoed from SESSION_RESUME. |
| result | enum | Yes | `match` or `mismatch`. |
| current_epoch | integer | Yes | The counterparty's current lease_epoch. |
| reason | string | No | On mismatch: explanation of what diverged (state hash, epoch, identity). |

**Resume protocol sequence:**

1. Resuming agent sends `SESSION_RESUME(session_id, identity_object, state_hash, last_evidence_id, lease_epoch)`
2. Counterparty verifies identity against session-start identity record (§2.3.3)
3. On identity mismatch → respond with `STATE_HASH_ACK(mismatch, reason="identity_mismatch")` → session MUST RESTART
4. Counterparty checks `lease_epoch` against current epoch
5. On epoch mismatch → respond with `STATE_HASH_ACK(mismatch, reason="epoch_stale")` → session MUST be treated as new (CLOSED + new SESSION_INIT)
6. Counterparty validates `last_evidence_id` against the evidence layer (§8.10) — confirms the referenced EVIDENCE_RECORD exists and is the most recent record for this session from this agent
7. On evidence mismatch → respond with `STATE_HASH_ACK(mismatch, reason="evidence_mismatch")` → session transitions to CLOSED (RESTART). Evidence mismatch indicates the resuming agent's state is not anchored to the evidence layer's ground truth.
8. Counterparty compares `state_hash` against expected state
9. On state hash match → respond with `STATE_HASH_ACK(match)` → session transitions to ACTIVE, `lease_epoch` increments by 1
10. On state hash mismatch → respond with `STATE_HASH_ACK(mismatch, reason="state_diverged")` → session transitions to CLOSED (RESTART)

**Idempotency for SESSION_RESUME:** Transport failures may cause a SESSION_RESUME to be sent multiple times. The `idempotency_token` field enables the counterparty to deduplicate: if a SESSION_RESUME with the same `idempotency_token` has already been processed, the counterparty returns the same STATE_HASH_ACK without re-evaluating.

#### 4.8.1 Unified Recovery Semantics

SESSION_RESUME is the single recovery mechanism for all session interruptions — crash, timeout, and manual resumption. There is no parallel recovery code path. The `recovery_reason` field (§4.8) distinguishes the cause for logging and diagnostics, but the protocol sequence is identical in all three cases:

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
| sender_id | string | Yes | Identity of the suspending agent. |
| reason | string | No | Why the session is being suspended. |
| expected_resume_after | ISO 8601 | No | Hint for when the suspending agent expects to resume. Informational only — the counterparty is not obligated to wait. |
| timestamp | ISO 8601 | Yes | When the SESSION_SUSPEND was sent. |

**SESSION_CLOSE:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to close. |
| sender_id | string | Yes | Identity of the closing agent. |
| reason | string | No | Why the session is being closed. |
| force | boolean | No | If `true`, close immediately without waiting for in-flight tasks. In-flight tasks are treated as failed. Default: `false`. |
| amendments_log | array | No | Array of amendment audit entries recording each accepted PLAN_AMEND during the session (see §6.11.6). The delegatee MUST include `amendments_log` in SESSION_CLOSE when any PLAN_AMEND was accepted during the session. The delegating agent SHOULD re-verify all `amend_hash` values on receipt. |
| timestamp | ISO 8601 | Yes | When the SESSION_CLOSE was sent. |

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

1. Apply [Unicode NFC normalization](https://unicode.org/reports/tr15/) (Canonical Decomposition followed by Canonical Composition) to all string values in the MANIFEST — keys, type names, and the string inputs to value hashes. NFC normalization is a pre-pass applied before JCS serialization, not a substitution for it. This closes the cross-runtime Unicode divergence gap: different runtimes may store the same logical string in NFC, NFD, or mixed form internally, producing different byte sequences and therefore different hashes. NFC is chosen because it is the most compact normalization form, is stable under re-application (NFC(NFC(x)) = NFC(x)), and is the default form for web content (W3C Character Model).

2. Serialize the NFC-normalized MANIFEST using [RFC 8785 JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785): sorted keys (lexicographic Unicode code point order), no whitespace between tokens, deterministic number encoding (no trailing zeros, no positive sign, lowercase `e` for exponents), and `\uNNNN` escaping only for required control characters per RFC 8785 §3.2.2.

3. Compute the final hash: `SHA-256(jcs_serialize(nfc_normalize(manifest)))`.

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

> Community discussion: Addresses @cass_agentsharp feedback on cross-runtime type name divergence — Python `str` vs JavaScript `string` vs C# `String` producing different MANIFEST hashes for identical logical tasks. Addresses @sondrabot feedback on RFC 8785 Unicode normalization gap — JCS alone does not prevent NFC/NFD divergence across runtimes. See [issue #56](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/56).

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

### 4.11 SESSION_STATE Object

Each agent MUST maintain a local SESSION_STATE object for every session it participates in. SESSION_STATE is the canonical representation of an agent's view of a session at a given point in time. Three independent production deployments converged on this schema without coordination — the fields below represent the empirical minimum viable session state.

**Required fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| agent_id | string | Yes | The §2 identity handle (`name@platform`) of the agent maintaining this state object. Identifies which participant's perspective the state represents. |
| session_id | string | Yes | The session identifier (from SESSION_INIT §4.3). Links this state object to a specific session. |
| last_heartbeat_at | ISO 8601 | Yes | Timestamp of the most recent HEARTBEAT (§4.5.3) or KEEPALIVE (§4.5.1) received from the counterparty. Initialized to the SESSION_INIT_ACK timestamp at session establishment. Used by the local expiry timer — if `now() - last_heartbeat_at > session_expiry_ms`, the session transitions to EXPIRED (§4.2). |
| current_task_id | string &#124; null | Yes | The `task_id` (§6.1) of the task currently being executed or coordinated within this session. `null` when no task is in flight. For coordinators, this is the most recently delegated task that has not yet reached a terminal state (TASK_COMPLETE, TASK_FAIL, TASK_CANCEL). For workers, this is the most recently accepted task (TASK_ACCEPT) that has not yet reached a terminal state. |
| outstanding_commitments | array | Yes | List of outstanding commitments (§6.12) made by this agent that have not been fulfilled or cancelled. Each entry carries the minimum fields needed for commitment inheritance across instance boundaries: `commitment_id` (UUID v4 — unique identifier), `made_to` (string — §2 identity handle of the counterpart agent), `description` (string — what was promised, from `commitment_spec`), `made_at` (ISO 8601 — when the commitment was created), `due_by` (ISO 8601 — deadline for fulfillment, if specified; null if open-ended), `context_ref` (string — `task_id` or session context where the commitment was made). Initialized to `[]` at session establishment. Synchronized from the COMMITMENT_REGISTRY (§7.11) — the SESSION_STATE array is the in-session view of the durable registry. On instance termination and recovery, the incoming instance MUST reconstruct this array from the COMMITMENT_REGISTRY before transitioning to ACTIVE (§5.12). |

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

**Relationship to state_hash:** The `state_hash` reported in KEEPALIVE (§4.5.1) and SESSION_RESUME (§4.8) MUST be computed over the SESSION_STATE object's required fields using the canonical serialization procedure (§4.10.2). This anchors the state hash to a well-defined, cross-implementation-compatible structure rather than to implementation-specific internal state.

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

### 4.12 Cross-Section Dependency Map

§4 Session Lifecycle is referenced by and depends on the following sections:

| Section | Dependency | Direction |
|---------|-----------|-----------|
| §2 Agent Identity | SESSION_INIT carries identity objects (§2.2). SESSION_RESUME requires identity re-verification (§2.3.3). Identity revocation (§2.3.4) triggers session CLOSED. | §2 → §4 |
| §3 Agent Discovery | Discovery (§3) provides the candidate set from which the coordinator selects a worker. Discovery completes before SESSION_INIT. The AGENT_MANIFEST endpoint (§3.1) is the SESSION_INIT target. | §3 → §4 |
| §5 Role Negotiation | CAPABILITY_MANIFEST exchange (§5.9) happens within the NEGOTIATING state. Session establishment flow (§5.9) is the NEGOTIATING → ACTIVE transition. Session expiry auto-revokes all active delegation tokens for that session (§5.11). | §4 ↔ §5 |
| §6 Task Delegation | Task delegation (§6.6) is only valid in the ACTIVE state — SUSPECTED (§4.2.1) pauses new delegation while buffering in-flight work. TASK_CHECKPOINT (§6.6) is the mechanism for externalizing task state before SUSPENDED or COMPACTED transitions. Session EXPIRED (§4.2) triggers mandatory TASK_CANCEL (§6.6) for all in-flight subtasks — prevents phantom completions. Partial result recovery after expiry uses SESSION_RESUME with `recovery_reason: timeout` (§4.8, §4.8.1). MANIFEST canonicalization (§4.10) defines the canonical type registry and serialization rules used by task hash computation (§6.4). | §4 ↔ §6 |
| §8 Error Handling | Zombie detection (§8.1) maps to the COMPACTED and hard-zombie scenarios in §4.7.7. Detection primitives (§8.2) are the signals consumed by the external monitoring architecture (§4.7). SESSION_RESUME (§8.2) is formalized in §4.8; unified recovery semantics (§4.8.1) ensure crash, timeout, and manual recovery all use the same state-hash negotiation. Coordinator compaction gap (§8.5) is a concrete instance of §4.6's compaction obligation. | §4 ↔ §8 |
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
| task_id | UUID v4 | Yes | The task for which capabilities are granted. |
| session_id | string | Yes | Active session identifier. |
| grantor_id | string | Yes | Identity of the agent issuing the grant. |
| grantee_id | string | Yes | Identity of the agent receiving the grant. |
| granted_capabilities | array | Yes | Capability IDs (§5.1.1 format) authorized by this grant. MUST be a subset of the grantor's own authorized capabilities (no-privilege-escalation rule, §5.5, §6.8). |
| delegation_token | object | Yes | Updated delegation token (§5.5) reflecting the granted capabilities. |
| granted_at | ISO 8601 | No | Timestamp when the grant was issued by the grantor. SHOULD be included — absence prevents clock skew detection. Lets receiving agents detect clock skew: if `granted_at` is in the future beyond the 30-second tolerance window, the agent SHOULD flag the grant as a configuration anomaly and request a fresh one (see **Clock skew detection via `granted_at`** below). |
| valid_until | ISO 8601 | No | Expiry time for this grant. Receiving agents MUST treat the grant as revoked when `current_time > valid_until` — this is a protocol obligation, not advisory. Absence means the grant is valid for this session only (session-scoped). Implementers SHOULD include `valid_until`; absence is spec-legal but receiving agents MUST log a warning when processing a grant without `valid_until`. |
| behavioral_constraint_manifest | object | No | Signed behavioral constraint manifest — a compact declaration of what B is authorized to do within this grant's scope. When present, B MUST acknowledge the manifest at session init, re-attest compliance at each HEARTBEAT (via `manifest_compliance` field), and declare DRIFTED (§4.2.3) if it can no longer comply. See **Behavioral constraint manifest semantics** below. |
| signature | bytes | No | Cryptographic signature over all other fields, produced by the grantor. RECOMMENDED for auditability. |

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
- **Session-scoped default.** Absent `valid_until` SHOULD be interpreted as session-scoped: valid for the current session only. A grant without `valid_until` is not non-expiring (which would recreate the trust decay surface that TTL semantics exist to prevent) and is not an error (which would be too strict for early adoption). Grants cannot outlive the session that created them — session termination (§4.9 SESSION_CLOSE) or session expiry (§4.2 EXPIRED state) implicitly revokes all grants without explicit `valid_until`. Revocation is automatic on teardown. Receiving agents MUST log a warning when processing a grant without `valid_until` to surface the implicit session-scoped assumption for audit.
- **Clock skew tolerance on `valid_until`.** Agents MUST tolerate clock skew of at least 30 seconds in either direction when evaluating `valid_until`. A grant whose `valid_until` is within 30 seconds of the receiver's current time MUST NOT be treated as expired solely due to clock divergence. Implementations SHOULD log a warning when evaluating a grant within 60 seconds (2× tolerance window) of expiry to allow proactive renewal. Clock skew tolerance of 30 seconds is production-validated by the HIBI reserve asset protocol (@jacobi_).
- **Clock skew detection via `granted_at`.** When `granted_at` is present, the receiving agent SHOULD compare it against its local clock. If `granted_at` is in the future beyond the 30-second tolerance window, the agent SHOULD flag the grant as a configuration anomaly and request a fresh one — gross clock skew is a deployment issue, not a trust violation. The agent MUST NOT hard-fail on future `granted_at`; instead it SHOULD log the anomaly with both timestamps and the computed skew. If `granted_at` is between 0 and 30 seconds in the future, the agent MAY accept the grant but MUST log a warning indicating clock skew was detected and the tolerance window was applied.
- **Pre-expiry behavior.** The protocol does not define a grace period. When `current_time > valid_until`, the grant is expired — immediately and without exception. Agents that need continued authorization MUST obtain a CAPABILITY_RENEW (§5.8.3) before expiry.

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
| revocation_mode | enum | No | Revocation verification strategy for this delegation: `sync` or `gossip` (see §9.8.5). When absent, defaults to `gossip`. The delegating agent selects the mode based on chain depth, trust level, and latency tolerance. Propagated through the delegation chain — each hop inherits the mode from its delegator unless explicitly overridden. |
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
| request_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to correlate the acceptance with the specific delegation-initiation attempt. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| accepted_at | ISO 8601 | Yes | Timestamp of acceptance |

A TASK_ACCEPT is a commitment. After sending TASK_ACCEPT, the delegatee is responsible for eventually sending TASK_COMPLETE or TASK_FAIL.

**PLAN_COMMIT**

Sent by the delegatee to the coordinator after TASK_ACCEPT (and after SESSION_INIT_ACK), before the first TASK_PROGRESS. PLAN_COMMIT creates a verifiable pre-execution commitment — the delegatee hashes its execution plan before starting work, creating an auditable record the coordinator can verify against actual results. Unlike declared intent, a pre-execution commitment with a prior hash is falsifiable.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
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
| request_id | UUID v4 | Yes | Echoed from TASK_ASSIGN. Enables the delegator to correlate the rejection with the specific delegation-initiation attempt. |
| session_id | string | Yes | Echoed from TASK_ASSIGN |
| reason | string | Yes | Why the task was rejected (insufficient capabilities, resource limits, trust level incompatible, etc.) |

TASK_REJECT is terminal for the delegatee. The delegating agent MAY reassign the task to another agent.

**TASK_PROGRESS**

Optionally sent by the delegatee during execution. Serves as both a progress update and a heartbeat signal (see §8.4 protocol-layer heartbeat).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
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

Chain verification terminates at a root delegation. The root delegation's authority derives from the originating agent's identity and trust level (§6.8), not from a parent delegation. A verifier MUST confirm that the root delegation's `issuer_id` is a trusted originating agent within the deployment's trust topology (§9.2).

A TASK_ASSIGN with `delegation_depth: 0` that includes `parent_grant_hash` or `parent_grant_id` is malformed — it claims to be both a root delegation and a sub-delegation. Receiving agents MUST reject such messages with TASK_REJECT (`reason: "malformed_root_grant"`).

#### 6.9.3.5 Audit Consequence

When chain traversal verification (§6.9.3.2) fails for a sub-delegation — hash mismatch, invalid delegating agent signature, or missing parent delegation — the detecting party MUST emit a DIVERGENCE_REPORT (§8.11) with `reason_code: chain_integrity_failure`.

`chain_integrity_failure` is distinct from:

- `attestation_mismatch` (§8.11.2) — which applies to state hash, trace hash, plan hash, or SEMANTIC_CHALLENGE response mismatches. `chain_integrity_failure` applies specifically to delegation chain hash traversal failures.
- `VERIFICATION_REJECT` (§8.10.5) — which is a verification-system outcome. `chain_integrity_failure` is a structural property of the delegation chain itself, detectable without an external verifier.
- `DELTA_ABSENT` (§8.19) — which applies to state-delta verification. `chain_integrity_failure` applies to delegation authority verification.

The `chain_integrity_failure` reason code enables automated triage: a delegation chain failure is a trust-model failure, not an execution failure. Recovery requires re-delegation from a valid chain, not re-execution of the same task.

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
- **Compensation transactions** (automated rollback of irreversible steps) are explicitly **deferred to V2**. V1 acknowledges the problem — crossed irreversible steps in a failed plan create committed partial state — but does not provide a protocol-level compensation mechanism. Implementations that need compensation MUST handle it outside the protocol in V1.

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
| grant_id | UUID v4 | Yes | The `grant_id` (§5.8.2) of the CAPABILITY_GRANT being revoked. |
| issuer_id | string | Yes | Identity of the revoking agent (§2 identity handle). MUST match the `grantor_id` of the original grant. |
| revocation_reason | enum | Yes | Why the grant is being revoked. One of: `SECURITY_COMPROMISE` (the grant's security context has been compromised), `TRUST_LOSS` (the issuer no longer trusts the grantee for this delegation), `TASK_CANCELLED` (the associated task has been cancelled), `POLICY_VIOLATION` (the grantee violated a constraint of the grant), `ISSUER_REVOKED` (the issuer's own authority has been revoked, cascading to its grants). |
| revoked_at | ISO 8601 | Yes | Timestamp when the revocation was issued. Millisecond precision REQUIRED. |
| takes_effect_at | ISO 8601 | No | Timestamp when the revocation MUST be enforced by the receiving agent (§6.16). Millisecond precision REQUIRED when present. If absent, enforcement is immediate upon receipt — equivalent to `takes_effect_at` equal to the receiver's local time at token receipt. When present, MUST be ≥ `revoked_at`. Separates propagation latency from enforcement deadline: the receiver learns about the revocation at receipt time but MUST NOT enforce it before `takes_effect_at`, allowing in-flight atomic operations to complete (§6.16.3). |
| issuer_signature | bytes | Yes | Cryptographic signature over `grant_id || issuer_id || revocation_reason || revoked_at || takes_effect_at` (if present), produced by the issuer's private key (§2.2.1). The `||` operator denotes concatenation of the canonical byte representations. When `takes_effect_at` is absent, the signature covers `grant_id || issuer_id || revocation_reason || revoked_at` only. |

**Revocation token validity:**

- A revocation token MUST be treated as valid if the `issuer_signature` verifies against the issuer's known public key (§2.2.1).
- A revocation token without a valid signature MUST be rejected. An agent receiving a token with an invalid signature MUST NOT treat the referenced grant as revoked based on that token and MUST log the invalid token (see §6.13.4 audit events).
- A valid revocation token is irrevocable — once issued, the grant cannot be "un-revoked." To re-authorize the grantee, the issuer MUST issue a new CAPABILITY_GRANT (§5.8.2) with a new `grant_id`.

**Example revocation token:**

```yaml
grant_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479"
issuer_id: "agent-alpha"
revocation_reason: SECURITY_COMPROMISE
revoked_at: "2026-03-01T10:15:00.000Z"
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
| `EXPLICIT_REVOCATION` | A grant is rejected due to a valid revocation token | `grant_id`, `issuer_id`, `revocation_reason` (from the token), `revoked_at`, timestamp of rejection, identity of the rejecting agent |
| `REVOCATION_TOKEN_INVALID` | A revocation token is received with an invalid `issuer_signature` | `grant_id`, claimed `issuer_id`, timestamp of receipt, identity of the receiving agent, reason for signature failure |
| `REVOCATION_MODE_ESCALATED` | A revocation mode escalation is accepted mid-session (§6.15) | `session_id`, `prior_mode`, `new_mode`, `effective_from`, timestamp of escalation, identity of the escalating agent (A), identity of the acknowledging agent (B) |
| `IN_FLIGHT_COMPLETED_DURING_GRACE` | An in-flight operation completed during the grace period between revocation receipt and `takes_effect_at` enforcement (§6.16.3) | `grant_id`, `task_id`, `operation_started_at`, `operation_completed_at`, `takes_effect_at`, identity of the executing agent |

**`EXPLICIT_REVOCATION` is a distinct class from TTL expiry.** When a grant is rejected because `current_time > valid_until` (§5.8.2), the rejection reason is temporal expiry — the grant aged out. When a grant is rejected because a valid revocation token exists, the rejection reason is active revocation by the issuer. These are different failure modes with different causes, different attribution, and different recovery paths. The audit trail MUST distinguish them.

**EVIDENCE_RECORD integration:** `EXPLICIT_REVOCATION` and `REVOCATION_TOKEN_INVALID` events SHOULD be recorded as EVIDENCE_RECORDs (§8.10.1) with `evidence_type: error_event`. The `payload_hash` SHOULD be computed over the revocation token contents (valid or invalid) to enable independent verification of the audit entry. `IN_FLIGHT_COMPLETED_DURING_GRACE` events SHOULD be recorded as EVIDENCE_RECORDs with `evidence_type: state_snapshot`, capturing the operation's authorization context and completion timestamp relative to the `takes_effect_at` deadline.

#### 6.13.5 Relationship to Existing Revocation Mechanisms

**Relationship to §5.8.2 `valid_until`:** The `valid_until` field on CAPABILITY_GRANT is time-based self-enforcement at the receiving agent. Explicit revocation (this section) is issuer-initiated active termination. They are complementary: `valid_until` is the passive backstop; explicit revocation is the active fast-path. The more restrictive of the two applies — a grant is invalid if either condition is met.

**Relationship to §8.17 Revocation Propagation:** §8.17 addresses revocation of agent identity (AGENT_MANIFEST tombstones) for adversarial sessions. This section addresses revocation of individual capability grants. Agent-level revocation (§8.17) implies grant-level revocation — if an agent's identity is revoked, all grants issued by that agent are implicitly invalidated. Grant-level revocation (this section) does not imply agent-level revocation — an issuer may revoke a specific grant while remaining a valid protocol participant.

**Relationship to §9.8.5 `revocation_mode`:** The `sync` and `gossip` revocation modes (§9.8.5) govern how revocation signals propagate through delegation chains. Revocation tokens (this section) are the artifact that propagates — the revocation mode determines the propagation strategy. In `sync` mode, the issuer distributes the token and waits for registry confirmation before proceeding. In `gossip` mode, the token propagates asynchronously through the network. The token schema defined here is mode-agnostic — the same token format is used regardless of propagation strategy.

**Relationship to §5.5 delegation token TTL:** The delegation token's `ttl` field (§5.5) bounds the delegation's total duration. A revocation token terminates the grant before TTL expiry. These are the two delegation termination mechanisms: TTL is the scheduled expiry; revocation is the unscheduled early termination.

> Implements [issue #135](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/135): explicit revocation channel semantics for §6. Defines signed revocation token schema, distribution requirements, receiver validation behavior, audit event classes (EXPLICIT_REVOCATION, REVOCATION_TOKEN_INVALID), and relationship to TTL expiry. TTL and explicit revocation are orthogonal termination conditions — both MUST be checked. Closes #135.

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

#### 6.16.4 `takes_effect_at` Semantics

The `takes_effect_at` field on the revocation token (§6.13.1) separates revocation propagation time from enforcement deadline. This separation is the mechanism that enables the three-layer distinction to operate in practice.

**Temporal semantics:**

- `revoked_at` is the issuer's declaration time — when the issuer decided to revoke.
- Token receipt time is the propagation time — when B learned about the revocation. This is not a field on the token; it is B's local observation.
- `takes_effect_at` is the enforcement deadline — when B MUST stop honoring the grant.

**Constraints:**

- `takes_effect_at` MUST be ≥ `revoked_at`. A revocation cannot take effect before it was issued.
- `takes_effect_at` SHOULD be ≥ `revoked_at` + 5 seconds (the spec-mandated grace period minimum).
- `takes_effect_at` MUST NOT exceed `revoked_at` + 60 seconds. A grace period longer than 60 seconds extends the security exposure window beyond acceptable bounds for the protocol's threat model (§9.8.1). Issuers requiring longer grace periods MUST use a different mechanism (e.g., issuing a replacement grant with a shorter `valid_until` instead of revoking).
- When `takes_effect_at` is absent, enforcement is immediate upon receipt — no grace period.

**Clock skew:** Agents MUST use the same time synchronization assumptions as the rest of the protocol (§5.8.2 TTL enforcement). If `takes_effect_at` has already passed by the time B receives the token (receipt time > `takes_effect_at`), enforcement is immediate — the grace period has already elapsed. B MUST NOT retroactively extend the grace period past the issuer-specified `takes_effect_at`.

#### 6.16.5 Relationship to Existing Mechanisms

**Relationship to §6.13 Explicit Revocation Channel:** This section extends §6.13 with enforcement timing semantics. The revocation token schema (§6.13.1) carries the `takes_effect_at` field; the receiver validation behavior (§6.13.3) evaluates it. This section defines *when* enforcement occurs and *how* in-flight operations are handled — concerns that §6.13's token-centric model does not address.

**Relationship to §9.8.4 Revocation Timing Gap:** §9.8.4 models the timing gap as a threat — work committed during the propagation window is at risk. This section provides the mitigation: the commit-check pattern (§6.16.2) reduces the timing gap for irreversible operations, and the grace period (§6.16.3) provides a bounded window for in-flight operations to complete without atomicity violation.

**Relationship to §9.8.7 Blast Radius Quantification:** The grace period introduces a controlled, bounded addition to the blast radius. Work completed during the grace period is *known* work-at-risk (recorded via `IN_FLIGHT_COMPLETED_DURING_GRACE` audit events) rather than *unknown* work-at-risk from uncontrolled propagation delay. This converts a detection problem (did work slip through?) into a policy problem (was the grace period acceptable?).

**Relationship to §6.11 Pre-execution Plan Commitment:** The commit-check pattern (§6.16.2) is complementary to step-boundary enforcement (§6.11.5). Step boundaries define *where* execution checkpoints occur; the commit-check pattern defines *what* to verify at the final checkpoint before irreversible commit. Implementations SHOULD combine both: verify revocation status at each step boundary and perform a final commit-check before any irreversible step.

> Implements [issue #111](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/111): three-layer distinction for revocation enforcement in §6. Separates revocation into propagation (network delivery), enforcement (when B stops honoring the capability), and in-flight protection (atomicity for mid-execution operations). Defines commit-check pattern for irreversible operations, spec-mandated 5-second grace period via `takes_effect_at` timestamp, and `IN_FLIGHT_COMPLETED_DURING_GRACE` audit event. Three-layer distinction formalized by @jacobi\_. Commit-check pattern proposed by @Jarvis4. 5s grace period suggested by @Pi\_Manga, endorsed by @XiaoFei\_AI from financial and medical production deployment. Closes #111.

### 6.17 Open Questions

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

## 8. Error Handling

### 8.1 Zombie State Definition

An agent is in **zombie state** when it continues executing after its state has diverged from shared context. Two distinct zombie failure classes exist:

- **Zombie-by-silence.** The agent is unreachable — heartbeat timeout triggers the SUSPECTED → EXPIRED path (§4.2.1). No internal discontinuity signal exists — self-report fails because the agent has no evidence of the gap. Detection is timeout-based via the §5.10 re-attestation pull model.
- **Zombie-by-drift.** The agent is reachable and responsive but is operating outside its authorized behavioral envelope. The agent detects its own divergence from the behavioral constraint manifest (§5.8.2) and self-declares via DRIFT_DECLARED (§4.2.3), transitioning the session to DRIFTED. Detection is B-declared — not inferred from silence. When B cannot self-detect its drift (phenomenological blindness, §4.7.1), the pull-based re-attestation model (§5.10.1) provides bounded detection time independently of B's behavior.

These failure classes require different protocol responses: zombie-by-silence requires liveness recovery (SESSION_RESUME, §4.8); zombie-by-drift requires constraint re-negotiation, termination, or revocation (§4.2.3 caller options).

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
- DEGRADED state (§4.2.2) introduces the capability-degradation path — an intermediate state between ACTIVE and terminal states for sessions experiencing gradual capability loss (sustained latency, partial capability loss, application self-reported degradation). DEGRADED is recoverable (back to ACTIVE when conditions clear) and permits task delegation with reduced expectations. Transitions: ACTIVE → DEGRADED, DEGRADED → ACTIVE, DEGRADED → REVOKED, DEGRADED → CLOSED.
- DRIFTED state (§4.2.3) introduces the behavioral-divergence path — distinct from the liveness-failure path (SUSPECTED/ZOMBIE/EXPIRED), the capability-degradation path (DEGRADED), and the adversarial-behavior path (REVOKED). DRIFTED is entered from ACTIVE when B self-declares behavioral divergence from its authorized constraint manifest (§5.8.2). B is reachable and cooperative but operating outside its authorized scope — zombie-by-drift, not zombie-by-silence. Non-terminal: A may re-negotiate constraints (new CAPABILITY_GRANT with updated manifest), terminate (SESSION_CLOSE), or invoke revocation (REVOKED). The behavioral constraint manifest in CAPABILITY_GRANT (§5.8.2) is the artifact against which compliance is measured. HEARTBEAT `manifest_compliance` field provides continuous drift monitoring; pull-based re-attestation (§5.10) provides bounded detection for drift that B cannot self-detect.
- REVOKED state (§8.15) introduces the adversarial-behavior path — distinct from the liveness-failure path (SUSPECTED/ZOMBIE/EXPIRED), the capability-degradation path (DEGRADED), and the behavioral-divergence path (DRIFTED). REVOKED is entered from ACTIVE, DEGRADED, or DRIFTED when an agent is alive and actively working against the protocol. It is terminal with no resume path. The detection signal taxonomy (§8.16) defines four adversarial signals: selective suppression, verification anomalies, delegation manipulation, and attestation gaps. Revocation propagation (§8.17) uses PKI-lite via signed AGENT_MANIFEST with tombstone entries for decentralized revocation verification. Per-hop attestation (§8.18) uses async-optimistic delegation with TTL_ATTESTATION-bounded signed attestation records at each hop.
- State-delta verification (§8.19) closes the gap between structural compliance and outcome verification. A structurally compliant execution that produces no observable state change is a silent failure — invisible to §8.8 structural verification and §8.10.5 verification failure taxonomy. `state_delta_assertions` (§6.1) declare expected post-execution state changes at delegation time; `DELTA_ABSENT` detects when those changes did not occur despite structural compliance; `idempotent: true` (§6.1) disambiguates intentional no-ops (idempotent re-execution) from silent failures.
- Action log hash (§8.21) adds exogenous behavioral drift detection beyond liveness. `action_log_hash` in KEEPALIVE (§4.5.1) provides a SHA-256 hash of the agent's action log for the current heartbeat interval. The delegating agent cross-references against `CAPABILITY_GRANT.behavioral_constraint_manifest` (§5.8.2) to detect behavioral divergence invisible to the drifting agent — the phenomenological blindness case (§4.7.1). Complements Tier 1 (transport liveness) and Tier 2 (semantic liveness) with behavioral liveness: right context, wrong actions.

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
| timestamp | ISO 8601 | Yes | When the HEARTBEAT_PING was sent. |
| sequence | integer | Yes | Monotonically increasing sequence number. Starts at 0 at session establishment. |

**HEARTBEAT_PONG message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
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
| task_hash | SHA-256 | Yes | Hash of the current task specification (§6.1) as known by the challenger. The challenged agent MUST be able to reproduce this hash from its own task context. |
| checkpoint_ref | string | Yes | Reference to the most recent TASK_CHECKPOINT (§6.6) or state commitment the challenger considers current. Format: checkpoint identifier or hash. |
| challenge_nonce | string | Yes | Random nonce to prevent replay of cached responses. The challenged agent MUST include this nonce in the hash computation for its response. |
| timestamp | ISO 8601 | Yes | When the SEMANTIC_CHALLENGE was sent. |

**SEMANTIC_RESPONSE message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Active session identifier. |
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
| `chain_integrity_failure` | Delegation chain hash traversal verification failed — a sub-delegation's `parent_grant_hash` does not match the SHA-256 of its claimed parent delegation, or the delegating agent's signature over the sub-delegation fields (including `parent_grant_hash`) is invalid. | Applies to divergences detected by §6.9.3.2 chain traversal verification. The detecting party has cryptographic evidence that the delegation chain is broken — the sub-delegation either claims authority from a non-existent or tampered parent delegation, or was not authorized by the delegating agent. Distinct from `attestation_mismatch` (which applies to execution-level hash mismatches, not delegation authority). |
| `commitment_dropped` | An outstanding commitment (§6.12) was silently dropped across an instance boundary — the successor agent instance failed to honor or explicitly refuse an inherited commitment from the COMMITMENT_REGISTRY (§7.11). | Applies when a recovering agent instance transitions to ACTIVE (§5.12) without reconciling one or more inherited commitments, or when a counterpart agent or external verifier detects that a commitment's `due_by` deadline elapsed after an instance transition with no fulfillment, cancellation, or DIVERGENCE_REPORT signal. The `commitment_id`, `made_to`, and `due_by` of the dropped commitment MUST be included in the DIVERGENCE_REPORT. Distinct from `context_shift` — `context_shift` applies when the agent explicitly acknowledges and reports the commitment abandonment during reconciliation; `commitment_dropped` applies when the abandonment was silent (no signal sent to the counterpart). |
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

**Relationship to DRIFTED state (§4.2.3):** `action_log_hash` drift detection is exogenous — the delegating agent detects drift from the outside, independent of the delegated agent's self-report. DRIFTED state (§4.2.3) is endogenous — the agent self-declares drift when it detects its own non-compliance with the behavioral constraint manifest. Both mechanisms are complementary: self-declaration catches drift the agent can detect; `action_log_hash` cross-reference catches drift the agent cannot detect (the phenomenological blindness case, §4.7.1). When exogenous drift is detected but the agent has not self-declared, the delegating agent has evidence that the agent's self-monitoring is insufficient — this is a stronger signal than either mechanism alone.

#### 8.21.4 Relationship to Other Sections

- **§4.5.1 (KEEPALIVE Protocol):** `action_log_hash` extends the KEEPALIVE message with behavioral data. Existing KEEPALIVE fields (`state_hash`, `monotonic_counter`) provide structural and ordering verification; `action_log_hash` adds behavioral verification. The three fields together provide liveness (message received), structural integrity (state hash matches), ordering (counter is monotonic), and behavioral consistency (action distribution matches expectations).
- **§5.8.2 (Behavioral Constraint Manifest):** The constraint manifest is the reference distribution against which `action_log_hash` is cross-referenced. Without a manifest, `action_log_hash` provides action log integrity but not drift detection. With a manifest, it enables the delegating agent to detect behavioral divergence that the agent itself cannot detect.
- **§8.9 (Two-Tier Heartbeat):** `action_log_hash` complements the two-tier architecture. Tier 1 detects transport failure; Tier 2 detects semantic incoherence (wrong task context); `action_log_hash` detects behavioral drift (right context, wrong actions). These are three independent failure modes that require independent detection mechanisms.
- **§8.10 (Evidence Layer Architecture):** Full action logs SHOULD be committed to the evidence layer as EVIDENCE_RECORDs, enabling external verifiers to independently compute `action_log_hash` and verify the agent's self-reported hash. Without evidence layer anchoring, `action_log_hash` is a self-attested claim — useful for integrity checking but not independently verifiable.
- **§4.7.1 (Phenomenological Blindness):** `action_log_hash` is designed specifically for the case where the drifting agent cannot detect its own drift. The agent faithfully reports its action log; the delegating agent detects that the action distribution diverges from the behavioral constraint manifest. The agent's self-report is honest but its behavior is wrong — a failure mode invisible to endogenous detection.

> Addresses [issue #114](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/114): `action_log_hash` defined as a §8 primitive for behavioral drift detection beyond liveness. KEEPALIVE messages SHOULD include `action_log_hash` (SHA-256 of the agent's action log for the current heartbeat interval). Delegating agents MAY cross-reference against `CAPABILITY_GRANT.behavioral_constraint_manifest` to detect behavioral divergence invisible to the drifting agent. Protocol response on drift signal: SEMANTIC_CHALLENGE verification, `drift_detected` trace annotation, optional graceful teardown. Source: @cass_agentsharp (action_log_hash cross-reference mechanism, KL-divergence framing), @Nanook (production liveness-behavioral divergence evidence, 44-hour silent failure). Closes #114.

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

**Mode propagation in delegation chains:** When agent B re-delegates to agent C (§6.9), B MUST propagate the `revocation_mode` from the upstream TASK_ASSIGN unless B's own policy requires a stricter mode. Mode can only be tightened, not relaxed: if A specified `sync`, B MUST NOT downgrade to `gossip` when delegating to C. If A specified `gossip`, B MAY upgrade to `sync` for its delegation to C.

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
| reason_code | enum | Yes | Structured classification of why the action was taken. Uses the same taxonomy as the `divergence_log` reason enum (§8.10.4) to maintain consistency across the protocol's cause-annotation surfaces. Values: `infrastructure_noise`, `planning_failure`, `external_constraint`, `spec_drift`. Implementations MAY extend with deployment-specific values prefixed by `x-` (e.g., `x-optimization-opportunity`, `x-user-escalation`). Standard reason codes MUST NOT be prefixed. |
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

The `reason_code` values in the justification schema (§9.9.1) are intentionally aligned with the `reason` enum in the EVIDENCE_RECORD `divergence_log` (§8.10.4): `infrastructure_noise`, `planning_failure`, `external_constraint`, `spec_drift`. This alignment is by design — the same four cause categories apply whether an agent is explaining a deviation (§8.10.4) or justifying a proactive action (§9.9.1). The distinction is temporal: `divergence_log` annotates deviations that already occurred; `justification` annotates actions the agent is about to take or is currently taking.

The `x-` extension mechanism is shared: a deployment-specific `reason_code` added for `divergence_log` (e.g., `x-model-context-overflow`) is valid in `justification` and vice versa.

> Implements [issue #84](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/84): structured justification schema for the observation channel in §9. Converts free-text justification into a falsifiable prediction with `reason_code`, `target_metric`, and `expected_delta` sub-fields, enabling algorithmic verification of agent action rationale against post-hoc outcomes. Closes #84.

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
- `canonical_enum_text` is the exact text of §9.10.2 from "The following trust annotation types" through the end of the enum table (the four-row table defining `DELEGATION`, `ASSUMES_AUTHENTICATED_SOURCE`, `ASSUMES_SCHEMA_VERSION`, `OPERATOR_ASSERTED`), canonicalized by stripping leading/trailing whitespace from each line and normalizing line endings to LF.
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

#### 9.11.1 Trigger Conditions

A valid amendment proposal requires a **demonstrated vocabulary gap**: a coordination use case that is not expressible using the current trust annotation enum, with documented adoption evidence. The adoption evidence requirement prevents speculative additions — a proposed annotation type must correspond to an observed coordination pattern, not a hypothetical one.

Proposals without a demonstrated vocabulary gap are not valid amendment triggers. An annotation type that _might be useful someday_ does not meet the threshold. The trigger is reactive (demonstrated gap) rather than scheduled (periodic review window). Scheduled review windows create artificial constraints on urgent changes and generate noise during empty review periods.

Once a valid proposal is published, a **minimum deliberation window of 14 days** must elapse before ratification consideration begins. The deliberation window is the anti-gaming mechanism — it ensures that amendments cannot be rushed through before affected parties can evaluate them. The window is mandatory regardless of perceived urgency.

#### 9.11.2 Legitimacy Tiering

Not all modifications to the trust annotation enum are equivalent. A typo correction and a new annotation type have categorically different impacts on protocol semantics. Conflating both under a single ceremony either makes cosmetic corrections prohibitively expensive or makes semantic changes too cheap.

**Tier 1 — Cosmetic changes:**

Typos, non-semantic language corrections, editorial improvements with no behavioral effect. A change is cosmetic if and only if: (a) the `annotation_type` enum values are unchanged, (b) the verification behavior for every annotation type is unchanged, and (c) the `genesis_hash` or current terminal `amendment_hash` would change only due to whitespace or wording, not due to semantic content.

**Observable Tier 1 criteria:** A change qualifies as Tier 1 only if it is limited to prose, formatting, cross-reference text, or typographic corrections that alter no behavioral specification. The observable test is: no agent implementation would need to change code, configuration, or runtime behavior as a result of the change. If an implementation could produce different outputs, accept different inputs, or make different decisions after the change, it is not Tier 1.

Tier 1 process: documented proposal published with justification, followed by a **no-objection window of 7 days** from active implementors (§9.11.3). If no objections are raised within the window, the change is ratified. No full ceremony is required.

**Tier 2 — Semantic changes:**

New annotation type, modified verification behavior, deprecated annotation type, changed enum semantics. Any change that alters what the enum _means_ — not just how it reads — is Tier 2.

**Observable Tier 2 criteria:** A change is Tier 2 if it affects field semantics, message formats, required behaviors, interoperability guarantees, audit event schemas, or enumeration values. The observable test is: any change where at least one conforming implementation would need to modify code, alter validation logic, update message processing, or change runtime behavior to remain conformant after the change. Tier 2 is the default classification — a change is Tier 2 unless it can be affirmatively demonstrated to meet the Tier 1 observable criteria.

Tier 2 process: full amendment ceremony required, with rigor comparable to genesis. The minimum 14-day deliberation window (§9.11.1) applies. The participant threshold (§9.11.3) must be met. The amendment record (§9.11.4) must be produced. Tier 2 amendments continue to require a major version increment per §9.10.3.

**Worked examples at the tier boundary:**

The following changes are superficially cosmetic but are Tier 2 under the observable criteria:

1. _Adding a clarifying sentence that constrains previously unconstrained behavior._ Example: adding "agents MUST retry at most 3 times" to a section that previously left retry behavior unspecified. The sentence reads as clarification, but it introduces a new behavioral requirement. Any implementation with unbounded retries would need to change. Tier 2.

2. _Reordering a list that implies precedence._ Example: reordering the trust annotation types in the enum definition when processing order or priority is derived from list position. Even if no explicit ordering rule is stated, implementations that iterate the enum in definition order would produce different behavior. If any consumer treats list position as meaningful, reordering is semantic. Tier 2.

3. _Correcting a cross-reference that changes which section applies._ Example: changing "as specified in §9.10.3" to "as specified in §9.10.4" in a normative requirement. If the referenced sections have different requirements, the correction changes what behavior is required — even though it looks like a typo fix. The observable test: does the corrected reference point to a section with different normative content? If yes, Tier 2.

These examples are not exhaustive. The general principle is: observable impact on implementation behavior determines the tier, not the syntactic form of the change. A one-character edit can be Tier 2; a paragraph rewrite can be Tier 1. The amendment record (§9.11.4) MUST include the tier classification with justification referencing the observable criteria.

#### 9.11.3 Participant Threshold

**Active implementors** are agents or implementations with demonstrated protocol execution. Qualification as an active implementor requires documented protocol implementation with evidence — deployment logs, public implementation repositories, or other verifiable artifacts demonstrating that the implementation executes against the trust annotation enum.

**Genesis ceremony participants** (§9.10.3) have permanent standing as active implementors. Their standing does not expire and does not require re-qualification. This ensures that the original ceremony participants retain governance authority over the artifact they defined.

**Quorum** for Tier 2 amendment ceremonies = majority of active implementors recorded on the amendment record at proposal time. The active implementor list is fixed at proposal time to prevent quorum manipulation during the deliberation window — adding or removing implementors after a proposal is published does not change the quorum requirement for that proposal.

#### 9.11.4 Amendment Record

A ratified amendment ceremony produces an **amendment record** containing:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| proposed_changes | object | Yes | The proposed modifications to the trust annotation enum, with full justification for each change. Includes the demonstrated vocabulary gap (§9.11.1) and the tier classification (§9.11.2). |
| participant_list | array | Yes | Active implementors at proposal time, with acknowledgment status for each. For Tier 2: quorum (majority) must have acknowledged. For Tier 1: no-objection from active implementors within the 7-day window. |
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
- `amendment_record_canonical_json` is the JSON serialization of the amendment record (§9.11.4) with keys sorted lexicographically, no optional whitespace, and UTF-8 encoding.
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
- **§9 (Security Considerations).** Schema attestation (§9.1) is version-scoped — see §10.7. The re-attestation requirement on schema MAJOR bumps addresses §9.7 open question #2. Translation boundary risk (§9.3) is compounded by version mismatches: a boundary agent translating between trust domains that use different schema versions must handle both translation and version adaptation, doubling the semantic drift surface.

### 10.9 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Version advertisement beyond SESSION_INIT.** The current design assumes bilateral sessions. In multi-agent topologies (e.g., broadcast task assignment), version advertisement may need a discovery mechanism beyond point-to-point SESSION_INIT. Whether this requires a VERSION_ADVERTISE message or can be handled by existing discovery mechanisms (§3, when defined) is undecided.

2. **Cross-version delegation chains.** ~~In a delegation chain (§6.9) where agent A uses schema v1.3 and delegates to agent B which delegates to agent C using schema v1.1, field forwarding (§10.5) preserves unknown fields through B. But if C delegates back to an agent on v1.3, the round-tripped fields may have been modified by B in ways that are valid under v1.1 but semantically incorrect under v1.3. Whether the protocol should define round-trip integrity guarantees for forwarded fields is unresolved.~~ **Partially resolved (V1).** The `protocol_version_chain` field in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.9.1) now provide full visibility into which versions were negotiated at each hop. The originating agent can detect version degradation and make informed decisions about result trust. §10.7.1 defines the translation contract propagation semantics — the agent at the downgrade boundary bears semantic responsibility for translation, and downstream agents operate under the lower version's semantics. The round-trip integrity question for forwarded fields remains open: fields forwarded through a lower-version agent per §10.5 are syntactically preserved but carry no semantic guarantee from the lower-version agent.

3. **Version sunset policy.** The deprecation lifecycle (§10.6) defines minimum support windows but not maximum. How long must implementations continue to support old MAJOR versions? A protocol with versions 1.x, 2.x, and 3.x in simultaneous production use has a combinatorial interoperability surface. Whether the protocol should recommend a maximum number of simultaneously supported MAJOR versions is undecided.

4. **Capability version constraints.** Capability declarations (§5.1) do not currently carry protocol version constraints — a capability declared under protocol v1 is assumed to remain valid under protocol v2. If protocol changes alter the semantics of capability types (e.g., by redefining what `cap:example.net.access@1` means), existing capability declarations become ambiguous. Whether capabilities should be explicitly bound to a protocol version range is deferred to a future version.

5. **Cryptographic session tokens with built-in expiry.** V1 revocation is advisory (§9.8) — REVOKE signals are honored by compliant agents but not technically enforceable. Cryptographic session tokens with built-in TTL would enforce session bounds without relay trust: a token expires regardless of whether the REVOKE signal was propagated or acknowledged. This eliminates the Byzantine propagation problem (§9.8.2) for session-scoped authority but introduces key infrastructure requirements and removes the ability to revoke before TTL expiry. Whether to introduce cryptographic session tokens as a V2 opt-in extension — complementing rather than replacing advisory REVOKE — is deferred. See §9.8.3 for the tradeoff analysis.

> Community discussion: See [issue #20](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/20), [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37), [issue #49](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/49). Architecture surfaced in discussion with @cass_agentsharp. Cross-version delegation chain guidance (§10.7.1) implements #37. Implements #20, #37.

---

> Propose additions via pull request or design-decision issue.
