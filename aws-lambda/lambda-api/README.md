# lambda-api

Context for building REST APIs on AWS Lambda using FastAPI, Mangum, and
AWS SAM — the rules, patterns, and conventions an AI coding assistant
should follow before generating any code in this kind of project.

Part of the [`python-contexts`](../../) collection. Independent from the
Django context in this repository — shares only foundational principles,
not specific rules.

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
# at the root of your Lambda API project
mkdir -p .context/lambda-api/patterns

cp /path/to/python-contexts/aws-lambda/lambda-api/SKILL.md          .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/architecture.md   .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/stack.md          .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/conventions.md    .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/rules.md          .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/patterns/*.md     .context/lambda-api/patterns/
```

### 3. Create a root instruction file

If you're using Claude Code, create `CLAUDE.md` at your project root:

```markdown
# Project name

You are a senior engineer on this project. Follow all the contexts below
before generating any code.

@.context/lambda-api/SKILL.md
@.context/lambda-api/architecture.md
@.context/lambda-api/stack.md
@.context/lambda-api/conventions.md
@.context/lambda-api/rules.md
@.context/lambda-api/patterns/new-endpoint.md
@.context/lambda-api/patterns/error-handling.md
@.context/lambda-api/patterns/testing.md
@.context/lambda-api/patterns/deploy.md
@.context/lambda-api/patterns/local-development.md
```

Then add **one** of the two data-access pattern files, depending on your
project's database:

```markdown
@.context/lambda-api/patterns/data-access-sql.md
```
or
```markdown
@.context/lambda-api/patterns/data-access-dynamodb.md
```

### 4. Commit it to your project

```bash
git add .context/ CLAUDE.md
git commit -m "add lambda-api context"
```

### When the context changes

```bash
# in this repo
git pull

# in your project, re-copy the changed files
cp /path/to/python-contexts/aws-lambda/lambda-api/*.md          .context/lambda-api/
cp /path/to/python-contexts/aws-lambda/lambda-api/patterns/*.md .context/lambda-api/patterns/

git add .context/
git commit -m "update lambda-api context"
```

## Option B — Chat-based usage (Claude.ai Projects, etc.)

1. Create a new Project in your chat UI of choice.
2. Name it something like `lambda-api context`.
3. Upload every file in this folder (including everything under
   `patterns/`), or skip the data-access pattern file that doesn't apply
   to your project.
4. Always work inside that Project — the context loads automatically in
   every conversation within it.

## Structure

```
lambda-api/
  SKILL.md              → entry point (general instructions + index)
  architecture.md         → layers, bounded-context-per-Lambda rule, folder structure
  stack.md                  → required libraries and why
  conventions.md              → naming, service pattern, schema strategy
  rules.md                      → what is FORBIDDEN, with ❌/✅ examples
  patterns/
    new-endpoint.md               → step-by-step to add a new route
    error-handling.md               → domain exceptions + FastAPI exception handler
    testing.md                        → testing pyramid, bypassing Mangum, mocking AWS
    deploy.md                          → deploy script, SAM template, secrets handling
    local-development.md                → running the app locally, Uvicorn vs sam local
    data-access-sql.md                    → SQL-specific: connection pooling, RDS Proxy, migrations
    data-access-dynamodb.md                 → DynamoDB-specific: single-table design, access patterns
```

## Philosophy

A Lambda function has a fundamentally different lifecycle than a
long-running server — it's born, runs briefly, and dies. This context
applies the same underlying discipline used across this collection
(business logic isolated and testable, one source of truth, dependencies
added only when justified) to that different shape of application: cold
start cost, packaging boundaries, and connection handling all get
Lambda-specific answers here.

One Lambda equals one bounded context — never an arbitrary route count.
See `architecture.md` for the full reasoning and examples.

## Choosing SQL or DynamoDB

Almost everything in this context is identical regardless of database
choice — only the data-access layer differs. Read `patterns/data-access-
sql.md` or `patterns/data-access-dynamodb.md` depending on your project,
not both, unless you're actively comparing the two for a new project
decision.

## Contributing

When changing a pattern:

1. Edit the relevant file and open a pull request explaining the
   reasoning.
2. Once merged, consumers re-copy the changed files into `.context/`
   (Claude Code) or re-upload them (chat-based Projects).
