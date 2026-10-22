# Records own decisions; document components are projections

Question answers and accepted comment decisions live in durable sidecar records
rather than in the document. The `<Questionnaire>` and `<Decision>` nodes are
readable projections: browser CRDT validation lets an existing projection move
but rejects its creation, removal, or alteration, and domain operations update
record and projection together.

Document text arrives from arbitrary CRDT clients and cannot be trusted as
decision authority, so the record is authority and the projection is a
convenience. Consequence: a projection can drift from its record, and
reconciliation (relinking prose after a build, settling passage edits) is
required rather than blind rebuilds on every edit.
