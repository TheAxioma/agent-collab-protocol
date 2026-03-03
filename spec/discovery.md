<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

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

