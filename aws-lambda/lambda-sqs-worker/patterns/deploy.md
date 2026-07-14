# Deploy

## The script

Same shape as any other context in this collection: environment-specific
`.env` files, `set -euo pipefail`, project name as a variable.

```bash
#!/bin/bash
# scripts/deploy.sh
set -euo pipefail

ENV_TYPE="${1:-staging}"
ENV_FILE=".env.${ENV_TYPE}"
PROJECT_NAME="${PROJECT_NAME:-my-worker}"

if [ ! -f "$ENV_FILE" ]; then
  echo "Environment file $ENV_FILE not found!"
  exit 1
fi

export $(grep -v '^#' "$ENV_FILE" | xargs)

sam build
sam deploy \
  --parameter-overrides \
    EnvType="$ENV_TYPE" \
    ProjectName="$PROJECT_NAME" \
  --stack-name "${PROJECT_NAME}-${ENV_TYPE}" \
  --capabilities CAPABILITY_IAM
```

## `template.yaml` reference

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  order-events-worker

Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production
  ProjectName:
    Type: String

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
        IDEMPOTENCY_TABLE_NAME: !Ref IdempotencyTable

Resources:
  WorkerFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub "${ProjectName}-${EnvType}"
      CodeUri: .
      Handler: app.handler.handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref IdempotencyTable
        - SQSPollerPolicy:
            QueueName: !GetAtt MainQueue.QueueName
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt MainQueue.Arn
            BatchSize: 10
            FunctionResponseTypes:
              - ReportBatchItemFailures

  MainQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${ProjectName}-queue-${EnvType}"
      VisibilityTimeout: 180
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
        maxReceiveCount: 3

  DeadLetterQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${ProjectName}-dlq-${EnvType}"
      MessageRetentionPeriod: 1209600

  IdempotencyTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "${ProjectName}-idempotency-${EnvType}"
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
      TimeToLiveSpecification:
        AttributeName: expiration
        Enabled: true
```

Note `Handler: app.handler.handler`, matching the `app/handler.py`
structure from `architecture.md`. `Policies` uses SAM's policy templates
(`DynamoDBCrudPolicy`, `SQSPollerPolicy`) to scope IAM permissions to
exactly the queue and table this function needs — avoid a broad
`AmazonSQSFullAccess`-style policy.

## FIFO queue variant

If the project uses a FIFO queue instead of standard, the queue
definitions change (queue name must end in `.fifo`, `FifoQueue: true`
required, `ContentBasedDeduplication` optional depending on whether the
producer supplies a `MessageDeduplicationId`):

```yaml
MainQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: !Sub "${ProjectName}-queue-${EnvType}.fifo"
    FifoQueue: true
    ContentBasedDeduplication: true
    VisibilityTimeout: 180
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
      maxReceiveCount: 3

DeadLetterQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: !Sub "${ProjectName}-dlq-${EnvType}.fifo"
    FifoQueue: true
    MessageRetentionPeriod: 1209600
```

The Lambda function definition and event source mapping are otherwise
unchanged — `ReportBatchItemFailures` and the `BatchProcessor` code work
identically for both queue types.

## Summary

| Rule | Reason |
|---|---|
| `PROJECT_NAME` is a variable, never hardcoded | Avoids deploying to the wrong stack when the script is copied into a new project |
| `set -euo pipefail` in the deploy script | Stops on the first error instead of continuing with a broken build |
| IAM policies scoped via SAM policy templates, not broad managed policies | Least privilege — the function only accesses its own queue and table |
| `.env.<environment>` files are never committed; `.env.example` always is | Prevents committing real secrets, documents expected configuration |
| DLQ and `FunctionResponseTypes: [ReportBatchItemFailures]` are always present in the template | Required for correct partial-failure and poison-message behavior |
