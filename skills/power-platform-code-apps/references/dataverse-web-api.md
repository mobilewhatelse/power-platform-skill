# Dataverse Web API: tables, columns, relationships

All calls below go to `<environmentUrl>/api/data/v9.2/...` with headers:

```
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
OData-MaxVersion: 4.0
OData-Version: 4.0
```

Get the token with `az account get-access-token --resource "<environmentUrl>" --query accessToken -o tsv`. Refresh it on any `401`.

## 1. Publisher and solution

Create a publisher first (unless reusing an existing one) — it defines your customization prefix:

```http
POST /publishers
{
  "uniquename": "contosoapp",
  "friendlyname": "Contoso App",
  "customizationprefix": "ctso",
  "customizationoptionvalueprefix": 10000
}
```

Then a solution tied to that publisher:

```http
POST /solutions
{
  "uniquename": "ContosoIncidentManagement",
  "friendlyname": "Contoso Incident Management",
  "version": "1.0.0.0",
  "publisherid@odata.bind": "/publishers(<publisher-guid>)"
}
```

Every subsequent create call (tables, roles, etc.) should be scoped to this solution by passing the `MSCRM.SolutionUniqueName` HTTP header, or by explicitly adding the component afterwards (see [solution-alm.md](solution-alm.md)):

```
MSCRM.SolutionUniqueName: ContosoIncidentManagement
```

## 2. Create a table (entity)

```http
POST /EntityDefinitions
{
  "@odata.type": "Microsoft.Dynamics.CRM.EntityMetadata",
  "SchemaName": "ctso_Incident",
  "DisplayName": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Incident", "LanguageCode": 1033 }] },
  "DisplayCollectionName": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Incidents", "LanguageCode": 1033 }] },
  "OwnershipType": "UserOwned",
  "HasActivities": false,
  "HasNotes": false,
  "Attributes": [
    {
      "@odata.type": "Microsoft.Dynamics.CRM.StringAttributeMetadata",
      "SchemaName": "ctso_Title",
      "MaxLength": 200,
      "FormatName": { "Value": "Text" },
      "RequiredLevel": { "Value": "ApplicationRequired" },
      "IsPrimaryName": true,
      "DisplayName": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Title", "LanguageCode": 1033 }] }
    }
  ]
}
```

**Gotcha — `400 Required field 'PrimaryAttribute' is missing`:** don't set a top-level `PrimaryNameAttribute` string. Instead set `IsPrimaryName: true` directly on the primary attribute's own object, as shown above.

## 3. Add columns to an existing table

```http
POST /EntityDefinitions(LogicalName='ctso_incident')/Attributes
```

Body shape depends on attribute type:

| Type | `@odata.type` | Notes |
|---|---|---|
| Text | `Microsoft.Dynamics.CRM.StringAttributeMetadata` | needs `MaxLength`, `FormatName:{Value:"Text"}` |
| Multiline text | `Microsoft.Dynamics.CRM.MemoAttributeMetadata` | needs `MaxLength` |
| Whole number | `Microsoft.Dynamics.CRM.IntegerAttributeMetadata` | needs `MinValue`/`MaxValue`/`Format` |
| Decimal | `Microsoft.Dynamics.CRM.DecimalAttributeMetadata` | needs `Precision` |
| Date only / DateTime | `Microsoft.Dynamics.CRM.DateTimeAttributeMetadata` | `Format: "DateOnly"` or `"DateAndTime"` |
| **Boolean** | `Microsoft.Dynamics.CRM.BooleanAttributeMetadata` | needs `OptionSet` of type `Microsoft.Dynamics.CRM.BooleanOptionSetMetadata` — **not** the generic picklist OptionSetMetadata; needs `TrueOption`/`FalseOption` with `Value`/labels |
| **Picklist / choice** | `Microsoft.Dynamics.CRM.PicklistAttributeMetadata` | needs `OptionSet` of type `Microsoft.Dynamics.CRM.OptionSetMetadata` with an `Options` array of `{Value, Label}` |
| Lookup | see relationships below | not created directly as an attribute |

Example boolean column:

