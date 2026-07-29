# Solution ALM: components, export, completeness

## Solution components

Every customization (table, role, sitemap, AppModule, Code App, etc.) is a row in the `solutioncomponents` table, linking a component GUID + type code to a solution. Common `componenttype` codes:

| Code | Component |
|---|---|
| 1 | Entity (table) |
| 20 | Role |
| 62 | SiteMap |
| 80 | AppModule |
| 300 | Canvas App / **Code App** |

Check what's in a solution:

```http
GET /solutioncomponents?$filter=_solutionid_value eq <solution-guid>&$select=componenttype,objectid
```

## Adding an existing component to a solution

If a component (e.g. a Code App) was created against the *default* solution instead of your named one — common when a tool like a Code App CLI creates its own registration without asking which solution to use — add it explicitly:

```http
POST /AddSolutionComponent
{
  "ComponentId": "<component-guid>",
  "ComponentType": 300,
  "SolutionUniqueName": "ContosoIncidentManagement",
  "AddRequiredComponents": false
}
```

Verify afterward:

```http
GET /solutioncomponents?$filter=objectid eq <component-guid> and _solutionid_value eq <solution-guid>
```

**Gotcha — a Code App can silently end up NOT in your named solution.** Symptom: everything works fine in the dev environment (because the Code App is present via the *Default Solution*), but an exported managed solution built from your named solution doesn't include it. Always check `solutioncomponents` for the Code App's `componenttype: 300` row explicitly scoped to your solution before considering the solution "complete" — don't assume presence in the environment implies presence in the solution.

## Exporting a managed solution

Via the UI (Solutions → your solution → Export → Managed), or:

```bash
pac solution export --path ./out.zip --name ContosoIncidentManagement --managed true
```

Always **Publish All Customizations** immediately before exporting — an export can otherwise capture stale/unpublished component state.

## Verifying a solution package is genuinely self-contained

Unzip and inspect directly rather than trusting the export dialog:

```bash
unzip -o out.zip -d out_inspect
cat out_inspect/solution.xml
```

Check:
- `<SolutionManifest><Managed>1</Managed>` and the expected `<Version>`.
- `<RootComponents>` lists every table, role, sitemap, AppModule, and the Code App (`type="300"`) you expect — count them.
- If the solution contains a Code App, the compiled bundle should be embedded under `CanvasApps/<schema-name>_CodeAppPackages/` (look for `index.html`, hashed JS/CSS under `assets/`, and any bundled image assets). If that folder is missing or empty, the Code App component wasn't actually included — go back to the "adding an existing component" step above.
- `<MissingDependencies>` entries referencing standard Microsoft platform packages (e.g. `AppModuleWebResources`, `AppFrameworkInfraExtensions`) are normal/benign — every Dataverse environment has these. Entries referencing YOUR OWN components are not benign and mean something is missing from the export.

## Importing into another environment

```bash
pac solution import --path ./out.zip
```

or via make.powerapps.com → Solutions → Import. After import, re-run the security role / user assignment steps for any users in the new environment — roles are imported as definitions but user assignments are NOT part of a solution and must be redone per environment.
