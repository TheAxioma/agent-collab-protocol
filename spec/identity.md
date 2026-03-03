<!-- Part of the Agent Collaboration Protocol spec. Index: [SPEC.md](../SPEC.md) -->

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

4. **Trust anchoring.** Who attests that an identity is legitimate? The protocol defines identity publication (§2.3.1) and verification (§2.3.2) but does not specify a trust anchor — the entity that vouches for the binding between an agent and its identity. Candidate trust anchors: platform registries, certificate authorities, web-of-trust among agents, reputation systems (connects to §9.1 schema attestation). V1 requires an operator-configured root trust anchor (§1.2) for capability grant chain termination. Identity-level trust anchoring — the distinct question of who vouches that an agent's identity claim is legitimate — remains deferred to deployment configuration.

