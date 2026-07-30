# Copilot Studio — Icons

A Copilot Studio agent has **two completely separate icon fields** that affect different surfaces. Changing one has no effect on the other.

## The two icon fields

| Field | Where it shows | Location | Format |
|-------|---------------|----------|--------|
| `iconbase64` | Copilot Studio maker portal only | `bot` record, top-level field | Square PNG, base64-encoded |
| `teams.colorIcon` | Teams and Microsoft 365 Copilot chat header/list | `bot.applicationmanifestinformation` JSON blob | Square PNG, base64-encoded |
| `teams.outlineIcon` | Teams tab strip (inverted color context) | Same JSON blob | Mostly-transparent white/monochrome PNG, 32×32 |

If you update `iconbase64` but not the Teams manifest icons, the Copilot Studio portal will show the new icon but Teams and M365 Copilot will still show the old one (or the default robot). Both fields must be updated independently.

## Update the authoring icon (`iconbase64`)

```js
const iconBytes = fs.readFileSync('./my-icon.png');
const iconBase64 = iconBytes.toString('base64');

await api('PATCH', `bots(${botId})`, { iconbase64: iconBase64 });
```

## Update the Teams/M365 icon (`applicationmanifestinformation`)

`applicationmanifestinformation` is a **JSON string** on the `bot` record — parse it, modify the `teams` section, and re-serialize. Do not use regex here (unlike the YAML instructions blob) — it's genuine JSON, safe to parse and stringify.

```js
const bot = await api('GET', `bots(${botId})?$select=applicationmanifestinformation`);
const manifest = JSON.parse(bot.applicationmanifestinformation);

manifest.teams.colorIcon   = fs.readFileSync('./color-icon.png').toString('base64');   // full color, square, 192×192 recommended
manifest.teams.outlineIcon = fs.readFileSync('./outline-icon.png').toString('base64'); // white monochrome, 32×32

await api('PATCH', `bots(${botId})`, {
  applicationmanifestinformation: JSON.stringify(manifest)
});
```

## Icon format requirements

- **`iconbase64` and `teams.colorIcon`**: must be a **square PNG**. Non-square or JPEG images pass the Dataverse PATCH with no error, but cause a fatal `MalformedBotChannelRegistrationIconException` at publish time (see [troubleshooting.md](troubleshooting.md)).
- **`teams.outlineIcon`**: 32×32 PNG, mostly transparent with white/monochrome content — used by Teams in inverted-color contexts (tab strip, dark mode). A simple white rounded-rectangle silhouette of the logo is sufficient; a full-detail logo at 32×32 is illegible anyway.

## Teams icon caching

Teams and M365 Copilot cache bot avatars server-side (CDN), independently of client-side cache. After a successful publish with the new icon:
- The **chat header** typically updates within minutes.
- The **chat list** (small avatar next to the conversation preview) may take significantly longer — potentially hours.
- Restarting the Teams app or signing out/in clears the **client** cache but not the server/CDN cache.

If the chat header shows the new icon but the chat list still shows the old one: this is normal CDN propagation delay, not an error.

## Always publish and verify

After updating either icon field, publish and check `synchronizationstatus`:

```bash
pac copilot publish --bot <botid>
```

```js
const status = await api('GET', `bots(${botId})?$select=synchronizationstatus`);
console.log(status.synchronizationstatus); // should be "Succeeded"
```

A PATCH that succeeded does not mean a publish will succeed — icon validation is asynchronous (see [troubleshooting.md](troubleshooting.md)).
