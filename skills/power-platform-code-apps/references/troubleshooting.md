# Troubleshooting quick reference

Error message / symptom → likely cause → fix. Ordered roughly by where in the workflow you'll hit them.

| Symptom | Cause | Fix |
|---|---|---|
| `400 Required field 'PrimaryAttribute' is missing` (creating a table) | Tried to set `PrimaryNameAttribute` as a top-level string | Set `IsPrimaryName: true` directly on the primary attribute object instead. See [dataverse-web-api.md](dataverse-web-api.md) |
| `400 key ... not valid for resource RelationshipMetadataBase` | Addressed a relationship by `LogicalName` | Relationships must be addressed by `MetadataId` GUID — filter by `SchemaName` first to get the GUID |
| `0x8004f01f ... referenced by 1 other components` (deleting a table) | AppModule and/or Code App still reference the table as a component/data source | Remove AppModule components → republish → remove Code App data source → rebuild+redeploy Code App → delete relationships → delete table. See [model-driven-app.md](model-driven-app.md) |
| `400 query parameters not supported on complex type` (on `.../Privileges`) | Tried `$select` on an entity's `Privileges` navigation | Don't use `$select` here — request the full response and filter client-side |
| `400 Cannot read the value '3' as a quoted JSON string value` (AddPrivilegesRole) | Sent `Depth` as an integer | `Depth` must be the string `"Basic"`/`"Local"`/`"Deep"`/`"Global"` |
| `405 Operation not supported on EntityMetadata` (updating `IsAuditEnabled` or similar) | Used PATCH | Use PUT against `EntityDefinitions(<guid>)/IsAuditEnabled` with the full `@odata.type` annotation |
| `400 Invalid property 'metadataid' ... on type 'Microsoft.Dynamics.CRM.entity'` (AddAppComponents) | Used `metadataid` as the bind key | Use `entityid` for table components on an AppModule |
| Sitemap groups show as `Unknown0` / `Unknown1` | `Title LCID` doesn't match the org's base language | `GET /organizations?$select=languagecode` and match every `Title LCID` to it |
| `Entity 'appmodule' with Id=... Does Not Exist` right after creating an AppModule | Misleading error — usually really `DuplicateAppModuleUniqueName`; the record was created but your lookup only checked published apps | Also query `POST /appmodules/Microsoft.Dynamics.CRM.RetrieveUnpublishedMultiple` |
| `403 CodeAppOperationNotAllowedInEnvironment` (on `power-apps push`) | "Enable code apps" feature not turned on for the environment | Enable it in Power Platform Admin Center → Environments → Settings; wait a few minutes for propagation |
| `power-apps init` fails: "power.config.json already exists" | Project already scaffolded | Skip `init`; edit `power.config.json` fields directly (e.g. `environmentId`) |
| SDK runtime error after upgrading `@microsoft/power-apps` (e.g. missing `initialize` export) | Upgraded the SDK past what the app code was written against | Pin the SDK to the version the app was built with; add `@microsoft/power-apps-cli` as a separate devDependency instead of upgrading the SDK |
| Image/logo renders in local dev but is a broken image after `power-apps push` | File lives in `public/` and is referenced by string path, not bundled | Move to `src/assets/` and `import` it as an ES module. See [code-app-deployment.md](code-app-deployment.md) |
| `delete-data-source` deletes generated files for tables you wanted to KEEP | CLI bug in some versions — wipes all legacy-style `models`/`services` files, not just the target table's | Restore the wiped files for kept tables from a pristine backup/export; check for dangling field/import references (e.g. a lookup pointing at the now-deleted table) |
| Exported managed solution "works" in dev but a Code App is missing from the package | Code App component was created in the environment's Default Solution, not your named solution | `POST /AddSolutionComponent` with `ComponentType: 300` and your `SolutionUniqueName`; verify via `solutioncomponents` |
| `401` mid-script | Azure CLI access token expired (typically ~1 hour) | Re-run `az account get-access-token` and retry; write scripts to refresh the token automatically on 401 rather than failing |
| `400 0x8006088a "The 'startswith' function isn't supported for Metadata Entities"` (listing tables) | Tried `$filter=startswith(LogicalName,'ctso_')` on `EntityDefinitions` | No server-side prefix filter exists for metadata — fetch with `$filter=IsCustomEntity eq true` and filter by prefix client-side. See [dataverse-web-api.md](dataverse-web-api.md) |
| `RelationshipDefinitions?$filter=ReferencingEntity eq '...'` returns `200` with an empty array | Missing the `/Microsoft.Dynamics.CRM.OneToManyRelationshipMetadata` type-cast segment — `ReferencingEntity` doesn't exist on the base type, so the filter silently matches nothing instead of erroring | Always query `RelationshipDefinitions/Microsoft.Dynamics.CRM.OneToManyRelationshipMetadata?$filter=ReferencingEntity eq '...'`. See [dataverse-web-api.md](dataverse-web-api.md) |

## General debugging approach

1. Read the full Dataverse error response body, not just the HTTP status — it usually names the exact field/property that's wrong.
2. When an error message seems inconsistent with what you did (e.g. "entity does not exist" right after you just created it), consider that it may be a **misleading wrapper around a different underlying error** — check for duplicate-name conflicts, unpublished-state visibility gaps, or eventual-consistency delays before assuming your create request failed outright.
3. When in doubt about a Web API resource's exact key/property names, fetch `<environmentUrl>/api/data/v9.2/$metadata` and search it directly — it's the ground truth for every entity type, key, and bound action shape.
