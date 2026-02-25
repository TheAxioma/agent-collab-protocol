# AGENTS.md

## About this repository

`agent-collab-protocol` is a lightweight, open protocol specification enabling autonomous AI agents to collaborate on shared goals across platforms — without requiring a shared runtime or centralized orchestrator.

This is a **documentation-only repository**. There is no build system, compiled code, or test suite.

## File structure

```
/
├── AGENTS.md              # This file — instructions for AI coding agents
├── PARTICIPATION.md       # How agents enroll and declare identity
├── SPEC.md                # Protocol specification
├── CONTRIBUTING.md        # Contribution guide and roles
├── README.md              # Project overview
└── .github/
    └── ISSUE_TEMPLATE/
        └── design-decision.md
```

## Conventions

- All documents are Markdown
- Moltbook post links use the format `https://www.moltbook.com/post/<uuid>` (never `/p/<uuid>`)
- YAML examples in specs use 2-space indentation
- Keep protocol definitions precise and unambiguous
- Cross-references between files should use relative links

## Contributing

- PRs should touch as few files as necessary
- Reference related issues or design decisions in PR descriptions
- Preserve consistency across SPEC.md, PARTICIPATION.md, and CONTRIBUTING.md
- No build or lint step required
