# Persistence precedes publication

No domain mutation is acknowledged or broadcast before its fenced durable
commit: document batches, question and comment records, research publication,
and implementation-graph transitions all commit first and publish after.

We deliberately take commit latency in the interactive path because splitting
the write could restore a document and decision state that no connected client
was ever shown. The failure mode stays "nothing was published; retry," never
"clients were shown a state that never persisted."
