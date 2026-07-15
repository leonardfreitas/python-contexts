# User Configuration

## Use `platformdirs` for the config location, never a hardcoded path

A CLI that needs to remember preferences between runs (a default source
path, a preferred output format, credentials location) should store that
configuration in the OS-appropriate location — not a hand-rolled
`~/.mytool` that ignores platform conventions.

```python
# app/config.py
from pathlib import Path
import tomllib
import tomli_w  # only if writing TOML is needed; tomllib is read-only
from platformdirs import user_config_dir

APP_NAME = "my-cli"

def config_path() -> Path:
    return Path(user_config_dir(APP_NAME)) / "config.toml"
```

`user_config_dir` resolves to the right place per OS automatically (e.g.
`~/.config/my-cli/` on Linux, `~/Library/Application Support/my-cli/` on
macOS, `%APPDATA%\my-cli\` on Windows) — the CLI's code never needs to
know or care which OS it's running on.

## Reading configuration

```python
# app/config.py (continued)
def load_config() -> dict:
    path = config_path()
    if not path.exists():
        return {}
    with path.open("rb") as f:
        return tomllib.load(f)
```

Missing config file is not an error — it means defaults apply. Reserve
`ConfigError` for a config file that exists but is malformed/invalid, not
for the absence of one.

## Writing configuration

```python
def save_config(data: dict) -> None:
    path = config_path()
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("wb") as f:
        tomli_w.dump(data, f)
```

## TOML over JSON or YAML for this use case

TOML is the format Python's own packaging ecosystem (`pyproject.toml`)
already uses, it's comment-friendly (unlike JSON), and it avoids YAML's
well-known ambiguity footguns (implicit type coercion, indentation
sensitivity) for a file a human might hand-edit. `tomllib` is in the
standard library from Python 3.11 — no extra dependency needed for
reading; writing needs `tomli_w` since the standard library's `tomllib`
is read-only by design.

## Rule

> Configuration is only ever read/written through `app/config.py`. No
> command or service opens a config file path directly — this keeps the
> config format and location a single, changeable implementation detail
> rather than something scattered across the codebase.

## Command-line flags override persisted config, which overrides built-in defaults

When a setting can come from more than one source, the precedence order
is explicit and consistent across the whole CLI:

```
CLI flag (--option value) > persisted config file > built-in default
```

```python
def resolve_setting(cli_value: str | None, config: dict, key: str, default: str) -> str:
    if cli_value is not None:
        return cli_value
    return config.get(key, default)
```
