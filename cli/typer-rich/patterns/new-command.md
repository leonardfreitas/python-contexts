# Adding a New Command

Step-by-step for adding a new command to an existing CLI built on this
context.

## 1. Domain exceptions (if this command introduces new failure modes)

```python
# app/exceptions.py
class ContextAlreadyExistsError(CliError):
    exit_code = 4
```

## 2. Service — the logic, as a function, no Typer/Rich imports

```python
# app/services/list_available_contexts.py
def list_available_contexts(contexts_root: Path) -> list[ContextInfo]:
    return [
        ContextInfo(name=p.relative_to(contexts_root).as_posix(), path=p)
        for p in contexts_root.glob("*/*/SKILL.md")
    ]
```

## 3. Command — thin, delegates immediately

```python
# app/commands/list_contexts.py
import typer
from rich.table import Table
from app.console import console
from app.services.list_available_contexts import list_available_contexts

app = typer.Typer()

@app.command(name="list")
def list_contexts():
    """List all available contexts."""
    contexts = list_available_contexts(CONTEXTS_ROOT)

    table = Table(title="Available contexts")
    table.add_column("Name")
    table.add_column("Path")
    for ctx in contexts:
        table.add_row(ctx.name, str(ctx.path))

    console.print(table)
```

If this is a new command group entirely, wire it into the root app:

```python
# app/cli.py
from app.commands.list_contexts import app as list_app
app.add_typer(list_app)
```

## 4. Tests — service first, then the command

```python
# tests/test_services.py
def test_list_available_contexts_finds_all_skill_files(tmp_path):
    (tmp_path / "django" / "api-drf").mkdir(parents=True)
    (tmp_path / "django" / "api-drf" / "SKILL.md").write_text("")

    contexts = list_available_contexts(tmp_path)

    assert len(contexts) == 1
    assert contexts[0].name == "django/api-drf"
```

```python
# tests/test_commands.py
def test_list_command_shows_table(monkeypatch):
    result = runner.invoke(app, ["list"])
    assert result.exit_code == 0
    assert "Available contexts" in result.stdout
```

## Checklist

- [ ] New failure modes inherit from `CliError` with an appropriate `exit_code`
- [ ] Service is a function, doesn't import `typer` or `rich`
- [ ] Command delegates immediately, no business logic inline
- [ ] Any interactive prompt lives in the command, not the service, and has a non-interactive escape hatch
- [ ] Service has a unit test with no `CliRunner` involved
- [ ] Command has a `CliRunner` test covering its happy path and its main error path
