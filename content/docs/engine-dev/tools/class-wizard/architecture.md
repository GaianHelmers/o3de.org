---
linkTitle: Architecture
title: Class Creation Wizard Architecture
description: How the Class Creation Wizard discovers templates and commands, and how its layered generation system fits together.
weight: 1
---

{{< note >}}
This information is for developers extending or modifying the **Class Creation Wizard** itself. If you're a user looking to generate a new class, see the [Class Creation Wizard User Guide](/docs/user-guide/editor/class-wizard/).
{{< /note >}}

## What It Does

At its core, the wizard:

1. **Detects your project and gem sources.** On launch, it reads your project's `project.json`, resolves gem paths through the O3DE manifest, and builds a map of every gem available to your project -- including external gems outside the engine directory.

2. **Identifies CMake build targets.** It scans each gem's `CMakeLists.txt` files, parsing `o3de_add_target`, `ly_add_target`, `add_library`, and `add_executable` macros. This lets the wizard know exactly where to register new source files, module descriptors, and dependencies.

3. **Generates classes from templates.** Each template defines the files to create, the variables to substitute, and the post-creation commands to run. The wizard stages files via O3DE's `create-from-template` system, applies variable substitution, evaluates conditional file exclusion with reference cleanup, and merges results into your gem's source tree.

4. **Runs integration commands.** After file generation, the wizard executes a sequence of commands defined by the template: registering files in CMake, adding module descriptors, inserting system component entries, adding gem dependencies, and more. Each command operates directly on your project's build files.

---

## Template and Command Discovery

### Template Discovery

The `WizardTemplateScanner` scans for `template.json` files under a `Templates/` directory in three locations (priority order):

| Priority | Location | Path Pattern |
|---|---|---|
| 1 (highest) | Engine | `<engine_path>/Templates/*/template.json` |
| 2 | Project | `<project_path>/Templates/*/template.json` |
| 3 | Gems | `<gem_path>/Templates/*/template.json` |

Gem paths are resolved via the O3DE manifest API's `manifest.get_project_enabled_gems()`. If the manifest API is unavailable, the wizard falls back to manually parsing the project's `project.json` and the user's `o3de_manifest.json`.

A `template.json` must contain a `"class_wizard"` block to be recognized. Templates without this block are ignored. Templates are deduplicated by resolved directory path and sorted alphabetically by display name.

### Command Discovery

The `CommandPluginLoader` scans for Python files in three locations, in priority order: the engine's `Tools/ClassCreationWizard/commands/` directory, then a flat `ClassWizardCommands/` directory under the project, then a `ClassWizardCommands/` directory under each gem (alphabetical by gem name). Each command file uses `@CommandRegistry.register()` to self-register. Commands are loaded via `importlib` with collision detection -- duplicate command names raise warnings and the first registration wins.

---

## GUI Internals

### Launching

From within the O3DE Editor, the wizard is reachable via **File > New Component**. Standalone, it's launched from the command line:

```bash
python ClassWizard.py --engine-path "C:\o3de" --project-path "C:\MyProject"
```

Opens the graphical interface. `--project-path` is required alongside `--engine-path` -- without it, the wizard cannot correctly resolve and scrub build targets.

### Generation Flow

1. **Template Selection.** The top combo box lists all discovered templates. "Basic Component" is always pinned first; the rest are sorted alphabetically by display name.

2. **Input Fields.** Dynamic fields are generated from each template's `input_vars` definition:

   | Input Type | GUI Widget |
   |---|---|
   | `text` | QLineEdit text field |
   | `dropdown` | BoundedComboBox with static choices |
   | `toggle` | QCheckBox |
   | `int` | QSpinBox |
   | `float` | QDoubleSpinBox |

3. **Create Button.** Validates all required fields, resolves variables, and runs the command pipeline.

4. **Status Panel.** Shows per-command status as the pipeline executes: pending (grey) to active to success (green) or fail (red).

### GUI Features

- **Fusion theme** with custom QSS stylesheet and SVG arrow icons
- **BoundedComboBox**: QComboBox subclass with MAX_POPUP_HEIGHT (400px), pre-constrains the popup view then resizes
- **Project condition awareness**: Input fields with `show_if` conditions only appear when the selected gem satisfies the condition (e.g., `hasEditor` shows editor-related toggles only for gems with an Editor module)

---

## Dynamic Variables

Templates define **input variables** -- toggles, text fields, dropdowns, and numeric fields -- that appear in the GUI or map to CLI flags. These variables flow through every part of the system:

- **File names and paths** -- `${Name}`, `${GemName}`, `${ComponentSuffix}`
- **Conditional file inclusion** -- optional interface headers, choosing between runtime and editor modules
- **Command arguments** -- pass user-provided values like file extensions, asset groups, or pixel dimensions directly into post-creation commands
- **In-file text replacement** -- the `replace_text` command substitutes placeholder tokens in generated source files with variable values

### Built-in Variables

The wizard's own resolver seeds exactly three base variables:

| Variable | Source | Example |
|---|---|---|
| `${Name}` | `--component-name` or GUI "Component Name" field | `PlayerHealth` |
| `${GemName}` | Selected gem namespace | `GS_Interaction` |
| `${ComponentSuffix}` | Template's `component_suffix` field | `Component` |

`${SanitizedCppName}` and similar variables come from the underlying `o3de create-from-template` staging step, not from the wizard itself.

See the [Template Descriptor Format](../template-descriptor/) for the full variable and condition syntax.

---

## Layered Architecture

The combination of O3DE's template language, the wizard's descriptor format, and the modular command plugin system creates a layered code generation architecture:

- **O3DE templates** handle raw file scaffolding and variable substitution in source code
- **[Template descriptors](../template-descriptor/)** (`template.json`) define what the wizard should do with those files -- which commands to run, which variables to collect, which files are conditional
- **[Command plugins](../commands/)** execute the actual build integration -- modifying CMake files, module descriptors, and registration code

Together, these layers allow complex class creation workflows to be defined entirely in JSON and Python, without modifying the wizard core. A gem author can ship a custom template that creates specialized component types, registers them in the correct build targets, adds cross-gem dependencies, and configures asset processing -- all through the template descriptor alone.
