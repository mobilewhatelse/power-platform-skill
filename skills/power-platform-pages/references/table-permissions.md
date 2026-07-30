# Power Pages — Table Permissions

Table permissions control which Dataverse tables the site's portal Web API (`/_api/...`) can access. A table permission record alone does nothing — it must be **associated with at least one web role**.

## Entity names

| | Enhanced (EDM) | Standard (SDM) |
|---|---|---|
| Permission entity | `mspp_entitypermissions` | `adx_entitypermissions` |
| Permission ID field | `mspp_entitypermissionid` | `adx_entitypermissionid` |
| Web role entity | `mspp_webroles` | `adx_webroles` |
| Web role ID field | `mspp_webroleid` | `adx_webroleid` |
| N:N nav property | `mspp_entitypermission_webrole` | `adx_entitypermission_webrole` |

## Detect which model is in use

```powershell
$permEntity = "mspp_entitypermissions"
$permIdField = "mspp_entitypermissionid"
$roleEntity  = "mspp_webroles"
$roleIdField = "mspp_webroleid"
$navProp     = "mspp_entitypermission_webrole"

$probe = try { Invoke-RestMethod "$apiBase/mspp_webroles?`$top=1" -Headers $h } catch { $null }
if (-not $probe) {
    $permEntity = "adx_entitypermissions"; $permIdField = "adx_entitypermissionid"
    $roleEntity  = "adx_webroles";         $roleIdField = "adx_webroleid"
    $navProp     = "adx_entitypermission_webrole"
}
```

## Scope values

| Value | Meaning |
|-------|---------|
| 756150000 | Global (all records) |
| 756150001 | Contact (current user's records) |
| 756150002 | Account (current user's account's records) |
| 756150003 | Self (for Contact table only) |
| 756150004 | Parent (child records of parent) |

## Create a permission record (idempotent)

```powershell
$permId = "e0000001-0000-0000-0000-000000000001"  # pre-assigned GUID for idempotency

$body = @{
    $permIdField                   = $permId
    mspp_entitylogicalname         = "eq_product"
    mspp_scope                     = 756150000   # Global
    mspp_read                      = $true
    mspp_write                     = $false
    mspp_create                    = $false
    mspp_delete                    = $false
    mspp_append                    = $false
    mspp_appendto                  = $false
    "mspp_websiteid@odata.bind"    = "/powerpagesites($websiteId)"
}
Invoke-RestMethod -Method Patch -Uri "$apiBase/$permEntity($permId)" -Headers $h `
    -Body ($body | ConvertTo-Json -Depth 5)
```

## Associate with a web role

```powershell
function Add-RoleRef ($permId, $roleId) {
    $body = @{ "@odata.id" = "$apiBase/$roleEntity($roleId)" } | ConvertTo-Json
    try {
        Invoke-RestMethod -Method Post `
            -Uri "$apiBase/$permEntity($permId)/$navProp/`$ref" `
            -Headers $h -Body $body | Out-Null
    } catch {
        $detail = $_.ErrorDetails.Message
        try { $detail = ($detail | ConvertFrom-Json).error.message } catch {}
        # "duplicate" or "already exists" = already associated, ignore silently
        if ($detail -notmatch 'duplicate|already exists') { Write-Warning $detail }
    }
}
```

## Find the Anonymous Users and Authenticated Users web roles

```powershell
$roles = (Invoke-RestMethod "$apiBase/$roleEntity?`$select=$roleIdField,mspp_name" -Headers $h).value
$anonRole = $roles | Where-Object { $_.mspp_name -match 'Anonymous' }
$authRole  = $roles | Where-Object { $_.mspp_name -match 'Authenticated' }
```

## Full example: create permissions for multiple tables

```powershell
$perms = @(
    @{ id="e0000001-0000-0000-0000-000000000001"; table="eq_product";      read=$true; write=$false; create=$false }
    @{ id="e0000001-0000-0000-0000-000000000002"; table="eq_document";     read=$true; write=$false; create=$false }
    @{ id="e0000001-0000-0000-0000-000000000003"; table="eq_serviceticket";read=$true; write=$true;  create=$true  }
)

foreach ($p in $perms) {
    $body = @{
        $permIdField                = $p.id
        mspp_entitylogicalname      = $p.table
        mspp_scope                  = 756150000
        mspp_read                   = $p.read
        mspp_write                  = $p.write
        mspp_create                 = $p.create
        mspp_delete                 = $false
        mspp_append                 = $false
        mspp_appendto               = $false
        "mspp_websiteid@odata.bind" = "/powerpagesites($websiteId)"
    }
    Invoke-RestMethod -Method Patch -Uri "$apiBase/$permEntity($($p.id))" -Headers $h `
        -Body ($body | ConvertTo-Json -Depth 5)

    Add-RoleRef $p.id $anonRole.$roleIdField
    Add-RoleRef $p.id $authRole.$roleIdField
    Write-Host "OK  $($p.table)"
}
```

## Why permissions don't work without role association

The portal evaluates permissions by: find all `mspp_entitypermissions` records for the requested table → filter to those associated with one of the current user's web roles → apply the most permissive matching record. If the permission record has no associated web roles, it matches no user, so the portal API returns 403 as if no permission existed.
