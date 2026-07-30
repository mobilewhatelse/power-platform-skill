# Copilot Studio — Troubleshooting

---

## `BotSynchronizationError` / `MalformedBotChannelRegistrationIconException` after publish

**Symptom:** `pac copilot publish` completes without a CLI error, but the `bot` record's `synchronizationstatus` field contains an error like `MalformedBotChannelRegistrationIconException: BCR Icon must be a valid PNG`.

**Cause:** The value written to `iconbase64` (or `teams.colorIcon`) is not a valid square PNG. The Dataverse `PATCH` call accepts any base64 string with no validation — the format check only happens asynchronously when the Bot Channel Registration is synced, which occurs at publish time.

**Fix:**
1. Ensure the source image is a square PNG (e.g. 192×192). Convert from JPEG or non-square if needed.
2. Re-write the corrected base64 to `iconbase64`.
3. Publish again and check `synchronizationstatus`.

**Key lesson:** A successful `PATCH` to a bot/botcomponent field is not proof the value is valid. Always verify by publishing and checking `synchronizationstatus`, not by assuming the write succeeded means the publish will succeed.

---

## Instructions update appears to save but doesn't take effect

**Cause 1:** The `data` YAML blob was overwritten incorrectly (e.g. `gptCapabilities` was corrupted or removed), and the platform silently rejects the update at sync time.

**Fix:** Always re-read the component after patching to confirm the new instructions are actually present in `data`. If the read-back doesn't match what you wrote, something went wrong in the YAML serialization.

**Cause 2:** The agent wasn't republished after the component update.

**Fix:** `pac copilot publish --bot <botid>` after any botcomponent change.

---

## `data` field modification breaks the agent

**Symptom:** After a script modifies the `.gpt.default` component's `data` field, the agent stops working or behaves unexpectedly in the portal.

**Cause:** The entire YAML blob was re-serialized by a library, which reformatted `gptCapabilities`, `aISettings`, or other sections in a way the platform's own parser doesn't accept.

**Fix:** Use a targeted regex replace scoped only to the `instructions:` block — don't parse and re-serialize the whole document. See [scripting.md](scripting.md).

---

## Teams/M365 Copilot still shows old icon after successful publish

**Cause:** Teams and M365 Copilot cache bot avatars on their own CDN, independent of the Dataverse record and the client cache. The update is correct server-side, but hasn't propagated yet.

**Not fixable by script.** Wait for CDN propagation (typically minutes for the chat header, potentially hours for the chat list). Signing out of Teams and back in clears the client cache, but not the CDN.

---

## Agent's Dataverse knowledge sources stop working after a scripted change

**Cause:** A `botcomponent` with `componenttype: 16` (knowledge source) was modified or re-created by script, but the referenced `skillConfiguration` backend resource was not reprovisioned.

**Fix:** Delete the scripted knowledge source component and re-add the knowledge source via the Copilot Studio portal's Knowledge tab. The portal triggers the backend provisioning that the API write alone does not. See [agent-structure.md](agent-structure.md).

---

## General: how to check publish errors

```js
const bot = await api('GET', `bots(${botId})?$select=synchronizationstatus,name`);
console.log(bot.synchronizationstatus);
```

`synchronizationstatus` contains either `"Succeeded"` or a JSON/string error description. This is the only place publish-time validation failures surface — they do not appear as CLI errors or API errors.
