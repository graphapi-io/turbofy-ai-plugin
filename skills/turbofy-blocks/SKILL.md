---
name: turbofy-blocks
description: Use when writing or editing a Turbofy block type's runtime React component — typically a file at block-types/<Name>/index.tsx inside a pulled Turbofy app. Covers the IBuildingBlockProps interface (config, dynamicData, locale, params, searchParams, slug, pageId), the auto-injected config.copies dictionary, locale/copy/navigation rules (no window.location, no t() wrappers, no hardcoded strings), client-side data fetching with @/api hooks (useTypeQuery, useListTypes, useListTypesByParent, useSearchTypes, useLinks, useTranslations, useFileDocuments, useUploadFile, useWsSubscription), the @/navigation Link/navigate helpers, UI/UX/accessibility/dark-mode design rules (typography, spacing, grid, color, motion, empty/loading states), and per-block-instance localization overrides. Use whenever you open or modify a block index.tsx file, or you need to decide between server-side dynamicData and client-side hooks.
---

# Turbofy Blocks

TODO: port §4 (Block types — IBuildingBlockProps, coding guidelines, UI & UX, accessibility, dark mode, motion, empty/loading states, iconography, content rules), §7 (Data fetching patterns — config vs dynamicData, SPA vs SSR, resolution hooks, search, websocket, performance guidelines), §8 (Building block links — `$$std.batchLink`, `useLinks`, @/navigation), and §9 (Common gotchas) from the source agent doc into this skill.

Source: previously `.opencode/agents/apps-manager.md` in the `my.graphapi.io` monorepo.
