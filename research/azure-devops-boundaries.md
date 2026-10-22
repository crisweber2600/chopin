# Azure DevOps integration boundaries

Research date: **2026-10-22**. Scope: [Verify Azure DevOps repository, work-item, and authorization interfaces](https://github.com/crisweber2600/chopin/issues/4), under [Plan Chopin for Hermes, Azure DevOps, and ACP](https://github.com/crisweber2600/chopin/issues/1).
**Evidence, not a working integration:** public Microsoft documentation and complete raw source
were retrieved; no Azure organization was queried, authenticated, or changed. No services or
agents were launched, and no application or authentication configuration was changed. Upstream mock tests were inspected,
not executed. Recommendations below are explicitly proposals, not settled product decisions.

Microsoft local MCP source is pinned to commit
`3ee829cb41536fd29aabfb27954fb6331b5e69ff` (commit date 2026-10-22;
manifest version `2.11.0`, not an independently verified installed/npm release). The remote
MCP guide reports `ms.date` 2026-09-21, `updated_at` 2026-09-28, and content commit
`efe955e720f8919bfdea6fbf9cd0dd8f60420c76`. REST examples deliberately target **7.1**;
this is a researched baseline, not a claim that no newer preview exists.[^local][^remote]

## 1. Conclusion and preserved product boundary

**Supported APIs cover the requested capabilities:** repository/file/PR reads and work-item
query, creation, update, relation linking, and observation. REST is the most explicit contract;
Microsoft MCP exposes useful subsets, not interchangeable schemas or authorization.
The approved scope permits existing-item reads and explicitly approved create/update/link
work, including PRs. It does **not** permit automatic merge or deployment. API availability
neither authorizes writes nor chooses synchronization policy.[^files][^prs][^create][^update][^events]

Chopin must retain shared documents, Chat, Decisions, and research in its native Hermes plugin
UI, with the permitted shared backend managed through Aspire and ACP for both Planner and
coding agents. Those are map constraints, not Azure features. Current source binds repository
arguments on the host, checks owner/session generations, and separates durable document
identity from repository labels. **Proposal:** preserve those boundaries rather than replace
them with a shared MCP process's ambient identity. This investigation does not select backend
topology, field authority, or an ACP adapter.[^chopin]

## 2. Stable resource addressing

Use identifiers, not renamed display names, as persisted authority:

| Resource                | Proposed persisted key and discovery                                                                                                      |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Organization/collection | Cloud account GUID from Accounts List, plus validated service origin and current organization URL; keep Entra tenant identity separately. |
| Project                 | Organization/collection key plus project GUID; Projects List is caller-filtered.                                                          |
| Repository              | Organization/collection key plus repository GUID, retaining project GUID and current names as metadata.                                   |
| Work item               | Organization/collection key plus numeric ID; project GUID is current placement, not globally unique item identity.                        |
| Pull request            | Organization/collection, target repository GUID, numeric PR ID; retain source repository/ref separately for forks.                        |

Microsoft exposes account, project, and repository UUIDs. Work-item IDs are unique within an
organization or project collection. A deleted/recreated resource with the same name must not
inherit the old binding. GUID equality is proposed application policy, not a claim of identity
preservation across collection migrations. URL/organization rename and Server collection
migration need explicit rediscovery tests.[^accounts][^projects][^repos][^workidentity][^prs]

Read file content at an exact commit using Git Items' `versionDescriptor` rather than a moving
branch when citing evidence. Keep document UUID, Azure work-item `rev`, Git commit, PR ID,
and Chopin document/storage revisions distinct.[^files][^chopin]

## 3. Per-user identity and authorization

For Services, Microsoft recommends Entra user delegation for interactive applications;
operations remain within the user's Azure DevOps access level and permissions. Application
identities are different principals requiring their own organization membership/access.
New integrations must not adopt legacy Azure DevOps OAuth just because older REST reference
security sections show its URLs: new registrations stopped in April 2025. Treat access tokens
as opaque and honor acquisition expiry metadata.[^entra]

**Repository access does not imply Boards access.** Git permissions and area-path permissions
are separate. Boards explicitly distinguishes `View work items in this node` from
`Edit work items in this node`; a view Deny prevents seeing that area's items, including for
administrators. Project membership, licensing/access level, group inheritance, explicit Deny,
repository/ref policies, and target area all matter. A token's API scope is not an independent
resource grant. Readable repo plus forbidden area is a mandatory negative pilot test.[^permissions]

**Proposal:** authorize each exact operation as its initiating principal; bind approval to
organization/project/repository or item IDs, expected revision, payload, and credential/owner
generation. Recheck at execution, on long-lived connections, and after identity rotation.
Do not relay restricted Boards content to every repository collaborator: either require
recipient access or agree an explicit document-sharing policy. A shared backend/service account
or Planner owner's token does not automatically represent another editor.[^entra][^permissions][^chopin]

Local MCP documents `interactive`, `azcli`, `env`, `envvar`, and `pat` authentication.
Important source nuance: current `azcli` builds `DefaultAzureCredential` with the development
credential chain, optionally preferring `AzureCliCredential` for a tenant; its name alone is
not proof of one fixed account. Isolate credentials per user and verify returned identity.
`ado_mcp_project`/team are selection defaults, not ACLs; domains reduce exposed tools, not the
underlying principal's permissions.[^gettingstarted][^authsource][^local]

## 4. REST and local/remote MCP contracts

REST provides Git Items, PR listing and updates, WIQL, work-item batch hydration, JSON Patch
creation/updates, process metadata, reporting revisions, and service hooks. These contracts
can support a host-owned, argument-validating adapter without making an MCP vendor's entire
catalogue an agent capability.[^files][^prs][^prupdate][^wiql][^batch][^update][^reporting]

Remote MCP uses Streamable HTTP and Entra OAuth at `https://mcp.dev.azure.com/{organization}`;
standalone MSA organizations are excluded. Microsoft documents specific client/registration
paths; Claude Desktop and Codex cannot currently satisfy its required dynamic-registration
flow and are directed to local MCP. Hermes compatibility was **not tested**; transport support
alone proves neither OAuth nor multi-user delegation. Remote headers select toolsets/tools,
read-only mode, or Insiders; tool selection and read-only mode cannot express arbitrary
parameter-level write approval. The guide warns its catalogue can lag service changes.[^remote]

The following are **source-verified local contracts**, not live `tools/list` discovery.
Use the pinned definitions for complete optional fields and conditional requirements:

- `repo_repository`: `get|list`; `repo_file`: `get_content|list_directory`.
  `repo_pull_request`: `get|list|list_by_commits`, repository/project identifiers,
  PR ID, refs, status, `top`/`skip`, and optional work-item/label/changed-file expansion.
- `wit_work_item`: `get|get_batch|list_comments|my|list_revisions|list_for_iteration|get_type`;
  IDs, project, fields/expansion/as-of, and action-specific paging inputs.
- `wit_query`: `get|get_results|wiql`; local ad-hoc input is `wiql`, with `top`
  default 50. **Remote differs:** the documented ad-hoc tool is `wit_query_by_wiql`,
  while remote `wit_query` also advertises `search|list`.
- `wit_work_item_write`: `create|update|update_batch|add_child`. Creation uses
  `workItemType` and `fields[{name,value,format?}]`; single-item update uses `id`
  and `updates[{op,path,value?}]`, allowing `add|replace|remove|test` and JSON values.
  Batch updates instead use capitalized `Add|Replace|Remove` and string values.
- `wit_work_item_link_write`: `link|unlink|link_to_pull_request|add_artifact_link`.
  Conditional targets include item IDs, relation type, project/repository GUIDs, PR ID,
  or artifact components/URI. Omitted unlink URL removes **all** links of that type.
  Comment writes are a separate `wit_work_item_comment_write` tool.[^toolset][^witsource][^reposource][^remote]

### Consolidated PR writes are an approval hazard

`repo_pull_request_write.action` is exactly `create|update|update_reviewers|vote`.
Creation takes repository, source/target refs, title, optional description, draft, work items,
fork repository, and labels. Update also accepts `status: Active|Abandoned`, `autoComplete`,
`mergeStrategy`, `mergeCommitMessage`, `deleteSourceBranch` (default false),
`transitionWorkItems` (**default true**), `bypassPolicy` (default false), and `bypassReason`.
The implementation sets `autoCompleteSetBy` and completion options when autocomplete is true.
Allowlisting this tool name therefore allows a merge-scheduling path.[^reposource]

**Proposal:** expose separate host-enforced approved operations, rejecting autocomplete,
completion options, bypass, unsolicited votes/reviewer changes, and branch deletion.
Preserve an existing PR's state unless approved otherwise; do not silently cancel someone
else's autocomplete. Inspect pre-existing autocomplete before writes or votes that could
satisfy it. No automatic completion or deployment capability belongs in either agent's
allowed contract. Azure policy is defense in depth, not a substitute for payload validation.
Push/PR publication can trigger existing pipelines, so review trigger consequences before
approving publication; withholding pipeline tools alone is insufficient.[^prupdate][^triggers]

## 5. Work-item metadata, conflicts, and idempotency

Discover system/inherited process IDs through `/_apis/work/processes`, then actual project
types, type fields with expanded properties, state names/categories, and relation reference
names. Classification Nodes supplies area/iteration identities and paths; retain IDs separately
from renameable paths and fetch sufficient child depth.[^processlist][^classifications] Processes differ, and custom required fields/defaults/allowed
values exist; do not hard-code Task/Bug, `Active`/`Closed`, or a universal transition graph.
Metadata discovery does not guarantee a transition is allowed for that caller and current
item; server rule validation remains authoritative.[^types][^fields][^states][^relations]

Creation and update use `application/json-patch+json`. For a read-modify-write, prepend
`{"op":"test","path":"/rev","value":<numeric revision read>}` in the **same patch** as
field/relation changes. On conflict, reload and reconcile rather than retry without the test.
`validateOnly` permits rule checking without persistence; `bypassRules` and
`suppressNotifications` must not become routine agent options. Validation is not a reservation
against concurrent change.[^create][^update]

The pinned Microsoft mock tests check preservation of numeric `/rev`, an injected 412
precondition failure, and injected 409 conflict details. This is evidence of adapter behavior,
**not** a live concurrency test. Local single-item updates can carry revision tests; batch
updates and specialized link helpers expose no equivalent revision input. **Proposal:** use
single-item guarded patches for approved relation writes, including index-based removals.
Test two readers at revision R: first update succeeds, second fails without partial mutation;
repeat with link insertion/removal and changed relation indices.[^wittests][^witsource]

Relations use reference names, not UI labels. PR relations use `ArtifactLink`, attribute name
`Pull Request`, and a `vstfs:///Git/PullRequestId/` URI with encoded project/repository/PR
components; a browser hyperlink is different. Cross-project links require resolving both
resources, not assuming the work item's project contains the PR.[^relations][^witsource]

Neither reviewed create contract exposes an idempotency key. **Inference:** retries after an
ambiguous timeout can duplicate items/PRs; multi-item helpers are not an application-level
transaction. Proposed durable publication intents should record approved payload, correlation,
remote IDs, read-back, and per-step completion before announcing success. Reconcile uncertain
outcomes; titles or source/target branch matches alone do not establish which request created
an object. A create-plus-link failure must remain a recoverable partial result.[^create][^prcreate][^chopin]

## 6. Complete reads, change observation, and recovery

Pagination is endpoint-specific: PR lists use `$top`/`$skip`; Projects List accepts a
continuation token; local comments expose a token and revisions use `top`/`skip`. Work-item
batch reads allow at most **200 IDs** and `errorPolicy: Omit` can conceal missing/forbidden
results. WIQL returns IDs/relations and `asOf`, not fully hydrated fields; its REST contract
has `$top` but no generic offset/token. **Proposal:** bounded, ordered query partitions plus
hydration and explicit completeness tracking; never label a default first page complete.
Local WIQL/saved-query execution requests eventual read consistency, making immediate query
absence unsuitable as proof that a write failed.[^prs][^projects][^batch][^wiql][^witsource]

Service hooks include work-item created, updated, deleted, restored, and commented events,
plus Git/PR events. They are not durable exactly-once synchronization: retries can repeat
delivery; probation loses new events, and HTTP 410 disables a subscription. Reporting
revisions offers continuation watermarks, `isLastBatch`, and `includeDeleted` for catch-up.
**Proposal:** authenticated ingestion, event/revision deduplication, durable checkpoints,
periodic reconciliation, and authorization-aware fan-out. Subscription provisioning needs
separate approval; hook sender credentials never authorize document readers.[^events][^hooks][^reporting]

Services throttling can delay a successful HTTP 200 carrying `Retry-After`, or reject with
429. Honor delay/rate headers, bound concurrency, back off transient reads, and reconcile
writes before retrying an unknown outcome. Do not interpret a throttle as denied access or
an empty collection.[^limits]

Removed workflow state, recycle-bin deletion, restoration, and permanent destruction are
different. REST supports restore, while Microsoft says MCP cannot delete/restore/destroy
items. Delete/restore/destroy have distinct permissions/access-level restrictions.
**Proposal:** preserve external references as unavailable/tombstoned rather than delete
Chopin documents or Decisions. A 404 alone cannot distinguish deletion, wrong target, or
lost visibility; never recreate or restore automatically. Recovery after permanent destruction
cannot be promised.[^delete][^restore]

## 7. Services/Server boundary and remaining verification

This report's target assumption is **Azure DevOps Services**. Server has deployment-specific
collection origins, versions, and Windows/.NET/PAT authentication; Services Entra application
identity patterns do not transfer automatically. Specify API versions per endpoint and check
the deployed release's matrix. Current local MCP constructs a `dev.azure.com` origin and its
PAT interceptor rejects arbitrary hosts: neither official MCP interface is established here
as a generic Server solution.[^authguidance][^versions][^authsource][^indexsource]

Before implementation acceptance, an explicitly authorized disposable pilot must verify:

1. Two distinct users, revocation/expiry, repo-readable but area-forbidden access, and
   no cross-user cache or shared-document leakage.
2. Actual Hermes/ACP client's local/remote discovery, OAuth, elicitation, schemas, and
   enforcement after agent/tool configuration translation; standard and Insiders separately.
3. Approved create/update/link/PR read-back, stale-revision refusal, partial failure,
   timeout reconciliation, pagination, process customization, hook replay/gaps, and throttling.
4. Negative autocomplete/completion/bypass/deployment tests, existing-PR autocomplete,
   pipeline triggers, rename/delete/recovery, and exact identifier preservation.

These are **untested gates**, not claims of supported live integration. Field/status authority,
publication approval granularity, sharing restricted Boards data, synchronization direction,
and background identity remain human decisions. Nothing in this research authorizes their
implementation or a production write.

## Documentation validation

Parent verification installed frozen workspace dependencies with install scripts disabled and ran
`bun run ci` successfully using the repository-pinned Bun 1.4.2. Formatting, lint, tokens, and design
checks passed (71 existing lint warnings, zero errors). Reference definitions, relative-link casing,
and whitespace were checked separately. These are documentation/source checks, not authenticated
Azure, Hermes OAuth, ACP-agent, or multi-user integration tests.

## Primary sources

All external sources below are Microsoft documentation or Microsoft-owned source. Learn
pages were retrieved in full; linked API versions and the source commit fix the evidence
scope. The local project references document the inspected base, not upstream Azure claims.

[^local]: [Microsoft MCP README, pinned source](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/README.md) and [manifest](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/package.json).

[^remote]: [Microsoft remote MCP guide](https://learn.microsoft.com/en-us/azure/devops/mcp-server/remote-mcp-server?view=azure-devops).

[^chopin]: Inspected Chopin base `47138debe85b3263323077fa172430bb74e59b01`: [architecture](../docs/architecture.md), [authentication](../docs/authentication.md), [repository binding](../apps/server/src/harness/github-tools.ts), [owner validation](../apps/server/src/agent/active-owner.ts), and [repository tools](../apps/server/src/agent/repository.ts); confirmed product constraints in [Plan Chopin for Hermes, Azure DevOps, and ACP](https://github.com/crisweber2600/chopin/issues/1) and [Verify Azure DevOps repository, work-item, and authorization interfaces](https://github.com/crisweber2600/chopin/issues/4).

[^accounts]: [Accounts List](https://learn.microsoft.com/en-us/rest/api/azure/devops/account/accounts/list?view=azure-devops-rest-7.1).

[^projects]: [Projects List](https://learn.microsoft.com/en-us/rest/api/azure/devops/core/projects/list?view=azure-devops-rest-7.1).

[^repos]: [Get Repository](https://learn.microsoft.com/en-us/rest/api/azure/devops/git/repositories/get-repository?view=azure-devops-rest-7.1).

[^workidentity]: [About work items and types](https://learn.microsoft.com/en-us/azure/devops/boards/work-items/about-work-items?view=azure-devops).

[^files]: [Git Items Get](https://learn.microsoft.com/en-us/rest/api/azure/devops/git/items/get?view=azure-devops-rest-7.1).

[^prs]: [PR List](https://learn.microsoft.com/en-us/rest/api/azure/devops/git/pull-requests/get-pull-requests?view=azure-devops-rest-7.1).

[^prupdate]: [PR Update](https://learn.microsoft.com/en-us/rest/api/azure/devops/git/pull-requests/update?view=azure-devops-rest-7.1).

[^prcreate]: [PR Create](https://learn.microsoft.com/en-us/rest/api/azure/devops/git/pull-requests/create?view=azure-devops-rest-7.1).

[^entra]: [Entra authentication](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/entra?view=azure-devops).

[^permissions]: [Permissions reference, including Git and area nodes](https://learn.microsoft.com/en-us/azure/devops/organizations/security/permissions?view=azure-devops).

[^gettingstarted]: [Pinned local authentication/client guide](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/docs/GETTINGSTARTED.md).

[^authsource]: [Pinned authentication implementation](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/src/auth.ts).

[^indexsource]: [Pinned server entry point](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/src/index.ts).

[^toolset]: [Pinned local tool catalogue](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/docs/TOOLSET.md).

[^reposource]: [Pinned repository schemas and handlers](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/src/tools/repositories.ts).

[^witsource]: [Pinned work-item schemas and handlers](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/src/tools/work-items.ts).

[^wittests]: [Pinned work-item mock tests](https://github.com/microsoft/azure-devops-mcp/blob/3ee829cb41536fd29aabfb27954fb6331b5e69ff/test/src/tools/work-items.test.ts).

[^create]: [Work Item Create](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items/create?view=azure-devops-rest-7.1).

[^update]: [Work Item Update, revision-test examples](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items/update?view=azure-devops-rest-7.1).

[^types]: [Work Item Types List](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-item-types/list?view=azure-devops-rest-7.1).

[^fields]: [Type Fields List with properties](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-item-types-field/list?view=azure-devops-rest-7.1).

[^states]: [Type States List](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-item-type-states/list?view=azure-devops-rest-7.1).

[^relations]: [Relation Types List](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-item-relation-types/list?view=azure-devops-rest-7.1).

[^wiql]: [WIQL Query](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/wiql/query-by-wiql?view=azure-devops-rest-7.1).

[^batch]: [Work Item Batch Get](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items/get-work-items-batch?view=azure-devops-rest-7.1).

[^events]: [Service-hook event catalogue](https://learn.microsoft.com/en-us/azure/devops/service-hooks/events?view=azure-devops).

[^hooks]: [Service-hook failure and probation behavior](https://learn.microsoft.com/en-us/azure/devops/service-hooks/troubleshoot?view=azure-devops).

[^reporting]: [Reporting revisions and continuation](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/reporting-work-item-revisions/read-reporting-revisions-get?view=azure-devops-rest-7.1).

[^limits]: [Services rate limits](https://learn.microsoft.com/en-us/azure/devops/integrate/concepts/rate-limits?view=azure-devops).

[^delete]: [Remove, delete, restore, destroy](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/remove-delete-work-items?view=azure-devops).

[^restore]: [Recycle Bin Restore](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/recyclebin/restore-work-item?view=azure-devops-rest-7.1).

[^authguidance]: [Services versus Server authentication guidance](https://learn.microsoft.com/en-us/azure/devops/organizations/security/authentication-guidance?view=azure-devops).

[^versions]: [REST version compatibility](https://learn.microsoft.com/en-us/azure/devops/integrate/concepts/rest-api-versioning?view=azure-devops).

[^triggers]: [Azure Repos pipeline triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops).

[^processlist]: [Processes List](https://learn.microsoft.com/en-us/rest/api/azure/devops/processes/processes/list?view=azure-devops-rest-7.1).

[^classifications]: [Classification Nodes Get](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/classification-nodes/get-classification-nodes?view=azure-devops-rest-7.1).
