# Interactive Prompts

## Default to Rich's own `Prompt` and `Confirm`, not Typer's built-in prompting

Typer supports basic prompting via `typer.prompt()`/`typer.confirm()`, but
since this context already standardizes on Rich for all output, use
Rich's `Prompt`/`Confirm` classes instead — this keeps prompt styling
consistent with every other piece of output in the CLI (same color
theme, same console instance).

```python
from rich.prompt import Prompt, Confirm
from app.console import console

name = Prompt.ask("What should we call this context?", console=console)

overwrite = Confirm.ask(
    f"[yellow].context/ already exists — overwrite?[/yellow]",
    console=console,
    default=False,
)
```

## Validated input via `choices`

```python
env = Prompt.ask(
    "Which environment?",
    choices=["dev", "staging", "production"],
    default="dev",
    console=console,
)
```

Rich re-prompts automatically on an invalid choice — don't hand-roll a
`while True` validation loop for this common case.

## Prompts belong in the command layer, not in services

Same boundary as everywhere else in this context: a service function
should never call `Prompt.ask()` internally — that would make it
impossible to call the service from a test, or from a future non-
interactive mode (e.g. a `--yes` flag that should skip confirmation
entirely), without also faking terminal input.

```python
# ✅ the command handles the prompt, the service stays pure
@app.command()
def init(context: str, force: bool = typer.Option(False, "--yes", "-y")):
    destination = Path(".context")
    if destination.exists() and not force:
        if not Confirm.ask(f"{destination} already exists — overwrite?", default=False):
            raise typer.Exit(code=0)

    copy_context_files(source=resolve_context_path(context), destination=destination)
```

```python
# ❌ the service prompts internally — now untestable without faking stdin,
# and impossible to run non-interactively
def copy_context_files(source: Path, destination: Path) -> list[Path]:
    if destination.exists():
        if not Confirm.ask("Overwrite?"):  # wrong layer
            return []
    ...
```

## Always provide a non-interactive escape hatch

Every command that would prompt interactively needs a flag to skip the
prompt (commonly `--yes`/`-y`, or accepting the value directly as an
option) — this is what makes the command usable in scripts and CI, where
no human is present to answer a prompt.

## When to reach for `questionary` instead

Rich's `Prompt`/`Confirm` cover single-value text input, choice-from-list,
and yes/no confirmation well. For richer interactions — multi-select
checkboxes, a fuzzy-searchable list, arrow-key navigation through nested
options — `questionary` is the documented conditional dependency (see
`stack.md`). Don't reach for it by default; most CLIs never need more
than what Rich already provides.
