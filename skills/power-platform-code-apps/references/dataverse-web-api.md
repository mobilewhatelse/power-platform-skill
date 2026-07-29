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
