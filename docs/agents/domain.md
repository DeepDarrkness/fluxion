# Domain Docs

This repository uses a single-context domain-documentation layout.

## Before Exploring

Read these when they exist and are relevant:

- `CONTEXT.md` at the repository root.
- ADRs under `docs/adr/` that touch the area being changed.

If these files do not exist, proceed silently. Domain-modeling creates them lazily when terminology or consequential decisions are resolved.

## File Structure

```text
/
|-- CONTEXT.md
|-- docs/
|   `-- adr/
|       |-- 0001-example-decision.md
|       `-- 0002-another-decision.md
`-- src/
```

## Vocabulary

Use canonical terms from `CONTEXT.md` in issues, specifications, code, tests, and documentation. Do not introduce synonyms for terms the glossary explicitly distinguishes or avoids.

If a required concept is missing, reconsider whether it is project vocabulary or identify it for domain-modeling.

## ADR Conflicts

Surface any contradiction with an existing ADR explicitly rather than silently overriding it. A superseding decision should identify the ADR it replaces.
