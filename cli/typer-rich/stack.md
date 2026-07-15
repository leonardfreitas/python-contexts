# Stack

## Core — always present

| Library | Role |
|---|---|
| **Typer** | Command parsing, type-hint-driven argument/option definitions, automatic `--help` generation |
| **Rich** | All terminal output — colored text, tables, panels, progress bars, spinners |
| **pytest** | Testing |
| **platformdirs** | Resolves OS-appropriate config/cache/data directories — see `patterns/user-config.md` |
| **ruff** | Lint + format |
| **mypy** | Static typing |

## Why Rich is core, not conditional

This context was chosen specifically because Rich is the intended default
experience, not an enhancement bolted on later. Once a project accepts
Rich as its output layer, mixing in plain `print()` calls elsewhere
creates visibly inconsistent output (some lines colored/formatted,
others not) — so Rich is used for all output from the first command
written, not added once the CLI "feels like it needs it."

## Typer already includes what you'd otherwise add separately

Typer is built on top of Click and already provides `--help` generation,
shell completion, and type-based validation out of the box — there's no
need to reach for `argparse` or hand-roll help text.

## Conditional — only added when the project justifies it

| Library | Add it when |
|---|---|
| `questionary` | Interactive prompts need something Rich's own `Prompt`/`Confirm` don't cover well — e.g. multi-select checkboxes, fuzzy-searchable lists. Default to Rich's built-in prompt classes first (see `patterns/interactive-prompts.md`); only reach for `questionary` when a specific interaction genuinely needs it |
| `tomli` | Only if targeting Python versions before 3.11 — `tomllib` is in the standard library from 3.11 onward and is preferred when available |

## Explicitly out of scope for this context's stack

Packaging and distribution tooling (build backends, PyPI publishing,
`pipx`/`uvx` considerations) is deliberately not covered here — see
`SKILL.md`. Don't infer a packaging stack from this file.
