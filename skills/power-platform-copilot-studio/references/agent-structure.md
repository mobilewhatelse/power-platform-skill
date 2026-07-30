# Copilot Studio — Agent Structure in Dataverse

A Copilot Studio agent is stored as a `bot` record with several child `botcomponent` records.

## Find an agent by name

```js
const bot = await api('GET', `bots?$filter=name eq '${agentName}'&$select=botid,schemaname,name,iconbase64,applicationmanifestinformation,synchronizationstatus`);
const botId = bot.value[0].botid;
```

## Find the agent's components

```js
const components = await api('GET', `botcomponents?$filter=_parentbotid_value eq ${botId}&$select=botcomponentid,name,schemaname,componenttype,data`);
```

### Component types

| `componenttype` | Purpose |
|----------------|---------|
| 9 | System topics (built-in conversation topics) |
| 15 | Orchestration / GPT component — holds `instructions` and `gptCapabilities` as a YAML blob |
| 16 | Knowledge source configuration |

### Find the GPT/orchestration component

```js
const gptComp = components.value.find(c => c.componenttype === 15 && c.schemaname.endsWith('.gpt.default'));
```

The `schemaname` follows the pattern `<botschemaname>.gpt.default`.

## The GPT component's `data` field

The `data` field is a YAML string with `kind: GptComponentMetadata`. The two main sections are:

- **`instructions`** — the agent's system prompt. A plain string (block scalar). Safe to script.
- **`gptCapabilities`** — knowledge sources, orchestration settings. Schema is not well-documented and depends on backend provisioning. Configure via the portal; script only what you can ground against a live example.

Example `data` shape:

```yaml
kind: GptComponentMetadata
instructions: |-
  You are a helpful assistant...
gptCapabilities: {}
aISettings:
  ...
```

## Knowledge sources — why they can't be fully scripted

Adding a Dataverse knowledge source via the portal creates a `botcomponent` with `componenttype: 16` and `data` like:

```yaml
kind: KnowledgeSourceConfiguration
source:
  kind: DataverseStructuredSearchSource
  skillConfiguration: <generated-guid>
```

That `skillConfiguration` GUID is tied to a backend search index provisioned when you click "Add" in the portal. Writing a matching YAML blob without that provisioning step does nothing useful — the search index doesn't exist. **This is a one-time portal step**, not something to reproduce via script.

## Creating an agent from scratch — not fully scriptable

`pac copilot create` requires a template extracted from an existing copilot. There is no reliable from-scratch creation path via the Web API or PAC CLI without an existing agent to clone. Create the agent once via the [Copilot Studio maker portal](https://copilotstudio.microsoft.com), then use scripts for everything after that.

## Publish an agent

```bash
pac copilot publish --bot <botid>
```

After publishing, check `synchronizationstatus` on the `bot` record for errors:

```js
const status = await api('GET', `bots(${botId})?$select=synchronizationstatus`);
console.log(status.synchronizationstatus);
```

A successful PATCH to any bot field does not guarantee a successful publish — see [troubleshooting.md](troubleshooting.md).
