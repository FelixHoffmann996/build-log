# Malformed User Reminder Payloads: Invalid JSON at a Node.js 256 KB Queue Limit

## TL;DR

For user reminders, fix malformed queue payloads by validating the webhook body before enqueueing, measuring the serialized UTF-8 bytes against the 256 KB limit, storing only a small reference payload, and making the consumer idempotent before adding retries. A retry cannot repair invalid JSON or an oversized body; it only repeats the same permanent failure.

The evaluation constraint matters: a weekly healthtech digest must reach each active customer once, while duplicate sends are worse than a delayed send. The simple design puts the full webhook body, rendered digest, and customer record into the queue. It works with small fixtures, then crosses the limit as real summaries grow. The safer design queues an immutable reminder reference and lets the worker load the content from controlled storage.

Small messages win.

## Start with a byte budget, not queue configuration

There are three boundaries in this job, and treating them as one is the root mistake. The HTTP boundary accepts a webhook body. The producer boundary turns validated input into a queue message. The worker boundary consumes that message and performs the send. Each boundary needs its own validation because each has a different failure meaning. Invalid JSON belongs at the HTTP boundary. If parsing fails, no reminder exists yet, so the handler should reject the request and avoid queueing anything. A valid JSON value with the wrong shape belongs at the schema boundary: an array, a missing `customerId`, or a date in an unexpected representation can all be valid JSON while still being invalid input. An oversized serialized message belongs at the producer boundary. None of these cases becomes more valid after waiting. The failed approach is to catch every error, return success to the webhook caller, and rely on queue retries. That hides a rejected input behind an accepted response and turns a deterministic validation error into repeated work. Exponential backoff spaces attempts out; it doesn't change the bytes or schema. Use it for a failure that may clear between attempts, not as a general error handler.

There is another subtle trap. JavaScript string length is not a payload byte count, and the queue limit applies to the serialized message that actually crosses the boundary. Measure `Buffer.byteLength(serialized, "utf8")` after `JSON.stringify`, including envelope fields such as version and idempotency key. Don't estimate from the source object, don't count characters, and don't wait for the queue client to be the first component that notices a large body. I'm not sure which field causes the growth in every system; only production size telemetry can answer that. In a weekly digest, likely candidates include generated summary text, duplicated profile data, and embedded template markup. The architectural fix doesn't depend on guessing correctly: keep those variable-sized fields outside the queue message.

| Failure | Classification | Producer action | Retry? |
| --- | --- | --- | --- |
| Body cannot be parsed as JSON | Permanent input error | Reject before enqueue | No |
| JSON fails reminder schema | Permanent input error | Return field-level validation details | No |
| Serialized message exceeds 256 KB | Permanent construction error | Store content and enqueue a reference | No |
| Worker loses a dependency temporarily | Transient execution error | Preserve the same idempotency key | Yes, with bounded backoff |
| Reminder was already sent | Duplicate delivery | Record a no-op outcome | No |

No retry.

This split prevents poison messages from consuming retry capacity. More important, it makes alerts useful: a validation counter points to a caller or deployment regression, while a retry counter points to a temporary execution problem. Mixing them produces a noisy queue and no diagnosis.

## How can Node.js schema validation keep invalid JSON webhook bodies under 256 KB?

Start with one canonical schema and use it twice: once in the webhook handler and once in the worker. The second check is deliberate. Queue contents can outlive a deployment, and a versioned envelope lets a worker reject an unsupported shape without pretending the send was attempted. The producer should accept the business input, persist a snapshot or rendering input, then enqueue identifiers and a content reference. The idempotency key must describe the business operation, not a delivery attempt. For this scenario, `weekly-digest:{customerId}:{weekStart}` is stable across a webhook retry, a queue redelivery, and a worker retry. A random key generated per attempt defeats deduplication. Likewise, a timestamp captured at enqueue time can create a new identity for the same weekly reminder. Validation order should be boring and explicit: enforce the request-body cap while reading, parse once, validate the parsed value, construct a minimal envelope, serialize once, measure exact bytes, and enqueue. If the framework has already parsed the body, retain middleware-level size limits and feed the parsed unknown value into the same schema validator. Don't stringify an untrusted value several times in separate layers; the value checked should be the value sent.

For the full digest, persist either the immutable rendered content or immutable inputs plus a renderer version. Persisting only a pointer to a mutable customer record can make a retry send different content from the first attempt. That may be acceptable for some reminder classes, but health messaging needs an explicit decision. If reproducibility matters, snapshot the send inputs and retain the snapshot according to the application's data policy. This is the key trade-off: a reference payload adds a storage read and lifecycle work. Inline payloads remain reasonable when a hard upper bound keeps every serialized message comfortably below 256 KB and the contents aren't sensitive enough to make queue copies undesirable. If low latency is the dominant constraint and messages are tightly bounded, stick with inline data. If digest size varies with generated text, use references.

## Put the byte contract in one TypeScript boundary

The example below keeps transport and storage behind generic interfaces. It doesn't assume a queue vendor, and it treats schema failure and payload size as permanent producer errors. The `enqueueWeeklyDigest` function receives parsed input because request-body enforcement and JSON parsing belong in the HTTP adapter; `parseWebhookJson` shows the adapter's parsing step without tying it to a framework.

