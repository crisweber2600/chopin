# Hermes hosting boundary for Chopin's collaborative editor

## Result and evidence boundary

**Hermes Desktop has a genuine native contribution surface, not just an iframe host.** It can
mount React panes and routed pages, navigation, commands, and contextual chrome. This establishes a
credible host for Chopin's finished UI, **not proof that the existing rich editor runs unchanged**.
The critical unresolved boundaries are single-file packaging, React DOM dependencies, authenticated
bidirectional collaboration, identity pinning, and disposal. [H1], [H2], [H3]

Research date: **2026-10-22**. Installed Hermes source was rechecked at
`d923a5eb487a17c0a68801bba9ef0f56c81040a8`, with no tracked/untracked changes reported. Chopin was
examined at `47138debe85b3263323077fa172430bb74e59b01`. Current official Desktop, dashboard,
and multi-connection documentation was also fetched. Revision-pinned source links below take
precedence for precise implementation claims; deployed documentation can differ. [H1], [H4], [H5]

**Verified here:** source inspection, manifests/import maps, repository instructions, architecture,
design guidance, and the research issue/map Notes. **Not tested:** plugin installation or execution,
editor bundling, two-user collaboration, remote sign-in, performance, or service lifecycle. No
prototype, service, configuration change, or production write was authorized or performed.

The agreed destination preserves shared documents, Chat, Decisions, and research; uses Azure
Repos/Boards with approved writes; requires ACP for both Planner and coding agents; defaults to
Hermes while allowing compatible alternatives; and permits a shared backend managed through
Aspire. An existing-site iframe is not the finished product. These are constraints, not findings
that Hermes already implements them. [C1]

## 1. Four different extension boundaries

| Boundary           | Verified contract                                                                                                | Consequence for Chopin                                                                        |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Desktop UI         | ESM `plugin.js`, default `{ id, register(ctx) }`; panes, routes, sidebar, palette, SDK UI kit                    | Native-feeling document workspace is a supported kind of contribution.                        |
| Dashboard UI       | Separate `dashboard/manifest.json`, prebuilt single-script/IIFE UI, optional CSS, `window.__HERMES_PLUGIN_SDK__` | Dashboard examples are not Desktop packages or an alternative fulfillment of the destination. |
| Python plugin      | `plugin.yaml`, plugin manager, tools/hooks and other agent-side facilities                                       | Backend/agent extension, not a rich-editor mount.                                             |
| REST/socket bridge | `dashboard/plugin_api.py` exporting a FastAPI router, mounted at `/api/plugins/<id>`                             | Shared backend namespace used by both UI SDKs; not a ready-made Chopin protocol adapter.      |

These distinctions are explicit in documentation and source. Dashboard `version` is plugin package
metadata, not a Desktop SDK version. Python `manifest_version` defaults to 1; the inspected parser's
supported maximum is 2, independent of `api_version`. Newer manifest versions warn and load rather
than enforcing compatibility. Desktop's inspected `HermesPlugin` contract has no manifest-version
or minimum-SDK field. Declare a tested host revision and feature-detect required APIs rather than
inventing a common “v2 plugin SDK.” [H1], [H3], [H4], [H6]

Standalone Desktop files live in an app-level root. Unified packages can carry
`desktop/plugin.js` alongside Python code; Electron materializes their Desktop half into that root.
The renderer does not load it afresh for each selected profile. Unified Desktop halves are opt-in;
Python enablement is separate. A remote Python installation does not automatically put its UI
files on the local Desktop. Installation, update, and removal must therefore account for both
machines and both enable states. [H1], [H2], [H7]

**Trust boundary:** runtime Desktop code executes with the renderer's app authority, not inside a
capability sandbox. Import restrictions and contribution error boundaries are not protection from
a malicious editor bundle. Package provenance, pinned-version review, and consent to install are
separate human decisions; remote document content must remain data, never plugin code. [H2], [C4]

## 2. Packaging: restricted imports do not prohibit bundled libraries

