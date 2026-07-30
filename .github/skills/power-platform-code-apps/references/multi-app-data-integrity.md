# Multi-app data integrity: when more than one app shares a table

The moment a second app (a Model-driven App added alongside a Code App, or vice versa) can write to a table an existing app already reads, every unstated assumption the first app made about that table's data becomes a real, exploitable gap. This file is about closing that gap in a durable way — the failure mode below was hit **three times in a row** on the same underlying issue before the actual fix stuck, which is exactly why each attempted fix and why it wasn't enough is worth reading in order.

## The core problem: a "never null" field isn't actually enforced the way you think

A Code App's own creation dialogs (client-side, e.g. `zod` validation) typically require every field they collect before they'll submit. That's real protection *for records created through that dialog* — it does nothing once a **second** app (a Model-driven App on the same table, a direct API integration, a different Canvas App) can also write records. If the Code App's rendering code assumes a lookup is always populated (`incident.owner.fullName`, no null check) and the second app ever saves that field blank, the Code App crashes with a null-reference error the moment it tries to render that record.

## Attempt 1 (fragile): async automation to fill the gap

Tempting fix: lower the field's `RequiredLevel` to `Recommended` (so a Model-driven-App user isn't blocked) and add a Power Automate cloud flow, triggered on create, that fills the field in if it's blank.

**Why this failed**: the flow is asynchronous — it runs some seconds after the record is created. A real, form-created record sits with the field genuinely `null` for that window. Any other app reading the table during that window (a dashboard refresh, a list view, another user opening the record) hits exactly the crash you were trying to prevent. Testing this via direct API calls with a manual wait *looks* like it works — it hides the real race condition instead of fixing it.

**Lesson**: async automation cannot close a "this field is briefly null" window. If any other reader assumes non-null, don't rely on a flow to backfill it — either keep the field hard-required (`ApplicationRequired`, and accept the user must fill it in), or use a genuinely synchronous mechanism (below).

## Attempt 2 (also fragile): `ApplicationRequired` without checking the form

Reverting to `ApplicationRequired` seems like it should fully close the gap. It doesn't, for two independent reasons — see [dataverse-web-api.md §10](dataverse-web-api.md#10-requiredlevel-is-enforced-client-side-and-only-for-fields-actually-on-the-form):

1. `ApplicationRequired` is a **client-side, UI-rendered** constraint. If the field isn't actually present on the form the user is looking at (see the near-duplicate-field gotcha in [model-driven-app.md](model-driven-app.md)), nothing blocks the save — the constraint exists in metadata but was never exercised.
2. A published form change may not be visible in an already-open Unified Interface session until the user hard-refreshes or reopens the app — so even a genuinely correct fix can appear not to have taken effect for a session that predates the publish.

**Lesson**: after setting `ApplicationRequired` on a field, explicitly verify (by inspecting the live `formxml`) that the field is on every form real users actually use to create/edit records — for every app that touches this table, not just the one you're currently working on. Treat "I set RequiredLevel" and "this field can no longer be saved blank" as two separate claims, and verify the second one directly.

## Attempt 3 (works for string columns): native Autonumber

For a field like an incident/ticket number, the field's *value* doesn't need to come from the user at all — Dataverse can generate it itself, synchronously, as part of the same save transaction. See [dataverse-web-api.md §9](dataverse-web-api.md#9-autonumber-columns--the-only-genuinely-synchronous-auto-fill-mechanism). This has zero null-window, because there's no window — the value exists before the record is ever visible to any other reader. This only works for string columns, though; it doesn't help a lookup like "Owner" or a datetime like "Response Due".

## The actually durable fix: defend the reader, not just the writer

For fields Autonumber can't help with (lookups, datetimes, anything not a generated string), the fix that finally held was **not** another attempt to make the writer-side constraint airtight — it was making the *reading* app defensive about data it doesn't fully control the origin of:

```tsx
// Before: crashes if this Model-driven-App-created (or API-created,
// or bulk-imported) record has a blank owner
<Field label="Owner" value={incident.owner.fullName} />

// After: never crashes, regardless of which app created the record
// or what it left blank
<Field label="Owner" value={incident.owner?.fullName ?? 'Unassigned'} />
```

Apply optional chaining + a sane fallback everywhere a related record's field is read, and everywhere a related record's `.id` is used in a filter/comparison (`items.filter(i => i.relatedRecord?.id === x)`).

**Generalized lesson: when multiple apps share a Dataverse table, writer-side constraints (`RequiredLevel`, form design, client-side validation) reduce how often bad data appears, but only reader-side defensiveness (null-guards, fallback values) actually prevents a crash.** Constrain the writer where you reasonably can — it's good hygiene and reduces support burden — but never treat it as the only thing standing between your app and a null-reference crash. Grep the reading app's entire source for unguarded dereferences on every field that comes from a relationship, not just the one that already crashed.
