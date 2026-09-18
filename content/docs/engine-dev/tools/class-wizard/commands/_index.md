---
title: "Command System"
linkTitle: "Commands"
description: "How the Class Creation Wizard command system works, and a reference of all built-in commands."
weight: 500
---

Commands are the actions the Class Creation Wizard executes after generating template files. They handle build system integration -- registering files in CMake, adding module descriptors, inserting dependencies, and modifying generated source.

---

## How Commands Work

### Architecture

The command system is built on three classes in `command_plugin.py`:

- **`WizardCommand`** -- Abstract base class that all commands extend
- **`CommandContext`** -- Data object passed to every command's `execute()` method
- **`CommandRegistry`** -- Global registry that maps command names to their classes

Commands are invoked from the `process_commands` array in a template's `class_wizard` block. Each entry specifies a command name, its arguments, and an optional condition:

```json
{
    "command": "register_file_list",
    "args": { "component_name": "${Name}${ComponentSuffix}" }
}
```

### Command Discovery

The `CommandPluginLoader` scans for Python files in three locations, loaded in this order:

| Priority | Location | Namespace |
|---|---|---|
| 1 (highest) | `<EngineTools>/ClassCreationWizard/commands/*.py` | `engine` |
| 2 | `<Project>/ClassWizardCommands/*.py` | `project` |
| 3 | `<Gem>/ClassWizardCommands/*.py` | Gem name (alphabetical) |

**First registration wins.** If two plugins register the same command name, the first one loaded takes priority and a warning is logged. Files prefixed with `_` are skipped.

### Execution Flow

1. Template files are generated and staged
2. Conditional file exclusion and cleanup runs
3. Commands execute in order from `process_commands`
4. Registration commands (those with `is_registration_command = True`) only run when `--automatic-register` is enabled
5. Each command receives a `CommandContext` and returns `True` (success) or `False` (failure)

> **Note -- Conditional file exclusion is automatic.** You do not invoke a command to exclude files. Conditional file exclusion and reference cleanup (EBus scrubbing, editor include removal) is handled automatically before `process_commands` runs, based on the `condition` and `cleanup_hint` fields in each `copyFiles` entry. See the [Template Descriptor Format](../template-descriptor/) for details.

---

## Built-in Commands

### Registration Commands

Registration commands only run when the `--automatic-register` flag is set (or the GUI checkbox is enabled). They modify CMake and module files to integrate the new class into the build.

---

#### register_file_list

Adds the generated `.h` and `.cpp` files to the gem's CMake build target.

| Arg | Type | Description |
|---|---|---|
| `component_name` | string | Base filename (without extension) to register |

**Behavior:** Locates the selected build target's `FILES_CMAKE` list file. Inserts `Source/{component_name}.h` and `Source/{component_name}.cpp` into the `set(FILES ...)` block. If the target uses inline sources instead of a cmake list file, appends a `target_sources(...)` block to `CMakeLists.txt`.

---

#### register_module_descriptor

Adds a `CreateDescriptor()` call to the gem's module file so the component is instantiated at startup.

| Arg | Type | Description |
|---|---|---|
| `component_name` | string | Full component class name |
| `module_kind` | string | `"runtime"` (default) or `"editor"` -- selects `{Namespace}Module.cpp` or `{Namespace}EditorModule.cpp` |

**Behavior:** Adds `#include "{component_name}.h"` and inserts `{component_name}::CreateDescriptor()` into the `m_descriptors.insert(...)` block.

---

#### register_system_component

Adds the component to the `GetRequiredSystemComponents()` list so it activates automatically.

| Arg | Type | Description |
|---|---|---|
| `component_name` | string | Full component class name |
| `module_kind` | string | `"runtime"` (default) or `"editor"` |

**Behavior:** Adds `#include` and inserts `azrtti_typeid<Namespace::ComponentName>()` into the `return AZ::ComponentTypeList{ ... }` block.

---

#### register_interface_header

Registers an interface header (e.g. `PlayerHealthInterface.h`) in the gem's INTERFACE or API build target.

| Arg | Type | Description |
|---|---|---|
| `component_name` | string | Base name (without suffix) -- the wizard looks for `{component_name}Interface.h` |

**Behavior:** Locates the interface header under `Include/{Namespace}/`. Scans CMake targets to find an INTERFACE or API target. Adds the header to that target's file list.

---

### General Commands

These commands run regardless of the `--automatic-register` setting.

---

#### add_gem_dependency

Adds a gem dependency to the current build target's `BUILD_DEPENDENCIES` block in CMake.

