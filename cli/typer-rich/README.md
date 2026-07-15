# typer-rich

Context for building command-line tools in Python with Typer and Rich —
the rules, patterns, and conventions an AI coding assistant should follow
before generating any code in this kind of project.

Part of the [`python-contexts`](../../) collection. Fully self-contained
— doesn't reference or depend on any other context in this repository.

## Two ways to use this

| | Option A — Claude Code ✦ recommended | Option B — Claude.ai (or any chat UI) |
|---|---|---|
| Where it runs | Terminal / VS Code / Cursor / JetBrains | Browser |
| Setup | Copy this folder into your project + a root instruction file | Upload files to a Project |
| Updates | Re-copy the folder after a `git pull` | Manual re-upload |
| Advantage | The instruction file is committed, the whole team shares the same setup | No extra installation |

## Option A — Claude Code (recommended)

### 1. Clone this repository

```bash
git clone <url-of-this-repo>
```

### 2. Copy the context folder into your project

```bash
# at the root of your CLI project
mkdir -p .context/typer-rich/patterns

cp /path/to/python-contexts/cli/typer-rich/SKILL.md          .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/architecture.md   .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/stack.md          .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/conventions.md    .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/rules.md          .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/patterns/*.md     .context/typer-rich/patterns/
```

### 3. Create a root instruction file

If you're using Claude Code, create `CLAUDE.md` at your project root:

```markdown
# Project name

You are a senior engineer on this project. Follow all the contexts below
before generating any code.

@.context/typer-rich/SKILL.md
@.context/typer-rich/architecture.md
@.context/typer-rich/stack.md
@.context/typer-rich/conventions.md
@.context/typer-rich/rules.md
@.context/typer-rich/patterns/new-command.md
@.context/typer-rich/patterns/error-handling.md
@.context/typer-rich/patterns/output.md
@.context/typer-rich/patterns/user-config.md
@.context/typer-rich/patterns/interactive-prompts.md
@.context/typer-rich/patterns/testing.md
```

### 4. Commit it to your project

```bash
git add .context/ CLAUDE.md
git commit -m "add typer-rich context"
```

### When the context changes

```bash
# in this repo
git pull

# in your project, re-copy the changed files
cp /path/to/python-contexts/cli/typer-rich/*.md          .context/typer-rich/
cp /path/to/python-contexts/cli/typer-rich/patterns/*.md .context/typer-rich/patterns/

git add .context/
git commit -m "update typer-rich context"
```

## Option B — Chat-based usage (Claude.ai Projects, etc.)

1. Create a new Project in your chat UI of choice.
2. Name it something like `typer-rich context`.
3. Upload every file in this folder (including everything under
   `patterns/`).
4. Always work inside that Project — the context loads automatically in
   every conversation within it.

## Structure

```
typer-rich/
  SKILL.md              → entry point (general instructions + index)
  architecture.md         → layers, command structure, folder tree
  stack.md                  → required libraries and why
  conventions.md              → naming, service pattern, output conventions
  rules.md                      → what is FORBIDDEN, with ❌/✅ examples
  patterns/
    new-command.md               → step-by-step to add a new command
    error-handling.md              → typed CLI errors, single top-level handler, exit codes
    output.md                        → Rich conventions: console, tables, panels, progress
    user-config.md                     → reading/writing persistent user configuration
    interactive-prompts.md               → prompts and confirmations via Rich
    testing.md                             → testing commands with CliRunner, testing services directly
```

## Philosophy

A CLI command is a thin entry point, not a place for business logic — the
same discipline used across this collection applies here too, with a
terminal instead of an HTTP request or a queue message as the trigger. A
command parses input, calls a service, and formats the result; it does
not decide business outcomes itself.

## Not covered here

Packaging and distribution (PyPI, `pipx`, `uvx`, standalone binaries) is
intentionally left to a separate, dedicated context. This context covers
the CLI's internal structure and behavior only.

## Contributing

When changing a pattern:

1. Edit the relevant file and open a pull request explaining the
   reasoning.
2. Once merged, consumers re-copy the changed files into `.context/`
   (Claude Code) or re-upload them (chat-based Projects).
