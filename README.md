# python-contexts

A collection of reusable AI context repositories for Python projects — the
architecture rules, patterns, and conventions an AI coding assistant (like
Claude Code) should follow before generating code.

Instead of re-explaining your architecture decisions in every chat, or
hoping an AI assistant infers the "right" way to structure a service layer,
error handling, or transactions, you point it at a versioned, reviewable
set of markdown files. The assistant reads the context once and generates
code that matches how your team actually works — not a generic tutorial
pattern.

## Why this exists

AI coding assistants are very good at writing code that *works*, and much
less consistent at writing code that matches a specific team's standards —
where business logic should live, how errors propagate, when a transaction
is needed, what a test at each layer should and shouldn't cover. Left
unguided, the same assistant will happily generate five different
architectural styles across five different conversations.

A context repository fixes that: the decisions are made once, written down
with their reasoning (not just the rule, but *why*), and reused across every
project and every conversation.

## Structure

This repository is organized by language, and within each language, by
framework/stack:

```
python-contexts/
  django/
    api-drf/          → Django + Django REST Framework API context
    (more contexts may be added here over time, e.g. a Celery-worker-only
    context, or an admin-only Django project context)
  aws-lambda/
    lambda-api/       → REST API on AWS Lambda (FastAPI + Mangum + SAM) context
    lambda-sqs-worker/ → Async SQS-consumer worker on AWS Lambda (SAM) context
  cli/
    typer-rich/       → Command-line tool (Typer + Rich) context
```

Each context folder is self-contained: it has its own `README.md` with
installation instructions, and can be adopted independently of the others.

## Available contexts

| Context | Path | Description |
|---|---|---|
| Django + DRF API | [`django/api-drf`](./django/api-drf) | Layered architecture (view/serializer/service/model), domain exceptions, transactions, testing pyramid, deployment, permissions |
| AWS Lambda API | [`aws-lambda/lambda-api`](./aws-lambda/lambda-api) | REST API on AWS Lambda with FastAPI + Mangum + SAM, one-Lambda-per-bounded-context, SQL/DynamoDB data-access patterns, testing, deploy |
| AWS Lambda SQS Worker | [`aws-lambda/lambda-sqs-worker`](./aws-lambda/lambda-sqs-worker) | Async message-processing worker on AWS Lambda consuming SQS with SAM, batch processing, idempotency, error handling, SQL/DynamoDB data-access patterns |
| CLI (Typer + Rich) | [`cli/typer-rich`](./cli/typer-rich) | Command-line tools with Typer and Rich, command structure, output formatting, interactive prompts, user config, error handling, testing |

## How to use a context

Each context supports two consumption modes:

- **Claude Code** (or any coding agent that reads a project-level context
  file) — copy the context folder into your project and reference it from
  a root instruction file (e.g. `CLAUDE.md`). This is the recommended
  approach: the context is committed to your repository, versioned, and
  shared identically across your whole team.
- **Chat-based usage** (e.g. Claude.ai Projects) — upload the context files
  as project knowledge and reference them in conversation.

See the README inside each context folder for exact setup steps.

## Philosophy

A few principles apply across every context in this repository, regardless
of language or framework:

- **Business logic is isolated and testable outside of any specific
  transport layer** (HTTP, message queue, CLI). Frameworks are tools to be
  leveraged, never owners of the domain.
- **Every rule documents its reasoning**, not just the constraint. A rule
  without a "why" gets silently violated the first time it's inconvenient.
- **Dependencies are added only when justified**, not "in case we need it
  later." Conditional infrastructure that nobody uses is a liability, not
  a feature.
- **A single source of truth beats two synchronized ones.** Whenever a
  design would require two representations of the same information to stay
  in sync manually, the context favors collapsing them into one.

## Review

Every rule here goes through review before it's adopted — first-pass review
comes from a maintainer with over a decade of hands-on Python experience,
and changes are also checked by other engineers with significant production
experience. The point isn't to claim these rules are the only right answer,
but to make sure each one reflects something that has actually held up in
practice, not just a theory about how code "should" be structured.

## Contributing

When changing a pattern:

1. Edit the relevant file and open a pull request explaining the reasoning
   for the change, not just the change itself.
2. Once merged, consumers of the context re-copy the updated files (Claude
   Code) or re-upload them (chat-based Projects).

## License

MIT — use this freely, adapt it to your own team's conventions, and feel
free to fork it into your own context repository.
