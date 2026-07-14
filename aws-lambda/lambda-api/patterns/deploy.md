# Deploy

## The script

`scripts/deploy.sh` builds and deploys the SAM application to a specific
environment, loading configuration from an environment-specific `.env`
file.

```bash
#!/bin/bash
# scripts/deploy.sh
set -euo pipefail

ENV_TYPE="${1:-staging}"
ENV_FILE=".env.${ENV_TYPE}"
PROJECT_NAME="${PROJECT_NAME:-my-api}"

if [ ! -f "$ENV_FILE" ]; then
  echo "Environment file $ENV_FILE not found!"
  exit 1
fi

export $(grep -v '^#' "$ENV_FILE" | xargs)

sam build
sam deploy \
  --parameter-overrides \
    EnvType="$ENV_TYPE" \
    JwtSecretKey="$JWT_SECRET_KEY" \
    EncryptionKey="$ENCRYPTION_KEY" \
    JwtExpireHours="$JWT_EXPIRE_HOURS" \
  --stack-name "${PROJECT_NAME}-${ENV_TYPE}" \
  --capabilities CAPABILITY_IAM
```

Usage:

```bash
./scripts/deploy.sh staging
./scripts/deploy.sh production
```

The parameters passed via `--parameter-overrides` above (`JwtSecretKey`,
`EncryptionKey`, `JwtExpireHours`) are illustrative — every real project
defines its own set based on what its `template.yaml` declares as
parameters. The pattern that matters is the mechanism, not this specific
list.

## Why `set -euo pipefail`

Without it, if `sam build` fails, the script can still continue and attempt
`sam deploy` — a partial/broken build could get deployed. `set -e` stops
the script on the first error, `set -u` fails if a variable is referenced
but never defined (catches a forgotten value in an `.env` file), and
`pipefail` makes sure an error inside a pipe isn't silently swallowed.

## Why the project name is a variable, never hardcoded

`PROJECT_NAME` must come from a variable (with a sane default), never be
hardcoded as a literal string inside the script. A hardcoded name is easy
to forget to change when the script is copied into a new project, and it
silently deploys to — or worse, overwrites — the wrong stack.

## Secrets in `--parameter-overrides`: known limitation and mitigation

> Any value passed via `--parameter-overrides` is visible in the shell's
> command history and in the process list while the command is running.
> This is fine for non-sensitive values (like `EnvType`), but it's a real
> exposure for secrets (like a JWT signing key or an encryption key).

**Required mitigation at the `template.yaml` level:** mark every sensitive
parameter with `NoEcho: true`. This keeps the value out of CloudFormation
logs and the console — it does not fix the shell-history exposure, but
it's the minimum required step:

```yaml
Parameters:
  JwtSecretKey:
    Type: String
    NoEcho: true
```

**Path to maturity (not required on day one):** once a project is handling
real production secrets, migrate from passing raw values via
`--parameter-overrides` to referencing a managed secrets store (e.g. SSM
Parameter Store or Secrets Manager) directly from the template:

```yaml
JwtSecretKey:
  Type: AWS::SSM::Parameter::Value<String>
  Default: /my-api/jwt-secret-key
```

Starting with `--parameter-overrides` + `NoEcho: true` is the accepted
default for a new project — it's simple and safe enough for most cases.
Treat the Secrets Manager/Parameter Store migration as a conditional
upgrade (same philosophy as any other conditional dependency in this
context collection): add it when the project's risk profile justifies the
extra infrastructure, not preemptively.

## Environment files

One `.env.<environment>` file per environment (e.g. `.env.dev`,
`.env.staging`, `.env.production`). None of them are committed:

```gitignore
# .gitignore
.env.dev
.env.staging
.env.production
```

`.env.example` **is** committed, documenting every expected key with no
real values — this is the template a new developer or a CI pipeline uses
to know what to fill in.

## `template.yaml` reference

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  my-api

Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production
  JwtSecretKey:
    Type: String
    NoEcho: true
  EncryptionKey:
    Type: String
    NoEcho: true
  JwtExpireHours:
    Type: Number

Globals:
  Function:
    Runtime: python3.13
    Timeout: 30
    MemorySize: 256
    LoggingConfig:
      LogFormat: JSON
      ApplicationLogLevel: INFO
      SystemLogLevel: INFO
    Environment:
      Variables:
        ENV_TYPE: !Ref EnvType
        JWT_SECRET_KEY: !Ref JwtSecretKey
        ENCRYPTION_KEY: !Ref EncryptionKey
        JWT_EXPIRE_HOURS: !Ref JwtExpireHours

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub "my-api-${EnvType}"
      CodeUri: .
      Handler: app.handler.handler
      Events:
        Api:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY
```

`Handler: app.handler.handler` matches the folder structure defined in
`architecture.md` — `app/handler.py` containing `handler = Mangum(app)`.
Adjust the path if your project uses a different root package name.

## Summary

| Rule | Reason |
|---|---|
| `PROJECT_NAME` is a variable, never hardcoded in the script | Avoids deploying to the wrong stack when the script is copied into a new project |
| `set -euo pipefail` in every deploy/local script | Stops on the first error instead of continuing with a broken build |
| Every sensitive template parameter has `NoEcho: true` | Minimum required mitigation against leaking secrets in CloudFormation logs/console |
| `.env.<environment>` files are never committed; `.env.example` always is | Prevents committing real secrets, documents expected configuration |
| Secrets Manager/Parameter Store is a conditional upgrade, not a day-one requirement | Matches the "add complexity only when justified" philosophy used across every context in this collection |
