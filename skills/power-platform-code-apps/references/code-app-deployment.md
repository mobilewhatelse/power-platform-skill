# Code App deployment: SDK/CLI setup, data sources, static assets

## SDK vs CLI package confusion

Power Apps Code Apps depend on **two separate npm packages** that must be version-compatible with each other and with the app code:

- `@microsoft/power-apps` — the runtime SDK the app code imports (`initialize()`, data client, etc.)
- `@microsoft/power-apps-cli` — the CLI (`power-apps init/push/add-data-source/...`), a **separate package** in newer versions

**Gotcha:** an old app scaffolded against `@microsoft/power-apps@0.5.x` breaks if you blindly upgrade to `1.x` — the newer SDK's runtime API changed (e.g. the `initialize` export moved/renamed) and app code written against 0.5.x stops compiling/running. If the app already works against a pinned SDK version, keep the SDK pinned and add the CLI as a separate devDependency instead of upgrading both:

```json
{
  "dependencies": { "@microsoft/power-apps": "^0.5.2" },
  "devDependencies": { "@microsoft/power-apps-cli": "^0.13.0" }
}
```

If both packages register a `power-apps` bin and you get a bin-name collision, invoke the CLI directly instead of via `npx`:

```bash
node node_modules/@microsoft/power-apps-cli/dist/Bin.js push
```

## power.config.json

Key fields:

```json
{
  "environmentId": "<target-environment-guid>",
  "appDisplayName": "My App",
  "buildPath": "./dist",
  "buildEntryPoint": "index.html",
  "connectionReferences": {},
  "databaseReferences": {
    "default.cds": {
      "dataSources": {
        "<friendlyKey>": {
          "entitySetName": "ctso_incidents",
          "logicalName": "ctso_incident",
          "isHidden": false
        }
      }
    }
  }
}
```

- `environmentId` must point at the target environment — if `push` fails with an environment mismatch, fix this and retry.
- If a project already has a `power.config.json`, don't run `power-apps init` (it refuses with "already exists") — just edit the fields directly.

## Adding/removing Dataverse data sources

```bash
npx power-apps add-data-source -a dataverse -t <table-logical-name>
npx power-apps delete-data-source -a dataverse -n <plural-registered-key>
```

**Gotcha:** `delete-data-source` takes the **plural** registered key (the `dataSources` object key in `power.config.json`, e.g. `incidents`), not the singular logical name (`incident`).

**Gotcha — `delete-data-source` can over-delete.** In some CLI versions it wipes ALL legacy-style generated model/service files under `src/generated/models` and `src/generated/services` (not just the targeted table's files), breaking the build for every OTHER table you wanted to keep. If this happens:
1. Keep a pristine copy of the original generated output (e.g. the original export zip, or a git commit made right after scaffolding) before running any destructive CLI command.
2. Restore the wiped `-model.ts`/`-service.ts` files for tables you're keeping from that pristine copy.
3. Check for dangling references: a model file for a kept table may reference a field pointing at a deleted table (e.g. a lookup) — manually remove that field and its import from both the model and its zod validator.

Regenerating hooks/validators (if your codebase separates those from models/services) is usually unaffected by this bug — only models/services in the older generation pattern are at risk. Always diff/build after running any generation command.

## Static asset bundling — the #1 "broken image after deploy" cause

Vite's `public/` folder is copied **verbatim** to `dist/` at build time, and this works fine locally (`vite dev`/`vite preview`). But the Power Apps Code App deploy tool (`power-apps push`) appears to only reliably upload files that are part of Vite's **bundled, hashed asset graph** — i.e. files under `dist/assets/*` that got there because something in your source imported them as a module.

**Symptom:** a logo/image referenced as `<img src="/logo.png" />` or `` src={`${import.meta.env.BASE_URL}logo.png`} `` renders fine in local dev but shows as a broken image once deployed via `power-apps push`, while CSS and JS (which ARE bundled via imports) work fine.

**Fix:** move the asset into `src/assets/` and import it as an ES module instead of referencing a `public/`-relative path:

```tsx
// broken after deploy:
<img src="/logo.png" alt="Logo" />

// also broken after deploy (same root cause, still a public/ path):
<img src={`${import.meta.env.BASE_URL}logo.png`} alt="Logo" />

// works:
import logo from '@/assets/logo.png';
// ...
<img src={logo} alt="Logo" />
```

This forces Vite to hash and place the file under `dist/assets/`, which `power-apps push` does upload. Apply the same fix to favicons referenced from `index.html` if they go missing after deploy (though `index.html`'s own `<link rel="icon">` pointing at a `public/`-root file is typically fine since `index.html` itself is always deployed — the risk is specifically with paths referenced *from within bundled JS/TSX*).

## Deployment sequence

```bash
npm install
npm run build     # fix all TypeScript errors before proceeding — don't deploy a broken build
npx power-apps push
```

**Gotcha — `403 CodeAppOperationNotAllowedInEnvironment`:** the target environment doesn't have the **"Enable code apps"** feature turned on. Power Platform Admin Center → Environments → select environment → Settings → find and enable the code apps feature. Allow a few minutes for propagation before retrying.

**Gotcha — auth/token issues:** `npx power-apps logout` then retry `push` to force a fresh interactive login.
