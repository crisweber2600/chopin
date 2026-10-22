# PostgreSQL is the only runtime storage adapter

Chopin's durable boundary is the `StorageAdapter` port, and PostgreSQL is the
only selectable runtime adapter; `MemoryStorage` exists solely for unit and
contract tests. The adapter guarantees collaboration depends on — revision-
checked commits, renewable leases with fencing, ordered update replay — need
real transactional semantics, and a halfway adapter risks restoring document and
decision state no connected client was ever shown.

Consequence: every adapter guarantee is proven against real PostgreSQL
(`bun run test:postgres`), and every deployment runs PostgreSQL rather than an
embedded store.
