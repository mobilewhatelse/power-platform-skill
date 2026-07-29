# Model-driven apps: sitemap + AppModule

Sometimes you want a model-driven app (native Dataverse forms/views) alongside a Code App — e.g. for admin/back-office screens — pointing at the same tables. This is done via an `AppModule` record plus a sitemap.

## 1. Build the sitemap XML

```xml
<SiteMap>
  <Area Id="area_main" ResourceId="Area_Main_Title">
    <Group Id="group_operations" ResourceId="Group_Operations_Title">
      <SubArea Id="subarea_incident" Entity="ctso_incident" ResourceId="Incident_Title" />
      <SubArea Id="subarea_actionitem" Entity="ctso_actionitem" ResourceId="ActionItem_Title" />
    </Group>
  </Area>
</SiteMap>
```

`ResourceId` labels need `Title` elements with an explicit `LCID` in the surrounding `<Titles>` — or simpler, just inline plain resource strings and set them via the `Sitemap` record's `sitemapxml` with `<Titles><Title LCID="1033" Title="Operations"/></Titles>` blocks per node.

**Gotcha — group/subarea labels show as `Unknown0`/`Unknown1`:** this happens when the `Title LCID` doesn't match the environment's actual base language. Don't assume `1033` (English) or guess from the UI locale — check the real base language first:

```http
GET /organizations?$select=languagecode
```

and set every `Title LCID` to match that value.

## 2. Create the sitemap record

```http
POST /sitemaps
{
  "sitemapname": "ctso_operations_sitemap",
  "sitemapxml": "<SiteMap>...</SiteMap>"
}
```

## 3. Create the AppModule

```http
POST /appmodules
{
  "name": "Contoso Operations",
  "uniquename": "ctso_operations_app",
  "webresourceid": null,
  "url": "main.aspx"
}
```

**Gotcha — garbled `Entity 'appmodule' with Id=... Does Not Exist` on first attempt:** this usually actually means `DuplicateAppModuleUniqueName` (error code `0x8005011F` / `-2147155681`) — the record often WAS created, but your follow-up "find by name" query only checked published apps. Also check unpublished ones:

```http
POST /appmodules/Microsoft.Dynamics.CRM.RetrieveUnpublishedMultiple
```

## 4. Link the sitemap and add table components

```http
PATCH /appmodules(<appmodule-guid>)
{ "appmodulesitemap_association@odata.bind": "/sitemaps(<sitemap-guid>)" }
```

Add each table as an app component:

```http
POST /appmodules(<appmodule-guid>)/Microsoft.Dynamics.CRM.AddAppComponents
{
  "AppComponents": [
    { "entityid@odata.bind": "/EntityDefinitions(LogicalName='ctso_incident')" }
  ],
  "Reset": false
}
```

**Gotcha:** the entity reference property is `entityid` (not `metadataid`) when the target's `@odata.type` is `Microsoft.Dynamics.CRM.entity`. Using `metadataid` fails with `400 Invalid property 'metadataid' ... on type 'Microsoft.Dynamics.CRM.entity'`. Confirm the correct key name for any table by checking the `$metadata` document (`EntityType Name="entity"` → `Key: entityid`) if in doubt.

To remove components later (e.g. before deleting a table — see below), use the mirror action:

```http
POST /appmodules(<appmodule-guid>)/Microsoft.Dynamics.CRM.RemoveAppComponents
{ "AppComponents": [ { "entityid@odata.bind": "/EntityDefinitions(LogicalName='ctso_incident')" } ] }
```

## 5. Validate and publish

```http
POST /appmodules(<appmodule-guid>)/Microsoft.Dynamics.CRM.ValidateApp
POST /PublishXml
{ "ParameterXml": "<importexportxml><appmodules><appmodule>...</appmodule></appmodules></importexportxml>" }
```

(`PublishAllXml` with no body also works and is simpler if you don't need a targeted publish.)

## 6. Grant access

Associate a security role with the AppModule so users with that role see it in the app picker:

```http
POST /appmodules(<appmodule-guid>)/appmoduleroles_association/$ref
{ "@odata.id": "<environmentUrl>/api/data/v9.2/roles(<role-guid>)" }
```

## Removing a table that's referenced by an AppModule

Deleting a table that still has `AppModuleComponent` entries fails with `0x8004f01f ... referenced by 1 other components`. Order matters:

1. `RemoveAppComponents` for the table on every AppModule that references it.
2. Republish (`PublishAllXml`).
3. If a Code App also has this table registered as a data source, remove it there too (see [code-app-deployment.md](code-app-deployment.md)) and rebuild+redeploy the Code App — Dataverse treats the Code App's own data-source registration as a dependency, separate from the model-driven AppModule.
4. Delete any relationships pointing at/from the table (by GUID — see [dataverse-web-api.md](dataverse-web-api.md)).
5. Only then delete the table itself, then `PublishAllXml`.