The runtime loader scans imports, rewrites allowed specifiers to host-owned shim modules, and
imports a JavaScript Blob. The exact inspected map permits `@hermes/plugin-sdk`, `react`,
`react/jsx-runtime`, **and `react/jsx-dev-runtime`**. Relative modules, bare third-party packages,
CSS imports, and URL imports are not mapped. Raw TSX/JSX is not transpiled at load time. [H2], [H3]

**Inference:** a build could compile JSX and inline third-party JavaScript into one ESM artifact,
leaving only permitted imports. The loader does not reject code merely because it originated in
Lexical or Yjs. This is a plausible packaging route, **not demonstrated compatibility**. Every
remaining dynamic import, worker, asset URL, CSS/font dependency, or Node-only assumption must be
accounted for without bypassing admission checks. The in-tree Vite “bundled plugin” mode is a
separate distribution choice, not a prerequisite for all bundling. Its actual lint rule blocks
selected internal import patterns; its broader documentation wording should not be mistaken for
an exhaustive package-dependency prohibition. [H1], [H2], [H3], [H8]

Concrete editor risks:

- **React identity:** Hermes supplies its singleton React. Its Desktop manifest pins React and
  React DOM `19.2.7`; Chopin's catalog pins `19.2.4`. Use the host React in a prospective artifact
  and verify peer/runtime compatibility; matching the major version is not proof. [H3], [H9], [C2]
- **React DOM is a real gap:** Chopin imports `createPortal` and `flushSync` from `react-dom`.
  That specifier is absent from the runtime map, and the inspected SDK does not export those
  functions. Simply externalizing both React and React DOM will fail. Whether a carefully bundled
  React DOM subset works safely with host React, or an SDK addition/adaptation is preferable,
  requires a bounded proof and possibly an upstream decision. Do not reach into private globals.
  [H3], [H10], [C3]
- **Lexical/Yjs graph:** Chopin pins Lexical `0.48.0`, MDXEditor `4.1.0` with a repository patch,
  Yjs `13.6.31`, and y-protocols `1.0.7`. The patch, dialect node registrations, and a deduplicated
  dependency graph belong in the proof; source availability alone does not validate a new bundle.
  Preserve the server-compatible schema rather than substituting a host editor. [C2], [C4]
- **Size/loading:** current source reads complete plugin files up to 16 MiB; older-shell fallback
  rejects truncated reads associated with the 512 KiB preview limit. Module evaluation has a
  10-second deadline. These are ceilings, not acceptable latency or memory targets. Measure the
  real artifact before choosing distribution. [H2], [H11]

## 3. Collaboration transport is not supplied by `ctx.socket`

`ctx.rest` addresses only the plugin namespace and uses Desktop's authenticated REST bridge.
It resolves ambient connection/profile routing per call. Its public options do not expose an
explicit connection-owner pin. `ctx.socket` resolves a connection, receives JSON messages,
reconnects with backoff, and returns **only a disposer**, not a send function. Token-mode sockets
use a query credential. **OAuth remotes deliberately get no socket**, because core manages their
single-use WebSocket tickets. Therefore it cannot directly replace Chopin's bidirectional room
transport. [H12], [H13]

Chopin's existing reusable seam is `Transport.on/send/ask` with connection readiness. Its
`PlanProvider` retains updates until acknowledged, reopens on connection, and treats epoch resets
separately from ordinary reconnects. Existing architecture requires server validation and fenced
persistence before acknowledgment/publication; Decisions remain authoritative sidecar records.
Replacing the wire does not remove those obligations. [C4], [C5]

There are at least two candidates to compare, not select here:

1. **Plugin namespace adapter:** REST commands plus authenticated event hints and durable
   catch-up/poll reads, backed by a shared collaboration service. This needs explicit
   ordering/ack/replay design and may need new SDK capabilities for connection pinning.
2. **Dedicated shared-service client:** a native pane connects to the collaboration service through
   a separately designed user-authenticated HTTP/WebSocket boundary. This avoids pretending
   `ctx.socket` is duplex, but still needs proof of renderer network policy, login, origin handling,
   credential custody, and deployment compatibility.

