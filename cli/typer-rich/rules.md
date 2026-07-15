# Rules — What is FORBIDDEN

An executive summary of every "hard" rule scattered across the other
files. For a quick check, start here; for the full reasoning behind each
rule, see the referenced file.

## Layers (`architecture.md`)

- ❌ Business logic inside a `@app.command()` function beyond
  parsing/calling a service/formatting output
- ❌ `services/` importing `typer` or `rich`
- ❌ A new `Console()` instantiated inside a command instead of importing
  the shared instance from `app/console.py`

## Error handling (`patterns/error-handling.md`)

- ❌ A command with its own `try/except` that prints a formatted error
  itself — error rendering happens once, centrally
- ❌ Raising a bare `Exception` instead of a `CliError` subclass for
  anything the user should see a clear message about
- ❌ A blanket `except Exception` that swallows unexpected bugs into a
  generic message, hiding real tracebacks during development

## Output (`patterns/output.md`)

- ❌ Hand-formatted tabular output with manual string padding instead of
  `rich.table.Table`
- ❌ Manually overriding Rich's automatic `NO_COLOR`/non-terminal
  detection without an explicit, deliberate `--no-color` flag as a
  feature
- ❌ Status/error output written to stdout instead of the dedicated
  `error_console` (stderr)

## User configuration (`patterns/user-config.md`)

- ❌ A hardcoded config path (e.g. `~/.mytool`) instead of
  `platformdirs.user_config_dir`
- ❌ Any code outside `app/config.py` reading or writing the config file
  directly
- ❌ Treating a missing config file as an error instead of "defaults
  apply"

## Interactive prompts (`patterns/interactive-prompts.md`)

- ❌ A service function calling `Prompt.ask()`/`Confirm.ask()` internally
  — prompting is a command-layer concern
- ❌ A command that prompts interactively with no non-interactive escape
  hatch (a `--yes`/`-y` flag or equivalent)
- ❌ Using `typer.prompt()`/`typer.confirm()` instead of Rich's
  `Prompt`/`Confirm` once a project has standardized on Rich for output

## Testing (`patterns/testing.md`)

- ❌ A service with no direct unit test (only covered indirectly through
  a `CliRunner` test)
- ❌ A command with no test covering at least one error path
- ❌ Skipping tests for interactive prompts instead of simulating input
  via `CliRunner`'s `input` parameter
