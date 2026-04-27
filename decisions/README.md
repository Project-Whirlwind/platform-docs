# Architecture Decision Records

An ADR captures a significant decision: what was decided, the context that forced the decision, and the consequences — good and bad.

ADRs are **immutable once accepted**. If a decision is revisited, a new ADR supersedes the old one. The old one stays so we know why we changed course.

---

## Format

```markdown
# ADR-NNN: Title

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context
What situation, constraint, or problem forced this decision?
What options were considered?

## Decision
What did we decide?

## Consequences
What becomes easier? What becomes harder? What are we accepting as a tradeoff?
```

## Status definitions

- **Proposed** — under discussion, not yet binding
- **Accepted** — binding for new work
- **Deprecated** — no longer recommended but not actively reversed
- **Superseded** — replaced by a newer ADR (link to it)

## When to write an ADR

Write one when:
- You're choosing between two viable approaches
- The decision will be hard to reverse
- You expect future engineers to ask "why did we do it this way"
- You're introducing a new technology or dependency
