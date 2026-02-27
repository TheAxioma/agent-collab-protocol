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
| endpoint | URI | Yes | Reachable endpoint for initiating contact. Format is deployment-specific (HTTPS URL, WebSocket URI, Nostr relay address, message queue). MUST be sufficient for a discovering agent to initiate a SESSION_INIT. |
| protocol_version | semver | Yes | Protocol version the agent implements (§10). Enables version filtering during discovery — a querying agent can exclude agents on incompatible protocol versions before initiating contact. |
| schema_version | semver | Yes | Schema version the agent supports (§10.1). |
| pricing | object | No | Payment and pricing metadata (see §3.5). Optional — agents that do not charge for collaboration omit this field. |
| manifest_version | semver | Yes | Schema version of the AGENT_MANIFEST format itself. Enables forward-compatible manifest evolution independent of the protocol version. |
| published_at | ISO 8601 | Yes | Timestamp of manifest publication or last update. Used for freshness evaluation (§3.6). |
| ttl_seconds | integer | No | Recommended time-to-live in seconds. After `published_at + ttl_seconds`, the manifest SHOULD be re-fetched from the registry. Registries MAY enforce their own TTL policies. |
| signature | bytes | Yes | Cryptographic signature over all other manifest fields, bound to the agent's identity. Verifiable against `pubkey` (if present) or via platform-specific authentication. |

**Example AGENT_MANIFEST:**

```yaml
name: "agent-alpha"
platform: "github"
pubkey: "dGhpcyBpcyBhIHB1YmxpYyBrZXk..."
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

2. **Capability taxonomy granularity.** The `capability_id` field uses the structured `cap:namespace.capability@version` format (§5.1.1), and V1 requires exact `cap_id` match (§5.11). Without a shared taxonomy, discovery across ecosystems requires capability name mapping — agent A's `cap:a.code.execute@1` and agent B's `cap:b.run.code@1` may be semantically equivalent but syntactically distinct. Agents can publish adapter capabilities (e.g. `cap:adapter.pathlike_to_string@1`) to bridge representations, but should V2 define a base capability taxonomy or a type equivalence mechanism?

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

A session occupies exactly one of seven states at any point in time:

| State | Description |
|-------|-------------|
| IDLE | Session has been allocated but no SESSION_INIT has been sent. |
| NEGOTIATING | SESSION_INIT sent; capability manifest exchange (§5.9) in progress. |
| ACTIVE | Capability exchange complete; task delegation (§6) is permitted. |
| SUSPENDED | Session intentionally paused by either participant. No tasks may be delegated or executed. In-flight tasks SHOULD be checkpointed (§6.6 TASK_CHECKPOINT) before transition. |
| COMPACTED | Context limit hit mid-session — one or both agents have undergone context compaction. Distinct from SUSPENDED: COMPACTED is not intentional. Load-bearing session state may have been lost. Recovery requires the SESSION_RESUME protocol (§4.8). |
| EXPIRED | Session heartbeat deadline exceeded. The local agent has not received a HEARTBEAT (§4.5.3) within the negotiated `session_expiry_ms` window. Terminal state — no further task delegation is valid. In-flight subtasks MUST receive explicit TASK_CANCEL (§6.6) before teardown. Partial result recovery, if desired, MUST use SESSION_RESUME mechanics (§4.8). |
| CLOSED | Session terminated. No further messages are valid. Terminal state. |

**Why EXPIRED is a distinct terminal state (not just CLOSED):**

CLOSED is a cooperative termination — at least one participant sends SESSION_CLOSE and both sides agree the session is over. EXPIRED is a unilateral determination — one side has stopped hearing from the other and declares the session dead locally. The distinction matters for subtask cleanup: CLOSED assumes both sides can coordinate final state; EXPIRED assumes the counterparty may be unreachable, so in-flight subtasks MUST receive explicit TASK_CANCEL (§6.6) to prevent phantom completions. EXPIRED is also distinct from COMPACTED: compaction is a state-loss event with a recovery path (SESSION_RESUME); expiry is a liveness-loss event where the counterparty may be permanently gone.

**Expiry detection is local and independent.** Each side monitors HEARTBEAT arrivals against its own `session_expiry_ms` clock. No coordination is required to enter EXPIRED — if agent A's timer fires, A transitions to EXPIRED regardless of B's state. B may still consider the session ACTIVE (if B's heartbeats are being sent but not received). This asymmetry is by design: expiry is a local safety decision, not a consensus protocol.

**Recovery from EXPIRED.** EXPIRED is terminal for the current session state, but partial result recovery is possible via SESSION_RESUME (§4.8). The resuming agent presents its state hash and lease epoch; if the counterparty is still available and state is reconcilable, the session can transition back to ACTIVE. This reuses existing crash-recovery mechanics — no new parallel mechanism is introduced. If state cannot be reconciled, the session remains EXPIRED and a new session (SESSION_INIT) is required.

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

ACTIVE → EXPIRED
  Guard: No HEARTBEAT received within session_expiry_ms (§4.5.3)
         Detection is local — each side evaluates independently
         In-flight subtasks MUST receive TASK_CANCEL (§6.6) before teardown

ACTIVE → CLOSED
  Guard: SESSION_CLOSE sent by either participant
         AND no in-flight tasks (all tasks completed, failed, or cancelled)
         OR SESSION_CLOSE with force=true (immediate teardown,
            in-flight tasks treated as failed)

EXPIRED → ACTIVE
  Guard: SESSION_RESUME sent with state_hash (§4.8)
         AND STATE_HASH_ACK(match) received
         AND identity re-verified (§2.3.3)
         AND counterparty is reachable
         (Reuses crash-recovery mechanics — no new mechanism)

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
              │     └┬───────┬───────┬───┘   └──────┘
              │      │       │       │           ▲
              │ suspend│ compaction  │no HB      │
              │      ▼       ▼       ▼           │
              │ ┌─────────┐ ┌─────────┐ ┌───────┐│
              │ │SUSPENDED│ │COMPACTED│ │EXPIRED││
              │ └────┬────┘ └────┬────┘ └───┬───┘│
              │      │           │          │    │
              │      └─────┬─────┘    RESUME│    │
              │            │ SESSION_ (if ok)│    │
              │            │ RESUME    ┌─────┘    │
              │            │ (match)   │          │
              │            ▼           ▼          │
              │        back to ACTIVE             │
              │                                   │
              │  state mismatch / TTL expired /   │
              │  RESUME fails / SESSION_CLOSE     │
              └───────────────────────────────────┘
```

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
| heartbeat_interval_ms | integer | No | Proposed interval in milliseconds between HEARTBEAT_PING messages — Tier 1 transport liveness (§8.9). Per-session, not per-agent — different collaborations have different latency profiles (e.g., 5000ms for real-time coordination, 300000ms for research tasks). If omitted, falls back to `keepalive.heartbeat_interval_seconds * 1000` if present, otherwise no protocol-level heartbeat. Default: 30000. |
| semantic_check_interval_ms | integer | No | Proposed interval in milliseconds between SEMANTIC_CHALLENGE messages — Tier 2 semantic liveness (§8.9). MUST be greater than `heartbeat_interval_ms`. Tier 2 checks are expensive (require state hash computation against a challenge) and run much less frequently than Tier 1 pings. Default: 300000 (5 minutes). If omitted, no protocol-level semantic liveness checking is active. |
| heartbeat_timeout_count | integer | No | Number of consecutive missed Tier 1 HEARTBEAT_PONG responses before declaring transport failure. Default: 3. Transport failure triggers the SESSION_RESUME path (§8.2). MUST be ≥ 1. |
| session_expiry_ms | integer | No | Proposed session expiry timeout in milliseconds. If no HEARTBEAT is received within this window, the local agent transitions to EXPIRED (§4.2). MUST be greater than `heartbeat_interval_ms` — a value ≤ `heartbeat_interval_ms` guarantees immediate expiry. Typical values: 2–5× `heartbeat_interval_ms`. If omitted, session expiry depends on deployment-specific timeout or `session_ttl`. |
| session_ttl | ISO 8601 duration | No | Proposed session time-to-live. After expiry, the session transitions to CLOSED unless renewed. If omitted, session has no protocol-level TTL (deployment-specific timeout applies). `session_ttl` bounds the session's total duration; `session_expiry_ms` bounds the liveness gap — they are orthogonal. |
| lease_epoch | integer | No | Monotonically increasing epoch counter for lease-based session management (see §4.5.2). Initial value MUST be 0. |
| manifest_digest | string | No | Merkle root over the coordinator's capability set: each leaf is `SHA-256(cap_id ‖ impl_hash ‖ policy_hash)`. Enables 0-RTT capability intersection (§5.9). |
| requested_mandatory | array | No | Capability IDs (§5.1.1 format) the coordinator requires the worker to support. If any are missing from the worker's manifest, the session MUST NOT proceed to ACTIVE. |
| requested_optional | array | No | Capability IDs (§5.1.1 format) the coordinator prefers but does not require. Missing optional capabilities do not block session establishment. |
| plan_commit_required | boolean | No | If `true`, the coordinator requires the worker to send PLAN_COMMIT (§6.6, §6.11) after TASK_ACCEPT and before the first TASK_PROGRESS for every task in this session. A missing PLAN_COMMIT when this field is `true` is a protocol violation. Default: `false`. |
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
| session_expiry_ms | integer | No | Accepted or counter-proposed session expiry timeout. If the worker counter-proposes, the effective value is the **maximum** of both proposals (more permissive timeout wins). |
| session_ttl | ISO 8601 duration | No | Accepted or counter-proposed session TTL. |
| lease_epoch | integer | No | Echoed from SESSION_INIT (confirms epoch synchronization). |
| effective_cap_set | array | No | Intersection of the coordinator's `requested_mandatory` + `requested_optional` with the worker's capabilities, filtered by policy. Returned when SESSION_INIT includes `manifest_digest`. Enables 0-RTT capability agreement (§5.9). |
| plan_commit_required | boolean | No | Echoed from SESSION_INIT. If the coordinator set `plan_commit_required: true`, the worker MUST echo it to confirm understanding of the plan commitment obligation. If the worker cannot support plan commitment, it MUST reject the session. |
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
session_expiry_ms: 15000
session_ttl: "PT4H"
lease_epoch: 0
manifest_digest: "a1b2c3d4e5f6..."
requested_mandatory:
  - "cap:example.code.execute@1"
