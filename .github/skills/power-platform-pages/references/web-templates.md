# Power Pages — Web Templates, Page Templates, and Routing

## Web template upload

### Enhanced Data Model (EDM) — two-step upload

EDM stores the HTML source in a **file column** (`filecontent`), not an inline text field. Two separate calls are required:

```powershell
$templateId = "c1000001-0000-0000-0000-000000000001"  # pre-assigned GUID
$html = [System.IO.File]::ReadAllText(".\home.html", [System.Text.Encoding]::UTF8)

# Step 1: upsert the metadata record (content field = empty JSON object placeholder)
$body = @{
    powerpagecomponentid       = $templateId
    name                       = "EQ-Home"
    content                    = "{}"
    powerpagecomponenttype     = 9
    "powerpagesiteid@odata.bind" = "/powerpagesites($websiteId)"
} | ConvertTo-Json
Invoke-RestMethod -Method Patch -Uri "$apiBase/powerpagecomponents($templateId)" -Headers $h -Body $body

# Step 2: upload HTML source to the file column
$fileHeaders = @{
    Authorization    = "Bearer $token"
    "OData-MaxVersion" = "4.0"
    "OData-Version"  = "4.0"
    "Content-Type"   = "application/octet-stream"
    "x-ms-file-name" = "EQ-Home.html"
}
$bytes = [System.Text.Encoding]::UTF8.GetBytes($html)
Invoke-RestMethod -Method Put -Uri "$apiBase/powerpagecomponents($templateId)/filecontent" -Headers $fileHeaders -Body $bytes
```

**Gotcha:** PATCH-ing `mspp_source` or `content` with the raw HTML in EDM silently succeeds but the template never renders — the actual source lives in the file column, not in `content`.

### Standard Data Model (SDM) — single PATCH

```powershell
$body = @{
    adx_webtemplateid        = $templateId
    adx_name                 = "EQ-Home"
    adx_source               = $html
    "adx_websiteid@odata.bind" = "/adx_websites($websiteId)"
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Method Patch -Uri "$apiBase/adx_webtemplates($templateId)" -Headers $h -Body $body
```

---

## Page templates

A page template links a web page to a web template. In EDM, page templates are also `powerpagecomponents` (type=3); their `content` is a JSON string:

```powershell
$ptId  = "c2000001-0000-0000-0000-000000000001"
$wtId  = "c1000001-0000-0000-0000-000000000001"
$contentJson = "{`"webtemplateid`":`"$wtId`",`"type`":2,`"usewebsiteheaderandfooter`":false}"

$body = @{
    powerpagecomponentid       = $ptId
    name                       = "EQ-Home"
    powerpagecomponenttype     = 3
    content                    = $contentJson
    "powerpagesiteid@odata.bind" = "/powerpagesites($websiteId)"
} | ConvertTo-Json
Invoke-RestMethod -Method Patch -Uri "$apiBase/powerpagecomponents($ptId)" -Headers $h -Body $body
```

**Required fields in content JSON:**
- `type: 2` — Web Template (not Rewrite=1 or URL=3)
- `usewebsiteheaderandfooter: false` — required for full-page custom templates

In SDM, set `mspp_type = 756150001` and `mspp_usewebsiteheaderandfooter = false` on the `mspp_pagetemplates` record.

---

## Routing: root pages and content pages

Power Pages uses a **root/content page pair** for each URL. Breaking either side breaks routing.

| Property | Root page (`mspp_isroot = true`) | Content page (`mspp_isroot = false`) |
|----------|----------------------------------|--------------------------------------|
| `mspp_partialurl` | set (e.g. `warranty-check`) | **must be null** — content pages derive their URL from the root |
| `mspp_rootwebpageid` | points to **self** | — |
| `mspp_masterwebpageid` | — | points to the root page |
| `mspp_pagetemplateid` | set | set (same template as root, or different) |

### Fix broken routing

```powershell
$rootId    = "..."   # root page GUID
$contentId = "..."   # content page GUID
$ptId      = "..."   # page template GUID

# 1. Root page points to itself
Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_webpages($rootId)" -Headers $h `
    -Body '{"mspp_rootwebpageid@odata.bind":"/mspp_webpages(' + $rootId + ')"}'

# 2. Clear partialurl on content page
Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_webpages($contentId)" -Headers $h `
    -Body '{"mspp_partialurl":null}'

# 3. Link content page to root
Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_webpages($contentId)" -Headers $h `
    -Body '{"mspp_masterwebpageid@odata.bind":"/mspp_webpages(' + $rootId + ')"}'

# 4. Set page template on both
$ptBody = '{"mspp_pagetemplateid@odata.bind":"/mspp_pagetemplates(' + $ptId + ')"}'
Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_webpages($rootId)"   -Headers $h -Body $ptBody
Invoke-RestMethod -Method Patch -Uri "$apiBase/mspp_webpages($contentId)" -Headers $h -Body $ptBody
```

---

## Cache

Power Pages caches templates and page structure for approximately **5 minutes**. After any upload or routing fix, trigger a cache purge and wait:

```powershell
# Trigger cache refresh (fire-and-forget — ignore errors)
try { Invoke-WebRequest -Uri "https://<site>.powerappsportals.com/_services/about" -UseBasicParsing -TimeoutSec 30 } catch {}
Start-Sleep -Seconds 300
```

There is no API to force an immediate cache flush — waiting is the only option.
