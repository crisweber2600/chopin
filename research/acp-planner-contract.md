# ACP contract for the Planner, workers, and coding agents

## Finding and scope

**Standard ACP can carry Planner turns and coding runs; today's Chopin cannot do both with isolated workers unchanged.** This is architectural feasibility, not tested interoperability. Either a direct ACP `HarnessV1` adapter or an ACP bridge can satisfy the Planner requirement, but only with host-authoritative tools, explicit execution policies, structured-result validation, and a worker execution path. Keeping only Chopin's custom WebSocket or disabling research does not satisfy the destination. The requirements preserve shared documents, Chat, Decisions, research, a native Hermes Desktop plugin, Hermes by default, and selectable compatible agents; Azure writes require explicit authorization, with no automatic merge/deploy. [R]

Evidence checked **October 22, 2026**:

- Chopin: `47138debe85b3263323077fa172430bb74e59b01`; all repository paths/lines below refer to this revision.
- Installed Hermes source inspected: `d923a5eb487a17c0a68801bba9ef0f56c81040a8`.
- Public Hermes `main` checked separately: `5043721040a4b97269df0775b226910f2cd0bb54` (October 22). It has a materially different production ACP path; the installed checkout is not evidence of latest behavior. [H1]
- ACP source: `87da85c4634a1f9b519e760042886316b82a0a23`; Gemini CLI: `a713410aba5d594594e20d5c822e3aab8f822089` (both October 22). [A1][G1]

**Fact** means source/documentation inspected; **proposal/inference** means a possible target contract; **untested** means no executable compatibility result. No agents, services, credential stores, or authentication changes were exercised. Dependency-only documentation checks are recorded below.

## 1. The three protocols are not interchangeable

**Facts.** Chopin's harness registry contains only `copilot-sdk`, `atomic`, and `remote` (`apps/server/src/harness/harnesses.ts:34–46`). The remote protocol declares `chopin.agent.v1`, `/api/v1/agent`, custom `turn`, stream-part, and `host-tool-call` messages; it is not ACP (`packages/agent-link/src/protocol.ts:7–16,88–131,231–260`; `apps/server/src/harness/remote/link.ts:513–529`). Conversely, `apps/connector/src/acp.ts:121–126,170–254` already launches coding-agent subprocesses, negotiates ACP, creates sessions with MCP servers, forwards updates/permissions, and sends cancellation. It is an ACP **client**, not a Planner harness.

ACP controls conversations; MCP exposes executable tools. An ACP tool notification describes agent activity, not an instruction for Chopin to execute it. Host functions therefore need a scoped MCP surface or a deliberately negotiated extension, not execution triggered by display events. Standard stdio ACP uses newline-delimited JSON-RPC; a custom transport must retain ACP messages and lifecycle. A proprietary WebSocket with unrelated messages does not become ACP by renaming it. [A2][A3]

**Version trap.** The current ACP repository contains current/stable v1 and draft v2 schemas. In v1, `session/prompt` completes with `stopReason`; draft v2 instead acknowledges insertion with `messageId`, and foreground completion arrives through `state_update`. The v2 comparison is prospective, not an additional approved implementation target. Never finish a turn merely because the prompt RPC returns without checking the negotiated version. Hermes and Gemini inspected sources use the v1-shaped `agentCapabilities` handshake; no v2 interoperability is established. [A1][A4] [H2][G2]

## 2. Direct harness versus standard-ACP bridge

| Route  | Proposed boundary                                                                                                                         | Consequences                                                                                                                                                                                                                                            |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Direct | Shared backend's `HarnessV1` adapter → standard ACP → selected agent; agent → scoped MCP → host tools                                     | One adapter can serve Planner and disposable workers. Must implement lifecycle/event translation, instruction setup, result collection, policy checks, cancellation, and cleanup. Model credentials/execution must have an explicit per-owner boundary. |
| Bridge | Existing remote harness → `chopin.agent.v1` → owner-side bridge → standard ACP → selected agent; scoped MCP forwards permitted host calls | Preserves the existing host/owner separation and can place execution near Desktop. Adds translation and failure boundaries. It remains genuine Planner ACP only when every Planner model turn actually crosses the ACP leg.                             |

