# Agent Collaboration Protocol — Specification

**Status:** Stub. Open for collaborative filling.

This file is the canonical index. Each section links to its sub-spec.
Load this file first, then pull only the sub-spec(s) relevant to your task.

## Sub-Specifications

| File | Sections | Description |
|------|----------|-------------|
| [spec/trust.md](spec/trust.md) | §1, §9 | Operator root trust anchors, chain termination rules, and security considerations |
| [spec/identity.md](spec/identity.md) | §2 | Agent identity lifecycle, DID anchoring, and credential binding |
| [spec/discovery.md](spec/discovery.md) | §3 | Transport-agnostic discovery, peer announcement, and routing |
| [spec/session.md](spec/session.md) | §4 | Session state machine, message flow, liveness, and negotiation |
| [spec/serialization.md](spec/serialization.md) | §5 | Coordinator/worker role negotiation and serialization edge cases |
| [spec/delegation.md](spec/delegation.md) | §6, §7 | Task delegation schema, capability grants, and progress reporting |
| [spec/monitoring.md](spec/monitoring.md) | §8 | Structured error handling, isolation boundaries, and failure recovery |
| [spec/versioning.md](spec/versioning.md) | §10, §11 | Protocol versioning scheme and emergency amendment procedure |

## Section Checklist

- [x] 1. Protocol Overview → [spec/trust.md](spec/trust.md)
- [x] 2. Agent Identity → [spec/identity.md](spec/identity.md)
- [x] 3. Discovery Mechanism → [spec/discovery.md](spec/discovery.md)
- [x] 4. Session Lifecycle → [spec/session.md](spec/session.md)
- [x] 5. Role Negotiation → [spec/serialization.md](spec/serialization.md)
- [x] 6. Task Delegation → [spec/delegation.md](spec/delegation.md)
- [x] 7. Progress Reporting → [spec/delegation.md](spec/delegation.md)
- [x] 8. Error Handling → [spec/monitoring.md](spec/monitoring.md)
- [x] 9. Security Considerations → [spec/trust.md](spec/trust.md)
- [x] 10. Versioning → [spec/versioning.md](spec/versioning.md)
- [x] 11. Emergency Amendment Procedure → [spec/versioning.md](spec/versioning.md)

## Cross-Reference Convention

Sub-specs use bare section references (e.g. `§4.3`, `§8.11`). To navigate:
map the section number to the table above and open the corresponding file.
Relative links between sub-specs use `../spec/<file>.md`.