| Arg | Type | Description |
|---|---|---|
| `dependency` | string | Dependency specifier, e.g. `Gem::GS_Cinematics.API` |

**Behavior:** Finds the `o3de_add_target` / `ly_add_target` block for the current build target. Inserts the dependency under `BUILD_DEPENDENCIES > PRIVATE` if not already present. Includes a **self-dependency guard** -- if the dependency resolves to the same gem as the current namespace, the command is skipped.

---

#### copy_file

Copies a file from one location to another within the gem directory.

| Arg | Type | Description |
|---|---|---|
| `source` | string | Source path relative to gem root |
| `dest` | string | Destination path relative to gem root |

---

#### copy_setreg

Ensures the `Registry/` directory exists for setreg file placement.

| Arg | Type | Description |
|---|---|---|
| `setreg_name` | string | Name of the `.setreg` file |

**Behavior:** Determines the correct `Registry/` directory based on the gem's directory structure (`Gem/`, `Code/`, or root-level).

---

#### register_asset_setreg

Configures the O3DE Asset Processor to recognize a custom data asset file extension.

| Arg | Type | Description |
|---|---|---|
| `asset_name` | string | Asset class name |
| `asset_ext` | string | File extension to register (default: `"mydata"`) |

**Behavior:** Reads the asset class UUID from its C++ header (via `AZ_RTTI` / `AZ_TYPE_INFO` macros). Writes an `RC` entry into the gem's `.setreg` JSON under `Amazon > AssetProcessor > Settings`, mapping the file extension to the asset type GUID.

---

#### register_generic_asset

Registers a `GenericAssetHandler` in the gem's `DataAssetSystemComponent`.

| Arg | Type | Description |
|---|---|---|
| `asset_name` | string | Asset class name |
| `asset_ext` | string | File extension (default: `"mydata"`) |
| `asset_group` | string | Asset browser group (default: `"Other"`) |

**Behavior:** Adds `#include` for the asset header, inserts a `GenericAssetHandler` registration block into `Activate()`, and adds a `::Reflect(context)` call in `Reflect()`.

---

#### replace_text

Performs find-and-replace on a generated source file. Useful for injecting variable values into template placeholders that are not standard O3DE template variables.

| Arg | Type | Description |
|---|---|---|
| `component_name` | string | Filename to search for in the generated output |
| `text_to_replace` | string | The literal text to find |
| `replacement` | string | Static replacement text (use this **or** `replacement_var`) |
| `replacement_var` | string | Name of an input variable whose value becomes the replacement |

**Example** -- replacing a channel placeholder with a user-provided value:

```json
{
    "command": "replace_text",
    "args": {
        "component_name": "${Name}_Reactor.h",
        "text_to_replace": "${PulseChannel}",
        "replacement_var": "pulse_channel"
    }
}
```

---

#### copy_file_to

General-purpose "copy one staged file to any destination" command. Intended as the single primitive that replaces the older `copy_file` / `copy_setreg` / `copy_asset_files` / `copy_variant_files` family in new templates -- those commands remain registered for templates already using them.

| Arg | Type | Description |
|---|---|---|
| `source` | string | Path inside the live staging directory |
| `destination` | string | Path under the resolved `anchor` |
| `anchor` | string | `dest_root` (default), `gem_root`, `gem_assets`, `gem_registry`, or `engine_root` |
| `is_templated` | bool | Apply `${variable}` substitution to file contents (default `true`) |
| `skip_existing` | bool | Don't overwrite an existing destination file (default `true`) |
| `create_dirs` | bool | Create missing destination directories (default `true`) |

**Behavior:** Copies a single file from the staging directory to a destination resolved against one of five named anchors -- the build target's source tree (`dest_root`), the gem root, `<gem>/Assets/`, `<gem>/Registry/`, or the engine root.

---

#### copy_glob_to

Glob-based companion to `copy_file_to` -- copies every staged file matching a pattern into a destination directory, preserving relative structure.

| Arg | Type | Description |
|---|---|---|
| `source_glob` | string | Glob pattern relative to the staging directory (supports `**`) |
| `dest_anchor` | string | Same anchor set as `copy_file_to` (default `dest_root`) |
| `dest_subdir` | string | Path under the anchor that becomes the new root of the matched tree |
| `strip_prefix` | string | Path prefix to strip from each matched file's relative path before joining onto `dest_subdir` |
| `is_templated` / `skip_existing` / `create_dirs` | bool | Same as `copy_file_to` |

**Behavior:** Use this when an entire subtree of staged files shares one destination anchor and subdirectory. For one-off copies use `copy_file_to` directly.

---

#### copy_asset_files