These are **design alternatives**, not existing features. Direct mapping starts from `HarnessV1` and current agent declarations; bridging starts from the remote turn payload, including instructions, tool schemas, model, and `responseFormat` (`apps/server/src/harness/agents.ts:89–122,168–198`; `apps/server/src/harness/remote/link.ts:513–535`). Neither standard ACP v1 `PromptRequest` nor `NewSessionRequest` supplies a portable system-prompt or JSON-schema-output field. Putting instructions in user text is not equivalent to installing trusted system instructions. Dedicated agent launch/configuration or a negotiated extension needs explicit validation; it cannot be advertised as baseline ACP portability. [A1]

**Worker blocker, not a flag fix.** `apps/server/src/config.ts:291–296` refuses remote mode with background jobs, and `apps/server/src/harness/remote/adapter.ts:13–22` resolves only registered document Planner sessions. Deleting that guard would not provide worker execution. A bridge route needs either a separate ACP worker harness selected independently of Planner transport, or an extended owner-side worker broker supporting job claims and independently isolated sessions. The direct route also needs worker policy provisioning. Both must retain research rather than hide the gap with `BACKGROUND_JOBS=off`.

## 3. Capability and authority contract

**Proposal:** qualify agents by role, not a single “supports ACP” flag. Record executable/version, negotiated protocol, content/MCP transports, load support, model selection, policy identity, and result mechanism. Missing capabilities are unsupported. Use MCP stdio as the v1 baseline; use HTTP only when advertised. Do not advertise client filesystem/terminal capabilities unless implemented and authorized. [A5][A6]

This alone is **not a sandbox**. ACP permission requests are optional agent requests, not a mandatory interception point for every operation. Denying client filesystem capabilities does not remove the agent's own shell/filesystem. Neither an empty working directory nor a prompt prohibition proves isolation. [A3][G4]

The host should bind every tool authorization to document/repository identity, owner generation, credential revision, active turn/job and allowed operation. Recheck membership/role, revocation, expiry, and explicit write approval at execution, not just session attachment. Existing `ActiveOwnerBindings` already checks/revokes these bindings (`apps/server/src/agent/active-owner.ts:52–98,140–213`); port its semantics to Azure instead of forwarding a broad Azure token into model prompts. Use short-lived, least-authority MCP grants; keep model-provider credentials distinct from repository/work-item credentials. This is a **proposed preservation** of the current boundary, not an Azure implementation.

Permission UI should offer only appropriate request options, correlate consent to the owner and exact action, deny unattended escalation, and cancel pending requests when execution is cancelled. Approval of a tool request must not stand in for human Decisions or graph approval. Server-side validation, record authority, and persistence-before-publication remain independent of ACP (`AGENTS.md:142–196,304–330`). [A3][A7]

## 4. Hermes default: verified support and limitations

**Facts.** Official Hermes documentation describes `hermes acp` as a stdio server, inheriting Hermes provider configuration and a broad `hermes-acp` toolset: filesystem, terminal, web/browser, memory, skills, code execution, and delegation. It documents per-platform tool selection and separately added editor MCP servers. A default personal Hermes ACP session is therefore not a restricted Chopin Planner or private worker. [H3]

Rechecking public source changes the preliminary picture: production `HermesACPAgent()` now returns `GatewayACPAgent`; the stdio process is a viewer of gateway-owned execution. `new_session` checks `acp-editor-policy-v1` and creates an ACP-source policy; editor MCP is bound to that session. Toolsets are resolved/frozen by execution policy. These are useful implementation mechanisms, **not a portable ACP tool-inventory or isolation capability**. [H1][H7] [H4]

The inspected handshake advertises load, images, fork/list/resume but does not explicitly advertise HTTP MCP. A conforming client must use the negotiated advertisement, not infer HTTP support from internal implementation. Current source includes revision-fenced MCP rebinding and reconnect snapshot adoption. The official host guide instead describes changed MCP specifications as refusing reattachment; its internals page also retains the older in-process lifecycle. **Documentation/source drift is unresolved:** pin the chosen Hermes build and test these behaviors before relying on either interpretation. [H2][H3] [H5][H8]

