# Architecture

## The layers

```
Terminal → Typer command (thin) → Service (business logic) → Rich (output)
```

| Layer | Responsibility | Does NOT do |
|---|---|---|
| **Command (Typer)** | Parses arguments/options, calls the service, formats the result via Rich | Business logic |
| **Service** | The actual logic — file operations, validation, orchestration | Anything Typer/Rich-specific — doesn't know a terminal exists |
| **Console/output** | A shared Rich `Console` instance, used only from the command layer | Business logic |

The litmus test is the same one used throughout this collection: if the
logic needs to be callable from a test with no `CliRunner` involved, or
from a different entry point entirely (a script, a future non-CLI
caller), it's a service.

```python
# app/services/copy_context_files.py
def copy_context_files(source: Path, destination: Path) -> list[Path]:
    """Returns the list of files copied. Knows nothing about the terminal."""
    destination.mkdir(parents=True, exist_ok=True)
    copied = []
    for file in source.rglob("*.md"):
        target = destination / file.relative_to(source)
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(file.read_text())
        copied.append(target)
    return copied
```

```python
# app/commands/init.py
@app.command()
def init(context: str):
    """Copy a context into the current project."""
    copied = copy_context_files(source=resolve_context_path(context), destination=Path(".context"))
    console.print(f"[green]Copied {len(copied)} files.[/green]")
```

## Command structure — sub-apps by command group

Typer supports nested `Typer()` instances to organize related commands
under a common prefix (e.g. `mytool init ...`, `mytool list ...`). Use one
sub-app per logical group of commands, not a single flat app with every
command mixed together once the tool grows past a handful of commands.

```python
# app/cli.py
import typer
from app.commands.init import app as init_app
from app.commands.list_contexts import app as list_app

app = typer.Typer()
app.add_typer(init_app, name="init")
app.add_typer(list_app, name="list")

if __name__ == "__main__":
    app()
```

For a CLI small enough to only ever have a handful of top-level commands
(no natural grouping), a single flat `Typer()` app with each command as a
function is fine — don't force sub-app nesting where there's nothing to
group.

## Folder structure

```
my-cli/
  pyproject.toml

  app/
    cli.py                    # the root Typer app, wires up sub-apps/commands
    console.py                  # the shared Rich Console instance

    commands/                    # one file per command or command group
      init.py
      list_contexts.py

    services/                     # business logic, one function per operation
      copy_context_files.py
      resolve_context_path.py

    config.py                      # user configuration read/write (see patterns/user-config.md)
    exceptions.py                   # CliError and subclasses

  tests/
    test_services.py
    test_commands.py
```

### Why `console.py` holds a single shared `Console` instance

Rich's `Console` object carries state relevant to consistent output
(width detection, color support, whether output is being piped). Creating
a new `Console()` in every command risks inconsistent behavior (e.g. one
command respecting `NO_COLOR` and another not). One shared instance,
imported wherever output happens, keeps behavior consistent across the
whole CLI.

```python
# app/console.py
from rich.console import Console

console = Console()
error_console = Console(stderr=True)
```

### Why `services/` never imports `typer` or `rich`

Same rule as every other context in this collection, applied to a
terminal-based interface instead of HTTP or a queue: the layer that
contains business logic should be callable and testable without any
knowledge of the interface driving it.

## Rule

> A `@app.command()` function's body is limited to: parsing/validating
> input beyond what Typer's type hints already handle, calling exactly
> one service function, and formatting the result via Rich. Anything more
> than that belongs in `services/`.
