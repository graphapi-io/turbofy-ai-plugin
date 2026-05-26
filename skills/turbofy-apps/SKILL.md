---
name: turbofy-apps
description: Use when building or modifying Turbofy apps via the Turbofy MCP — creating an app, pulling it locally, editing app.ts (pages, block-instance placement, app settings, i18n), pushing changes, migrating legacy @cms_* tables, or understanding the Apps CMS data model (App, Page, BuildingBlock, BuildingBlockType, Localization, Image, SlugMapping). Loads the MCP tool list, the unified workspace file layout (~/.turbofy/workspaces/<env>/<wsId>/apps/<appId>/), and the pull → edit → push workflow. Use whenever you are calling Turbofy_app_* or Turbofy_workspace_* tools, or you need to reason about how the pieces of an app fit together.
---

# Turbofy Apps

Your main goal is to create and develop apps inside the Turbofy platform. The companion skills `turbofy-blocks` (writing block React components) and `turbofy-dynamic-fields` (server-side dynamic-field JavaScript) cover those areas in detail.

---

## 1) What are Apps?

Apps are a powerful feature of Turbofy that enables users to dynamically create and run web applications directly within the Turbofy dashboard — no publishing or deployment required. These apps have direct access to users' data, making them seamlessly integrated with the platform.

### Data model

This section describes the core tables, their purpose, key fields, and how they relate to each other.

Tables are referenced **by name** in Turbofy MCP (e.g. `Page`, `Localization`).

The **source of truth for the Apps CMS app definition** is the decompiled workspace files (`app.ts` and `block-types/` under the app directory after `Turbofy_app_pull`). The MCP writes them to `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`. Use them to understand pages, blocks, block types, and macros.

Important: the core Apps CMS tables are **system types** and are **not part of the workspace schema**:

- System CMS tables: `App`, `Page`, `BuildingBlockType`, `BuildingBlock`, `Localization`, `Image`
- Their IDs are stable and come from `CmsOfTypeEnum.*` (not workspace-specific)
- `Turbofy_table_list` reflects the workspace schema only, so it may not show system CMS tables

If a workspace schema contains legacy `@cms_*` directive tables, app tooling will refuse to operate until the workspace is migrated to system CMS types. Contact the workspace owner to migrate before continuing.

In dynamic-field code, tables are often referenced **by table id** (e.g. `$$std.getRecord(<tableId>, ...)`). For workspace schema tables, IDs are workspace-specific; resolve them via:

- `Turbofy_table_list` (table names + ids + fields)

Exception: for the **system CMS tables**, use the stable IDs from `CmsOfTypeEnum` (do not look them up via `Turbofy_table_list`).

#### App

An **App** is the top-level container for a set of pages.

Key fields:

- `id`
- `name`
- `createdAt`, `updatedAt`

An app **has many Pages**.

#### Page

A **Page** represents a route in the webapp (e.g. homepage, product detail, cart).

Key fields:

- `id`
- `name`
- `slugConfig` (`Json`) — config used to resolve/construct route slugs. Format: `{ "<lang>": { "slug": "/path/[param]" }, "paramsCollectionMap": { "<param>": { "ofType": "<table-ID>", "slugField": "<field>" } } }`. For dynamic pages `paramsCollectionMap` is **required** — it maps `[param]` placeholders to collections. **`ofType` must be the table ID (not the table name)**; use `Turbofy_table_list` to find it. `slugField` is optional — when set, the named record field (e.g. `"name"`) is used as a human-readable slug segment.
- `localizedConfig` (`@dynamic_field`, typed as `String`) — returns page metadata derived from `slugConfig` and dynamic args. In this workspace it computes:
  - `canonicalPath` — the canonical URL path (with slugs resolved)
  - `params` — resolved entity IDs for dynamic segments (e.g. `{ articleId: "1234-..." }`). These are passed as `$$args.params` to building block dynamic fields.
  - `notFound` — `true` if any referenced entity doesn't exist
- `createdAt`, `updatedAt`

A page **belongs to an App** and **has many BuildingBlocks**.

#### BuildingBlockType

