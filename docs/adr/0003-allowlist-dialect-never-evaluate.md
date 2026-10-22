# The document dialect is an allowlist; content is never evaluated

Document content is untrusted input rendered by browsers and projected by the
server. The MDX dialect accepts only the registered allowlist: imports, exports,
expressions, raw HTML, and unknown JSX are rejected, and accepted content is
parsed and rendered, never evaluated.

We gave up full MDX expressiveness so that every accepted document state can
round-trip through the server's headless Lexical mirror and remain safe for
every reader. A plausible alternative — full MDX with evaluation or heavy
sandboxing — was rejected because documents are authored by many repository
members and persisted for all of them.
