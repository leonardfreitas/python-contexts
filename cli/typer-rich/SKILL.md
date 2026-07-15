# Typer + Rich CLI Context

You are a senior engineer building a command-line tool in Python using
Typer for command parsing and Rich for terminal output. Follow all the
contexts below before generating any code.

This context is fully self-contained. It does not reference, depend on,
or assume the presence of any other context in this collection.

This is the entry point. Read in this order:

1. `architecture.md` — layers, command structure, folder organization
2. `stack.md` — required libraries and why
3. `conventions.md` — naming, service pattern, output conventions
4. `rules.md` — what is FORBIDDEN, with ❌ / ✅ examples
5. `patterns/` — step-by-step recipes for recurring tasks

## Core philosophy

A CLI command is a thin entry point, not a place for business logic —
the same discipline used across every context in this collection applies
here too, just with a terminal instead of an HTTP request or a queue
message as the trigger. A command parses input, calls a service, and
formats the result; it does not decide business outcomes itself.

If you are generating code and logic appears to be leaking into a
`@app.command()` function beyond parsing and output formatting — stop and
move it into `services/`. This keeps business logic testable without
spinning up Typer's `CliRunner`, and reusable if the same logic is ever
needed from a non-CLI context (a script, a different entry point).

## Packaging and distribution are out of scope for this context

This context covers building the CLI's command structure, logic, and
output. How the CLI is packaged and published (PyPI, `pipx`, `uvx`,
standalone binaries) is intentionally left to a separate, dedicated
context — don't infer packaging conventions from this one.

## Patterns index

- `patterns/new-command.md` — step-by-step to add a new command or command group
- `patterns/error-handling.md` — typed CLI errors, single top-level handler, exit codes
- `patterns/output.md` — Rich conventions: console, tables, panels, progress
- `patterns/user-config.md` — reading/writing persistent user configuration
- `patterns/interactive-prompts.md` — prompts and confirmations via Rich
- `patterns/testing.md` — testing commands with `CliRunner`, testing services directly

## Structure of this context

```
typer-rich/
  SKILL.md              → this file, entry point + index
  architecture.md         → layers, command structure, folder tree
  stack.md                  → required libraries and why
  conventions.md              → naming and code patterns
  rules.md                      → what is FORBIDDEN
  patterns/
    new-command.md
    error-handling.md
    output.md
    user-config.md
    interactive-prompts.md
    testing.md
```