**Inference:** use explicitly provisioned execution policies, separate from an unrestricted personal chat. Evaluate isolation of memory, skills, plugins, ambient MCP, network, provider fallback and credentials as well as tool names. Current gateway policy supports toolset selection and bypass flags, but the public ACP session parameters do not make these a universal per-job control surface. Hard isolation may require separate execution instances/sandboxing; no configuration change is prescribed here. [H4][A1]

## 5. Preserve summary and research execution

Current agents specify empty tools for private structured stages and only host `web_search` for public evidence, with distinct schemas (`apps/server/src/harness/agents.ts:135–245`). Research opens fresh worker sessions, reauthorizes before generation, combines deadline/credential/job abort signals, validates sources and closes resources (`apps/server/src/jobs/research-workspace.ts:893–1029`). The job contract fences late artifacts and separates public briefs from private document analysis/synthesis (`docs/background-jobs.md:362–386,487–538`).

**Proposed minimum ACP worker contract:**

1. Fresh session per stage/attempt, never a fork of Planner history; bounded summary reductions may reuse their own session. Session separation must also exclude cross-session memory and ambient tools.
2. Public evidence sees only the approved brief and a bounded host search tool. Private analysis/synthesis sees designated private inputs and no web, repository, shell, filesystem, skills, plugins or ambient MCP.
3. Explicit model route and owner binding; revocation aborts model and host tools. No fallback to another person's provider subscription.
4. Strictly validate a bounded result against the existing schema, provenance rules, deadline and claim generation; only the host settles/publishes it.

ACP v1 has no standard schema-output argument. A **proposed** portable collection strategy is a schema-declared, job-scoped MCP `submit_result` tool, intercepted by the host and not allowed to mutate documents. Alternatively parse/validate bounded final JSON and reject malformed output; model compliance remains untested. A submission tool is a narrowly scoped capability, so adopting it changes the current literally empty-tool private-worker mechanism and requires an explicit isolation-contract decision. No adapter should silently claim equivalent native structured output. [A1]

Research also currently depends on GitHub MCP `web_search` (`docs/background-jobs.md:506–523`; `jobs/research-workspace.ts:932–969`). Removing Copilot/GitHub requires selecting a replacement search provider while retaining normalized source provenance and public/private separation; changing the Planner transport alone is insufficient.

## 6. Events, cancellation and recovery

**Proposal:** translate message/thought chunks into chat streams; key tool state by session/turn/tool-call ID, merge partial updates, and preserve permission/error/cancelled states. ACP v1 tool names/raw outputs are not guaranteed complete; its `ToolCall.name` is optional. Obtain authoritative host-tool results from the MCP execution ledger, never human-readable titles. Deduplicate notifications versus actual execution so each host action runs once. Preserve unknown usage rather than invent tokens/credit accounting. Existing remote code rejects undeclared/repeated host calls and aborts host tools at turn end (`apps/server/src/harness/remote/link.ts:331–439`). [A1][A3]

For v1 cancellation, send `session/cancel`, cancel permission waiters, accept trailing updates, and await the prompt's cancelled completion with bounded teardown. The connector already escalates after three seconds (`apps/connector/src/acp.ts:200–211`), but killing a current Hermes stdio viewer must not be assumed to kill gateway-owned execution. Verify canonical cancellation and late-result fencing separately. [A7][H1]

Load is optional in v1 and replays prior conversation; it is not exactly-once tool execution or a durable event cursor. Keep Chopin's durable transcript, mutation receipts and job claims authoritative; distinguish replay from new updates, never re-execute tools from history, and do not automatically replay interrupted mutating prompts (`AGENTS.md:224–228`). Hermes reconnect internals have their own snapshot/watermark handling; that is agent-specific, not proof of generic ACP replay semantics. [A6][H5]

## 7. Independent portability check and remaining decisions

**Gemini CLI** independently documents `gemini --acp`. Its inspected dispatcher advertises load, richer prompt content and HTTP/SSE MCP. However, session configuration merges configured MCP servers with client-provided servers; filesystem operations can fall back to the native filesystem when proxy capabilities are absent. This disproves the assumption that supplying only Chopin MCP or empty client capabilities restricts every compatible agent. Its prose also references an obsolete implementation path and handshake-time MCP registration; source and ACP session setup place server definitions in session requests. [G1][G2] [G3][G4] [A6]

