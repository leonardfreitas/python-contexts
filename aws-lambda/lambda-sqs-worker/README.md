# lambda-sqs-worker

Context for building asynchronous message-processing workers on AWS
Lambda that consume from SQS, using AWS SAM — the rules, patterns, and
conventions an AI coding assistant should follow before generating any
code in this kind of project.

Part of the [`python-contexts`](../../) collection. This context is fully
self-contained: it does not reference or depend on any other context in
this repository (including `lambda-api`), and can be adopted on its own
in a project with no relationship to any other context here.

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
# at the root of your worker project
mkdir -p .context/lambda-sqs-worker/patterns

cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/SKILL.md          .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/architecture.md   .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/stack.md          .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/conventions.md    .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/rules.md          .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/patterns/*.md     .context/lambda-sqs-worker/patterns/
```

### 3. Create a root instruction file

If you're using Claude Code, create `CLAUDE.md` at your project root:

```markdown
# Project name

You are a senior engineer on this project. Follow all the contexts below
before generating any code.

@.context/lambda-sqs-worker/SKILL.md
@.context/lambda-sqs-worker/architecture.md
@.context/lambda-sqs-worker/stack.md
@.context/lambda-sqs-worker/conventions.md
@.context/lambda-sqs-worker/rules.md
@.context/lambda-sqs-worker/patterns/new-message-handler.md
@.context/lambda-sqs-worker/patterns/batch-processing.md
@.context/lambda-sqs-worker/patterns/idempotency.md
@.context/lambda-sqs-worker/patterns/error-handling.md
@.context/lambda-sqs-worker/patterns/testing.md
@.context/lambda-sqs-worker/patterns/deploy.md
@.context/lambda-sqs-worker/patterns/local-development.md
```

Then add **one** of the two data-access pattern files, depending on your
project's database:

```markdown
@.context/lambda-sqs-worker/patterns/data-access-sql.md
```
or
```markdown
@.context/lambda-sqs-worker/patterns/data-access-dynamodb.md
```

### 4. Commit it to your project

```bash
git add .context/ CLAUDE.md
git commit -m "add lambda-sqs-worker context"
```

### When the context changes

```bash
# in this repo
git pull

# in your project, re-copy the changed files
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/*.md          .context/lambda-sqs-worker/
cp /path/to/python-contexts/aws-lambda/lambda-sqs-worker/patterns/*.md .context/lambda-sqs-worker/patterns/

git add .context/
git commit -m "update lambda-sqs-worker context"
```

## Option B — Chat-based usage (Claude.ai Projects, etc.)

1. Create a new Project in your chat UI of choice.
2. Name it something like `lambda-sqs-worker context`.
3. Upload every file in this folder (including everything under
   `patterns/`), or skip the data-access pattern file that doesn't apply
   to your project.
4. Always work inside that Project — the context loads automatically in
   every conversation within it.

## Structure

```
lambda-sqs-worker/
  SKILL.md              → entry point (general instructions + index)
  architecture.md         → layers, bounded-context rule, batch processing model, folder structure
  stack.md                  → required libraries and why
  conventions.md              → naming, message/service pattern
  rules.md                      → what is FORBIDDEN, with ❌/✅ examples
  patterns/
    new-message-handler.md        → step-by-step to add a new message type
    batch-processing.md             → BatchProcessor, partial batch failure, sequential vs parallel
    idempotency.md                    → why it's mandatory, how to configure it
    error-handling.md                   → RetryableError vs PermanentError, DLQ, redrive policy
    testing.md                            → testing pyramid, simulating batch events, mocking AWS
    deploy.md                               → deploy script, SAM template, queue/DLQ configuration
    local-development.md                      → sam local invoke, sample event payloads
    data-access-sql.md                          → SQL-specific data access
    data-access-dynamodb.md                       → DynamoDB-specific data access
```

## Philosophy

SQS delivers messages with an **at-least-once** guarantee, and a single
invocation can receive a **batch** of messages where some succeed and
others fail independently. This context exists to make both of those
properties explicit and safe by default: idempotency isn't optional,
partial batch failure is always reported precisely, and a message's fate
(retry vs permanent failure) is always an explicit decision in the code —
never an accident of an uncaught exception.

One worker equals one bounded context — never an arbitrary count of
message types. See `architecture.md` for the full reasoning and examples.

## Choosing SQL or DynamoDB

Almost everything in this context is identical regardless of your
business data's database choice — only the data-access layer differs.
Read `patterns/data-access-sql.md` or `patterns/data-access-dynamodb.md`
depending on your project. Note that the idempotency store is always
DynamoDB regardless of this choice (see `patterns/idempotency.md`) — that
table's technology and your business data's technology are independent
decisions.

## Contributing

When changing a pattern:

1. Edit the relevant file and open a pull request explaining the
   reasoning.
2. Once merged, consumers re-copy the changed files into `.context/`
   (Claude Code) or re-upload them (chat-based Projects).
