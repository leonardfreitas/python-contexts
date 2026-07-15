# Testing

## Testing pyramid by layer

| Layer | Test type | Terminal involved? |
|---|---|---|
| Service | Unit | No — call the function directly |
| Command | Integration | Yes — via Typer's `CliRunner` |

## Services → unit test, called directly, no CLI machinery

```python
def test_copy_context_files_copies_all_markdown(tmp_path):
    source = tmp_path / "source"
    source.mkdir()
    (source / "SKILL.md").write_text("content")

    destination = tmp_path / "dest"
    copied = copy_context_files(source=source, destination=destination)

    assert len(copied) == 1
    assert (destination / "SKILL.md").exists()
```

No `CliRunner`, no simulated terminal input, no Rich console involved —
this is the fast path and should be the majority of the test suite.

## Commands → integration test via `CliRunner`

```python
from typer.testing import CliRunner
from app.cli import app

runner = CliRunner()

def test_init_command_copies_files(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    result = runner.invoke(app, ["init", "django/api-drf"])

    assert result.exit_code == 0
    assert "Copied" in result.stdout
```

`CliRunner` captures stdout/stderr and the exit code without spawning a
real subprocess — fast enough to run for every command's happy path and
key error paths.

## Testing error handling

```python
def test_init_command_fails_with_clear_message_for_unknown_context():
    result = runner.invoke(app, ["init", "nonexistent/context"])

    assert result.exit_code == 3  # ContextNotFoundError.exit_code
    assert "not found" in result.stdout
```

This validates the full path from a raised `CliError` through the central
handler to the formatted output and exit code — the same "glue" test
philosophy used for route/handler tests in every other context in this
collection.

## Testing interactive prompts

`CliRunner.invoke()` accepts an `input` parameter to simulate what a user
would type in response to a prompt:

```python
def test_init_prompts_before_overwriting_existing_context(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    (tmp_path / ".context").mkdir()

    result = runner.invoke(app, ["init", "django/api-drf"], input="n\n")

    assert "already exists" in result.stdout
    assert result.exit_code == 0  # user declined, command exits cleanly
```

## Rule

> Every service has a direct unit test with no `CliRunner` involved.
> Every command has at least one `CliRunner` test covering its happy path,
> and one covering its most likely error path. Interactive prompts are
> tested by simulating input via `CliRunner`'s `input` parameter, not
> skipped because "it's just a prompt."