**Paper portability matrix:** both agents have a documented/source-backed stdio conversation path and load support; Gemini explicitly advertises HTTP MCP, while inspected Hermes does not. Neither has demonstrated Chopin worker isolation, schema-result fidelity, host-tool correlation, owner separation or recovery. Keep them selectable only for roles whose contract passes; do not silently downgrade Planner ACP or research to make selection succeed. [H2][G2]

Human decisions still needed: direct harness versus bridge; execution placement under Aspire; whether Planner retains personal Hermes capabilities; per-owner model billing/credential ownership; acceptable agent-specific provisioning versus negotiated extensions; structured-result mechanism; replacement search provider; reconnect retention and approval scope. Native Desktop routes/panes are available, but Desktop gateway RPC is not itself an ACP Planner implementation. [R][H6]

Future acceptance evidence must cover both agents' Planner read/edit and denied-write turns; all isolated worker stages with malformed results; real search provenance; concurrent owners; permission cancellation; revoked credentials; dropped links/restarts; replay without duplicate mutation; and cancellation that stops execution rather than only the viewer. **None was run in this planning task.**

## Validation performed

Source inspection and public documentation retrieval only; upstream runtime tests were read, not
executed. The initial researcher lacked `dprint`. Parent verification then installed the frozen
workspace dependencies with install scripts disabled and ran `bun run ci` successfully using the
repository-pinned Bun 1.4.2: formatting, lint, tokens, and design checks passed (71 existing lint
warnings, zero errors). Citation-label resolution, whitespace, and privacy-path checks passed;
there are no relative Markdown links. These documentation checks do not establish ACP runtime
interoperability, tool isolation, or recovery; all runtime acceptance tests remain.

## Sources

Repository citations above use exact paths/line ranges at the declared Chopin revision.

[R]: https://github.com/crisweber2600/chopin/issues/1
[A1]: https://github.com/agentclientprotocol/agent-client-protocol/tree/87da85c4634a1f9b519e760042886316b82a0a23/schema
[A2]: https://agentclientprotocol.com/protocol/transports
[A3]: https://agentclientprotocol.com/protocol/tool-calls
[A4]: https://agentclientprotocol.com/protocol/v2/prompt-lifecycle
[A5]: https://agentclientprotocol.com/protocol/initialization
[A6]: https://agentclientprotocol.com/protocol/session-setup
[A7]: https://agentclientprotocol.com/protocol/prompt-turn
[H1]: https://github.com/NousResearch/hermes-agent/blob/5043721040a4b97269df0775b226910f2cd0bb54/acp_adapter/server.py#L245-L249
[H2]: https://github.com/NousResearch/hermes-agent/blob/5043721040a4b97269df0775b226910f2cd0bb54/acp_adapter/gateway_server.py#L138-L147
[H3]: https://hermes-agent.nousresearch.com/docs/user-guide/features/acp
[H4]: https://github.com/NousResearch/hermes-agent/blob/5043721040a4b97269df0775b226910f2cd0bb54/gateway/session_policy.py#L627-L680
[H5]: https://github.com/NousResearch/hermes-agent/blob/5043721040a4b97269df0775b226910f2cd0bb54/acp_adapter/gateway_server.py#L173-L405
[H6]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk
[H7]: https://github.com/NousResearch/hermes-agent/blob/5043721040a4b97269df0775b226910f2cd0bb54/acp_adapter/gateway_server.py#L269-L292
[H8]: https://hermes-agent.nousresearch.com/docs/developer-guide/acp-internals
[G1]: https://github.com/google-gemini/gemini-cli/blob/a713410aba5d594594e20d5c822e3aab8f822089/docs/cli/acp-mode.md
[G2]: https://github.com/google-gemini/gemini-cli/blob/a713410aba5d594594e20d5c822e3aab8f822089/packages/cli/src/acp/acpRpcDispatcher.ts#L40-L103
[G3]: https://github.com/google-gemini/gemini-cli/blob/a713410aba5d594594e20d5c822e3aab8f822089/packages/cli/src/acp/acpSessionManager.ts#L379-L434
[G4]: https://github.com/google-gemini/gemini-cli/blob/a713410aba5d594594e20d5c822e3aab8f822089/packages/cli/src/acp/acpFileSystemService.ts#L51-L83
