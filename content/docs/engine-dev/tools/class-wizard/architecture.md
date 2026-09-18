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

The Class Creation Wizard builds directly on O3DE's original templating system -- the same `template.json` format and [`copyFiles`](../template-descriptor/#copyfiles) staging mechanism used by the plain `o3de create-from-template` CLI. It doesn't replace that system. It breaks "generate a class" into more granular stages, each with its own purpose, by layering wizard-specific structure on top of the standard format.

That layer starts with the file list itself. A `copyFiles` entry can carry wizard-only details -- `condition`, `cleanup_hint`, `isEditor`, `isInterface`, `excludeFromMerge` -- that O3DE's own template engine doesn't understand and copies through untouched. The wizard is what actually reads and acts on them, in a pass that runs *after* O3DE's own staging finishes: this is how an optional interface header or an EditorComponent variant gets included or dropped, and how references to it get scrubbed from the files that remain (see [Conditions](../template-descriptor/#conditions) and [Cleanup Hints](../template-descriptor/#cleanup-hints)).

Everything else the wizard adds lives in one place: the [`"class_wizard": {}` block](../template-descriptor/#the-class_wizard-block). Its presence is what marks a template as wizard-compatible at all -- a `template.json` without it is invisible to the wizard, even if `copyFiles` is otherwise well-formed. Inside that block:

- [**`input_vars`**](../template-descriptor/#input_vars) let a template expose its own, nonstandard inputs -- fields beyond the three the wizard always provides (`${Name}`, `${GemName}`, `${ComponentSuffix}`) -- as GUI widgets and CLI flags.
- [**`process_commands`**](../template-descriptor/#process_commands) lists, in chronological order, the steps needed to actually finish the job: everything that has to happen to the generated files and the surrounding gem once they're on disk.

### Execution Order

Put together, a single **Create** click or CLI invocation runs through this sequence:

1. **Discovery.** The wizard scans for every `template.json` with a `class_wizard` block, and every command plugin, across the engine, project, and gems.
2. **Input collection.** You pick a template and a destination, and fill in its `input_vars` -- resolved alongside the three built-in variables.
3. **Staging.** Nothing touches your gem yet. O3DE's own `create-from-template` mechanism copies every `copyFiles` entry into a **temporary staging directory**, substituting `${variable}` tokens in paths and content as it goes. Every file the template lists exists here -- including any that will never survive to your gem.
4. **Conditional exclusion and cleanup -- still in staging.** The wizard evaluates each staged file's `condition`. Files that fail are deleted from the staging directory outright. Then, for each file just deleted, the wizard scrubs references to it out of the files that remain, per its `cleanup_hint` -- stripping `#include` lines, EBus handler inheritance, and `BusConnect`/`BusDisconnect` calls. All of this happens on the temp copy; nothing scrutinized this way ever reaches your source tree.
5. **Merge.** Only the files that survived staging are copied into your gem's source tree. (`excludeFromMerge` files are the one exception -- they stay in the staging directory only, for a command like [`copy_file_to`](../commands/built-in-commands/#copy_file_to) to place somewhere other than the default location.)
6. **Commands -- against the real destination, not the staging copy.** `process_commands` now run in order against your actual gem. Each is gated by its own `condition`, and [registration commands](../commands/built-in-commands/#registration-commands) are additionally gated by `--automatic-register`.

### Commands Are Additive, Not Destructive

This is the detail that makes the command layer powerful rather than just convenient: a registration command never blindly overwrites, and it never assumes it's starting from nothing. [`register_system_component`](../commands/built-in-commands/#register_system_component), for example, first checks whether *this exact component* is already listed in the target module's `GetRequiredSystemComponents()` block. If it is, the command logs that and stops there -- running the wizard twice is safe. If it isn't, the command locates that block (already present in the module file from when the gem itself was scaffolded, long before this template ran) and injects one new entry into it, fixing up the trailing comma on whatever entry was there before. It never recreates the block and never disturbs entries that other templates, or other wizard runs, already put there. [`register_module_descriptor`](../commands/built-in-commands/#register_module_descriptor) follows the identical pattern for `CreateDescriptor()` calls.

[Data Asset](../template-descriptor/templates/data-asset/) calls `register_system_component` twice -- once for the runtime module, once for the editor module -- and both calls land the same way: find the existing block, add one entry, leave everything else alone. Generate a second Data Asset in the same gem later, and both calls repeat that -- additively, without touching the first one's registration.

### A Full-Stack Example

[Data Asset](../template-descriptor/templates/data-asset/) exercises nearly all of this at once: a conditional interface file with cleanup, three differently-shaped `input_vars` (a toggle, a required text field, a free-text field), and seven `process_commands` -- an unconditional gem dependency, two file registrations, a module descriptor, two system component registrations (runtime *and* editor, unconditionally), a [built-in command](../commands/built-in-commands/) that wires up a `GenericAssetHandler`, and one command gated behind its own `add_bus_interface` toggle. Once you've read this page, that template reads as a straightforward composition of the pieces above -- see its [full breakdown](../template-descriptor/templates/data-asset/) for the exact schema.

---

## Template and Command Discovery

This is how the wizard finds what it can offer, before you ever open it. If you're authoring a template or a command rather than just using one, this is where your files need to live -- see the [Template Descriptor Format](../template-descriptor/) for the schema a template needs, or [Command System](../commands/) to add a new command.

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

## Dynamic Variables

If you're deciding what a template should ask the user for, this is the mechanism -- the full field-by-field syntax lives in [Template Descriptor Format > input_vars](../template-descriptor/#input_vars).

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

This is the mental model for the rest of this guide. Pick your layer below, then go read its dedicated page:

- **O3DE templates** handle raw file scaffolding and variable substitution in source code
- **[Template descriptors](../template-descriptor/)** (`template.json`) define what the wizard should do with those files -- which commands to run, which variables to collect, which files are conditional
- **[Command plugins](../commands/)** execute the actual build integration -- modifying CMake files, module descriptors, and registration code

Together, these layers allow complex class creation workflows to be defined entirely in JSON and Python, without modifying the wizard core.

### A Self-Contained Creation System, Scoped to a Gem

This is what the layering is actually *for*. A gem's codebase has its own conventions -- naming patterns, base classes, EBus wiring, registration idioms -- that are specific to that gem and have no reason to live in the engine. The wizard's three layers exist so a gem author can bind all of that together:

- **The codebase** -- the gem's actual C++ source: its conventions, structure, and the shape of the classes it already has.
- **Templating** -- to reproduce the extensible, repeatable pieces of that codebase, so a new class comes out looking exactly like the gem's existing ones.
- **Commands** -- to satisfy whatever integration work is too complex for file-copying alone: wiring a new class into a gem-specific registry, dependency list, or bespoke system that only that gem's code understands.

None of this needs to be generic, and none of it needs to live in the engine. A `Templates/` directory and a `ClassWizardCommands/` directory, both scoped inside a single gem module, are enough to stand up a complete, gem-specific creation system -- one that produces classes indistinguishable from the gem's hand-written code, using commands built specifically to understand that gem's own patterns.

It deploys the same way every other part of a gem does: nothing to register, nothing to configure. The moment that gem is enabled for a project -- [discovered the same way as everything else](#template-and-command-discovery) -- its templates and commands are available in the wizard, exactly as if they'd shipped with the engine. That's the same on-demand, plug-in model O3DE's own gem system uses everywhere else; the wizard just extends it to code generation.

---

## GUI Internals

{{< note >}}
This section is about the implementation of the wizard's own PySide6 GUI application -- widget classes, layout, and how the process launches. It has nothing to do with using the wizard; see the [User Guide](/docs/user-guide/editor/class-wizard/) for that.
{{< /note >}}

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