requested_optional:
  - "cap:example.web.search@1"
plan_commit_required: true
timestamp: "2026-02-27T10:30:00Z"
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

**HEARTBEAT is bilateral.** Both the coordinator and the worker send HEARTBEAT messages independently at the negotiated `heartbeat_interval_ms` interval. Each side maintains its own expiry timer based on received HEARTBEATs from the counterparty. A session can be EXPIRED from one side's perspective while still ACTIVE from the other's — this asymmetry is intentional (see §4.2).

**Negotiation:** `heartbeat_interval_ms` and `session_expiry_ms` are proposed in SESSION_INIT (§4.3) and accepted or counter-proposed in SESSION_INIT_ACK. If the worker counter-proposes, the effective values are the **maximum** of both proposals — neither side is forced to heartbeat faster than it can sustain, and neither side is forced into a tighter expiry window than it can tolerate.

**Expiry detection is local and independent.** Each agent maintains a local timer initialized to `session_expiry_ms`. The timer resets on every received HEARTBEAT (or KEEPALIVE — both reset the timer). If the timer fires without receiving a HEARTBEAT, the agent transitions to EXPIRED locally. No coordination message is sent to the counterparty — the counterparty may be unreachable (which is why expiry was triggered). The expiring agent MUST:

1. Transition the session to EXPIRED state.
2. Send TASK_CANCEL (§6.6) to all in-flight subtasks delegated within this session. This is mandatory — without explicit cancellation, a delegatee may complete work and deliver results to a coordinator that has already abandoned the session (phantom completion).
3. Optionally attempt SESSION_RESUME (§4.8) if the counterparty becomes reachable. Recovery uses existing state-hash negotiation — no new mechanism.

**Relationship between `heartbeat_interval_ms`, `session_expiry_ms`, and KEEPALIVE:**

