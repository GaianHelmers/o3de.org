---
title: "Command System"
linkTitle: "Commands"
description: "How the Class Creation Wizard command system works, and a reference of all built-in commands."
weight: 500
---

Commands are the actions the Class Creation Wizard executes after generating template files. They handle build system integration -- registering files in CMake, adding module descriptors, inserting dependencies, and modifying generated source.

`process_commands` is optional -- the schema allows an empty list for a template that needs nothing beyond staging its files. Most real templates instead lean on the [pool of built-in commands](#built-in-commands) below. When none of them fit what you need, [author your own](#writing-custom-commands) -- the [Command Authoring Guide](command-authoring/) covers exactly what a command needs to implement to be fully compliant with the core system.

---

## How Commands Work

### Anatomy of a Command

Every command is one Python class, and every piece of it has a specific job:

| Piece | Purpose |
|---|---|
| `@CommandRegistry.register("name")` | Self-registers the class the instant it's defined -- not a separate step. See below. |
| Subclassing `WizardCommand` | Makes it a command at all -- required methods and default properties both come from here. |
| `name`, `description`, `version`, `author` | Identity and metadata. Only `name` (must match the decorator) and `description` (shown in `--template-help`) matter beyond documentation. |
| `is_registration_command` | The one flag deciding whether `--automatic-register` gates this command. |
| `__init__(...)` | The command's schema -- its parameters are exactly what a template's `args` object can supply. |
| `execute(self, ctx)` | The action itself, run against the shared `CommandContext`. |

That decorator is what makes the system extensible: it isn't a separate registration step, it runs the instant the class is *defined*. When the [`CommandPluginLoader`](#command-discovery) imports a command file, Python executes the file top to bottom as a normal part of that import -- and executing a `class Foo(WizardCommand): ...` statement under `@CommandRegistry.register(...)` is what adds `Foo` to the registry. There's no manifest to update and no list to keep in sync: dropping a correctly-decorated `.py` file into a `commands/` or `ClassWizardCommands/` directory is enough to make it available, the next time the wizard scans that directory -- registration and discovery are the same event.

```json
{
    "command": "register_file_list",
    "args": { "component_name": "${Name}${ComponentSuffix}" }
}
```

See the [Command Authoring Guide](command-authoring/#2-define-the-command-class) for what each piece looks like in a real, working example, with every line explained.

### Command Discovery

The `CommandPluginLoader` scans for Python files in three locations, loaded in this order:

| Priority | Location | Namespace |
|---|---|---|
| 1 (highest) | `<EngineTools>/ClassCreationWizard/commands/*.py` | `engine` |
| 2 | `<Project>/ClassWizardCommands/*.py` | `project` |
| 3 | `<Gem>/ClassWizardCommands/*.py` | Gem name (alphabetical) |

**First registration wins.** If two plugins register the same command name, the first one loaded takes priority and a warning is logged. Files prefixed with `_` are skipped.

### Execution, Command by Command

`process_commands` runs in the chronological order it's written in the template -- that ordering is the templating format's own promise, and it's what lets one command's output become the next one's assumption (register the files, *then* point a module descriptor at them, *then* add that to the system component list). Any [scoped command blocks](../template-descriptor/#scoped-commands-advanced) are flattened into that same flat, ordered sequence before anything runs.

Before the first command runs, the wizard builds a single [`CommandContext`](command-authoring/#the-commandcontext) -- one shared object, not one per command -- carrying the destination, the resolved variables, the build target, and the staging paths. Then, for each command in order:

1. If it's a [registration command](built-in-commands/#registration-commands) and `--automatic-register` wasn't passed, it's skipped -- not even constructed.
2. Its own `condition` is evaluated; if false, it's skipped, and the wizard logs why.
3. Its `args` are resolved (`${variable}` substitution), and `CommandRegistry.create(name, resolved_args)` constructs the command -- which does nothing more than `command_class(**resolved_args)`. This is why a command's `__init__` signature doubles as its schema: see [Anatomy of a Command](#anatomy-of-a-command) above and [Constructor Arguments](command-authoring/#constructor-arguments) for the authoring side of it.
4. `execute(ctx)` runs against the shared context.

That last step has a detail worth knowing: `execute()`'s `True`/`False` return isn't actually checked by the loop that calls it. A command that returns `False` has told you, via its own logging, that something didn't go as planned -- but the wizard doesn't stop or roll anything back because of it. The whole `process_commands` sequence is wrapped in one try/except; only an unhandled exception stops the run outright, and even then the temporary staging directory is always deleted in a `finally` block, success or failure.

This is the last stage of the sequence described in [Architecture > Execution Order](/docs/engine-dev/tools/class-wizard/architecture/#execution-order) -- everything above happens after your files are already sitting in your gem.

> **Note -- Conditional file exclusion is automatic.** You do not invoke a command to exclude files. Conditional file exclusion and reference cleanup (EBus scrubbing, editor include removal) is handled automatically before `process_commands` runs, based on the `condition` and `cleanup_hint` fields in each `copyFiles` entry. See the [Template Descriptor Format](../template-descriptor/) for details.

---

## Built-in Commands

The wizard ships with a pool of ready-made commands -- registration commands that only run with `--automatic-register`, and general commands that always run. See the [Built-in Commands reference](built-in-commands/) for the full list, every argument, and exactly what each one does.

---

## Writing Custom Commands

You can extend the command system by writing your own command plugins. See the [Command Authoring Guide](command-authoring/) for a complete walkthrough.
