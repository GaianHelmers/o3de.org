---
title: "Level Component"
linkTitle: "Level Component"
description: "Generate a component that attaches to the level entity."
weight: 30
---

A component that attaches to the level entity rather than an individual game entity. Use this for per-level services -- weather systems, lighting controllers, or other level-wide logic that should exist once per level rather than once per entity.

{{< image-width src="/images/user-guide/editor/class-wizard/templates/level-template-screen.png" width="450" alt="The wizard with Level Component selected" >}}

**What you'll fill in:**

| Field | What it does |
|---|---|
| Add Bus Interface | On by default. Generates a companion EBus interface header so other components can talk to this one. |
| Add Editor Comp. | Only shown if your gem has an Editor module. Generates an `EditorComponent` wrapper so this component shows up and can be edited in the Editor's Entity Inspector. |

Fill in a **Component Name** and pick the **Gem** you're adding it to, then select **Create**.

For the full generated file list, process commands, and `template.json` schema, see [Level Component](/docs/engine-dev/tools/class-wizard/templates/level-component/) in the Developer Guide.