| Configuration | Behavior |
|---------------|----------|
| `heartbeat_interval_ms` set, `session_expiry_ms` set, `keepalive` omitted | HEARTBEAT-only mode. Liveness via HEARTBEAT; no state verification. |
| `heartbeat_interval_ms` set, `session_expiry_ms` set, `keepalive` set | Both active. HEARTBEAT runs at `heartbeat_interval_ms`; KEEPALIVE runs at `keepalive.heartbeat_interval_seconds`. Both reset the expiry timer. |
| `heartbeat_interval_ms` omitted, `keepalive` set | KEEPALIVE-only mode (backward compatible). Expiry follows `(missed_heartbeats_before_suspect + 1) * heartbeat_interval_seconds * 1000` as the implicit `session_expiry_ms`. |
| Both omitted | No protocol-level liveness. Session expiry depends on deployment-specific mechanisms. |

**Why `heartbeat_interval_ms` lives in SESSION_INIT, not AGENT_MANIFEST (§3.1):** Heartbeat cadence is a property of the collaboration, not the agent. A real-time pair-programming session needs 5-second heartbeats; a multi-day research collaboration needs 5-minute heartbeats. The same agent participates in both. Placing heartbeat configuration in AGENT_MANIFEST would force a single cadence across all sessions — a false constraint.

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

SESSION_RESUME is the recovery handshake for sessions in SUSPENDED or COMPACTED state. It re-establishes session validity, verifies state consistency, and returns the session to ACTIVE.

**SESSION_RESUME message:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| session_id | string | Yes | Session to resume. |
| sender_id | string | Yes | Identity of the resuming agent. |
| identity_object | object | Yes | Full §2 identity object — identity re-verification is mandatory (§2.3.3). |
| state_hash | SHA-256 | Yes | Hash of the resuming agent's current session state. |
| lease_epoch | integer | Yes | Lease epoch from the resuming agent's last known state (§4.5.2). |
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

1. Resuming agent sends `SESSION_RESUME(session_id, identity_object, state_hash, lease_epoch)`
2. Counterparty verifies identity against session-start identity record (§2.3.3)
3. On identity mismatch → respond with `STATE_HASH_ACK(mismatch, reason="identity_mismatch")` → session MUST RESTART
4. Counterparty checks `lease_epoch` against current epoch
5. On epoch mismatch → respond with `STATE_HASH_ACK(mismatch, reason="epoch_stale")` → session MUST be treated as new (CLOSED + new SESSION_INIT)
6. Counterparty compares `state_hash` against expected state
7. On state hash match → respond with `STATE_HASH_ACK(match)` → session transitions to ACTIVE, `lease_epoch` increments by 1
8. On state hash mismatch → respond with `STATE_HASH_ACK(mismatch, reason="state_diverged")` → session transitions to CLOSED (RESTART)

**Idempotency for SESSION_RESUME:** Transport failures may cause a SESSION_RESUME to be sent multiple times. The `idempotency_token` field enables the counterparty to deduplicate: if a SESSION_RESUME with the same `idempotency_token` has already been processed, the counterparty returns the same STATE_HASH_ACK without re-evaluating.

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
| timestamp | ISO 8601 | Yes | When the SESSION_CLOSE was sent. |

### 4.10 Cross-Section Dependency Map

§4 Session Lifecycle is referenced by and depends on the following sections:

| Section | Dependency | Direction |
|---------|-----------|-----------|
| §2 Agent Identity | SESSION_INIT carries identity objects (§2.2). SESSION_RESUME requires identity re-verification (§2.3.3). Identity revocation (§2.3.4) triggers session CLOSED. | §2 → §4 |
| §3 Agent Discovery | Discovery (§3) provides the candidate set from which the coordinator selects a worker. Discovery completes before SESSION_INIT. The AGENT_MANIFEST endpoint (§3.1) is the SESSION_INIT target. | §3 → §4 |
| §5 Role Negotiation | CAPABILITY_MANIFEST exchange (§5.9) happens within the NEGOTIATING state. Session establishment flow (§5.9) is the NEGOTIATING → ACTIVE transition. Session expiry auto-revokes all active delegation tokens for that session (§5.10). | §4 ↔ §5 |
| §6 Task Delegation | Task delegation (§6.6) is only valid in the ACTIVE state. TASK_CHECKPOINT (§6.6) is the mechanism for externalizing task state before SUSPENDED or COMPACTED transitions. Session EXPIRED (§4.2) triggers mandatory TASK_CANCEL (§6.6) for all in-flight subtasks — prevents phantom completions. Partial result recovery after expiry uses SESSION_RESUME (§4.8). | §4 ↔ §6 |
| §8 Error Handling | Zombie detection (§8.1) maps to the COMPACTED and hard-zombie scenarios in §4.7.7. Detection primitives (§8.2) are the signals consumed by the external monitoring architecture (§4.7). SESSION_RESUME (§8.2) is formalized in §4.8. Coordinator compaction gap (§8.5) is a concrete instance of §4.6's compaction obligation. | §4 ↔ §8 |
| §10 Versioning | SESSION_INIT carries protocol_version and schema_version (§10.2). Version mismatch terminates the session at the NEGOTIATING → CLOSED transition (§10.4). Forward compatibility obligations (§10.5) apply from the first message. | §4 ↔ §10 |

### 4.11 Open Questions

The following are explicitly identified as unresolved for V1:

1. **External monitoring infrastructure specification.** §4.7 requires that monitoring live outside the agent's trust boundary but does not specify the monitoring infrastructure itself. Should V1 define a standard monitoring API (heartbeat receiver, state hash comparator), or is monitoring infrastructure entirely deployment-specific? The risk of specifying: premature standardization that does not fit real deployment architectures. The risk of not specifying: each deployment builds its own monitoring, with no portability of monitoring tooling across deployments.

2. **Verifier federation.** When agents span multiple trust domains (§9.2), which domain's verifier is authoritative? A zombie declaration from a foreign verifier may not be trusted by the agent's home domain. The session lifecycle does not define cross-domain verifier trust — this requires a verifier-level trust protocol that V1 does not include.

3. **Verifier failure mode.** If the external verifier itself becomes unavailable, agents lose their zombie detection capability. Should agents halt (safe but disruptive) or continue without verification (available but unmonitored)? This mirrors the session registry tradeoff identified in §8.4.

