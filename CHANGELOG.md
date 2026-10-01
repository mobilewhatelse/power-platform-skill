# Changelog

## 0.2.1

- Fix: user feedback reported that the skills did not work in GitHub Copilot, whose harness does not understand some of the frontmatter properties (its documentation lists only `name`, `description`, `license`, and `allowed-tools` as text). Removed `allowed-tools`, `argument-hint`, and `user-invocable` from every `SKILL.md`; frontmatter is now only `name`, `description`, `license`.
- Added `tools/check_skills.py`, which fails on non-portable frontmatter fields, name/folder mismatches, over-long descriptions, and unsafe YAML in descriptions.
- Docs: corrected the README and CLAUDE.md statement that the extra fields were Copilot-specific and ignored by Claude Code.

## 0.2.0

- Added `references/connector-data-sources.md` to `power-platform-code-apps` — wiring a Code App to a non-Dataverse connector data source (e.g. an existing Copilot Studio agent): the pre-existing-connection requirement, a Git Bash path-mangling gotcha in `add-data-source`, a client-side singleton bug (`PowerDataSourcesInfoProvider`) that silently breaks connector calls even when everything is correctly provisioned server-side, the undocumented-but-required `notificationUrl` field for `ExecuteCopilotAsyncV2`, and an in-flight-guard pattern for slow connector calls.
- Added matching entries to `power-platform-code-apps`'s `troubleshooting.md`, including a general Dataverse gotcha: a field named like an autonumber (e.g. `...Number`, the primary-name attribute) isn't necessarily Dataverse's native AutoNumber type — check `AutoNumberFormat` before assuming it self-populates.

## 0.1.0

- Added `.claude-plugin/marketplace.json` — the three existing skills (`power-platform-code-apps`, `power-platform-pages`, `power-platform-copilot-studio`) are now installable in one bundle via `/plugin marketplace add` + `/plugin install` in Claude Code and GitHub Copilot CLI, alongside the existing file-based install paths.
