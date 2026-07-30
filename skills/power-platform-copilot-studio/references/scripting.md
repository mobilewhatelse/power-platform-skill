# Copilot Studio — Scripting Instructions and Agent Settings

## Updating instructions

The agent's instructions live in the `.gpt.default` botcomponent's `data` YAML blob. Use a **targeted regex replace** on just the `instructions:` block — do not parse and re-serialize the whole document, since that risks corrupting `gptCapabilities` or `aISettings`.

```js
function yamlBlockScalar(text) {
  const indented = text.split('\n').map(line => `  ${line}`).join('\n');
  return `|-\n${indented}`;
}

async function setInstructions(gptComponentId, instructions) {
  const comp = await api('GET', `botcomponents(${gptComponentId})?$select=data`);

  if (!comp.data.startsWith('kind: GptComponentMetadata')) {
    throw new Error('Unexpected data shape — refusing to overwrite. Inspect manually first.');
  }

  const newBlock = `instructions: ${yamlBlockScalar(instructions)}`;
  const newData = comp.data.replace(/instructions:(\n(?!\S).*)*/, newBlock);

  await api('PATCH', `botcomponents(${gptComponentId})`, { data: newData });
}
```

**Why `|-` (block scalar)?** Multi-line instructions with `\n` embedded in a quoted scalar cause YAML parse errors in the portal and potentially at sync time. The `|-` block scalar (literal block, strip trailing newlines) is stable across any instruction text regardless of special characters, as long as each line is indented by at least 2 spaces.

**Why regex, not a YAML library?** No widely-available JS YAML library round-trips this specific YAML shape cleanly — they tend to reformat the `gptCapabilities` object or change quoting, which can break the platform's own YAML parser. A regex replace scoped to just the `instructions:` block is safer: it touches nothing else.

## Verify before reporting success

Always re-read the component after patching and confirm the instructions are present:

```js
const check = await api('GET', `botcomponents(${gptComponentId})?$select=data`);
console.log('New data:\n', check.data);
```

## Publish after changes

Instructions only take effect after publishing:

```bash
pac copilot publish --bot <botid>
```

Then verify there are no sync errors:

```js
const bot = await api('GET', `bots(${botId})?$select=synchronizationstatus`);
if (bot.synchronizationstatus !== 'Succeeded') {
  console.error('Publish failed:', bot.synchronizationstatus);
}
```
