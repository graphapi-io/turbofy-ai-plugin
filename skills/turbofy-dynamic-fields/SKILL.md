---
name: turbofy-dynamic-fields
description: Use when writing or editing Turbofy dynamic-field code — JavaScript strings executed server-side in a QuickJS VM. Triggers include writing defaultConfig or defaultDynamicData on an appBuilder.blockType, per-instance config or dynamicData on an appBuilder.block, the localizedConfig of a page, or any other field declared as @dynamic_field. Covers the runtime model ($$self, $$args, $$std globals; how return values and errors are resolved), the $$std API ($$std.getRecord, $$std.listRecords, $$std.listRecordsByParent, $$std.batchGetRecords, $$std.batchGetRecordsByInputs, $$std.translate, $$std.batchTranslate, $$std.getImage, $$std.batchLink, $$std.get, $$std.getDynamicArg, $$std.getCurrentRecord), reserved dynamicArgs keys (fields, fieldsCode, skipDynamicResolver, lang, cmsOfTypes, slug, params, searchParams), serializable-shape rules, and the snapshot gotcha for cross-field reads. Use whenever you are authoring or debugging a dynamic field.
---

# Turbofy Dynamic Fields

TODO: port §6 (Writing dynamic field code — runtime model, `$$std` API reference, reserved `dynamicArgs` keys, dynamic field patterns, debugging checklist) from the source agent doc into this skill.

Source: previously `.opencode/agents/apps-manager.md` in the `my.graphapi.io` monorepo.
