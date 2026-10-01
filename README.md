# Power Platform Skills — Claude Code + GitHub Copilot

A skill collection compatible with both **[Claude Code](https://claude.com/claude-code)** and **[GitHub Copilot](https://github.com/features/copilot)**, packaging field-tested, copy-pasteable patterns for building on the **Power Platform** — Code Apps, Model-driven Apps, Power Pages, Copilot Studio, and Dataverse — all scripted directly against the Dataverse Web API.

No project-specific or organization-specific content — just the mechanics of the platform, distilled from real end-to-end build-and-deploy sessions.

## Using these skills

### Claude Code CLI / GitHub Copilot CLI (plugin marketplace)
```
/plugin marketplace add mobilewhatelse/power-platform-skill
/plugin install power-platform@power-platform-skill
```
Gets you an update command (`/plugin update`) and version tracking — see [CHANGELOG.md](CHANGELOG.md) for what changed between versions.

### Claude Code (file-based, no plugin install)
Add this repo as a skills source in your project's `.claude/settings.json`, or reference individual skill files directly in chat. Skills are in `.github/skills/<skill-name>/SKILL.md`.

### GitHub Copilot (VS Code)
Clone or add this repo. Copilot auto-discovers skills from `.github/skills/` and makes them available as slash commands (e.g. `/power-platform-pages`). Requires VS Code with GitHub Copilot extension in agent mode.

### Both
Skills use the shared `SKILL.md` format with only the portable frontmatter fields `name`, `description`, and `license`, so the files load unchanged in both tools. Harness-specific fields are deliberately left out (GitHub Copilot does not support them in a skill); `tools/check_skills.py` enforces this.

## Skills in this repo

### `power-platform-code-apps` — Code Apps + Model-driven Apps + Dataverse

For building **Power Apps Code Apps** (React/Vite) and/or **standalone Model-driven Apps** backed by Dataverse — schema provisioning, security roles, native auditing, model-driven forms/subgrids/dashboards, multi-app data integrity, and solution ALM.

Entry point: [`.github/skills/power-platform-code-apps/SKILL.md`](.github/skills/power-platform-code-apps/SKILL.md)

| Reference | Content |
|---|---|
| [`dataverse-web-api.md`](.github/skills/power-platform-code-apps/references/dataverse-web-api.md) | Tables, columns, relationships, RequiredLevel gotcha |
| [`security-and-audit.md`](.github/skills/power-platform-code-apps/references/security-and-audit.md) | Security roles, core-platform-privileges gap, native auditing |
| [`model-driven-app.md`](.github/skills/power-platform-code-apps/references/model-driven-app.md) | Sitemap, AppModule, FormXml, subgrids, dashboards |
| [`multi-app-data-integrity.md`](.github/skills/power-platform-code-apps/references/multi-app-data-integrity.md) | Shared-table null-field risks between multiple apps |
| [`solution-alm.md`](.github/skills/power-platform-code-apps/references/solution-alm.md) | Solution components, export, package verification |
| [`code-app-deployment.md`](.github/skills/power-platform-code-apps/references/code-app-deployment.md) | SDK/CLI versions, static-asset bundling trap, deploy sequence |
| [`connector-data-sources.md`](.github/skills/power-platform-code-apps/references/connector-data-sources.md) | Wiring a Code App to a non-Dataverse connector (e.g. Copilot Studio) — pre-existing connection requirement, path-mangling gotcha, a client-side singleton bug that breaks connector calls silently, required-but-undocumented fields |
| [`troubleshooting.md`](.github/skills/power-platform-code-apps/references/troubleshooting.md) | Error messages → causes → fixes |

---

### `power-platform-pages` — Power Pages

For building and scripting **Power Pages** sites — web templates, routing, anonymous access, table permissions, site settings, and the Enhanced vs Standard Data Model split.

Entry point: [`.github/skills/power-platform-pages/SKILL.md`](.github/skills/power-platform-pages/SKILL.md)

| Reference | Content |
|---|---|
| [`data-model.md`](.github/skills/power-platform-pages/references/data-model.md) | Enhanced (mspp_/powerpagecomponent) vs Standard (adx_) — auto-detect pattern, entity mapping, component type codes |
| [`web-templates.md`](.github/skills/power-platform-pages/references/web-templates.md) | Upload HTML templates (EDM two-step file column vs SDM single PATCH), page templates, routing root/content page pair |
| [`anonymous-access.md`](.github/skills/power-platform-pages/references/anonymous-access.md) | AutoLogin gotcha, required site settings for mixed access, web page access control rules, cache delay |
| [`table-permissions.md`](.github/skills/power-platform-pages/references/table-permissions.md) | Create permission records, associate with web roles via $ref (N:N), scope values, idempotent GUID pattern |
| [`troubleshooting.md`](.github/skills/power-platform-pages/references/troubleshooting.md) | Auth redirect loops, 404 sub-pages, 403 portal API, template upload silently ignored, cache delays |

---

### `power-platform-copilot-studio` — Copilot Studio agents

For scripting **Copilot Studio agents** via the Dataverse Web API — bot/botcomponent entity structure, updating instructions (YAML blob, targeted regex approach), icons (authoring vs. Teams/M365 — two separate fields), publish-time validation gotchas, and knowledge source limitations.

Entry point: [`.github/skills/power-platform-copilot-studio/SKILL.md`](.github/skills/power-platform-copilot-studio/SKILL.md)

| Reference | Content |
|---|---|
| [`agent-structure.md`](.github/skills/power-platform-copilot-studio/references/agent-structure.md) | bot + botcomponent data model, component types, GPT component YAML shape, why knowledge sources can't be fully scripted, why from-scratch creation needs the portal |
| [`scripting.md`](.github/skills/power-platform-copilot-studio/references/scripting.md) | Instructions update via targeted regex on the YAML blob, YAML block scalar pattern, why not to parse+re-serialize |
| [`icons.md`](.github/skills/power-platform-copilot-studio/references/icons.md) | `iconbase64` vs `applicationmanifestinformation.teams.colorIcon/outlineIcon`, format requirements, Teams CDN caching delay |
| [`troubleshooting.md`](.github/skills/power-platform-copilot-studio/references/troubleshooting.md) | BotSynchronizationError, publish-time validation vs write-time, broken YAML after re-serialization, Teams icon cache |

---

## Where things happen

| Task | Where |
|---|---|
| Design/prompt a starting Code App | [vibe.powerapps.com](https://vibe.powerapps.com) |
| Build and manage Power Pages sites | [make.powerpages.microsoft.com](https://make.powerpages.microsoft.com) |
| Environment admin | [Power Platform Admin Center](https://admin.powerplatform.microsoft.com) |
| Tables, security roles, solutions (UI) | [make.powerapps.com](https://make.powerapps.com) |
| Everything above, scripted | Dataverse Web API (`<env>/api/data/v9.2/...`) |
| Build & deploy a Code App | `npm run build` + `npx power-apps push` |

## Core principle

Script the Dataverse Web API directly rather than clicking through portal UIs. Authenticate on demand with `az account get-access-token` (never persist tokens to disk), and write provisioning scripts to be idempotent (check-then-create / pre-assigned GUIDs with PATCH upsert) so they're safely re-runnable.

## Using a skill

Copy the relevant `skills/<name>/` directory into a Claude Code skills directory (project-local `.claude/skills/` or a plugin), or point Claude Code at this repo. Each skill's `SKILL.md` is the entry point; it loads only the reference docs relevant to the current task.

## License

MIT — see [LICENSE](LICENSE).
