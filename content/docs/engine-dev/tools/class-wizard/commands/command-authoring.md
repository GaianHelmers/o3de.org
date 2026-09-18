---
title: "Command Authoring Guide"
linkTitle: "Command Authoring"
description: "How to create custom command plugins for the Class Creation Wizard."
weight: 200
---

The Class Creation Wizard uses a modular plugin architecture. Each command is a self-contained Python class that registers itself with the wizard at load time. You can add new commands to the engine, your project, or any gem -- no changes to the wizard core required.

## Writing a Command

### 1. Create the File

Place your command file in the appropriate `ClassWizardCommands/` directory:

```
MyGem/
  ClassWizardCommands/
    my_custom_command.py
```

As indicated above, the engine's own `commands/` directory is just the first location the wizard checks -- not the sole place to create a command. A gem can carry its own `ClassWizardCommands/` folder the same way a project can, and it's picked up automatically as long as it's named and placed correctly -- a `.py` file, not prefixed with `_`, directly inside that `ClassWizardCommands/` directory. This matters most when a command is tightly coupled to a specific gem's own codebase; see [Architecture > A Self-Contained Creation System, Scoped to a Gem](/docs/engine-dev/tools/class-wizard/architecture/#a-self-contained-creation-system-scoped-to-a-gem) for why that's the real point of this being pluggable at the gem level at all. See [Command Discovery](/docs/engine-dev/tools/class-wizard/commands/#command-discovery) for the full scan order and priority across the engine, your project, and every gem.

### 2. Define the Command Class

Every command follows the same shape. Here's what each piece is actually for.

**Register and subclass.**

```python
from command_plugin import CommandRegistry, WizardCommand, CommandContext

@CommandRegistry.register("my_custom_command")
class MyCustomCommand(WizardCommand):
    """One-line summary of what this command does."""
```

The decorator's argument, `"my_custom_command"`, is the exact string a template's `process_commands` entries will use in their `"command"` field -- it's independent of the class name, and nothing checks that the two stay in sync with the `name` property below. Subclassing [`WizardCommand`](#the-wizardcommand-interface) is what makes this a command at all: it's where the abstract methods you must implement, and the default property values you get for free, both come from.

**Metadata.**

```python
    @property
    def name(self) -> str:
        return "my_custom_command"

    @property
    def description(self) -> str:
        return "Brief description shown in --template-help output"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"
```

`name` is the one property here that matters beyond documentation -- keep it identical to the decorator's argument, since letting them drift apart is a silent bug rather than an error. `description` is the only one a user ever actually sees, surfaced in `--template-help` output. `version` and `author` are pure bookkeeping, useful mainly if you're maintaining a command across multiple gems; both are safe to leave at their defaults otherwise.

**The registration flag.**

```python
    # Set True if this command should only run with --automatic-register
    is_registration_command = False
```

This single flag decides whether `--automatic-register` gates your command. Leave it `False` -- the default -- for anything that isn't modifying CMake or module files; `replace_text`, `add_gem_dependency`, and every asset-copying command in the [built-in set](../built-in-commands/) all leave it unset for exactly that reason.

**The constructor.**

```python
    def __init__(self, target_file: str, value: str = "default"):
        self.target_file = target_file
        self.value = value
```

This signature *is* your command's schema -- see [Constructor Arguments](#constructor-arguments) below. Every parameter here is a key a template author can pass in `args`. Give optional ones a default; leave required ones without one, so a template that omits them fails loudly at construction instead of failing confusingly later.

**The action.**

```python
    def execute(self, ctx: CommandContext) -> bool:
        ctx.logger(f"Running my_custom_command on {self.target_file}")

        # Access the gem's source directory
        target = ctx.dest_root / "Source" / self.target_file
        if not target.exists():
            ctx.logger(f"File not found: {target}")
            return False

        # Do your work here
        content = target.read_text(encoding="utf-8")
        content = content.replace("PLACEHOLDER", self.value)
        target.write_text(content, encoding="utf-8")

        ctx.logger(f"Replaced PLACEHOLDER with {self.value}")
        return True
```

This is the one method every command must implement. It receives the single [`CommandContext`](#the-commandcontext) shared by every command in the run, and does whatever the command actually promises -- here, a find-and-replace on a generated file. The `True`/`False` return is informational, not a control signal: nothing in the wizard branches on it. `ctx.logger()` is what actually surfaces success or failure to whoever's watching, in both the GUI status panel and the CLI's console output -- so log clearly, especially on the failure path.

Put together, those five pieces are the whole class:

```python
from command_plugin import CommandRegistry, WizardCommand, CommandContext


@CommandRegistry.register("my_custom_command")
class MyCustomCommand(WizardCommand):
    """One-line summary of what this command does."""

    @property
    def name(self) -> str:
        return "my_custom_command"

    @property
    def description(self) -> str:
        return "Brief description shown in --template-help output"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"

    is_registration_command = False

    def __init__(self, target_file: str, value: str = "default"):
        self.target_file = target_file
        self.value = value

    def execute(self, ctx: CommandContext) -> bool:
        ctx.logger(f"Running my_custom_command on {self.target_file}")

        target = ctx.dest_root / "Source" / self.target_file
        if not target.exists():
            ctx.logger(f"File not found: {target}")
            return False

        content = target.read_text(encoding="utf-8")
        content = content.replace("PLACEHOLDER", self.value)
        target.write_text(content, encoding="utf-8")

        ctx.logger(f"Replaced PLACEHOLDER with {self.value}")
        return True
```

### 3. Use It in a Template

```json
{
    "command": "my_custom_command",
    "args": {
        "target_file": "${Name}Component.cpp",
        "value": "${SomeInputVar}"
    }
}
```

---

## The CommandContext

Every command receives a `CommandContext` with these fields:

| Field | Type | Description |
|---|---|---|
| `dest_root` | `Path` | Root directory of the target gem (e.g. `D:\Project\Gem`) |
| `namespace` | `str` | Gem namespace / name (e.g. `"GS_Interaction"`) |
| `component_name` | `str` | Name of the component being created |
| `build_target` | `CMakeTarget` | The selected CMake build target -- has `name`, `raw_name`, `kind`, `file` (the `Path` to its `CMakeLists.txt`), and `files_cmake_list` |
| `variables` | `dict` | All resolved variables -- base vars (`Name`, `GemName`, `ComponentSuffix`) plus user input values |
| `logger` | `callable` | Logging function -- call `ctx.logger("message")` |
| `engine_path` | `Path` | Path to the O3DE engine root |
| `copy_files` | `list` | `(resolved_path, CopyFileDef)` pairs for every condition-passing file from `copyFiles`. Commands that need a generated file's actual path (rather than assuming `Source/`) read this. |
| `template_path` | `Path` or `None` | Path to the source template directory (containing `template.json` and `Template/`). Used by commands that read template-side files outside the normal staging pipeline, e.g. `copy_asset_files`. |
| `stage_dir` | `Path` or `None` | Path to the live staging directory, valid only during the `process_commands` phase. Files marked `excludeFromMerge: true` are still present here. `copy_file_to` and `copy_glob_to` read from this. |

---

## The WizardCommand Interface

All commands extend the `WizardCommand` abstract base class:

```python
class WizardCommand(ABC):
    @abstractmethod
    def execute(self, ctx: CommandContext) -> bool:
        """Run the command. Return True on success."""
        ...

    @property
    @abstractmethod
    def name(self) -> str:
        """The registered command name."""
        ...

    @property
    def is_registration_command(self) -> bool:
        """If True, only runs when --automatic-register is set."""
        return False

    @property
    def description(self) -> str:
        """Brief description for help output."""
        return ""

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return ""
```

You can also set `is_registration_command` as a class attribute instead of a property:

```python
@CommandRegistry.register("my_registration_cmd")
class MyRegistrationCmd(WizardCommand):
    is_registration_command = True
    # ...
```

---

## Constructor Arguments

Constructor parameters map directly to the `args` object in the template JSON. The wizard calls `CommandRegistry.create(name, args)` which instantiates your class with `**args`. So if your template says:

```json
{
    "command": "my_command",
    "args": { "target_file": "Foo.cpp", "value": "bar" }
}
```

Your class must accept those as `__init__` parameters:

```python
def __init__(self, target_file: str, value: str = "default"):
```

Use default values for optional arguments.

---

## Variable Resolution

All string values in `args` are resolved **before** your constructor is called. `${Name}`, `${GemName}`, `${ComponentSuffix}`, and any template `input_vars` are substituted automatically. Your command receives final, resolved strings.

To access raw variable values at execution time (e.g. for the `replace_text` pattern), use `ctx.variables`:

```python
channel = ctx.variables.get("pulse_channel", "DefaultChannel")
```

---

## Conditional Execution

Commands can be gated by a `condition` in the template JSON:

```json
{
    "command": "register_interface_header",
    "condition": "add_bus_interface",
    "args": { "component_name": "${Name}" }
}
```

Condition syntax:

| Pattern | Meaning |
|---|---|
| *(empty or omitted)* | Always runs |
| `"var_name"` | Runs if the variable is truthy |
| `"!var_name"` | Runs if the variable is falsy |
| `"${var} == 'value'"` | Equality check |
| `"${var} != 'value'"` | Inequality check |

---

## Tips

- **Keep commands focused.** One command should do one thing. Compose complex workflows by chaining multiple commands in the template's `process_commands` array.
- **Log clearly.** Use `ctx.logger()` throughout -- users see this output in both GUI and CLI modes.
- **Return False on failure.** The wizard doesn't inspect this value -- it never stops or reports on it for you, and always moves on to the next command. `ctx.logger()` is the only thing that actually tells anyone a command failed, so log it yourself before returning `False`. Don't raise exceptions unless something is truly unrecoverable.
- **Use `CMakeAnalyzer`** if you need to parse or modify CMake files. It's available from `command_plugin`:
  ```python
  from command_plugin import CMakeAnalyzer
  targets = CMakeAnalyzer.scan_targets(ctx.build_target.file.parent, ctx.namespace)
  ```
  `scan_targets(gem_path, gem_name)` takes the directory to search and the gem name used to resolve `${GemName}`-style tokens in target names -- it returns a `List[CMakeTarget]`, not a single target.
- **Test with CLI first.** Run with `--automatic-register` and check the output before using the GUI.

---

## Blank Command Template

A complete, minimal, fully working command -- no required arguments, does nothing but log and succeed. Copy this into a new `.py` file in a `ClassWizardCommands/` directory, rename `my_command` / `MyCommand` throughout, and build out `execute()` from there.

```python
from command_plugin import CommandRegistry, WizardCommand, CommandContext


@CommandRegistry.register("my_command")
class MyCommand(WizardCommand):
    """One-line summary of what this command does."""

    @property
    def name(self) -> str:
        return "my_command"

    @property
    def description(self) -> str:
        return "Shown in --template-help output"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"

    # Set True only if this command should be gated behind --automatic-register.
    is_registration_command = False

    def execute(self, ctx: CommandContext) -> bool:
        ctx.logger("my_command executed")
        return True
```

This is usable immediately, with no `args` at all:

```json
{
    "command": "my_command"
}
```

Add an `__init__(self, ...)` when the command needs input from the template -- see [Constructor Arguments](#constructor-arguments) -- and replace the body of `execute()` with whatever the command actually needs to do, using `ctx` to reach the destination gem, the resolved variables, and the logger.
