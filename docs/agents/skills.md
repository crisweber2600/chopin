# Matt Pocock skills installation

The project-local skill collection comes from `mattpocock/skills`, verified against
commit `5beb35b040e3de97e54fa9e386d7e42d15c295c2`. It contains 38 skills and was installed
with the upstream-recommended `skills` CLI, pinned to version `1.7.4`:

```sh
npx --yes skills@1.7.4 add mattpocock/skills --agent codex hermes-agent --skill '*' --yes
```

Canonical files live in `.agents/skills/`; `.hermes/skills/` contains relative
symlinks to the same files. `skills-lock.json` records the source and per-skill
content hashes. Existing Chopin skills are preserved.

## Local configuration

- [GitHub issue tracker](issue-tracker.md): `crisweber2600/chopin`, not upstream.
- [Default triage vocabulary](triage-labels.md): the five canonical roles.
- [Single-context domain docs](domain.md): one shared glossary and ADR directory,
  created when real terms and decisions are agreed.

Start with `/ask-matt` to select a workflow. The setup configuration is consumed by
`to-spec`, `to-tickets`, `triage`, `wayfinder`, and domain-modeling workflows.
Repository-specific instructions and security boundaries in `AGENTS.md` remain in
force.

## Maintenance

Preserve the imported skill files as upstream content. The formatter excludes only
the named third-party skill directories and their Hermes links, following the
existing vendored `gh-stack` convention; first-party skills and project guidance
remain checked.

To update, run `npx --yes skills@1.7.4 update --project --yes`; re-run `add` to include
new skills. Inspect the resulting diff, update the explicit vendored-directory
exclusions if the collection changes, verify all links and source hashes, and run
`bun run ci`. Installation alone does not execute the optional setup, migration,
hook, or experimental skills.

The upstream MIT notice is preserved in [matt-pocock-license.txt](matt-pocock-license.txt).