```json
{
  "@odata.type": "Microsoft.Dynamics.CRM.BooleanAttributeMetadata",
  "SchemaName": "ctso_IsClosed",
  "DisplayName": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Is Closed", "LanguageCode": 1033 }] },
  "OptionSet": {
    "@odata.type": "Microsoft.Dynamics.CRM.BooleanOptionSetMetadata",
    "TrueOption": { "Value": 1, "Label": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Yes", "LanguageCode": 1033 }] } },
    "FalseOption": { "Value": 0, "Label": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "No", "LanguageCode": 1033 }] } }
  }
}
```

## 4. Create a lookup / relationship (Many-to-One)

Relationships are created as a single call that implicitly creates the lookup attribute on the "many" side:

```http
POST /RelationshipDefinitions
{
  "@odata.type": "Microsoft.Dynamics.CRM.OneToManyRelationshipMetadata",
  "SchemaName": "ctso_incident_affectedsystem",
  "ReferencedEntity": "ctso_affectedsystem",
  "ReferencingEntity": "ctso_incident",
  "Lookup": {
    "@odata.type": "Microsoft.Dynamics.CRM.LookupAttributeMetadata",
    "SchemaName": "ctso_AffectedSystemId",
    "DisplayName": { "@odata.type": "Microsoft.Dynamics.CRM.Label", "LocalizedLabels": [{ "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel", "Label": "Affected System", "LanguageCode": 1033 }] }
  }
}
```

**Gotcha — relationships are addressed by GUID, not by name, for GET/DELETE.** `RelationshipDefinitions(LogicalName='...')` returns `400 key ... not valid for resource RelationshipMetadataBase`. Instead:

```http
GET /RelationshipDefinitions?$filter=SchemaName eq 'ctso_incident_affectedsystem'&$select=MetadataId
DELETE /RelationshipDefinitions(<MetadataId-guid>)
```

This matters when you need to delete a table that has inbound/outbound lookups — delete the relationship (by GUID) before deleting the table.

## 5. Creation order

Build a dependency graph and create in tiers:

- **Tier 0**: reference/lookup tables with no dependencies (e.g. Region, Category, Status-type tables).
- **Tier 1**: primary entities that reference Tier 0.
- **Tier 2**: dependent/child tables that reference Tier 1 (and each other).

Within a script, create all Tier 0 tables and their columns first, then all relationships pointing at them, before moving to Tier 1, etc.

## 6. Publish

After schema changes, publish so the changes take effect in forms/views/APIs immediately:

```http
POST /PublishAllXml
```

(No body required.) Safe to call after every batch of changes; it's a full republish, not incremental, so for large environments prefer batching changes and publishing once at the end of a script rather than after every single call.

## 7. Idempotent existence checks

Before creating, always check:

```http
GET /EntityDefinitions?$filter=LogicalName eq 'ctso_incident'&$select=MetadataId
GET /EntityDefinitions(LogicalName='ctso_incident')/Attributes?$filter=LogicalName eq 'ctso_isclosed'&$select=MetadataId
```

A 404/empty result means "doesn't exist yet, safe to create"; a non-empty result means "already exists, skip".

## 8. Reading an existing schema (docs, drift-checks, tooling)

Don't trust a stale data-model export or an old design doc — the live environment is the only source of truth once anyone has clicked around in make.powerapps.com. Query it directly whenever you need to generate documentation, build a diagram, or verify a schema hasn't drifted from what's on paper.

List every custom table under your prefix:

```http
GET /EntityDefinitions?$filter=IsCustomEntity eq true&$select=LogicalName,SchemaName,DisplayName,DisplayCollectionName,MetadataId
```

**Gotcha — `startswith` is not supported on metadata entities.** `EntityDefinitions?$filter=startswith(LogicalName,'ctso_')` fails with `400 0x8006088a "The 'startswith' function isn't supported for Metadata Entities"`. There's no server-side prefix filter — pull everything with `IsCustomEntity eq true` and filter by prefix client-side (`.filter(e => e.LogicalName.startsWith('ctso_'))`).

