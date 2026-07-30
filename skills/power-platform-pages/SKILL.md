---
name: power-platform-pages
description: Build, script, and deploy Power Pages (portals) backed by Dataverse — web templates, routing, table permissions, anonymous/authenticated access, site settings, and the Enhanced vs Standard Data Model split. Use when the user is working with a Power Pages site, mspp_/adx_ entities, portal auth settings, or Liquid/HTML web templates.
---

# Power Platform — Power Pages

This skill packages field-tested patterns for building and scripting **Power Pages** sites against the Dataverse Web API — from uploading web templates to fixing anonymous access and wiring table permissions to web roles.

It contains no project-specific business content — only the mechanics of the platform.

## When to use this

- The user is building or modifying a Power Pages site (web templates, routing, table permissions, web roles, site settings).
- The user needs to upload/replace HTML templates without going through the Power Pages Studio GUI.
- The user has anonymous access or auth redirect issues (site forces login even for public pages).
- The user needs to configure table permissions so the site's Web API calls (`/_api/...`) can read or write Dataverse rows.
- The user hits a routing 404 on sub-pages, a "403 Forbidden" from the portal Web API, or a template that uploads but never renders.

## Prerequisites

- Azure CLI (`az`) logged in as System Customizer or System Administrator: `az login`
- The target environment has a provisioned Power Pages site (created in [make.powerpages.microsoft.com](https://make.powerpages.microsoft.com))
- You know the `websiteId` (GUID of the `powerpagesites` or `adx_websites` record)

## Key concepts

| Concept | Details |
|---------|---------|
| **Data model** | Two variants exist: Enhanced (`mspp_`/`powerpagecomponent`) and Standard (`adx_`). Always auto-detect. See [references/data-model.md](references/data-model.md). |
| **Web Templates** | HTML+Liquid source stored as Dataverse records. EDM upload requires two steps (metadata PATCH + file PUT). SDM is a single PATCH to `mspp_source`. See [references/web-templates.md](references/web-templates.md). |
| **Routing** | Power Pages uses root/content page pairs. Broken routing is almost always a missing `mspp_masterwebpageid` link or a non-null `mspp_partialurl` on the content page. See [references/web-templates.md](references/web-templates.md). |
| **Anonymous access** | Site settings alone are not enough — `Authentication/OpenIdConnect/AzureAD/AutoLogin` must be `false`. Access control rules on pages override site settings. See [references/anonymous-access.md](references/anonymous-access.md). |
| **Table permissions** | Must be both created AND associated with web roles via a N:N relationship. A permission record with no role assignment does nothing. See [references/table-permissions.md](references/table-permissions.md). |

## Workflow

1. **Detect the data model** — run a probe GET against `powerpagecomponents` and `adx_webtemplates` to know which entity set to use. Do this at the start of every script. See [references/data-model.md](references/data-model.md).
2. **Upload web templates** — write templates as plain `.html` files locally; upload via Dataverse Web API (EDM: two-step with file column; SDM: single PATCH). See [references/web-templates.md](references/web-templates.md).
3. **Fix routing** — create/patch page templates, root pages, and content pages with correct linkage. See [references/web-templates.md](references/web-templates.md).
4. **Configure access** — for mixed public/authenticated sites, set the `AutoLogin` site setting to `false` and remove any stray access control rules from public pages. See [references/anonymous-access.md](references/anonymous-access.md).
5. **Set table permissions** — create permission records with fixed GUIDs (idempotent), then POST a `$ref` to each target web role. See [references/table-permissions.md](references/table-permissions.md).
6. **Wait for cache** — Power Pages caches templates and site settings for ~5 minutes. Hit `/_services/about` to trigger a purge, then wait before retesting.
7. **Diagnose** — use component-type queries against `powerpagecomponents` to see exactly what's in the site. See [references/troubleshooting.md](references/troubleshooting.md).

## Core principle: script the Web API, don't click through the Studio

Every mutation in this skill can be done with a plain HTTP call:

```
<environment-url>/api/data/v9.2/<EntitySet>(<id>)
```

authenticated with:

```powershell
$token = az account get-access-token --resource $envUrl --query accessToken -o tsv
```

Never persist that token — fetch it fresh at script start. Scripts are safe to commit as-is (zero secrets).

Write scripts to be **idempotent**: pre-assign GUIDs, use PATCH (upsert semantics), ignore "already exists" errors on `$ref` associations.
