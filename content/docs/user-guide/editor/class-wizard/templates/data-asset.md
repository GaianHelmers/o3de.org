---
title: "Data Asset"
linkTitle: "Data Asset"
description: "Generate a custom data asset type with its own asset handler."
weight: 60
---

A custom data asset class, complete with a dedicated system component that registers its asset handler at startup. Use this when you need a new custom asset type -- for example, a designer-authored data file with its own extension -- rather than a component that attaches to entities. This template is also known as Generic Asset, after the `GenericAssetHandler` it registers.

{{< image-width src="/images/user-guide/editor/class-wizard/templates/data-asset-template-screen.png" width="450" alt="The wizard with Data Asset selected" >}}

**What you'll fill in:**

| Field | What it does |
|---|---|
| Add Bus Interface | On by default. Generates a companion EBus interface header for this asset type. |
| File Extension | Required. The file extension the Asset Processor will recognize for this asset type (e.g. `dataasset`), without the dot. |
| Asset Group | The category this asset type appears under in the Asset Browser (e.g. `DataAssets`). Any name works -- it's not limited to a fixed list. |

Fill in a **Component Name** and pick the **Gem** you're adding it to, then select **Create**.

For the full generated file list, process commands, and `template.json` schema, see [Data Asset](/docs/engine-dev/tools/class-wizard/templates/data-asset/) in the Developer Guide.