These are engineering hypotheses derived from the inspected seams, not advertised integrations.
Neither authorizes launching services outside the agreed Aspire lifecycle. An iframe could be an
explicitly approved diagnostic/control comparison only, never the final UI. [H12], [C1], [C5]

There is also a first-party `broadcast_plugin_event` bridge into the gateway event stream. It fans
out to every connected client of that process; it is not durable document delivery or per-reader
repository authorization. Use non-sensitive invalidation hints with authorized reads, not private
document payloads indiscriminately. It does not prove OAuth delivery end to end. [H14]

**Additional hosting risk:** Python APIs are not invariably in-process. In
`plugins.isolation: host`, user-plugin HTTP routes forward into a profile's plugin-host process;
responses are buffered and that forwarder does not provide streaming/WebSockets. In-process mode
can mount routers directly. This source distinction invalidates a blanket “Python backend always
runs inside the gateway” assumption and must enter the transport test matrix. The source also
contains first-request HTTP hot-mounting for newly enabled APIs, whereas the SDK troubleshooting
text recommends restart. Neither behavior proves backend code updates/unloading follow Desktop
hot reload: test those lifetimes independently. [H1], [H15], [H16]

## 4. Connection, session, profile, and human identity

Hermes supports several gateways with independent profiles/sessions in one Desktop. That is
multi-connection agent management, not evidence of a shared multi-person document model. Keep
Chopin document UUID, collaborative member identity, repository authorization, Hermes connection,
profile, durable session ID, and runtime session ID distinct. [H5], [H10], [C4]

The SDK exposes `connectionId`, `focusedSessionOwner`, `focusedSessionId`,
`focusedStoredSessionId`, and `focusedSessionProfile`. The complete owner is preferable to a bare
profile, which can collide across gateways. `host.request` targets the live gateway per call;
`host.requestProfile` supports qualified route descriptors. A pane must not assume tile focus,
active gateway, plugin REST scope, and its document's execution owner move together. [H10], [H13]

**Required design/test consequence:** pin an operation's authority at initiation, reject stale
responses after navigation, and key caches by collaboration endpoint, authenticated principal,
repository, and document as applicable, plus Hermes connection/profile for agent resources.
`ctx.storage` is plugin-namespaced local persistence, not a shared document database or a security
wall. Do not infer Azure user permissions from a selected Hermes profile or gateway token.
[H2], [H10], [H12], [C1], [C4]

Preserve shared root Chat without implicitly publishing private Hermes transcript content.
Transcript storage, projection, and consent rules remain to be decided. In the current Chopin model,
child research results retain their own documents/Decisions and use the root Chat with child focus,
not a second child Chat. ACP integration must explicitly map Planner ownership and permissions while preserving
research/summary execution; the Desktop JSON-RPC and plugin event APIs inspected here are not
proof of that ACP contract. [C1], [C4]

## 5. Native appearance and lifecycle ownership

Use SDK chrome and theme variables around the reused editor, rather than importing the whole web
application shell. Chopin's existing styles assume its shared tokens and MDXEditor structural CSS;
comment/image overlays portal into `document.body`, and table controls use viewport geometry.
A wrapper alone therefore does not isolate all CSS/portals. Prefix/map editor styles and address
portal containers, stacking, clipping, focus, and keyboard shortcuts explicitly. Shadow DOM is an
untested alternative, not an automatic fix for Lexical selections or document-level APIs.
[H1], [C3], [C6], [C7]

The loader disposes tracked contributions on reload/disable and rolls back failed registration.
`ctx.onDispose`, scoped events/timers/listeners, and component effect cleanup cover different
lifetimes. Raw sockets, Y.Docs/Awareness, provider subscriptions, observers, injected styles,
query work, and late async completions need explicit ownership. ESM evaluation can occur before
activation, so module scope must not initiate document writes or persistent resources. Disabling
the UI is not cancellation of already-authorized shared research or agent work unless a separate
server operation says so. [H2], [H16], [H17], [C4]

## 6. Bounded prototype verification plan — not executed

