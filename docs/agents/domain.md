# Domain docs

This repository uses a **single shared domain context**, even though its runtime
is a monorepo. The apps and packages collaborate on the same document-authoring
product.

## Read before exploring

- Root `GLOSSARY.md`, when it exists, for shared domain vocabulary.
- Relevant records in `docs/adr/`, when that directory exists, for decisions.
- Existing [architecture](../architecture.md), [authentication](../authentication.md),
  and [storage](../storage.md) documentation for the affected area.

`GLOSSARY.md` and `docs/adr/` are created lazily by domain-modeling work, when terms
or decisions are actually agreed. If absent, proceed silently; do not create empty
placeholders or invent decisions just to populate the layout. Keep existing
product and architecture documentation intact.

Use glossary vocabulary in issues, specifications, code, and review. Surface
conflicts with existing ADRs explicitly rather than silently overriding them.
Keep the root glossary and ADR directory shared until the user chooses a different
layout.
