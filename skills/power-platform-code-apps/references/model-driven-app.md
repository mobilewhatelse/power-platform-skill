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
  "webresourceid": "<any-valid-webresource-guid>",
  "clienttype": 4
}
```

**Gotcha — `webresourceid` is required, not nullable.** It's the app's tile icon. Omitting it (or passing `null`) fails with `400 Attribute 'webresourceid' cannot be NULL`. It's a plain `Edm.Guid` property, not a navigable lookup — set it directly (not via `@odata.bind`). Any existing web resource GUID works (e.g. reuse the one from another app in the same environment — `GET /appmodules?$select=webresourceid` to find one); it doesn't need to be your own custom icon.

**Gotcha — omitting `clienttype` defaults it to `2` ("legacy web client"),** which makes the app show a persistent "This app is designed for the legacy web client and might have features or customizations that aren't supported in Unified Interface" warning banner to every user. Every correctly-configured Unified Interface app (including Microsoft's own system apps like Solution Health Hub) uses `clienttype: 4` — always set it explicitly. It's an `Edm.Int32` in range 1–31 (bitmask-like), `ApplicationRequired`. It's `PATCH`-able after the fact if you forgot it, despite a documented `CannotUpdateAppModuleClientType` error existing for some scenarios — re-query after publishing to confirm the write actually landed, since a read immediately after the PATCH can still show the stale value.

**Gotcha — garbled `Entity 'appmodule' with Id=... Does Not Exist` on first attempt:** this usually actually means `DuplicateAppModuleUniqueName` (error code `0x8005011F` / `-2147155681`) — the record often WAS created, but your follow-up "find by name" query only checked published apps. Also check unpublished ones (this is a `GET`-only **function**, not an action — note the empty parens):

```http
GET /appmodules/Microsoft.Dynamics.CRM.RetrieveUnpublishedMultiple()
```

## 4. Link the sitemap and add table components

A sitemap is linked to an AppModule the **same way tables are** — as an `AddAppComponents` component — not via a navigation-property PATCH. There is no `appmodulesitemap_association` you can bind to; attempting one fails with a confusing `"An undeclared property 'appmodulesitemap_association' ... no property value was found in the payload"` error that reads like an XML/OData bug rather than "this property doesn't exist."

**Gotcha — `AddAppComponents` is an unbound action.** `POST /appmodules(<guid>)/Microsoft.Dynamics.CRM.AddAppComponents` 404s (`Resource not found for the segment`). The real signature is unbound, at the service root:

```http
POST /AddAppComponents
{
  "AppId": "<appmodule-guid>",
  "Components": [
    { "@odata.type": "Microsoft.Dynamics.CRM.sitemap", "sitemapid": "<sitemap-metadata-guid>" },
    { "@odata.type": "Microsoft.Dynamics.CRM.entity", "entityid": "<table-metadataid-guid>" }
  ]
}
```

**Gotcha — `entityid` / `sitemapid` here are plain GUID *values*, not `@odata.bind` navigation targets.** Both are declared as plain `Edm.Guid` properties on these polymorphic `crmbaseentity` component items, not as navigable lookups. Writing `"entityid@odata.bind": "/EntityDefinitions(LogicalName='ctso_incident')"` throws an opaque OData deserialization error (`An ODataPrimitiveValue was instantiated with a value of type 'ODataEntityReferenceLink'...`) instead of a helpful "not bindable" message. Fetch the real GUID first (`GET /EntityDefinitions(LogicalName='ctso_incident')?$select=MetadataId`) and pass it as a literal value, with an explicit `@odata.type` on each component so the polymorphic collection knows how to deserialize it.

**Gotcha — a sitemap has *two different GUIDs*.** `sitemapidunique` (used to address the record in URLs, e.g. `GET/PATCH /sitemaps(<id>)`) and `sitemapid` (the true metadata primary key, required when referencing the sitemap as an `AddAppComponents` component) are **not the same value**. Using `sitemapidunique` in the components array fails with `The ID ... doesn't exist or isn't valid for the component type "SiteMap"`. Fetch both (`$select=sitemapid,sitemapidunique`) and use the right one for the right call.

To remove components later (e.g. before deleting a table — see below), use the mirror **unbound** action `POST /RemoveAppComponents` with the same `AppId` + `Components` shape.

## 5. Validate and publish

```http
POST /PublishAllXml
{}
```

**Note:** `ValidateApp` (as a bound `appmodules(<guid>)/Microsoft.Dynamics.CRM.ValidateApp` call) isn't exposed as a callable Web API action in at least some Dataverse versions — it 404s (`Resource not found for the segment`). It's optional (a warnings/lint check, not required for the app to function) — safe to skip and just publish directly. `PublishXml` with a targeted `ParameterXml` also works if you want a narrower publish, but `PublishAllXml` is simpler and fine for low-frequency provisioning scripts.

## 6. Grant access

Associate a security role with the AppModule so users with that role see it in the app picker:

```http
POST /appmodules(<appmodule-guid>)/appmoduleroles_association/$ref
{ "@odata.id": "<environmentUrl>/api/data/v9.2/roles(<role-guid>)" }
```

## 7. Build a real main form (FormXml)

A table created via the Web API gets an auto-generated main form containing only the primary name field + owner — not the full column set. That's usually too sparse to be usable. Adding the rest of the columns is still **plain declarative FormXml** — the exact same mechanism the Maker Portal's drag-and-drop form designer produces, no JavaScript, no PCF controls. This matters for licensing: FormXml-only forms never require an *additional* Power Apps license beyond what's already needed to open any custom Dataverse app. What **does** typically require extra/premium licensing is a Custom Page (an embedded Canvas App inside a model-driven app), a PCF control that calls an external service, Power Automate premium connectors, or AI Builder/Copilot features — none of that is needed here.

**Don't hand-author FormXml from scratch — extend an existing valid one.** Fetch the table's current form first:

```http
GET /systemforms?$filter=objecttypecode eq 'ctso_incident'&$select=name,type,formid,formxml
```

`type eq 2` is the Main form. Even the sparse auto-generated one is a proven-valid skeleton (Dataverse itself generated it) — typically just `<form><tabs><tab>...<columns><column><sections><section><rows><row><cell><labels><label/></labels><control/></cell></row>...` with **no** `<header>`/`<footer>`/`<events>` required. Add more `<section>`/`<row>`/`<cell>` blocks to it rather than constructing the wrapper yourself — this sidesteps most schema mistakes.

**Control `classid` values aren't centrally documented with actual GUIDs** — don't guess them from memory. The safest way to get correct ones: fetch a well-established **out-of-box** table's main form in the same environment (e.g. `contact`'s "Contact" form) and harvest real `<control classid="...">` values for each field type you need, by finding a control whose `datafieldname` matches a known OOB column of that type:

| Field type | Example OOB field (on `contact`) | classid |
|---|---|---|
| Single line text | `jobtitle` | `{4273EDBD-AC1D-40d3-9FB2-095C621B552D}` |
| Multiline text (memo) | `description` | `{E0DECE4B-6FC8-4a8f-A065-082708572369}` |
| Choice / picklist | `preferredcontactmethodcode` | `{3EF39988-22BB-4f0b-BBBE-64B5A3748AEE}` |
| Two options / boolean | `donotbulkemail` | `{67FAC785-CD58-4f9f-ABB3-4B7DDC6ED5ED}` |
| Date/DateTime | `birthdate` | `{5B773807-9FB2-42db-97C3-7A91EFF8ADFF}` |
| Lookup | `parentcustomerid` | `{270BD3DB-D9AF-4782-9025-509E298DEC0A}` |

These are stable, well-known IDs (Microsoft's own client renders them the same way across environments), but grounding them against a live form beats trusting a remembered value.

Minimal cell/control shape to add a field to a section:

```xml
<row>
  <cell id="{new-guid}" showlabel="true">
    <labels><label description="Severity" languagecode="1033" /></labels>
    <control id="ctso_severity" classid="{3EF39988-22BB-4f0b-BBBE-64B5A3748AEE}" datafieldname="ctso_severity" />
  </cell>
</row>
```

Then `PATCH /systemforms(<formid>) { "formxml": "<form>...</form>" }` and `POST /PublishAllXml`.

**Gotcha — label text needs XML escaping.** An unescaped `&` (e.g. a section labeled "Status & Deadlines") fails with an opaque `400 Error in parsing formxml. Line 1. Position 1409 ... An error occurred while parsing EntityName` — the position points at the ampersand, but the message gives no hint it's an escaping problem. Escape `&`, `<`, `>`, `"` in every label/description string before interpolating it into the XML.

## Removing a table that's referenced by an AppModule

Deleting a table that still has `AppModuleComponent` entries fails with `0x8004f01f ... referenced by 1 other components`. Order matters:

1. `RemoveAppComponents` for the table on every AppModule that references it.
2. Republish (`PublishAllXml`).
3. If a Code App also has this table registered as a data source, remove it there too (see [code-app-deployment.md](code-app-deployment.md)) and rebuild+redeploy the Code App — Dataverse treats the Code App's own data-source registration as a dependency, separate from the model-driven AppModule.
4. Delete any relationships pointing at/from the table (by GUID — see [dataverse-web-api.md](dataverse-web-api.md)).
5. Only then delete the table itself, then `PublishAllXml`.