**Proposal, requiring separate authorization:** a time-boxed disposable pilot with one pinned
Desktop revision, synthetic documents/users, isolated storage, and no Azure writes or production
credentials. The time budget is not decided. Any services must be Aspire-managed. Stop after the
agreed gates; report failures rather than quietly rewriting the editor or selecting topology. [C1]

1. **Packaging gate:** build one native page containing the smallest real patched editor and
   collaboration provider. Inventory residual imports/assets; measure artifact size/load time;
   prove host React identity and exercise portals/`flushSync`. Include a table, custom dialect
   node, undo, paste, and composition input. Pass only with no hook/context/duplicate-library
   errors. A failed React DOM boundary becomes an explicit SDK/adaptation decision.
2. **Transport gate:** compare candidate adapters with local/token and OAuth remotes, and
   in-process/host-isolated Python APIs. Observe OAuth socket absence rather than expecting
   accidental success. Show acknowledged edits, bounded reconnect replay, epoch replacement,
   denied access, and catch-up after missed hints. Polling must recover durable state, not merely
   repaint metadata while edits are lost.
3. **Shared-state/identity gate:** two humans edit the same document; verify Chat, Decisions,
   presence, and a research child agree. Switch between two connections with identically named
   profiles while a read/write is in flight. Pass only if no response, cache entry, permission,
   or write crosses the original authority boundary. Test sign-out/revocation and read-only access.
4. **Lifecycle/UI gate:** repeat mount/unmount, pane moves, route changes, disable, and hot reload
   twenty times. Count sockets/listeners/observers/styles; verify no retained ownership or
   duplicate publication. Test two mounted editors, light/dark themes, narrow panes, scrolling,
   overlays, keyboard focus, and reduced motion. Record limitations without claiming a finished UI.

**Human choices left open:** collaboration service topology and endpoint selection; sign-in and
Azure authorization ownership; independent versus focus-following document/agent binding;
acceptable SDK/upstream changes and minimum Desktop revision; plugin update/trust policy; and
whether leaving/disable cancels, detaches, or leaves durable work running. ACP agent permissions
and work-item write approval remain separate decisions, not consequences of picking a pane.
[C1]

## Documentation validation

Parent verification installed frozen workspace dependencies with install scripts disabled and ran
`bun run ci` successfully using the repository-pinned Bun 1.4.2. Formatting, lint, tokens, and design
checks passed (71 existing lint warnings, zero errors); citation labels, relative-link casing, and
privacy-path checks passed. No plugin, editor, authentication, or collaboration integration was run.

## Sources

All Hermes source links pin the inspected revision. Live documentation was accessed 2026-10-22.

[H1]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk
[H2]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/contrib/runtime-loader.ts
[H3]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/sdk/runtime.ts
[H4]: https://hermes-agent.nousresearch.com/docs/user-guide/features/extending-the-dashboard
[H5]: https://hermes-agent.nousresearch.com/docs/user-guide/multi-connection-desktop
[H6]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/hermes_cli/plugins_manifest.py
[H7]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/electron/desktop-plugins-root.ts
[H8]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/eslint.config.mjs
[H9]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/package.json
[H10]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/sdk/index.ts
[H11]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/electron/main.ts#L17835
[H12]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/api/plugins.ts
[H13]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/api/client.ts
[H14]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/hermes_cli/plugin_events.py
[H15]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/hermes_cli/plugin_isolation.py
[H16]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/hermes_cli/web_server_dashboard.py#L790-L1047
[H17]: https://github.com/NousResearch/hermes-agent/blob/d923a5eb487a17c0a68801bba9ef0f56c81040a8/apps/desktop/src/contrib/plugin.ts
[C1]: https://github.com/crisweber2600/chopin/issues/1
[C2]: https://github.com/crisweber2600/chopin/blob/47138debe85b3263323077fa172430bb74e59b01/package.json
[C3]: https://github.com/crisweber2600/chopin/tree/47138debe85b3263323077fa172430bb74e59b01/packages/editor/src
[C4]: ../docs/architecture.md
[C5]: ../packages/editor/src/provider.ts
[C6]: ../apps/web/DESIGN.md
[C7]: ../packages/editor/src/styles.css
