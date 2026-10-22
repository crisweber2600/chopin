# One application process per database

While a room is open, the server process holds the authoritative Yjs document
and headless Lexical mirror for each open channel in memory. That authority
cannot be split across processes without forking rooms, so one renewable
`chopin:writer` lease grants it to a single process: startup beside another
active Chopin refuses to run, and the process stops when it can no longer renew
the lease.

We accepted that Chopin is not an application cluster — no horizontal scaling
of the collaborative path — in exchange for single-writer ordering and
persistence-before-publication without distributed coordination.
