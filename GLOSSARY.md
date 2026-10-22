# Chopin

A collaborative authoring system where people and a hosted agent co-author
repository-connected documents. One shared domain context across the monorepo.

## Language

### Authoring

**Document**:
The collaboratively authored, repository-connected artifact.
_Avoid_: plan (as the product noun), file, page

**Plan**:
A document being used for planning. Not the product noun for every document;
keep it for planning-specific workflows and literal implementation names.

**Dialect**:
The restricted MDX allowlist every persisted document state must satisfy.

**Record**:
The durable authority for one decision or comment, held outside the document.
_Avoid_: treating its document projection as authority

**Projection**:
A document's readable rendering of a durable record. It may move but is never
independently created, removed, or altered.

### Collaboration

**Channel**:
The durable collaboration container for one document, its Chat, decisions, and
repository identity.

**Room**:
The server's live in-memory representation of an open channel.

**Chat**:
A channel's durable message transcript.
_Avoid_: Conversation (as the surface's name)

**Epoch**:
The identity of one collaborative document history. A fresh epoch makes clients
discard incompatible updates.

### Agents

**Planner**:
Chopin's hosted document agent.
_Avoid_: Copilot agent, assistant

**Coding agent**:
An external MCP client that creates, revises, or implements a document from its
own local workspace.
_Avoid_: external agent, MCP agent

### Decisions and research

**Decision**:
A question raised from conversation with recorded answers or comment decisions.

**Research request**:
Durable work attached to one parent document; while pending it is neither a
channel, a Chat, nor a route.

**Child document**:
An ordinary document attached to one parent for navigation and context. It has
no Chat of its own.

**Anchor**:
A record's reference to a document block that survives surrounding edits;
ambiguous or missing matches are orphaned rather than guessed.

### Implementation

**Implementation graph**:
A versioned task graph bound to one document revision; a person's press of
Build is the approval.