A **BuildingBlockType** defines a category of building block (e.g. "Navigation", "ProductTable", "Footer"). There are multiple instances of the same block type across the application.

Key fields:

- `id`
- `name`
- `defaultConfig` (`@dynamic_field`, typed as `String`) — default configuration for blocks of this type. Use it for static server-side data such as copies, links, and layout settings. It should depend on `$$args.lang`, not on route params.
- `defaultDynamicData` (`@dynamic_field`, typed as `String`) — default server-side data for blocks of this type. Use it for data that depends on `$$args.params` and `$$args.searchParams`. In SSR apps, this provides the initial data snapshot; later interactions should use hooks.
- `sourceCodeUrl` (`String`) — URL of the block type's source code. Points to a `.tsx` file (single-file block) or a `.source.zip` archive (multi-file block).
- `compiledCodeUrl` (`String`) — URL of the compiled/bundled block code.
- `dependencies` (`Json`) — block type's dependency metadata.
- `createdAt`, `updatedAt`

A building block type **has many BuildingBlocks**.

#### BuildingBlock

A **BuildingBlock** is a UI component instance placed on a page.

Key fields:

- `id`
- `name` (`@dynamic_field`) — derived from its `BuildingBlockType` (here it resolves to the type's `name`).
- `config` (`@dynamic_field`, typed as `String`) — configuration for rendering. In this workspace it resolves to the type's `defaultConfig`.
- `position` (`String`) — determines rendering order within the page. Convention: use multiples of 10 and (optionally) hierarchical positions like `20/10` to render a child block at position `10` inside the block at position `20`.
- `dynamicData` (`@dynamic_field`) — server-resolved data for the block. Use it for route-dependent initial data (for example data keyed by `$$args.params` or `$$args.searchParams`), especially in SSR apps.
- `createdAt`, `updatedAt`

Relationships:

- A building block **belongs to a Page** and **belongs to a BuildingBlockType**.

#### Localization

The **Localization** table stores translated strings for the CMS.

Key fields:

- `id` — follows `${lang}_${kind}_${entityId}` (see naming patterns below).
- `dictionary` (`Json`) — translation key/value map. Used by `$$std.translate(...)` and `$$std.batchTranslate(...)`.
- `content` (`String`) — freeform content (project-specific usage).
- `createdAt`, `updatedAt`

**Naming patterns** (canonical):

| Pattern                          | Example IDs             | Purpose                                                                      |
| -------------------------------- | ----------------------- | ---------------------------------------------------------------------------- |
| `{lang}_blocktype_{blockTypeId}` | `en_blocktype_2vaEf74y` | Block type copies — written by `Turbofy_app_push` from `record.ts` `localizations`. |
| `{lang}_block_{blockId}`         | `en_block_ghi789`       | Per-instance block overrides.                                                |
| `{lang}_page_{pageId}`           | `en_page_def456`        | Page-level localized content.                                                |

Note: `appId` is not included in the pattern since localizations already belong to an app via the `appId` field.

`{lang}_blocktype_{blockTypeId}` records are managed by the workspace — never edit them directly. They are produced by `Turbofy_app_push` from each block type's `record.ts` `localizations` field, and surfaced back into `record.ts` by `Turbofy_app_pull`.

**Legacy IDs**: older workspaces may contain Localization rows with ad-hoc IDs (`en_product`, `de_footer`, `en_navigation`, `en_table`, `de_search`, `en_manufacturer_2vaEf74y...`). They predate the canonical scheme. Inspect with `Turbofy_data_list` on the Localization table (`ofType: CmsOfTypeEnum.Localization`) and migrate to one of the patterns above when touching them.

#### SlugMapping

The **SlugMapping** table maps human-readable URL slugs to entity IDs, scoped by type and language.

Key fields:

- `id` — composite key: `{appId}#{typeId}#{lang}#{slug}` (e.g. `1234-1234-1234-1234#gbqlrk#en#what-to-eat`). Mutable — update to change the slug without deleting the record.
- `key` (`String`, `@sortby`) — composite sort key: `{appId}#{typeId}#{lang}#{entityId}#{timestamp}` (e.g. `1234-1234-1234-1234#gbqlrk#en#1234-abcd#2026-03-20T17:29:51.407Z`). Stable — one record per entity per language.

Usage:

- **Slug → entity**: query by sort key `eq` to find the entity ID
- **Entity → slug**: direct `Turbofy_data_get` by ID to read the slug from `key`
- Managed via the **Slug Management** UI in the page editor

The `pageLocalizedConfigCode()` macro resolves slugs automatically. The engine later passes resolved entity IDs as `$$args.params` to building blocks.

#### Image

The **Image** table stores image assets.

Key fields:

- `id` (`ID`) — image identifier.
- `url` (`String`) — image URL.
- `width`, `height` (`Integer`) — dimensions.
- `createdAt`, `updatedAt`

---

## 2) MCP tools

Use the `turbofy` MCP for production work and the `turbofy-alpha` MCP for the alpha stage. Both expose the same set of tools, prefixed `Turbofy_*`.

### Core rules

**Pass the workspace ID explicitly.** The new MCP does not track an "active workspace" — every tool that needs a workspace accepts a `workspaceId` argument. Discover IDs with `Turbofy_list_workspaces`. Within a pulled app directory the workspace is implicit from the directory path.

**Read before write.** Before updating or deleting a record, fetch it (`Turbofy_data_get`) to confirm IDs and current state. Before creating related records, list the parent/collection (`Turbofy_data_list` with the right `ofType`) to avoid duplicates.

**Verify after write.** After create/update/delete, fetch/list again to confirm the intended state. Before `Turbofy_app_push`, call it with `dryRun: true` first to preview the diff.

**Treat IDs correctly.** Workspace-schema table IDs are workspace-specific — resolve via `Turbofy_table_list`. System CMS table IDs are stable — use `CmsOfTypeEnum.App | Page | BuildingBlockType | BuildingBlock | Localization | FileDocument`.

**Know common limits.** `Turbofy_data_list` is capped (commonly 100 per request). Paginate with the returned `nextToken`.

### Complete tool list

**Workspaces & orgs:**

- `Turbofy_list_organizations`
- `Turbofy_list_workspaces`
- `Turbofy_workspace_pull` — pull workspace schema into `~/.turbofy/workspaces/<env>/<workspaceId>/` (writes `schema.ts` + `schema.base.json`)
- `Turbofy_workspace_push` — typecheck, compile, validate, and push the local `schema.ts`

**Apps (unified workspace):**

- `Turbofy_app_init` — create app + default Home page remotely AND scaffold the local workspace in one step
- `Turbofy_app_pull` — pull app into `~/.turbofy/workspaces/<env>/<workspaceId>/apps/<appId>/`
- `Turbofy_app_push` — compile schema + blocks, upload, reconcile from the unified workspace (supports `dryRun: true`)

**Tables:**

- `Turbofy_table_list` — list workspace tables with field names/types

**Data (generic CRUD across all tables, including system CMS tables):**

- `Turbofy_data_list` — list records (`ofType`, optional `nextToken`, `limit`, etc.)
- `Turbofy_data_get` — fetch a single record
- `Turbofy_data_create` — create a record
- `Turbofy_data_add_many` — bulk create
- `Turbofy_data_update` — update a record
- `Turbofy_data_delete` — delete a record

**Files:**

- `Turbofy_file_upload` — upload a file and create a FileDocument record

**Reading specific entity types via `Turbofy_data_*`:**

When you need pages/block types/building blocks/localizations/images, use `Turbofy_data_list` and `Turbofy_data_get` with the appropriate system CMS type ID:

| Entity              | `ofType`                          |
| ------------------- | --------------------------------- |
| App                 | `CmsOfTypeEnum.App`               |
| Page                | `CmsOfTypeEnum.Page`              |
| BuildingBlockType   | `CmsOfTypeEnum.BuildingBlockType` |
| BuildingBlock       | `CmsOfTypeEnum.BuildingBlock`     |
| Localization        | `CmsOfTypeEnum.Localization`      |
| Image (FileDocument)| `CmsOfTypeEnum.FileDocument`      |
| SlugMapping         | `CmsOfTypeEnum.SlugMapping`       |

Apps, pages, block types, and block instances are normally managed via the workspace files + `Turbofy_app_push` — the generic data tools are for one-off reads/edits and for things the workspace doesn't own (e.g. arbitrary Localization rows, SlugMapping entries, FileDocument records).

---

## 3) App & schema workflow

### Primary workflow: `Turbofy_app_init` → `Turbofy_app_pull` / edit / `Turbofy_app_push`

**For a brand new app**, call `Turbofy_app_init`. This creates the app + a default Home page remotely and scaffolds the local workspace in one step.

The recommended way to modify an existing app's schema, pages, block types, block instances, **and block type source code** is the unified workspace:

1. **Pull** — `Turbofy_app_pull` writes everything to `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`:
   - `schema.ts` — GraphQL data schema (DSL)
   - `app.ts` — schema re-export + page definitions + blocks (DSL); imports block types from `./block-types/index.js`
   - `app.base.json` / `schema.base.json` — serialized baseline used by push for diffing; do not edit by hand
   - `package.json` + `tsconfig.json` — scaffolded Node/TypeScript config so the directory is self-contained
   - `block-types/index.ts` — barrel re-exporting every block type variable
   - `block-types/{Name}/record.ts` — one **record** per block type (the `appBuilder.blockType({...})` call)
   - `block-types/{Name}/index.tsx` (+ siblings) — optional runtime React code for the block type, pulled from the `.source.zip` archive when present
2. **Edit** — modify files directly (add/remove/update pages, block types, blocks, schema, block source code).
3. **Push** — `Turbofy_app_push` compiles schema, compiles+uploads each block type that has source code on disk, and reconciles the app state.

Call `Turbofy_app_push` with `dryRun: true` to preview changes before applying them.

Block source location is **convention-based**: push looks for `block-types/{Name}/index.{tsx,ts}`. If absent, the block type is treated as sourceless (valid — useful for content-only blocks where the runtime is provided elsewhere). When present, push compiles + uploads it and writes the resulting URLs back to the CMS automatically.

The reconciler diffs the edited file against the current API state and executes only the necessary operations.

### File layout

After `Turbofy_app_pull` or `Turbofy_app_init`, files live under `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`:

```
~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/
  schema.ts                 # GraphQL data schema (DSL)
  app.ts                    # schema re-export + pages + blocks (DSL); imports from ./block-types
  app.base.json             # serialized baseline app state (do not edit)
  schema.base.json          # serialized baseline schema declaration (do not edit)
  package.json              # scaffolded so the directory is self-contained
  tsconfig.json
  block-types/
    index.ts                # barrel: re-exports every block type variable
    Navigation/
      record.ts             # one row of the BuildingBlockType table, in code
      index.tsx             # runtime React code (optional)
    ProductTable/
      record.ts
      index.tsx
    Dashboard/              # multi-file block type
      record.ts
      index.tsx             # entry point
      helpers.ts
      chart-utils.ts
    Footer/
      record.ts             # sourceless block type — record only, no index.tsx
```

`<environment>` is the active MCP environment (`alpha` or `prod`) — the `turbofy` MCP uses `prod`, `turbofy-alpha` uses `alpha`. `<workspaceId>` and `<appId>` are the IDs reported by `Turbofy_list_workspaces` and `Turbofy_app_pull` / `Turbofy_app_init`.

`record.ts` is a per-block-type manifest that maps 1:1 to a row in the `BuildingBlockType` table. It imports `appBuilder` and exports a single `const`:

```ts
// block-types/Navigation/record.ts
import { appBuilder } from "@graphapi-io/dsl-builders";

export const navigationBlock = appBuilder.blockType({
  id: "...",
  name: "Navigation",
  defaultConfig: `...`,
});
```

`record.ts` does **not** belong to the runtime bundle. The block's `index.tsx` must not import it — push and the iframe runtime keep them separate by design.

`block-types/index.ts` is a generated barrel that re-exports each block type from its sibling `record.ts`:

```ts
// block-types/index.ts
export { navigationBlock } from "./Navigation/record.js";
export { dashboardBlock } from "./Dashboard/record.js";
export { footerBlock } from "./Footer/record.js";
```

In `app.ts`, block types are imported from the barrel and used in page definitions:

```ts
export { schema } from "./schema.js";
import { appBuilder } from "@graphapi-io/dsl-builders";
import {
  navigationBlock,
  dashboardBlock,
  footerBlock,
} from "./block-types/index.js";

const home = appBuilder.page({ ... });

export const app = appBuilder.buildApp({
  name: "My App",
  i18n: { locales: ["en", "de"], default: "en" },
  pages: [home, productDetail],
  blockTypes: [navigationBlock, dashboardBlock, footerBlock],
});
```

### Localization workflow

Localized copies for block types live **inside the workspace**, alongside the block type definition. The agent edits them directly in `record.ts`; `Turbofy_app_push` materializes them into Localization rows in the CMS, and `Turbofy_app_pull` reverses the round trip.

#### `i18n` on `buildApp` — the source of truth for supported locales

Every workspace declares its supported languages in `buildApp({ i18n: ... })`. This is the only place that lists locales for the app.

```ts
export const app = appBuilder.buildApp({
  name: "My App",
  i18n: {
    locales: ["en", "de"], // every locale the app supports
    default: "en",         // root-locale redirect target AND copies-fallback locale
  },
  pages: [home, productDetail],
  blockTypes: [navigationBlock, dashboardBlock, footerBlock],
});
```

Rules enforced by `buildApp`:

- `i18n.locales` must be non-empty.
- `i18n.default` must appear in `i18n.locales`.
- Every block type's `localizations` keys must be a subset of `i18n.locales`. Adding a key for an unsupported locale is a hard error — the only way to localize into a new language is to first add it to `i18n.locales`.

Push reconciles `i18n` into `App.settings.i18n` on the server. Pull reads it back from there. There is no other place to declare locales.

> `i18n.default` plays two runtime roles: (1) it controls the root-locale redirect, and (2) it is the locale used as the **fallback dictionary** for auto-injected `copies` (see "Auto-injected `copies`" below). They are intentionally the same value — there is no separate "fallback" knob.

#### Per-block-type `localizations` in `record.ts`

Each `record.ts` carries a `localizations` field — a record keyed by language code, whose values are flat dictionaries of copies:

```ts
// block-types/Navigation/record.ts
import { appBuilder } from "@graphapi-io/dsl-builders";

export const navigationBlock = appBuilder.blockType({
  id: "blktype_2vaEf74y",
  name: "Navigation",
  defaultConfig: `({ columns: 3 });`,
  localizations: {
    en: { home: "Home", about: "About", contact: "Contact" },
    de: { home: "Startseite", about: "Über uns", contact: "Kontakt" },
  },
});
```

At push time each non-empty entry becomes a `${lang}_blocktype_${blockTypeId}` Localization row. At runtime the dictionary for the current locale is exposed as `config.copies` to the React component (see the `turbofy-blocks` skill).

> **Auto-injected `copies` (hard lock).** When you set `defaultConfig` on a block type, the builder wraps your code so the runtime always returns `{ ...yourResult, copies: <localized dictionary> }`. You **never need to spread `blockTypeCopiesCode()` yourself** — and even if you tried to set a `copies` key in your object, it would be overridden. Same for block-instance `config`: any user-supplied code is wrapped to always emit `copies` as the merge of the block type's translations and the block instance's `localizations` overrides. Only `defaultConfig` and `config` are wrapped — `defaultDynamicData` and `dynamicData` are left untouched. The wrap is reversible: `Turbofy_app_pull` strips it back to your original source.
>
> **Fallback-language merge.** The wrap also fetches the dictionary for the app's `i18n.default` locale (`"en"` when `i18n` is unset) and merges it **under** the active locale. Any key missing in the active locale falls back to the default locale's value, so a half-translated locale degrades gracefully instead of returning `undefined`.
>
> For block instances the merge order is **language tier first, scope tier second** — every active-locale dict beats every fallback-locale dict; within a tier, block-instance beats block-type. From lowest → highest priority: block-type default → block-instance default → block-type active → block-instance active. This guarantees that translating a block type into the active locale wins over a default-locale-only block-instance override (otherwise the German page would render the English instance copy instead of the translated type copy).
>
> The default locale is baked into the wrap at `Turbofy_app_push` time from `i18n.default`, so changing it requires a re-push.
>
> **Single-batch fetch.** All required dictionaries (2 for block-type `defaultConfig`, 4 for block-instance `config`) are pulled in a single `$$std.batchGetRecordsByInputs(...)` call. That means **one runtime interruption** per dynamic field regardless of how many locales are merged — and the runner internally dedupes by `(ofType, id)` so a request where active === fallback collapses to a single underlying read.

#### Empty per-locale dictionaries are intentional

`Turbofy_app_pull` always seeds **every** locale declared in `i18n.locales` into each block type's `localizations`, even when the CMS has no data for that locale. You will see entries like:

```ts
localizations: {
  en: { joinAs: "You are", roomLabel: "Room" },
  de: { joinAs: "", roomLabel: "" },
}
```

The empty `de` strings are a contract: this app supports German, this block type has not been translated yet, and these are the keys that need translations. **Do not delete empty per-locale entries** — that would silently drop the language from the contract for that block type. Instead, fill in the translations.

If a workspace pull surfaces a locale you didn't expect (e.g. `fr: { ... }` while `i18n.locales` is `["en", "de"]`), it means a stray Localization row exists in the CMS. Pull preserves it so it doesn't get silently dropped, but `buildApp` will reject the push until you either add `"fr"` to `i18n.locales` or remove the locale from the block type.

#### Adding a new language

1. Add the locale code to `i18n.locales` in `buildApp` (and update `default` if needed).
2. Run `Turbofy_app_push`. This writes the updated `App.settings.i18n` to the server.
3. Run `Turbofy_app_pull`. Every block type's `record.ts` now has an empty dictionary for the new locale, ready to fill in.
4. Translate, then `Turbofy_app_push` again to materialize the localizations.

#### Per-block instance overrides on `appBuilder.block(...)`

Individual block instances can override the type's copies by setting `localizations` on the `block(...)` call inside a page. The shape mirrors block-type `localizations` — a record keyed by language code with flat dictionaries — but with two important differences:

- **It is optional.** Most blocks inherit copies from their type and leave `localizations` unset. Add it only when this particular instance on this particular page needs different copy than the default.
- **`Turbofy_app_pull` does not auto-seed empty dictionaries here.** Pull writes `localizations` only when at least one CMS override actually exists. This keeps page files clean — overrides stand out instead of drowning in empty placeholders.

```ts
// app.ts
const home = appBuilder.page({
  name: "Home",
  blocks: [
    appBuilder.block({
      type: navigationBlock,
      localizations: {
        en: { home: "Home" },
        de: { home: "Zuhause" }, // override only the keys that differ
      },
    }),
    // No `localizations` — inherits everything from `navigationBlock.localizations`.
    appBuilder.block({ type: footerBlock }),
  ],
});
```

Push materializes each non-empty dictionary into a `${lang}_block_${blockId}` Localization row. Validation in `buildApp` enforces the same locale subset rule as block types: every key under `localizations` must be in `i18n.locales`.

To remove an override, delete the `localizations` field (or the specific locale entry) and run `Turbofy_app_push`. The reconciler does not delete dropped Localization rows automatically — use `Turbofy_data_delete` on the Localization row (`ofType: CmsOfTypeEnum.Localization`) if you need the CMS row gone, otherwise the empty/stale row is harmless.

#### When to use which localization mechanism

- **Block type copies** — strings owned by a block type's React component (button labels, headings, error messages). Live in `record.ts` `localizations`. The builder auto-injects them into `config.copies` at runtime regardless of what you put in `defaultConfig`. **This is where almost all UI strings belong.**
- **Per-block instance overrides** — when one specific block instance on one specific page needs different copies than the block type default. Set `localizations` on the corresponding `appBuilder.block(...)` call in `app.ts` (writes `${lang}_block_${blockId}` rows on push). Or, for ad-hoc one-off edits, use `Turbofy_data_create` / `Turbofy_data_update` on `CmsOfTypeEnum.Localization`.
- **Page-level localized content** (`{lang}_page_{pageId}`) — page metadata (title, description) consumed by `pageLocalizedConfigCode()`. Manage via `Turbofy_data_*` on `CmsOfTypeEnum.Localization`.
- **Schema/data localizations** (e.g. translated entity descriptions) — store on the entity itself (the original "manufacturer description in `de`" use case). Use ad-hoc dictionaries the entity's dynamic fields can read via `$$std.translate(...)`.

### Schema workflow: pull → edit → push

1. **Pull** — call `Turbofy_workspace_pull` to write the current schema to `~/.turbofy/workspaces/<environment>/<workspaceId>/` as `schema.ts` + `schema.base.json`.
2. **Edit** — modify `schema.ts` using the builder DSL (see below). Do not touch `schema.base.json` — it is the diff baseline used by push.
3. **Push** — call `Turbofy_workspace_push` to typecheck, compile, validate, and push.

If the push fails, fix the reported errors in `schema.ts` and call push again.

### Builder DSL reference

The schema file imports and uses the builder:

```ts
import { dataBuilder as builder } from "@graphapi-io/dsl-builders";
```

#### Enums

```ts
const StatusEnum = builder.enumType("Status", ["Active", "Inactive"]);
```

#### Tables

```ts
const ProjectTable = builder.table(
  "Project",
  {
    name: builder.fields.string(),
    description: builder.fields.string(),
    status: builder.fields.enum(StatusEnum),
  },
  {
    directives: ["@required_oncreate", "@sortby"],
    publishingTypes: ["Queries", "Mutations"],
  },
);
```

#### Fields

Scalar: `string()`, `integer()`, `float()`, `boolean()`, `id()`, `email()`, `phone()`, `url()`, `date()`, `dateTime()`, `time()`, `timestamp()`, `json()`, `ipAddress()`

List: `listString()`, `listInteger()`, `listFloat()`, `listBoolean()`, `listId()`

Special: `enum(enumDeclaration)`, `dynamicField()`

All field methods accept optional opts:

```ts
builder.fields.string({ label: "Full Name", directives: ["@auth"] });
```

#### Parent-child relationships

```ts
const TaskTable = builder.table(
  "Task",
  {
    title: builder.fields.string(),
  },
  {
    firstParent: ProjectTable,
  },
);
```

Up to two parents are supported via `firstParent` and `secondParent`.

#### Build call

Every schema file must end with `builder.build()`:

```ts
export const schema = builder.build({
  enums: [StatusEnum],
  types: [ProjectTable, TaskTable],
  products: [],
});
```

### Critical schema rules

- **Do NOT provide IDs for new types, fields, or enums.** The builder auto-generates 6-character unique IDs. Existing types in the decompiled file already have `{ id: "..." }` — keep those as-is. Only omit IDs for things you are adding.
- **Declare parents before children.** Tables used as `firstParent` or `secondParent` must appear earlier in the file.
- **Default fields are automatic.** Every table automatically gets `id` (with `@connector`), `createdAt`, and `updatedAt`. Do not add these manually.
- **Include all types in the build call.** Every table and enum variable must appear in the `builder.build()` arrays, otherwise it will be silently dropped.

### Macros

The `app.ts` file imports macros from `@graphapi-io/declarations`:

```ts
import { macros } from "@graphapi-io/declarations";
const {
  blockConfigCode,
  blockDynamicDataCode,
  blockNameCode,
  blockTypeCopiesCode,
  pageLocalizedConfigCode,
} = macros;
```

| Macro                                     | Used in                     | Description                                                                                                                                                                                                                          |
| ----------------------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `blockTypeCopiesCode()`                   | _internal_                  | Resolves localized copies for the block type from `${lang}_blocktype_${$$self.id}` Localization records. Used as the schema default for `BuildingBlockType.defaultConfig` (when no user code is provided). **Do not spread it into `defaultConfig` yourself** — the dsl-builders auto-inject `copies` into every user-supplied `defaultConfig` (hard lock; see "Auto-injected `copies`"). |
| `blockConfigCode(BlockTypeTable.id)`      | _internal_                  | Inherits the parent block type's `defaultConfig` and merges block-instance copies. Used as the schema default for `BuildingBlock.config`. User-supplied `config` is wrapped automatically (the wrap merges block-type + block-instance translations into `copies`); you do not need to call this macro from `app.ts`.                                                                |
| `blockDynamicDataCode(BlockTypeTable.id)` | `BuildingBlock.dynamicData` | Inherits the parent block type's `defaultDynamicData`. Used in the schema definition and in per-instance `dynamicData` overrides to extend parent data.                                                                              |
| `blockNameCode(BlockTypeTable.id)`        | `BuildingBlock.name`        | Derives the block instance name from its block type. Used in the schema definition.                                                                                                                                                  |
| `pageLocalizedConfigCode()`               | `Page.localizedConfig`      | Computes localized page metadata (canonical path, notFound). Used in the schema definition.                                                                                                                                          |

---

## 4) Block instances

### Editing the DSL file

- **Add a block**: add a `block()` call inside a page's `blocks` array.
- **Remove a block**: delete the `block()` call.
- **Change position**: reorder blocks in the array (positions auto-compute) or set an explicit `position`.
- **Change config/dynamicData**: edit the `config` or `dynamicData` string on the block.

Example — adding a block to a page:

```ts
const home = appBuilder.page({
  id: "existing-page-id",
  name: "Home",
  blocks: [
    appBuilder.block({ id: "existing-block-id", type: hero }),
    appBuilder.block({ type: card, config: "({ title: 'New card' })" }), // NEW — no id means create
  ],
});
```

Blocks without an `id` in the DSL file are treated as new (create). Blocks with an `id` that no longer appear are deleted.

### Adjusting localization

Most localization edits happen in the workspace, not via MCP data tools:

- **Block type copies** — edit `block-types/{Name}/record.ts` `localizations`, then `Turbofy_app_push`. See "Localization workflow" above.
- **Per-block instance overrides** — edit `localizations` on the relevant `appBuilder.block(...)` call in `app.ts`, then `Turbofy_app_push`. See "Per-block instance overrides" above.
- **App-level supported locales** — edit `i18n.locales` in `buildApp`, then `Turbofy_app_push`.

Use the generic data tools below for ad-hoc edits or things the workspace doesn't own:

- `Turbofy_data_list` with `ofType: CmsOfTypeEnum.Localization` — list localizations (filter the returned items client-side by id prefix, e.g. starting with `en_page_<pageId>`)
- `Turbofy_data_get` with `ofType: CmsOfTypeEnum.Localization` — fetch a localization record
- `Turbofy_data_create` / `Turbofy_data_update` — create or update a localization record

These are the right tools for **page-level content** (`{lang}_page_{pageId}`) and one-off per-block instance edits when you don't want to round-trip through the workspace. Avoid using them to write `{lang}_blocktype_{blockTypeId}` rows directly — those are owned by the workspace and the next `Turbofy_app_pull` will reflect whatever is in `record.ts`. The same applies to `{lang}_block_{blockId}` rows when the block has `localizations` declared on the `block(...)` call in `app.ts`: pull will surface them and push will overwrite anything you wrote outside the workspace.

When a block is created via `Turbofy_app_push`, an empty `en_block_<blockId>` Localization row is created as a side-effect (so the editor UI has something to write to). It is harmless — pull skips empty dictionaries and the workspace remains the source of truth.

### Inspecting

Use the generic data tools to read CMS entities:

- Pages: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: CmsOfTypeEnum.Page`
- Block types: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: CmsOfTypeEnum.BuildingBlockType`
- Block instances: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: CmsOfTypeEnum.BuildingBlock`

For block instances on a specific page, list with `ofType: CmsOfTypeEnum.BuildingBlock` and filter client-side by `pageId`.

---

## See also

- **`turbofy-blocks`** — writing block-type runtime React components (`block-types/<Name>/index.tsx`): props, copies, UI/UX rules, client-side hooks.
- **`turbofy-dynamic-fields`** — server-side dynamic-field code (`defaultConfig`, `defaultDynamicData`, page `localizedConfig`): the `$$std` API, `$$args`, `$$self`, reserved `dynamicArgs` keys.
