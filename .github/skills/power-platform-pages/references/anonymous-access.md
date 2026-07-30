# Power Pages — Anonymous Access and Auth Settings

## The most common failure: AutoLogin

If the site redirects **every visitor** to Azure AD login even after setting anonymous-access site settings, the cause is almost always:

```
Authentication/OpenIdConnect/AzureAD/AutoLogin = true
```

This one setting overrides everything else. Set it to `false` first, then debug further if needed.

---

## Required site settings for mixed anonymous + Azure AD

For a site that allows anonymous browsing but also supports Azure AD login:

| Setting name | Value |
|---|---|
| `Authentication/OpenIdConnect/AzureAD/AutoLogin` | `false` |
| `Authentication/Registration/AzureADLoginEnabled` | `true` |
| `Authentication/Registration/ExternalLoginEnabled` | `true` |
| `Authentication/Registration/OpenRegistrationEnabled` | `true` |
| `Authentication/Registration/LocalLoginEnabled` | `false` |

### Upsert a site setting via API

```powershell
function Upsert-Setting ($name, $value) {
    $existing = Invoke-RestMethod "$apiBase/mspp_sitesettings?`$filter=mspp_name eq '$name' and _mspp_websiteid_value eq '$websiteId'&`$select=mspp_sitesettingid" -Headers $h
    if ($existing.value.Count -gt 0) {
        $id = $existing.value[0].mspp_sitesettingid
        Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_sitesettings($id)" -Headers $h `
            -Body (@{ mspp_value = $value } | ConvertTo-Json)
    } else {
        $newId = [System.Guid]::NewGuid().ToString()
        Invoke-RestMethod -Method Post -Uri "$apiBase/mspp_sitesettings" -Headers $h `
            -Body (@{
                mspp_sitesettingid           = $newId
                mspp_name                    = $name
                mspp_value                   = $value
                "mspp_websiteid@odata.bind"  = "/powerpagesites($websiteId)"
            } | ConvertTo-Json)
    }
}

Upsert-Setting "Authentication/OpenIdConnect/AzureAD/AutoLogin" "false"
```

For SDM replace `mspp_sitesettings` with `adx_sitesettings` and field prefix `adx_`.

---

## Web page access control rules

Even with correct site settings, a **web page access control rule** on the home page blocks anonymous access. These rules override site settings for that specific page.

### Check for access control rules

```powershell
# Check all rules for the site
$rules = Invoke-RestMethod "$apiBase/mspp_webpageaccesscontrolrules?`$filter=_mspp_websiteid_value eq '$websiteId'&`$select=mspp_webpageaccesscontrolruleid,_mspp_webpageid_value,mspp_right,mspp_scope" -Headers $h
$rules.value | Format-Table
```

### Delete a blocking rule

```powershell
Invoke-RestMethod -Method Delete -Uri "$apiBase/mspp_webpageaccesscontrolrules($ruleId)" -Headers $h
```

---

## Power Pages Studio — disable AutoLogin via UI

If the API change doesn't stick (the studio sometimes resets this), also disable it in the UI:

**Power Pages Studio → Set up → Identity providers → Microsoft Entra ID → uncheck "Log in automatically"**

This writes to the same site setting but ensures Studio's own state is consistent.

---

## Cache delay

Site settings changes take **2–3 minutes** to propagate through the portal cache. Do not re-test immediately after a setting change.
