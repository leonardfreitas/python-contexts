# api-drf

Context for building Django + Django REST Framework APIs — the rules,
patterns, and conventions an AI coding assistant should follow before
generating any code in this kind of project.

Part of the [`python-contexts`](../../) collection.

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
# at the root of your Django project
mkdir -p .context/api-drf/patterns

cp /path/to/python-contexts/django/api-drf/SKILL.md          .context/api-drf/
cp /path/to/python-contexts/django/api-drf/architecture.md   .context/api-drf/
cp /path/to/python-contexts/django/api-drf/stack.md          .context/api-drf/
cp /path/to/python-contexts/django/api-drf/conventions.md    .context/api-drf/
cp /path/to/python-contexts/django/api-drf/rules.md          .context/api-drf/
cp /path/to/python-contexts/django/api-drf/auth.md           .context/api-drf/
cp /path/to/python-contexts/django/api-drf/permissions.md    .context/api-drf/
cp /path/to/python-contexts/django/api-drf/patterns/*.md     .context/api-drf/patterns/
```

### 3. Create a root instruction file

If you're using Claude Code, create `CLAUDE.md` at your project root:

```markdown
# Project name

You are a senior engineer on this project. Follow all the contexts below
before generating any code.

@.context/api-drf/SKILL.md
@.context/api-drf/architecture.md
@.context/api-drf/stack.md
@.context/api-drf/conventions.md
@.context/api-drf/rules.md
@.context/api-drf/auth.md
@.context/api-drf/permissions.md
@.context/api-drf/patterns/new-module.md
@.context/api-drf/patterns/error-handling.md
@.context/api-drf/patterns/transactions.md
@.context/api-drf/patterns/testing.md
@.context/api-drf/patterns/urls-and-routers.md
@.context/api-drf/patterns/deploy.md
@.context/api-drf/patterns/admin.md
```

(Other coding agents that support a project-level instruction file can
reference the same files the same way.)

### 4. Commit it to your project

```bash
git add .context/ CLAUDE.md
git commit -m "add api-drf context"
```

Every developer who clones the project now has the context ready — they
just need the coding agent installed.

### When the context changes

```bash
# in this repo
git pull

# in your project, re-copy the changed files
cp /path/to/python-contexts/django/api-drf/*.md          .context/api-drf/
cp /path/to/python-contexts/django/api-drf/patterns/*.md .context/api-drf/patterns/

git add .context/
git commit -m "update api-drf context"
```

## Option B — Chat-based usage (Claude.ai Projects, etc.)

1. Create a new Project in your chat UI of choice.
2. Name it something like `api-drf context`.
3. In the project's knowledge/files section, upload every file in this
   folder (including everything under `patterns/`).
4. Always work inside that Project — the context loads automatically in
   every conversation within it.

### When the context changes

Re-upload the changed file in your Project.

## Structure

```
api-drf/
  SKILL.md              → entry point (general instructions + index)
  architecture.md        → layers, folder tree, layer boundaries
  stack.md                → required libraries, core vs conditional, with rationale
  conventions.md            → naming, service pattern, serializer strategy
  rules.md                    → what is FORBIDDEN, with ❌/✅ examples
  auth.md                       → custom user model, JWT
  permissions.md                 → groups, authorization boundaries
  patterns/
    new-module.md                 → step-by-step to scaffold a new domain module
    error-handling.md              → domain exceptions + central exception handler
    transactions.md                 → atomic() and on_commit()
    testing.md                       → what to test at each layer, testing pyramid
    urls-and-routers.md               → router vs manual path(), when to use each
    deploy.md                          → Dockerfile, migrations, multi-platform notes
    admin.md                            → what to expose in Django Admin and how
```

## Philosophy

Django, without meaning to, encourages "fat views" and "fat models" that
accumulate business logic over time. This context exists to keep business
logic isolated in a `services/` layer, testable outside the HTTP request
cycle — everything else (the ORM, DRF, the Admin) is treated as a tool to
be leveraged, never as the owner of the domain.

If you're generating code from this context and business logic seems to be
leaking into a view, serializer, or permission class — stop and move it
into `services/`. This isn't a style preference; it's the single most
important rule in this repository.

## Contributing

When changing a pattern:

1. Edit the relevant file and open a pull request explaining the reasoning.
2. Once merged, consumers re-copy the changed files into `.context/`
   (Claude Code) or re-upload them (chat-based Projects).
