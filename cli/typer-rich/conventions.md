# Conventions

## Services: functions, one per operation

Same default as every other context in this collection: a function, not a
class, unless there's a genuine need for configurable state across calls.

```
services/
  copy_context_files.py     # copy_context_files(source, destination) -> list[Path]
  resolve_context_path.py     # resolve_context_path(name: str) -> Path
```

## Commands: one file per command or command group

```python
# app/commands/init.py
import typer
from pathlib import Path
from app.console import console
from app.services.copy_context_files import copy_context_files
from app.services.resolve_context_path import resolve_context_path

app = typer.Typer()

@app.command()
def init(context: str = typer.Argument(..., help="Context path, e.g. django/api-drf")):
    """Copy a context into the current project's .context/ folder."""
    files = copy_context_files(source=resolve_context_path(context), destination=Path(".context"))
    console.print(f"[green]✓[/green] Copied {len(files)} files into .context/")
```

## Naming: verb-first for commands, matching CLI conventions

Command names are verbs or verb phrases (`init`, `list`, `update`), not
nouns — this matches the convention users expect from CLIs generally
(`git commit`, `npm install`) and keeps `--help` output scannable.

## Output conventions

- Success messages use a green checkmark prefix: `[green]✓[/green] ...`
- Errors are never printed with a bare `console.print` inside a command —
  they're raised as a `CliError` subclass and formatted once, centrally
  (see `patterns/error-handling.md`)
- Tables, panels, and progress bars are built with Rich's own primitives
  (`rich.table.Table`, `rich.panel.Panel`, `rich.progress.Progress`), not
  hand-formatted strings with manual padding

## Type hints drive Typer's parsing — don't duplicate validation Typer already does

```python
# ✅ Typer validates this is an int, converts it, and rejects bad input automatically
@app.command()
def scale(count: int = typer.Option(1, help="Number of instances")):
    ...
```

```python
# ❌ redundant manual parsing Typer already handles via the type hint
@app.command()
def scale(count: str = typer.Option("1")):
    count = int(count)  # Typer would have done this, and rejected bad input before this line ever ran
    ...
```

## Configuration access

Same principle as every other context here: no scattered direct file
reads for configuration. All persistent user configuration goes through
`app/config.py` — see `patterns/user-config.md`.
