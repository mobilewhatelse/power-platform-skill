# Power Pages — Troubleshooting

Quick lookup: symptom → cause → fix.

---

## Site redirects all visitors to Azure AD login

**Cause:** `Authentication/OpenIdConnect/AzureAD/AutoLogin = true`

**Fix:** Set to `false` via API (see [anonymous-access.md](anonymous-access.md)) **and** in Power Pages Studio → Set up → Identity providers → Microsoft Entra ID → uncheck "Log in automatically". Both must agree.

---

## Template uploads successfully but page shows 404 or old content

**Cause 1:** Portal cache not yet refreshed (~5 min delay).  
**Fix:** Wait, then hit `/_services/about` to prod the cache.

**Cause 2 (EDM):** Template was PATCH'd with HTML in the `content` field instead of uploaded via the `filecontent` file column.  
**Fix:** Re-upload using the two-step approach (metadata PATCH + PUT to `/filecontent` with `Content-Type: application/octet-stream`). See [web-templates.md](web-templates.md).

---

## Sub-page URL returns 404

**Cause:** Routing linkage is broken. Most likely:
- Content page still has `mspp_partialurl` set (must be null)
- Content page's `mspp_masterwebpageid` not pointing to the root page
- Root page's `mspp_rootwebpageid` not pointing to itself

**Fix:** See the "Fix broken routing" section in [web-templates.md](web-templates.md).

---

## Portal Web API (`/_api/...`) returns 403 Forbidden

**Cause 1:** No table permission record exists for that table.  
**Cause 2 (most common):** Permission record exists but is not associated with any web role.  
**Fix:** Create the permission AND POST the `$ref` to Anonymous Users / Authenticated Users. See [table-permissions.md](table-permissions.md).

---

## Table permission already exists error on `$ref` association

**Symptom:** POST to `/$ref` returns error mentioning "duplicate" or "already exists".  
**Cause:** The role is already associated — this is not an error.  
**Fix:** Ignore silently. Pattern: `if ($detail -notmatch 'duplicate|already exists') { throw }`.

---

## Site settings don't take effect immediately

**Cause:** Portal caches site settings; changes take 2–3 minutes.  
**Fix:** Wait before retesting. For `AutoLogin` in particular, also verify the setting is correct in Power Pages Studio (Studio sometimes resets it on publish).

---

## `az account get-access-token` returns non-JWT output

**Symptom:** Token starts with something other than `eyJ`, script fails.  
**Cause:** Not logged in, wrong tenant, or CLI error output mixed into the token string.  
**Fix:** Run `az login` (or `az login --tenant <tenantId>`), then re-run. Check with `$token -match '^eyJ'`.

---

## `powerpagecomponents` returns 404 but site exists

**Cause:** The environment uses the Standard Data Model (pre-2022 provisioned portal).  
**Fix:** Use `adx_webtemplates`, `adx_webpages`, `adx_pagetemplates` instead. See [data-model.md](data-model.md).

---

## Template renders correctly in Studio preview but not on live site

**Cause:** Studio preview uses a privileged context that bypasses table permissions and site settings. Live site applies all restrictions.  
**Fix:** Test on the live URL (not Studio preview) after each permission or auth-settings change.
