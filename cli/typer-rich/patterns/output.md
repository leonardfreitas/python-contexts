# Output Conventions

## One shared Console, imported everywhere output happens

```python
# app/console.py
from rich.console import Console

console = Console()
error_console = Console(stderr=True)
```

`error_console` writes to stderr — this matters for CLIs whose stdout
output might be piped into another program (`mytool list | grep foo`);
errors and status messages shouldn't pollute stdout that a downstream
command might be parsing.

## Tables

```python
from rich.table import Table
from app.console import console

table = Table(title="Available contexts")
table.add_column("Name")
table.add_column("Path")

for ctx in contexts:
    table.add_row(ctx.name, str(ctx.path))

console.print(table)
```

Don't hand-format tabular output with manual string padding — Rich's
`Table` handles column alignment, width, and terminal wrapping correctly
across different terminal widths.

## Panels for errors and important standalone messages

```python
from rich.panel import Panel

error_console.print(Panel(message, title="Error", border_style="red"))
```

Reserved for the central error handler (see `patterns/error-handling.md`)
and similarly important standalone messages — not used for routine
success output, which stays as a simple styled line.

## Progress bars and spinners for anything taking more than ~1 second

```python
from rich.progress import Progress

with Progress() as progress:
    task = progress.add_task("Copying files...", total=len(files))
    for file in files:
        copy_one_file(file)
        progress.advance(task)
```

For an operation whose duration is unknown upfront (not a fixed list of
items to iterate), use a spinner instead of a determinate progress bar:

```python
with console.status("Fetching context list..."):
    contexts = fetch_available_contexts()
```

## Color conventions

- Green for success (`[green]✓[/green]`)
- Red for errors (handled centrally via the error panel, not per-command)
- Yellow for warnings that don't stop execution
- Dim/gray for secondary/supplementary information

## Respecting `NO_COLOR` and non-interactive environments

Rich's `Console` already detects non-terminal output (e.g. when stdout is
redirected to a file or piped) and disables color/formatting
automatically — don't override this detection manually unless a specific
`--no-color` flag is an explicit, deliberate feature of the CLI.
