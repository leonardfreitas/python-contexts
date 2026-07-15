# Error Handling

## Typed CLI errors, not bare exceptions or scattered `console.print` calls

```python
# app/exceptions.py
class CliError(Exception):
    """Base class for any error that should stop the CLI with a clear message."""
    exit_code = 1

    def __init__(self, message: str, **context):
        self.message = message
        self.context = context
        super().__init__(message)


class ConfigError(CliError):
    exit_code = 2

class ContextNotFoundError(CliError):
    exit_code = 3
```

Different `exit_code` values let scripts/CI pipelines calling this CLI
distinguish failure reasons programmatically, rather than always seeing a
generic `1`.

## A single top-level handler, not per-command try/except

```python
# app/cli.py
import sys
import typer
from rich.panel import Panel
from app.console import error_console
from app.exceptions import CliError

app = typer.Typer()
# ... app.add_typer(...) calls ...

def main():
    try:
        app()
    except CliError as e:
        error_console.print(Panel(e.message, title="Error", border_style="red"))
        sys.exit(e.exit_code)

if __name__ == "__main__":
    main()
```

Individual commands never catch `CliError` themselves and never call
`console.print` to report an error — they let it propagate. This mirrors
the same "one central translation point" principle used for exception
handling in every other context in this collection: a service raises a
typed error, exactly one place decides how to present it.

```python
# ✅ command doesn't handle the error, just lets it propagate
@app.command()
def init(context: str):
    files = copy_context_files(source=resolve_context_path(context), destination=Path(".context"))
    console.print(f"[green]✓[/green] Copied {len(files)} files")

# app/services/resolve_context_path.py
def resolve_context_path(name: str) -> Path:
    path = CONTEXTS_ROOT / name
    if not path.exists():
        raise ContextNotFoundError(f"Context '{name}' not found", available=list_available_contexts())
    return path
```

## Rule

> No command function contains a `try/except` around a service call to
> print a formatted error itself. Every error the user should see is a
> `CliError` subclass, raised from wherever it's detected (usually a
> service), and rendered exactly once, at the top level of the
> application's entry point.

## Unexpected (non-`CliError`) exceptions

Let them propagate past the `except CliError` block and crash with a full
Python traceback during development — don't add a catch-all
`except Exception` that swallows unexpected bugs into a generic "something
went wrong" message. A visible traceback for a genuine bug is more useful
than a polished error message that hides what actually happened. If the
CLI is mature enough to want a friendlier crash report for end users, that
becomes an explicit, separate decision (e.g. an opt-in `--verbose` flag
controlling traceback visibility), not a blanket catch-all from day one.