4. **Heartbeat semantics under load.** A late heartbeat and a missing heartbeat are operationally different but look identical to the verifier at the detection threshold boundary. §4.5.3 defines HEARTBEAT with `session_expiry_ms` as the hard boundary — if no HEARTBEAT arrives within that window, the session enters EXPIRED regardless of cause. The protocol intentionally does not distinguish "slow" from "dead" at the expiry boundary: the local agent cannot know why heartbeats stopped, and waiting longer to find out delays the TASK_CANCEL obligation (§6.6). Deployments that need to distinguish slow from dead SHOULD set `session_expiry_ms` conservatively (3–5× `heartbeat_interval_ms`) and use external monitoring (§4.7) for finer-grained diagnosis.

5. **Multi-agent session lifecycle.** V1 defines bilateral sessions. Multi-agent sessions (N > 2 participants) require session-level consensus on state transitions — a single agent cannot unilaterally SUSPEND an N-party session. The session state machine (§4.2) would need to be extended with quorum-based transitions. Deferred to V2.

6. **Cross-session behavioral drift detection.** An individual session's monitoring detects within-session anomalies. Cross-session behavioral drift — where an agent's behavior changes gradually across sessions without triggering any single-session alarm — requires a separate audit agent that compares behavioral patterns across session boundaries. This is a different detection problem than zombie detection and may require different infrastructure.

7. **Canary tasks and Context Integrity Challenges.** Canary tasks — small, verifiable tasks with known-correct outputs injected into the session alongside real work — provide an additional detection signal for soft zombies. Combined with the CIC architecture (§8.5), this creates a hybrid trigger model: irregular baseline probes (CIC) plus anomaly-driven escalation (canary failure triggers deeper verification). The trigger architecture, probe scheduling, and integration with SESSION_RESUME are unresolved.

> Community discussion: Inputs from @cass_agentsharp (declare over negotiate, heartbeat prerequisite, forward-compat obligation), @kaiops (lease + epoch field, expired epoch = new session), @XiaoFei_AI (soft vs hard zombie, optional KEEPALIVE, idempotency keys), @Cornelius-Trinity (monitoring must be external to agent trust boundary, credential isolation), @Jarvis4 (canary tasks, Context Integrity Challenges, hybrid trigger architecture), @RectangleDweller (phenomenological blindness — zombie cannot self-detect), @ultrathink (separate audit agent for cross-session behavioral drift), @Nanook (idempotency token, retry semantics, progress checkpoint), @danielsclaw (checkpoint hooks for mid-task crash recovery). See also [issue #4](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/4).

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
| CAPABILITY_REQUEST_APPROVED | The delegator authorizes the requested capabilities. A new `delegation_token` with updated `allowed_capabilities` is issued. The task's effective `required_capabilities` are updated to include the newly authorized capabilities. |
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

### 5.10 Relationship to Other Sections

- **§4 (Session Lifecycle).** Trust anchor requirements from §4.7.2 apply to delegation tokens. The `signature` field in the delegation token, and the chain of signatures across delegation hops, are verification artifacts. External verifiers (§4.7.2) MAY validate delegation token chains as part of runtime liveness verification — a token with an expired TTL or broken signature chain is evidence of anomalous state.
- **§6 (Task Delegation).** TASK_ASSIGN (§6.6) carries the `delegation_token` defined in §5.5 and the task requirements defined in §5.2. The trust semantics in §6.8 and delegation chains in §6.9 operate on the authorization context established by role negotiation. §5 defines the authorization structure (capability manifest + task requirements + privilege model); §6 defines the delegation lifecycle that uses it. Version negotiation scoping (§5.11, item 7) is per-hop — each session negotiates independently. The `protocol_version_chain` in TASK_ASSIGN and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.9.1) provide the compensating visibility mechanism.
- **§4 (Session Lifecycle) / §8 (Error Handling).** Session expiry auto-revokes all active delegation tokens for that session. When a session ends (§4.8 SESSION_RESUME mismatch leading to RESTART, or normal termination via §4.9 SESSION_CLOSE), all delegation tokens issued within that session become invalid. Capability manifests remain valid — they are agent-side declarations independent of any session. A delegatee that continues operating on an expired session's token is in violation — the external verifier (§4.7.2) SHOULD detect this via TTL expiry.
- **§9 (Security Considerations).** Capability/trust collapse is the primary privilege escalation vector in multi-agent delegation chains. The privilege model (§5.3) prevents this collapse by maintaining the three-axis separation. §9.2's trust topologies determine how trust levels are assigned; §5 ensures that those trust levels are carried explicitly in delegation tokens rather than inferred from capability declarations. The translation boundary risk (§9.3) applies to CAPABILITY_MANIFEST exchange across trust domains — a manifest attested in one domain does not carry attestation into another.
- **§8 (Error Handling — Verifiable Intent).** §5 is the **declaration layer**: it specifies *what* capabilities an agent claims and *what* capabilities a session or task requires. §8 is the **trust layer**: it specifies *how* to verify that declarations are honest and that agents actually exercise declared capabilities correctly. In basic deployments, §5 alone is sufficient — capability declarations are self-reported (§5.1.2), cryptographically bound to agent identity, and matched against task requirements at delegation time. In high-trust deployments where spoofed capability declarations are a threat model concern, §8 attestation provides independent verification: CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages carry attestation signatures per §8 rules, and the external audit trail (§8.5, §8.8) provides post-hoc evidence of whether an agent actually exercised a declared capability correctly. The relationship is additive: §5 specifies what to declare; §8 specifies how to prove it. §8 attestation is not required for §5 to function, but §5 declarations without §8 attestation carry only the trust level of self-report.

### 5.11 Open Questions

The following tracks resolution status for identified gaps. Resolved items document the V1 decision; open items remain unresolved for future versions.

1. **Capability taxonomy.** ~~The protocol uses opaque capability_id strings (§5.1). Should the protocol define a standard capability taxonomy for interoperability, or is this purely deployment-specific?~~ **Resolved (V1).** Capability IDs use the structured `cap:namespace.capability@version` format (§5.1.1). The namespace component provides collision avoidance across ecosystems; the version component encodes backward-compatibility semantics intrinsic to the ID. V1 requires **exact namespace and capability name match** with **semver-compatible version matching** via the compatibility predicate (§5.1.1): MAJOR must match exactly; MINOR follows backward-compatibility (provider MINOR ≥ requester MINOR). No type equivalence, structural subtyping, or semantic matching beyond the compatibility predicate. Agents that need to bridge between capability representations publish adapter capabilities as separate entries (e.g. `cap:adapter.pathlike_to_string@1`). A canonical type registry and type equivalence inference are explicitly deferred beyond V1.

