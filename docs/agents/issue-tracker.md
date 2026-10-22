# Issue tracker: GitHub

Issues, specifications, and implementation tickets for this fork live in
`crisweber2600/chopin`. Use the authenticated `gh` CLI and explicitly scope commands
with `--repo crisweber2600/chopin`; upstream `githubnext/chopin` is not this fork's
work tracker.

## Operations

- Create: `gh issue create --repo crisweber2600/chopin --title "..." --body-file <path>`.
- Read: `gh issue view <number> --repo crisweber2600/chopin --json number,title,body,labels,comments`.
- List: `gh issue list --repo crisweber2600/chopin --state open --json number,title,body,labels,assignees`.
- Comment: `gh issue comment <number> --repo crisweber2600/chopin --body-file <path>`.
- Label: `gh issue edit <number> --repo crisweber2600/chopin --add-label "..." --remove-label "..."`.
- Close after verified completion: `gh issue close <number> --repo crisweber2600/chopin --comment "..."`.

When a skill says "publish to the issue tracker," create a GitHub issue here.
When it says "fetch the relevant ticket," read that issue and its comments.
Use the role mapping in [triage-labels.md](triage-labels.md).

## Parent issues and dependencies

Use native GitHub sub-issues and blocking dependencies when available. Resolve each
issue's numeric database ID with
`gh api repos/crisweber2600/chopin/issues/<number> --jq .id`; an issue number and a
node ID are not interchangeable with that database ID.

- Parent link: `gh api --method POST repos/crisweber2600/chopin/issues/<parent>/sub_issues -F sub_issue_id=<child-database-id>`.
- Blocking link: `gh api --method POST repos/crisweber2600/chopin/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-database-id>`.

Read back links after writing. If native links are unavailable, record
`Part of #<parent>` and `Blocked by: #<number>` in the child body, and maintain the
parent's task list. Work is ready only when its blockers are closed.

For `/wayfinder`, use a map issue and linked child tickets. Create its additional
`wayfinder:*` labels only when that workflow is actually invoked; the setup's
triage vocabulary remains the five defaults. Claim a ticket with
`gh issue edit <number> --repo crisweber2600/chopin --add-assignee @me` before working
it, and retain conclusions and evidence in the issue before closing it.

## Pull requests as a triage surface

**PRs as a request surface: no.** External PRs are not automatically treated as
feature requests. Use issues for requirements and the repository's existing PR
readiness and review process for implementation changes.