```ts
type DigestRequest = {
  customerId: string;
  weekStart: string;
  content: string;
};

type DigestMessageV1 = {
  version: 1;
  customerId: string;
  weekStart: string;
  contentRef: string;
  idempotencyKey: string;
};

interface ContentStore {
  putImmutable(key: string, content: string): Promise<string>;
  get(ref: string): Promise<string>;
}

interface ReminderQueue {
  send(serializedMessage: string): Promise<void>;
}

const MAX_QUEUE_BYTES = 256 * 1024;

function parseWebhookJson(rawBody: string): unknown {
  try {
    return JSON.parse(rawBody) as unknown;
  } catch {
    throw new Error("invalid_json");
  }
}

function requireDigestRequest(value: unknown): DigestRequest {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("invalid_reminder_schema");
  }

  const input = value as Record<string, unknown>;
  if (
    typeof input.customerId !== "string" || input.customerId.length === 0 ||
    typeof input.weekStart !== "string" || input.weekStart.length === 0 ||
    typeof input.content !== "string" || input.content.length === 0
  ) {
    throw new Error("invalid_reminder_schema");
  }

  return {
    customerId: input.customerId,
    weekStart: input.weekStart,
    content: input.content,
  };
}

function serializeWithinQueueLimit(message: DigestMessageV1): string {
  const serialized = JSON.stringify(message);
  const bytes = Buffer.byteLength(serialized, "utf8");
  if (bytes > MAX_QUEUE_BYTES) {
    throw new Error("queue_payload_over_256kb");
  }
  return serialized;
}

async function enqueueWeeklyDigest(
  parsedBody: unknown,
  store: ContentStore,
  queue: ReminderQueue,
): Promise<void> {
  const request = requireDigestRequest(parsedBody);
  const idempotencyKey = `weekly-digest:${request.customerId}:${request.weekStart}`;
  const contentRef = await store.putImmutable(idempotencyKey, request.content);

  const message: DigestMessageV1 = {
    version: 1,
    customerId: request.customerId,
    weekStart: request.weekStart,
    contentRef,
    idempotencyKey,
  };

  await queue.send(serializeWithinQueueLimit(message));
}
```

The `256 * 1024` threshold expresses the stated 256 KB contract in bytes. A real adapter should also impose a separate request-body limit before buffering the entire webhook; that protects the HTTP process, while the serialization check protects the queue contract. Those limits may intentionally differ because the inbound request contains `content`, whereas the queued envelope contains `contentRef`.

There is a consistency edge between storing content and sending the message. If queue submission doesn't happen after storage succeeds, the immutable object can be left without a queue reference. That's cleanup work, not a reason to put the content back into the message. A periodic reconciler can compare durable reminder intent with enqueue state, but its exact design depends on the transaction facilities available. Your mileage may vary. On the worker side, claim the idempotency key in durable storage with a state transition that cannot be won by two workers. Only the winner may send. Mark completion after the downstream send returns successfully, and retain enough state to distinguish `in_progress` from `sent`. The dangerous interval is after the external send succeeds but before the completion record is durable. If the downstream channel accepts an idempotency key, pass the same stable key through; if it doesn't, exactly-once delivery cannot be guaranteed by the queue alone.

## Use a reminder ledger before spending the retry budget

Retries should be bounded, delayed, and attached to the same operation identity. Exponential backoff reduces the rate of repeated attempts by increasing delay between them. Add jitter when many digest jobs can fail together, because synchronized retries can recreate the load spike. Keep an attempt ceiling and move exhausted work to a review path with the original idempotency key and failure classification intact. Do not retry `invalid_json`, `invalid_reminder_schema`, or `queue_payload_over_256kb`. They require a caller change or a producer change. Retrying them spends queue operations and delays the signal without changing the result. This distinction also keeps cost visible: count bytes written, storage reads, queue deliveries, and downstream attempts per completed digest rather than looking only at the number of scheduled jobs.

The scheduler should create intent, not perform the whole fan-out in one invocation. For a weekly digest, record the target week and enumerate active customers through bounded batches. Each customer gets a stable operation key. If the scheduler runs twice, the same intent keys should converge on the same work records rather than create a second campaign. If the active-customer definition can change during a run, decide whether membership is snapshotted at campaign start or evaluated per batch; either choice is defensible, but leaving it implicit makes reconciliation difficult. Keep observability close to those state changes. Log the message version, idempotency key, byte count, attempt number, and error class, but don't log the full health digest or raw webhook body. Emit separate counters for rejected JSON, rejected schema, oversized producer messages, deduplicated deliveries, retry attempts, and completed sends. A single `job_failed` metric erases the information needed to choose a fix. Before copying this design, measure the p50, p95, and maximum serialized message size; the distribution of content-object sizes; duplicate suppression count; attempts per completed digest; time from scheduled intent to send; and the age and count of orphaned content references. The maximum matters for the 256 KB boundary, while percentiles expose normal cost and latency. Review those numbers across a full weekly cycle because a small development fixture won't represent the longest customer digest.

Test the boundaries directly. Feed the parser truncated JSON. Pass valid JSON with a missing field. Generate a message exactly at the accepted byte threshold and one byte beyond it. Deliver the same message concurrently to two workers and assert that only one wins the idempotency claim. Then simulate a worker restart at each state transition. These tests are more valuable than another happy-path scheduler test because they exercise the places where duplicates and poison messages originate. The final choice is conditional. Reference payloads are not suitable when the extra storage lookup violates a proven latency budget and inline content has a strict, small upper bound. A queue is also the wrong center of gravity when the reminder must be sent synchronously as part of a user request. For a weekly digest with variable content and duplicate-sensitive delivery, however, a compact reference message plus durable idempotency state gives the cleanest failure model.

Measure first.

## References

- https://en.wikipedia.org/wiki/Exponential_backoff
- https://vercel.com/docs/cron-jobs

## Further reading

- Exponential backoff: https://en.wikipedia.org/wiki/Exponential_backoff
- Cron job scheduling documentation: https://vercel.com/docs/cron-jobs