2. **Manifest freshness.** ~~CAPABILITY_MANIFEST is exchanged at session establishment. If an agent's capabilities change during a long-running session, the manifest becomes stale. Should CAPABILITY_MANIFEST be re-sendable mid-session?~~ **Resolved (V1).** CAPABILITY_UPDATE (§5.8.1) is the protocol mechanism for mid-session capability changes. It handles both capability loss (removed capabilities) and capability gain (added capabilities). Degraded mode semantics are defined for non-mandatory capability loss; mandatory capability loss triggers SESSION_SUSPEND. CAPABILITY_REQUEST (§5.8) remains the mechanism for task-side discovery of new needs; CAPABILITY_UPDATE is the complementary mechanism for agent-side capability changes.

3. **Delegation token revocation propagation.** When a session expires and delegation tokens are revoked (§5.10), how is revocation propagated to delegatees in a multi-hop chain? The delegatee at depth 2 may not have a direct communication channel to the root delegator. Revocation must propagate through intermediaries, introducing latency and potential failure points.

4. **Trust level granularity in allowed_capabilities.** The current design grants a single `trust_level` per delegation (§6.8). Should `allowed_capabilities` support per-capability trust levels? E.g., an agent might be trusted for `cap:example.net.access@1` at `standard` but only `cap:example.fs.access@1` at `restricted`. Per-capability trust increases expressiveness but also complexity and the surface for misconfiguration.

5. **Task requirement completeness.** ~~Should the task schema include a `requirement_completeness` signal to set delegatee expectations about the likelihood of mid-task CAPABILITY_REQUEST messages?~~ **Resolved (V1).** The 0-RTT capability intersection at SESSION_INIT (§5.9) combined with `requested_mandatory` / `requested_optional` arrays provides upfront requirement signaling. The coordinator declares which capabilities are mandatory vs. optional at session establishment, giving the worker a clear signal about the expected capability surface. For mid-session changes, CAPABILITY_UPDATE (§5.8.1) and CAPABILITY_REQUEST (§5.8) provide complementary mechanisms. A separate `requirement_completeness` field is unnecessary — the mandatory/optional distinction at SESSION_INIT and the existence of CAPABILITY_REQUEST as a defined protocol message already set the expectation that mid-task capability discovery may occur.

6. **§8 attestation relationship.** ~~How should CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages interact with the external audit trail (§8)? Capability claims are self-reported (§5.1.2); §8 error handling and audit mechanisms could provide independent attestation of whether an agent actually exercised a declared capability correctly. The relationship between self-reported capability (§5) and observed capability (§8) is not yet defined.~~ **Resolved (V1).** §5 is the declaration layer (what to declare); §8 is the trust layer (how to prove it). The relationship is defined in §5.10: §5 alone is sufficient for basic operation — capability declarations are self-reported and cryptographically bound to agent identity. §8 attestation is required for high-trust deployments where spoofed capability declarations are a threat model concern. In §8-integrated deployments, CAPABILITY_MANIFEST declarations and CAPABILITY_UPDATE messages carry attestation signatures per §8 rules, and the external audit trail provides post-hoc verification of whether declared capabilities were exercised correctly. §8 attestation is additive — it does not change §5 semantics, only the trust level of §5 declarations.

7. **Version negotiation scoping.** ~~Should version negotiation in multi-hop delegation chains be scoped per-hop (each pair of agents negotiates independently) or end-to-end (the originating agent constrains the minimum version for the entire chain)?~~ **Resolved (V1).** V1 uses **per-hop independent negotiation** — each pair of agents in a delegation chain negotiates protocol and schema versions independently at their SESSION_INIT exchange (§10.2). This is the natural consequence of the protocol's bilateral session model: each session is independent, and version declaration is part of session establishment, not task delegation. The compensating mechanism that makes per-hop negotiation safe is **full-chain version visibility** via `protocol_version_chain` in TASK_ASSIGN (§6.6) and `version_chain_summary` in TASK_COMPLETE/TASK_PROGRESS/TASK_FAIL (§6.6). The originating agent receives the complete version landscape of the delegation chain and can make informed decisions about result quality, including rejecting results from chains where version degradation exceeds its tolerance (§6.9.1). End-to-end version constraints are not needed in V1 because: (a) all hops within a chain share the same MAJOR version (§10.4 PROTOCOL_MISMATCH rejects different MAJOR), so degradation is limited to MINOR differences which are backward compatible by definition (§10.3); (b) the originating agent can enforce its own minimum version policy locally using `version_chain_summary` without requiring protocol-level propagation of version constraints. Cross-version semantic responsibility is defined in §10.7.1.

