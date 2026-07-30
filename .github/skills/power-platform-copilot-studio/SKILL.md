---
name: power-platform-copilot-studio
description: Script and manage Copilot Studio agents via the Dataverse Web API — bot/botcomponent entity structure, setting instructions, icons (authoring vs. Teams/M365), and publish-time validation gotchas. Use when the user is working with a Copilot Studio agent, the `bots` or `botcomponents` Dataverse entities, or bot icons/manifests.
allowed-tools: [shell]
argument-hint: "which agent, icon, or botcomponent are you working on?"
user-invocable: true
---

# Power Platform — Copilot Studio

This skill packages field-tested patterns for scripting **Copilot Studio agents** against the Dataverse Web API — reading and modifying agent instructions, setting icons, understanding the bot/botcomponent data model, and the publish-time validation failures that don't surface as immediate API errors.

It contains no project-specific content — only the platform mechanics, distilled from a real end-to-end build.

## When to use this

- The user wants to script a Copilot Studio agent's Instructions without going through the maker portal.
- The user needs to set or update the agent's icon (authoring icon and/or Teams/M365 app icon — these are **different fields**).
- The user hit a `BotSynchronizationError` / `MalformedBotChannelRegistrationIconException` after publishing.
- The user wants to understand how a Copilot Studio agent is stored in Dataverse, or find an agent's component records.
- The user wants to know why a knowledge source can't be fully scripted.

## Prerequisites

- Azure CLI (`az`) logged in as a user with System Customizer or System Administrator on the target environment: `az login`
- The Copilot Studio agent must already exist (created once via [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)) — there is no reliable from-scratch script-only creation path (see [references/agent-structure.md](references/agent-structure.md))
- You know the agent's `botid` GUID (find it via `GET /bots?$filter=name eq '<agent name>'&$select=botid`)

## Key concepts

| Concept | Details |
|---------|---------|
| **Agent = bot + botcomponents** | A Copilot Studio agent is a `bot` record plus several child `botcomponent` records. The orchestration component (`componenttype: 15`) holds instructions as a YAML blob. See [references/agent-structure.md](references/agent-structure.md). |
| **Two separate icon fields** | The authoring icon (`iconbase64` on `bot`) and the Teams/M365 icon (`applicationmanifestinformation.teams.colorIcon`) are unrelated — changing one has no effect on the other. See [references/icons.md](references/icons.md). |
| **Publish-time validation** | Some field validations (icon format, manifest schema) only fire asynchronously at publish/sync time, not at write time. A successful PATCH does not mean the value is valid. See [references/troubleshooting.md](references/troubleshooting.md). |
| **Knowledge sources** | Adding a Dataverse knowledge source via the portal provisions backend resources (a search index, a `skillConfiguration` record) that can't be reproduced by writing a `botcomponent` alone. This is a one-time portal step, not scriptable. See [references/agent-structure.md](references/agent-structure.md). |
| **Instructions: YAML, not plain text** | The `.gpt.default` component's `data` field is a YAML blob. Use targeted regex replace to update just the `instructions:` block — don't parse and re-serialize the whole document. See [references/scripting.md](references/scripting.md). |

## Workflow

1. **Find the agent and its components** — `GET /bots` by name, then `GET /botcomponents` filtered by `_parentbotid_value`. See [references/agent-structure.md](references/agent-structure.md).
2. **Update instructions** — targeted regex replace on the `.gpt.default` component's `data` YAML blob. See [references/scripting.md](references/scripting.md).
3. **Update icons** — `iconbase64` for the authoring icon, `applicationmanifestinformation` for Teams. Both must be square PNG. See [references/icons.md](references/icons.md).
4. **Publish and verify** — `pac copilot publish --bot <id>`, then check `synchronizationstatus` on the `bot` record for any publish-time errors. Don't assume a successful PATCH means the publish will succeed.

## Core principle

Same as the rest of this repo: script the Dataverse Web API directly, authenticate with `az account get-access-token`, write idempotent scripts. For Copilot Studio specifically: **always verify by publishing after each icon change** — the platform validates icon format asynchronously, not at write time, and errors end up in `synchronizationstatus`, not in the API response.
