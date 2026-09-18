---
title: "Class Creation Wizard"
linkTitle: "Class Creation Wizard"
description: "Generate new O3DE C++ classes -- components, system components, and data assets -- from a set of ready-made templates."
weight: 450
---

The **Class Creation Wizard** generates the boilerplate for a new O3DE C++ class -- source files, CMake registration, and module descriptor entries -- from a template you pick and a few fields you fill in.

{{< image-width src="/images/user-guide/editor/class-wizard/class-wizard-gui.png" width="500" alt="The Class Creation Wizard window" >}}

New to the wizard? Start with [Getting Started](getting-started/) for a step-by-step walkthrough. Prefer the command line? See [CLI Quick Start](cli/).

## Launching

From the O3DE Editor, select **File > New Component**. Or launch it standalone from your O3DE engine's `Tools/ClassCreationWizard/` directory:

```bash
python ClassWizard.py --engine-path "C:\o3de" --project-path "C:\MyProject"
```

Both `--engine-path` and `--project-path` are required -- the wizard needs your project path to correctly resolve your build targets.

## Templates

| Template | What It Creates |
|---|---|
| [Basic Component](templates/basic-component/) | A standard game component. |
| [Level Component](templates/level-component/) | A component that attaches to the level entity. |
| [System Component](templates/system-component/) | An engine-level system component that activates automatically. |
| [LyShine Component](templates/lyshine-component/) | A UI component for the LyShine (UI 2.0) system. |
| [Data Asset](templates/data-asset/) | A custom data asset type with its own asset handler. Also known as Generic Asset. |
| [Attimage](templates/attimage/) | An attachment image asset for rendering features. |

Your project or gems may add their own templates too -- those show up in the same dropdown alongside these.

---

## Going Further

This page covers using the wizard from the Editor. For the full command-line reference, the `template.json` schema, the command plugin system, and how to author your own templates and commands, see the [Class Creation Wizard Developer Guide](/docs/engine-dev/tools/class-wizard/).
