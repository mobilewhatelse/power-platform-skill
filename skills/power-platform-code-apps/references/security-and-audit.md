# Security roles and native auditing

Prefer Dataverse's built-in security and audit mechanisms over hand-rolled "governance tables" (an `AuditLog` table, a `SecurityRoleControl` table, etc.) — the platform already does this, natively, more reliably, and with zero app code.

## Security roles

### 1. Find privilege GUIDs for a table

```http
GET /EntityDefinitions(LogicalName='ctso_incident')/Privileges
```

**Gotcha:** you cannot add `$select` to this call — Dataverse returns `400 query parameters not supported on complex type` if you try. Just request the whole (small) response and pick out the privileges you need by `Name` (e.g. `prvReadctso_incident`, `prvWritectso_incident`, `prvCreatector_incident`, `prvDeletector_incident`, `prvAppendctso_incident`, `prvAppendToctso_incident`).

### 2. Create a role

```http
POST /roles
{
  "name": "Case Worker",
  "businessunitid@odata.bind": "/businessunits(<root-business-unit-guid>)"
}
```

Get the root business unit via `GET /businessunits?$filter=_parentbusinessunitid_value eq null`.

### 3. Grant privileges on the role

```http
POST /roles(<role-guid>)/Microsoft.Dynamics.CRM.AddPrivilegesRole
{
  "Privileges": [
    { "PrivilegeId": "<prvReadctso_incident-guid>", "Depth": "Global" },
    { "PrivilegeId": "<prvWritector_incident-guid>", "Depth": "Local" },
    { "PrivilegeId": "<prvCreatector_incident-guid>", "Depth": "Local" }
  ]
}
```

**Gotcha — `Depth` is a STRING enum, not an integer.** Sending `Depth: 3` (or any int) fails with `400 Cannot read the value '3' as a quoted JSON string value`. Valid values are the strings `"Basic"`, `"Local"`, `"Deep"`, `"Global"` (roughly: user-owned only → business unit → business unit + child BUs → whole org).

Repeat `AddPrivilegesRole` calls per table/privilege combination, or batch several `Privileges` entries into a single call — both work.

### 4. Assign a role to a user

Easiest via PAC CLI rather than raw Web API:

```bash
pac admin assign-user --environment <env-id> --user <user-upn-or-guid> --role "Case Worker"
```

Or raw Web API: `POST /systemusers(<user-guid>)/systemuserroles_association` with `@odata.id` pointing at the role.

## Native auditing

### 1. Enable auditing at the organization level

```http
PATCH /organizations(<org-guid>)
{ "isauditenabled": true }
```

### 2. Enable auditing per table

**Gotcha — this MUST be a PUT, not a PATCH.** `PATCH` against `EntityDefinitions(<guid>)` for a managed property like `IsAuditEnabled` returns `405 Operation not supported on EntityMetadata`. Use:

```http
PUT /EntityDefinitions(<entity-metadataid-guid>)/IsAuditEnabled
{
  "@odata.type": "Microsoft.Dynamics.CRM.EntityMetadata",
  "IsAuditEnabled": { "Value": true }
}
```

You need the entity's `MetadataId` GUID (not the logical name) for the URL — fetch it first with `GET /EntityDefinitions(LogicalName='ctso_incident')?$select=MetadataId`.

Repeat for every table you want audited. Then `POST /PublishAllXml`.

### 3. Reading the audit trail

Audit records live in the `audit` table and are queryable like any other table, or browsable in the UI at a record's "Audit History" (classic) — no custom UI needed:

```http
GET /audits?$filter=_objectid_value eq <record-guid>&$orderby=createdon desc
```

This gives you field-level change history, who changed what and when, for free — the exact thing a hand-rolled `AuditEntry` table tries to replicate, but transactionally consistent with the platform and impossible to bypass from app code.