Copies a template's asset subtree (shaders, `.pass`, `.azasset`, materials, textures) directly into `<gem>/Assets/`, bypassing the normal `o3de create-from-template` staging path.

| Arg | Type | Description |
|---|---|---|
| `source_subdir` | string | Folder under the template root to walk (default `TemplateAssets`) |
| `dest_subdir` | string | Folder under the gem root to write into (default `Assets`) |
| `is_templated` | bool | Apply `${variable}` substitution to file contents (default `true`) |
| `skip_existing` | bool | Don't overwrite existing destination files (default `true`) |

**Behavior:** Asset files belong at `<gem>/Assets/...`, not under the C++ build tree that normal staging writes to. This command reads directly from a sibling `TemplateAssets/` folder (never touched by staging) and copies it into the gem's `Assets/` tree, applying `${Name}`/`${GemName}` substitution to both paths and contents.

---

#### copy_variant_files

Copies one of several parallel variant subtrees into the gem's source tree, selected by an input variable's value. Used when a template offers multiple integration modes that all target the same final file paths (see [Scoped Commands](../template-descriptor/#scoped-commands-advanced)).

| Arg | Type | Description |
|---|---|---|
| `variant_var` | string | Name of the input variable naming the active variant (matched case-sensitively to a subdirectory) |
| `variant_root` | string | Folder under the template root holding the variant subdirectories (default `Variants`) |
| `dest_subdir` | string | Folder under the gem root to write into (default `""`, i.e. gem root) |
| `is_templated` / `skip_existing` | bool | Same as `copy_file_to` |

**Behavior:** Reads the variant variable's value, copies only the matching `<template>/Variants/<value>/` subtree, and skips the others -- letting several mutually-exclusive file sets (e.g. different `RenderingSystemComponent` shapes) share the same destination paths.

---

#### add_pass_creator_call

Wires a custom Atom RPI pass class into a gem's `RenderingSystemComponent` so Atom's `PassSystem` knows how to instantiate it from a `.pass` template.

| Arg | Type | Description |
|---|---|---|
| `pass_name` | string | C++ class name of the pass |
| `system_component_name` | string | Override for the system component class name (default `${GemName}RenderingSystemComponent`) |

**Behavior:** Injects matching `AddPassCreator`/`RemovePassCreator` calls into `Activate()`/`Deactivate()`, adding the necessary `#include`s. Idempotent -- re-running with the same pass name is a no-op rather than duplicating the registration.

---

#### add_feature_processor_registration

Wires an Atom RPI `FeatureProcessor` into a gem's `RenderingSystemComponent`. Mirrors `add_pass_creator_call` for the `FeatureProcessorFactory` API surface.

| Arg | Type | Description |
|---|---|---|
| `feature_processor_name` | string | C++ class name of the FeatureProcessor |
| `system_component_name` | string | Override for the system component class name (default `${GemName}RenderingSystemComponent`) |

**Behavior:** Injects matching `RegisterFeatureProcessor`/`UnregisterFeatureProcessor` calls into `Activate()`/`Deactivate()`, adding the necessary `#include`s. Idempotent, same as `add_pass_creator_call`.

---

## Command Summary Table

| Command | Type | Purpose |
|---|---|---|
| `register_file_list` | Registration | Adds source files to CMake build target |
| `register_module_descriptor` | Registration | Adds `CreateDescriptor()` to module file |
| `register_system_component` | Registration | Adds to `GetRequiredSystemComponents()` |
| `register_interface_header` | Registration | Registers interface header in API target |
| `add_gem_dependency` | General | Adds gem dependency to CMake |
| `copy_file` | General | Copies file within gem directory (legacy -- see `copy_file_to`) |
| `copy_setreg` | General | Ensures Registry directory exists (legacy -- see `copy_file_to`) |
| `copy_file_to` | General | Copies one staged file to any anchor-rooted destination |
| `copy_glob_to` | General | Globs staged files into an anchor-rooted destination directory |
| `copy_asset_files` | General | Copies a template's asset subtree into `<gem>/Assets/` |
| `copy_variant_files` | General | Copies one variant subtree selected by an input variable |
| `register_asset_setreg` | General | Configures Asset Processor for custom extension |
| `register_generic_asset` | General | Registers GenericAssetHandler |
| `replace_text` | General | Find-and-replace in generated files |
| `add_pass_creator_call` | General | Registers an Atom RPI pass with `PassSystemInterface` |
| `add_feature_processor_registration` | General | Registers an Atom RPI FeatureProcessor |

---

## Writing Custom Commands

You can extend the command system by writing your own command plugins. See the [Command Authoring Guide](command-authoring/) for a complete walkthrough.