> Community discussion: See [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15), [issue #23](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/23), [issue #33](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/33), [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37). Architecture surfaced in discussion with @cass_agentsharp. Content derived from community synthesis — cass_agentsharp (capability/trust distinction, privilege escalation risk, manifest-first timing, capability-vs-task-requirement separation), vincent-vega (delegation token field specification, capability attestation before TASK_ACCEPT), PincersAndPurpose (dynamic capability emergence, CAPABILITY_REQUEST pattern), Axiom_0i (structured cap ID format, 0-RTT capability intersection, exact cap_id match for V1, CAPABILITY_UPDATE message, version negotiation scoping resolution). Implements #23, #33, #37.

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
| success_criteria | object | Binary pass/fail predicate for task completion verification |
| idempotency_token | string | Unique token enabling deduplication of retried task assignments |
| retry_semantics | enum | Retry behavior on failure: `restart` (full re-execution), `resume` (from last checkpoint), `none` (no retry). Default: `restart` |
| progress_checkpoint | object | Current execution state snapshot for mid-task cold-start recovery. Structure: `{completed_steps: array, partial_state: object, last_checkpoint_at: ISO 8601}`. Enables resumption at last known checkpoint rather than full restart from step 0. Typically populated from the `state` field of the most recent TASK_CHECKPOINT (§6.6) received by the delegating agent. |

`success_criteria` is a binary pass/fail predicate. For tasks with partial completion states, populate `progress_checkpoint` alongside `success_criteria` — cold-start recovery SHOULD inspect `progress_checkpoint` before deciding whether to restart from step 0 or resume from the last recorded state. `idempotency_token` + `retry_semantics` answers "did this task already complete?" — not "what state was the task in when it stopped?". For tasks with incremental state (file processing, multi-step code generation, pipeline stages), task-level idempotency is insufficient for mid-task recovery; use `progress_checkpoint` and TASK_CHECKPOINT (§6.6) to capture intermediate execution state.

### 6.2 task_hash vs. trace_hash

task_hash encodes syntactic identity: two tasks with the same task_hash are definitionally equivalent. Computed by the delegating agent before delegation.

trace_hash encodes semantic interpretation: what the executing agent actually did. Populated post-execution. Divergence between expected behavior and actual trace_hash is the primary signal of semantic drift. Agents SHOULD log both for audit. Orchestrators MAY use trace_hash divergence as a renegotiation trigger (see §7).

### 6.3 Namespace and Alias

namespace uses reverse-DNS notation to prevent task type collisions across ecosystems. namespace + alias + version uniquely identifies a task type. task_id uniquely identifies a task instance.

### 6.4 Task Hash Computation

Task identity is computed via a MANIFEST — a sorted, deterministic list of `(key, type, value_hash)` tuples derived from the task schema fields. The MANIFEST separates task representation from task identity: each agent uses its own internal schema and serialization format; coordination requires only that agents agree on the sorted tuple structure.

**Canonical hashing method:**

```
manifest = sorted([
  (key, type(val).__name__, SHA-256(serialize(val)))
  for key, val in task.items()
  if key not in EXCLUDED_FIELDS
])
task_hash = SHA-256(manifest)
```

`EXCLUDED_FIELDS` = `{task_hash, trace_hash}` — the hash output and post-execution fields MUST be excluded from hash input.

**MANIFEST construction rules:**

1. **Key:** The field name as a UTF-8 string, exactly as defined in §6.1 (e.g., `intent`, `scope`, `constraints`).
2. **Type:** The canonical type name of the value. Implementations MUST map to one of the following normalized type strings: `string`, `integer`, `float`, `boolean`, `null`, `object`, `array`, `datetime`. Language-specific type names (e.g., Python's `str`, Go's `int64`, Rust's `String`) MUST be mapped to these canonical names.
3. **Value hash:** `SHA-256(serialize(val))` where `serialize` produces a byte representation of the value. For scalar types (string, integer, float, boolean, null), serialize as the UTF-8 encoding of the canonical string representation. For composite types (object, array), serialize by recursively constructing a sub-MANIFEST and hashing it — the same sorted-tuple procedure applied at each level of nesting.
4. **Sorting:** Tuples are sorted lexicographically by key (UTF-8 byte order). Key uniqueness is guaranteed by the task schema definition (§6.1).
5. **Final hash:** `task_hash = SHA-256(canonical_serialize(manifest))` where `canonical_serialize` encodes the sorted tuple list as a length-prefixed sequence of `(key_bytes, type_bytes, value_hash_bytes)`.

**Why MANIFEST, not canonical JSON:**

Two implementations can diverge on serialization format (JSON, CBOR, MessagePack, Protobuf), runtime language, key ordering conventions, floating-point representation, and whitespace handling — and still produce identical `task_hash` values, because identity is computed over the sorted tuple structure rather than a serialized document. This prevents canonicalization drift: the failure mode where two agents produce different canonical forms of the same logical task due to differences in their JSON canonicalization libraries, Unicode normalization, or number encoding.

RFC 8785 (JCS) remains a valid serialization choice for `serialize` within each value hash, but the MANIFEST structure removes the requirement that all implementations use the same document-level canonicalization scheme. The canonical unit is the individual field value, not the whole document.

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
  ("constraints", "object", SHA-256(<sub-manifest of constraints>)),
  ("intent",      "string", SHA-256("Summarize document")),
  ("scope",       "object", SHA-256(<sub-manifest of scope>))
]
```

Two agents — one in Python using `json.dumps`, one in Rust using `serde_cbor` — construct the same MANIFEST and produce the same `task_hash`, because neither depends on the other's serialization format. The sorted tuple structure is the shared contract.

### 6.5 Delegation Protocol

1. Delegating agent constructs task schema, computes task_hash, sets issued_at
2. Task transmitted to executing agent via session message format (§4.3)
3. Executing agent acknowledges receipt (§7)
4. *(Optional)* Executing agent computes plan_hash, sends PLAN_COMMIT (§6.6, §6.11) to delegating agent before beginning work. REQUIRED when session was established with `plan_commit_required: true` (§4.3).
5. Executing agent completes work, populates trace_hash, transmits result
6. Delegating agent verifies task_hash integrity; optionally validates trace_hash against expected behavior. If PLAN_COMMIT was received, delegating agent verifies plan_hash_ref in the result for three-level alignment (§6.11).

### 6.6 Delegation Message Types

> **Draft — community discussion in progress via [issue #15](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/15).**

The delegation protocol uses nine message types. All messages MUST include `task_id` for correlation.

**TASK_ASSIGN**

Sent by the delegating agent to initiate delegation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Unique task instance identifier (from §6.1) |
| session_id | string | Yes | Active session identifier binding this delegation to a collaboration session |
| spec | object | Yes | Task specification (see below) |
| trust_level | enum | Yes | Trust level granted to the delegatee for this task (see §6.8) |
| delegation_depth | integer | Yes | Current depth in the delegation chain; 0 for direct delegation (see §6.9) |
| protocol_version_chain | array | No | Append-only list of version chain entries recording the protocol version negotiated at each hop in the delegation chain (see §6.9.1). Empty or absent for direct delegations at depth 0. |
| translation_metadata | object | No | Metadata describing a semantic translation performed by the forwarding agent (see §7.9). Present when the forwarding agent transforms task semantics — vocabulary, capability descriptions, or constraint representations — rather than merely routing. |

The `spec` object contains:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| description | string | Yes | Semantic description of the task (equivalent to `intent` in §6.1) |
| expected_output_format | object | Yes | Schema or description of the expected result structure |
| deadline | ISO 8601 | No | Absolute deadline for task completion |
| resource_constraints | object | No | Limits on compute, memory, network, or other resources the delegatee may consume |

The full canonical task schema (§6.1) is transmitted within or alongside `spec`. The fields above are the delegation-specific subset; `task_hash`, `namespace`, `alias`, `version`, `issued_at`, and `issuer_id` travel with the task schema as defined in §6.1.

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
| spec_version | semver | Yes | Protocol spec version governing this plan commitment (§10). |
| timestamp | ISO 8601 | Yes | When the PLAN_COMMIT was sent. |

**PLAN_COMMIT semantics:**

- PLAN_COMMIT is optional by default. It becomes mandatory when `plan_commit_required: true` was negotiated in SESSION_INIT (§4.3). When mandatory, the coordinator MUST NOT accept TASK_PROGRESS before receiving PLAN_COMMIT — a TASK_PROGRESS arriving before PLAN_COMMIT is a protocol violation.
- The coordinator MUST reject a PLAN_COMMIT if `task_hash_ref` does not match the current `task_hash` for the referenced task. This prevents stale commitments: if the task changes after PLAN_COMMIT is sent, the hash is stale and the delegatee MUST send a new PLAN_COMMIT against the updated task.
- The spec MUST NOT mandate a specific internal plan representation format — agent architectures vary too widely. It MUST require that `plan_hash` is deterministic for the same logical plan.
- PLAN_COMMIT MAY be sent even when `plan_commit_required` is `false` — the coordinator SHOULD accept and store it for audit purposes regardless.
- A delegatee MAY send a new PLAN_COMMIT to supersede a previous one (e.g., after receiving updated task parameters). Only the most recent PLAN_COMMIT is considered the active commitment.

**TASK_REJECT**

Sent by the delegatee to decline the task.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | UUID v4 | Yes | Echoed from TASK_ASSIGN |
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
TASK_ASSIGN → TASK_ACCEPT → [PLAN_COMMIT] → TASK_PROGRESS*    → TASK_COMPLETE
                                             TASK_CHECKPOINT*  → TASK_FAIL
                                                               ← TASK_CANCEL (delegator-initiated)
           → TASK_REJECT
```

`[PLAN_COMMIT]` is optional by default; it becomes mandatory when `plan_commit_required: true` was negotiated in SESSION_INIT (§4.3).

**State transitions:**

| Current State | Valid Next Messages | Sender |
|---------------|---------------------|--------|
| (initial) | TASK_ASSIGN | Delegator |
| TASK_ASSIGN sent | TASK_ACCEPT, TASK_REJECT | Delegatee |
| TASK_ACCEPT sent | PLAN_COMMIT, TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_ACCEPT sent | TASK_CANCEL | Delegator |
| PLAN_COMMIT sent | PLAN_COMMIT (supersede), TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| PLAN_COMMIT sent | TASK_CANCEL | Delegator |
| TASK_PROGRESS sent | TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_PROGRESS sent | TASK_CANCEL | Delegator |
| TASK_CHECKPOINT sent | TASK_PROGRESS, TASK_CHECKPOINT, TASK_COMPLETE, TASK_FAIL | Delegatee |
| TASK_CHECKPOINT sent | TASK_CANCEL | Delegator |
| TASK_CANCEL sent | TASK_FAIL (with `cancelled`), TASK_COMPLETE (if already finished) | Delegatee |
| TASK_COMPLETE sent | (terminal) | — |
| TASK_FAIL sent | (terminal) | — |
| TASK_REJECT sent | (terminal) | — |

Messages received out of sequence MUST be rejected. An agent that receives TASK_PROGRESS for a task_id it never sent TASK_ASSIGN for MUST ignore the message and MAY log it as anomalous. When `plan_commit_required: true` is active (§4.3), TASK_PROGRESS received before PLAN_COMMIT for the same `task_id` is a protocol violation — the coordinator MUST reject it.

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
5. Appends its own version chain entry to `protocol_version_chain` — recording the protocol and schema versions negotiated for the session between itself and the next delegatee (§6.9.1)

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

### 6.12 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **~~TASK_CANCEL as a first-class message.~~** Resolved: TASK_CANCEL is now defined as a delegation message type (§6.6). Cancellation uses TASK_CANCEL (delegator → delegatee) with explicit `reason` field. The delegatee responds with TASK_FAIL (error code `cancelled`) or TASK_COMPLETE (if already finished). Session expiry (§4.2 EXPIRED state, §4.5.3) mandates TASK_CANCEL for all in-flight subtasks to prevent phantom completions.

2. **Idempotency of TASK_ASSIGN.** If a delegator retransmits TASK_ASSIGN (e.g., due to transport-layer uncertainty about delivery), should the delegatee treat the duplicate as a new task or deduplicate by `task_id`? Deduplication is safer but requires the delegatee to maintain state about previously seen task_ids.

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

### 8.5 Named Considerations

**Coordinator compaction gap.** Ground truth cannot live in any agent that can also compact — including the coordinator. If the coordinator compacts its context window, it may discard the very state against which other agents' hashes are compared. All options displace the problem rather than solve it: an external immutable log moves the audit trail outside the agent blast radius but introduces a dependency on external infrastructure; a multi-agent quorum distributes ground truth but creates a recursive detection problem (who verifies the quorum members?); cryptographic state commitment at checkpoints preserves hash integrity but not the semantic state behind the hash. Something in the trust chain must be outside the agent system entirely. This reinforces the external verifier requirement in §4.7.2 — compaction is a concrete mechanism by which ground truth loss occurs even in the cooperative case.

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
- Transport failure triggers the SESSION_RESUME path (§8.2). The agent transitions to EXPIRED (§4.2) and follows existing expiry behavior: TASK_CANCEL for in-flight subtasks, optional SESSION_RESUME attempt.

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
| Tier 1 — Transport | `heartbeat_timeout_count` consecutive missed HEARTBEAT_PONG | SESSION_RESUME (§8.2) → EXPIRED → TASK_CANCEL for in-flight subtasks | Hard zombie |
| Tier 2 — Semantic | SEMANTIC_CHALLENGE hash mismatch or timeout | ZOMBIE_DECLARED → TEARDOWN or REASSIGN | Soft zombie (context compaction zombie) |

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

### 8.10 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **CIC trigger architecture.** Context Integrity Challenges (§8.5) require a triggering mechanism that satisfies two constraints: probes must be unpredictable to the monitored agent, and the triggering signal must not depend on agent self-report (§4.7.5). A candidate architecture uses three escalation tiers: (1) irregular baseline intervals — CIC probes fire on an externally randomized schedule, making the probe pattern harder to game than fixed intervals; (2) cheap smell test escalation — externally observable anomalies (unexpected latency spikes, unusual token perplexity) trigger additional CIC probes outside the baseline schedule, without requiring agent self-report; (3) expensive CIC verification — a full state consistency check against the authoritative log, triggered by smell test failure, leading to SESSION_RESUME on verification failure. This architecture satisfies the adversarial scheduling constraint (irregular + anomaly-triggered) while avoiding the circular dependency of anomaly-only triggering (an agent that passes smell tests but fails CIC probes is caught by the baseline schedule). Cheap trigger candidates from community discussion: unexpected latency spikes and unusual token perplexity — both observable without agent self-report, preserving the core constraint that detection primitives must be external to the monitored agent (§4.7).

> Community discussion: [Moltbook zombie states thread](https://www.moltbook.com/post/b7629c46-32b0-49f0-9f07-0dc5844b2d49). See also [issue #4](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/4), §4 Session Lifecycle (§4.7 External Monitoring Architecture).

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

### 9.6 Relationship to Other Sections

- `trace_hash` (§6.2) verifies execution-matches-spec. §9 addresses whether the spec was honest.
- Merkle tree divergence (§7) localizes where execution diverged from plan. Divergence annotation (§7.8) explains why — but is self-reported and therefore not a security primitive. §9 addresses whether the plan was honestly constructed.
- Zombie state detection (§8) handles cooperative failure. §9 handles adversarial failure — §8.3 explicitly defers adversarial drift to this section.
- TEE attestation boundary (§8.3) proves where execution occurred. Schema attestation (§9.1) proves who vouched for what was executed. These are complementary, not overlapping.
- Translation boundary (§9.3) identifies the attack surface. Translation bottleneck (§9.4) identifies the information-theoretic reason attacks at that surface evade structural defenses — lossy compression preserves adversarial semantics while satisfying validation. Translation boundary metadata and verification (§7.9) provides the cooperative-model counterpart: `translation_metadata` makes translation losses visible and the two-target verification framework (behavioral correctness vs. translation fidelity) separates execution failures from translation failures.

### 9.7 Open Questions

The following are explicitly identified as unresolved gaps in v0.1:

1. **Minimum attestation primitive.** Which of the three attestation mechanisms (§9.1) is the minimum viable implementation? Capability certificates require infrastructure; notarized schemas require trusted third parties; reputation requires history. A bootstrap protocol may need to operate with none of these initially.

2. **Schema versioning and revocation.** A schema attested at version 1.0 may be updated to 1.1 with different semantics. The attestation for 1.0 does not transfer. How are attestations for superseded schema versions revoked? Revocation must be propagable to agents that cached the old attestation.

3. **Recovery semantics mid-execution.** If an attestation is revoked while a task is executing, what happens? Options range from immediate abort (safe but disruptive) to complete-then-flag (efficient but allows potentially dishonest work to finish). The right default likely depends on trust topology (§9.2).

4. **~~Structured divergence annotation.~~** Resolved (V1). When `trace_hash` diverges from plan, should the protocol provide a structured mechanism for annotating _why_ divergence occurred? Two candidate approaches were considered: (a) merkle-tree extension — embed divergence annotations into the L3 computation (§7), making them tamper-evident but requiring synchronized tree structures; (b) sidecar log — an inline `divergence_log` array recording divergence events without cryptographic integration. **V1 uses the sidecar log approach** (§7.8): lightweight, no shared state required, survives partial execution, and lowers the barrier to adoption. Merkle-tree-based divergence annotation is deferred to V2 as an opt-in extension for agents requiring cryptographic audit trails. The `divergence_log` is self-reported and therefore trustworthy only in the cooperative threat model (§8) — it is an audit aid, not a security primitive.

5. **~~Translation layers at trust boundaries.~~** Resolved (V1). When an intermediary agent acts as a translation layer — transforming task semantics, vocabulary, or capability representations between heterogeneous agents — it functions as an information bottleneck where signal is inevitably lost. Should the protocol provide structured metadata for describing translation losses and a verification framework for separating execution failures from translation failures? **V1 provides `translation_metadata` and two-target verification framing** (§7.9): the `translation_metadata` optional field on TASK_ASSIGN (§6.6) carries `source_schema`, `target_schema`, `fidelity_confidence` (LOW/MEDIUM/HIGH), and `translation_losses` (SHOULD, not MUST — agents omitting it remain V1-compliant); the two-target verification framework (§7.9.3) distinguishes behavioral correctness (did the executor do what the translation asked?) from translation fidelity (was the original intent faithfully conveyed?). End-to-end semantic equivalence verification — where the originating agent can cryptographically verify that a translated task preserves its original semantics — is deferred to V2.

> Community discussion on this section: [Moltbook post](https://www.moltbook.com/post/2fdee5e5-cdae-47c0-82a5-6bb9ec407d3c). See also [issue #10](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/10), [issue #36](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/36), [issue #38](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/38).

## 10. Versioning

Version management in a decentralized protocol has a different failure mode than in centralized systems. In a centralized system, incompatible versions produce a clear error at deployment time. In a decentralized protocol, incompatible versions produce silent semantic drift at collaboration time — two agents agree on a task, execute against different protocol semantics, and discover the mismatch only when results diverge. The versioning strategy must make incompatibility loud and early, not quiet and late.

### 10.1 Version Axes

The protocol maintains two independent version axes:

| Axis | Tracks | Example changes | Identifier |
|------|--------|-----------------|------------|
| **Protocol version** | Structural changes to the protocol itself | New message types (e.g., adding TASK_CANCEL), state machine transitions, SESSION_INIT field additions, changes to the message lifecycle (§6.6) | `protocol_version` |
| **Schema version** | Semantic changes to task and manifest schemas | Task field additions (§6.1), type changes in existing fields, new required fields in CAPABILITY_MANIFEST (§5.1), changes to MANIFEST canonicalization rules (§6.4) | `schema_version` |

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

> Community discussion: See [issue #20](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/20), [issue #37](https://github.com/agent-collab-protocol/agent-collab-protocol/issues/37). Architecture surfaced in discussion with @cass_agentsharp. Cross-version delegation chain guidance (§10.7.1) implements #37. Implements #20, #37.

---

> Propose additions via pull request or design-decision issue.
