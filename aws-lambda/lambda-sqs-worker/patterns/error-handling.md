# Error Handling

## The question that matters here isn't "what status code" — it's "should this message be retried"

Unlike an HTTP-facing context, there's no client waiting for a response.
Every exception's real consequence is: does this message go back to the
queue for another attempt, or is it a lost cause that will just keep
failing?

## The exception hierarchy

```python
# app/exceptions.py
class WorkerError(Exception):
    """Base class for any error raised while processing a message."""
    default_message = "Worker error"

    def __init__(self, message: str = None, **context):
        self.message = message or self.default_message
        self.context = context
        super().__init__(self.message)


class RetryableError(WorkerError):
    """Transient failure — safe and expected to succeed on a later retry.
    Examples: a downstream API timeout, a database temporarily unavailable."""
    default_message = "Transient failure, safe to retry"


class PermanentError(WorkerError):
    """Will never succeed no matter how many times it's retried.
    Examples: a malformed payload that will never become valid, a
    referenced entity that was permanently deleted."""
    default_message = "Message cannot be processed, will not succeed on retry"
```

## Both categories are still reported as a batch item failure

There's no code-level mechanism to route a message straight to the DLQ,
skipping retries — that's controlled by queue configuration
(`maxReceiveCount`), not by the exception type. Both `RetryableError` and
`PermanentError` result in the message being reported as failed in
`batchItemFailures`; the distinction exists to document intent (and can
drive different logging/alerting) rather than to change code-level
routing.

```python
def record_handler(record):
    body = json.loads(record.body)
    try:
        message = OrderCreatedMessage.model_validate(body["payload"])
    except ValidationError as e:
        raise PermanentError(f"Malformed message: {e}") from e

    try:
        handle_order_created(message)
    except ExternalServiceTimeout as e:
        raise RetryableError(str(e)) from e
```

## Never silently swallow an exception inside `record_handler`

> Rule: no `try/except` inside message processing catches an exception
> and treats it as success. Every failure, retryable or permanent, must
> propagate so `BatchProcessor` reports it as a `batchItemFailure`.
> Silently swallowing an exception here means data loss — the message is
> acknowledged as processed and removed from the queue despite never
> actually being handled correctly.

## DLQ and redrive policy — where "give up" actually happens

Since there's no code-level DLQ routing, the real mechanism for "stop
retrying and stop losing throughput to a poison message" lives in the SAM
template:

```yaml
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
```

After `maxReceiveCount` failed attempts, the message moves to the DLQ
automatically — a `PermanentError` will reach the DLQ after the same
number of attempts as a `RetryableError` that never resolves. Set
`maxReceiveCount` with both failure types in mind: high enough that a
transient error has a real chance to succeed on retry, low enough that a
permanent error doesn't waste excessive throughput before giving up.

## `VisibilityTimeout` must exceed the Lambda's timeout

> Rule: the queue's `VisibilityTimeout` must be greater than the Lambda
> function's configured timeout — AWS's guidance is at least 6x, to
> accommodate the function's own internal retry behavior. If
> `VisibilityTimeout` is too short, a message can become visible again
> (and get picked up by a second concurrent invocation) before the first
> invocation even finishes processing it — causing duplicate concurrent
> processing that idempotency (see `patterns/idempotency.md`) is the
> actual safety net for, but that a correctly configured
> `VisibilityTimeout` avoids triggering unnecessarily in the first place.

## Summary

| Rule | Reason |
|---|---|
| `RetryableError` vs `PermanentError` document intent | Drives logging/alerting distinctions even though both are reported the same way to `BatchProcessor` |
| No exception is silently swallowed inside message processing | Prevents data loss disguised as success |
| Every main queue has a DLQ with `maxReceiveCount` configured | Prevents a poison message from consuming throughput indefinitely |
| `VisibilityTimeout` ≥ 6x the Lambda timeout | Avoids duplicate concurrent processing of the same message |
