# Power Platform Code Apps + Dataverse — Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill packaging field-tested, copy-pasteable patterns for building **Power Apps Code Apps** (React/Vite apps) backed by **Dataverse** — schema provisioning, security, native auditing, model-driven app sitemaps, and solution ALM, done directly against the Dataverse Web API.

No project-specific or organization-specific content — just the mechanics of the platform, distilled from real end-to-end build-and-deploy sessions.

## What's a Power Apps Code App?

A **Code App** is a normal React (or other supported framework) app, written and built with your own tooling (Vite, npm, TypeScript), that Power Platform can host, secure, and connect to Dataverse/connectors just like a low-code Canvas App — via the `@microsoft/power-apps` SDK and `power-apps` CLI. It's the "pro-dev" alternative to Canvas Apps and model-driven apps, useful when you want full control over UI code but still want Power Platform's data layer, security model, and ALM.

You can start one from scratch, or generate a starting point at [vibe.powerapps.com](https://vibe.powerapps.com) (prompt-based scaffolding) and take it from there — this skill assumes the latter is common but works either way.

## Where things happen

| Task | Where |
|---|---|
| Design/prompt a starting Code App | [vibe.powerapps.com](https://vibe.powerapps.com) |
| Environment admin (enable code apps, manage users) | [Power Platform Admin Center](https://admin.powerplatform.microsoft.com) |
| Tables, security roles, solutions (UI) | [make.powerapps.com](https://make.powerapps.com) |
| Everything above, scripted | Dataverse Web API (`<env>/api/data/v9.2/...`) — see this skill's references |
| Build & deploy the Code App | Local machine: `npm run build` + `npx power-apps push` |

## Using this skill

Copy `skills/power-platform-code-apps/` into a Claude Code skills directory (project-local `.claude/skills/` or a plugin), or point Claude Code at this repo. The entry point is [`SKILL.md`](skills/power-platform-code-apps/SKILL.md); it links out to focused reference docs so only the relevant one gets loaded for a given task:

- [`references/dataverse-web-api.md`](skills/power-platform-code-apps/references/dataverse-web-api.md) — creating tables, columns, relationships via raw Web API calls
- [`references/security-and-audit.md`](skills/power-platform-code-apps/references/security-and-audit.md) — security roles, privileges, native Dataverse auditing
- [`references/model-driven-app.md`](skills/power-platform-code-apps/references/model-driven-app.md) — sitemap + AppModule for model-driven apps alongside a Code App
- [`references/solution-alm.md`](skills/power-platform-code-apps/references/solution-alm.md) — solution components, export, verifying package completeness
- [`references/code-app-deployment.md`](skills/power-platform-code-apps/references/code-app-deployment.md) — SDK/CLI version gotchas, data sources, the static-asset bundling trap, deploy sequence
- [`references/troubleshooting.md`](skills/power-platform-code-apps/references/troubleshooting.md) — quick lookup table of error messages → causes → fixes

## Core principle

Script the Dataverse Web API directly rather than clicking through the portal UI for repetitive schema/security work. Authenticate on demand with `az account get-access-token` (never persist tokens to disk), and write provisioning scripts to be idempotent (check-then-create) so they're safely re-runnable.

## License

MIT — see [LICENSE](LICENSE).
