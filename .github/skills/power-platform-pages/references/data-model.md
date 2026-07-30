# Power Pages — Enhanced vs Standard Data Model

Power Pages environments use one of two data models. **Always auto-detect** at the start of every script — never hard-code entity names.

## Detection

```powershell
$useEDM = $false
$edm = try { Invoke-RestMethod "$apiBase/powerpagecomponents?`$top=1&`$select=powerpagecomponentid" -Headers $h } catch { $null }
$std = try { Invoke-RestMethod "$apiBase/adx_webtemplates?`$top=1&`$select=adx_webtemplateid" -Headers $h } catch { $null }

if ($edm)      { $useEDM = $true;  Write-Host "Enhanced Data Model (powerpagecomponent)" }
elseif ($std)  { $useEDM = $false; Write-Host "Standard Data Model (adx_webtemplate)" }
else           { Write-Warning "Could not detect data model — defaulting to Standard" }

# Detect site entity
$siteEntity = "adx_websites"
$siteTest = try { Invoke-RestMethod "$apiBase/powerpagesites?`$top=1&`$select=powerpagesiteid" -Headers $h } catch { $null }
if ($siteTest) { $siteEntity = "powerpagesites" }
```

## Entity mapping

| Concept | Enhanced (EDM) | Standard (SDM) |
|---------|---------------|----------------|
| Site | `powerpagesites` | `adx_websites` |
| All components | `powerpagecomponents` | separate entity per type |
| Web template | type=9 in `powerpagecomponents` | `adx_webtemplates` |
| Page template | type=3 in `powerpagecomponents` | `adx_pagetemplates` |
| Web page | type=1 in `powerpagecomponents` | `adx_webpages` |
| Web file | type=8 in `powerpagecomponents` | `adx_webfiles` |
| Table permission | `mspp_entitypermissions` | `adx_entitypermissions` |
| Web role | `mspp_webroles` | `adx_webroles` |
| Site setting | `mspp_sitesettings` | `adx_sitesettings` |
| Web page ACL rule | `mspp_webpageaccesscontrolrules` | `adx_webpageaccesscontrolrules` |

## EDM component type codes

Used in `powerpagecomponenttype` field:

| Code | Type |
|------|------|
| 1 | Web Page |
| 3 | Page Template |
| 8 | Web File |
| 9 | Web Template |

## EDM: read all components for a site

Useful for diagnosing what exists before scripting:

```powershell
foreach ($type in @(1, 3, 9)) {
    $label = @{1="WEB PAGES"; 3="PAGE TEMPLATES"; 9="WEB TEMPLATES"}[$type]
    Write-Host "`n== $label ==" -ForegroundColor Cyan
    $r = Invoke-RestMethod "$apiBase/powerpagecomponents?`$filter=powerpagecomponenttype eq $type and _powerpagesiteid_value eq '$websiteId'&`$select=powerpagecomponentid,name,content" -Headers $h
    foreach ($c in $r.value) {
        Write-Host "  $($c.powerpagecomponentid)  $($c.name)"
        Write-Host "  $($c.content)"
    }
}
```

## EDM: field prefix

EDM table permissions and web roles use the `mspp_` prefix (not `adx_`), even though they are separate entities from `powerpagecomponents`. Example: `mspp_entitypermissions`, `mspp_webroles`.