List a table's custom columns (skips the ~20 standard system attributes every table has — createdon, ownerid, statecode, versionnumber, etc.):

```http
GET /EntityDefinitions(LogicalName='ctso_incident')/Attributes?$filter=IsCustomAttribute eq true&$select=LogicalName,AttributeType,DisplayName,RequiredLevel
```

List the relationships where a table owns the lookup (i.e. the table you'd see a "Region" or "Owner" column on):

```http
GET /RelationshipDefinitions/Microsoft.Dynamics.CRM.OneToManyRelationshipMetadata?$filter=ReferencingEntity eq 'ctso_incident'&$select=SchemaName,ReferencedEntity,ReferencingAttribute
```

**Gotcha — omitting the type-cast segment doesn't error, it silently returns nothing.** `RelationshipDefinitions?$filter=ReferencingEntity eq '...'` (without `/Microsoft.Dynamics.CRM.OneToManyRelationshipMetadata`) is a "successful" `200` with an empty `value: []` array, since `ReferencingEntity` only exists on the OneToMany subtype and the base `RelationshipMetadataBase` filter matches nothing. Easy to misread as "this table has no relationships" — always include the cast.

The result also includes standard system relationships (`business_unit_ctso_incident`, `owner_ctso_incident`, `lk_ctso_incident_createdby`, `team_ctso_incident`, etc.) — filter to `SchemaName` starting with your prefix to keep only the custom ones you actually authored.

## 9. Autonumber columns — the only genuinely synchronous "auto-fill" mechanism

If you need a column (e.g. a ticket/incident number) to always be populated without the user typing it, don't reach for a Power Automate flow that fills it in after create — see the "synchronous vs. asynchronous" gotcha in [multi-app-data-integrity.md](multi-app-data-integrity.md) for why that's fragile. Instead, convert the column to a native **Autonumber**:

```http
PUT /EntityDefinitions(LogicalName='ctso_incident')/Attributes(<attribute-MetadataId-guid>)
{
  "@odata.type": "Microsoft.Dynamics.CRM.StringAttributeMetadata",
  "AutoNumberFormat": "INC-{SEQNUM:5}"
}
```

Then `POST /PublishAllXml`, and optionally seed the counter above whatever ad-hoc values already exist in the table (default seed is `1000`):

```http
POST /SetAutoNumberSeed
{ "EntityName": "ctso_incident", "AttributeName": "ctso_incidentnumber", "Value": 50000 }
```

This only works on **string** columns — there's no Autonumber equivalent for datetime/integer columns; those still need either manual entry or an async flow (accept the flow's inherent null-window for those, or make the column `Recommended` rather than `ApplicationRequired` if blocking on it is worse than allowing it blank).

**Gotcha — once set, Autonumber always overwrites whatever value a client sends, including a client-generated placeholder.** If another app (a Code App, say) generates its own guessed number client-side and sends it on create, Dataverse silently replaces it with the real autonumber — the client's own success message may briefly show the wrong value until it re-reads the record, but the persisted data is always correct. This is a one-time, disclosed trade-off worth explaining to whoever owns the other app, not a bug to work around.

## 10. `RequiredLevel` is enforced client-side, and only for fields actually on the form

Setting a column's `RequiredLevel` to `ApplicationRequired` makes Unified Interface block Save with a red asterisk — but **only if that field actually appears on the form the user is looking at.** The platform itself accepts a `POST`/`PATCH` with the field omitted regardless of `RequiredLevel` — there is no server-side enforcement. Two ways this bites you in practice:

- A field genuinely isn't on the form you think it is (see the "near-duplicate field" gotcha in [model-driven-app.md](model-driven-app.md)) — the constraint silently does nothing.
- The metadata was published, but a user's already-open Unified Interface session may still be showing a cached copy of the form until they hard-refresh or reopen the app.

After marking any field `ApplicationRequired`, verify it's actually present on **every** form real users create/edit records from — don't assume the metadata change alone is sufficient:

```http
GET /systemforms(<formid>)?$select=formxml
```

...and check the `formxml` string for `datafieldname="ctso_yourfield"`. If multiple apps/forms exist for the same table, check all of them.
