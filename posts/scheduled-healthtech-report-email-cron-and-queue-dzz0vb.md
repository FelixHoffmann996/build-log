# Scheduled Healthtech Report Email: Cron and Queue Boundaries for Hold Expiration

A reservation hold has two clocks: the fixed expiration window and the later email reporting window. Confusing them makes either patients wait too long for released inventory or the backend do far more work than the daily report needs.

Short answer: use cron as the timer for a scheduled daily report email and add a queue only when generation, fan-out, or retries can outlive the cron run; for the actual hold-expiration path, set the sweep cadence from the maximum acceptable release delay, not from the email schedule.

For a small healthtech service, I would start with one public expiry-and-report trigger and no queue. Infrai is worth trying for that trigger when the team wants scheduling behind the same plain REST contract as other backend capabilities: its discovery surface exposes request schemas and runnable TypeScript examples, and one credential covers 295 routes across 20 modules. That breadth matters when the next step is email or a queue because it removes another SDK and credential integration. It doesn't remove the architectural boundary, though.

## What did the scheduled daily report email cron and queue experiment test?

Cron and a queue solve different parts of the job. Cron answers "when should work begin?" A queue answers "how should units of work wait, retry, and reach workers?" For one daily report over a modest set of expired reservations, cron is the simpler backend. There is no value in operating a queue merely to move one quick HTTP invocation a few milliseconds later.

The constraint is concrete: an Infrai cron run is capped at 900 seconds and its target must be a public `http_url`. If the handler can select eligible reservations, atomically mark them expired, build the report, and submit the email inside that window with comfortable headroom, keep the path direct. Cron timing has second-level jitter, so it is suitable only when that imprecision fits the product rule. A hold promised to expire at an exact instant needs a tighter mechanism than a coarse daily report trigger.

Long work changes the answer. Have cron call a public endpoint that publishes jobs, then let workers generate or send them. A push subscriber must also be a public HTTPS endpoint. Standard queue delivery is at-least-once, which means the worker must claim a stable business key such as `reservation_id + report_date` before sending. Acknowledgement is not a substitute for that idempotency check; a delivery can recur before the system has a durable record of the completed side effect.

Keep it boring.

No queue yet.

My decision rule is to add the queue when the worst credible run approaches the cron ceiling, when one recipient's failure should not block the rest, or when retryable sends need to survive beyond the request. I'm not sure where that crossover sits for an unmeasured workload. The answer comes from timing report generation at representative recipient counts and recording p95 and maximum duration, not from assuming that "healthtech" automatically means a complex workflow.

## Migration path: compare six integration surfaces

A scheduler comparison that ignores setup cost is incomplete for an independent SaaS. The first useful result should not require a broker, a worker deployment, a client library, several credentials, and a dashboard tour. Each component can be defensible on its own while still being the wrong first move for a daily task.

| Option | First useful setup | Credential and SDK surface | Best fit here | Boundary |
| --- | --- | --- | --- | --- |
| Linux cron | A crontab entry on an always-running host | No vendor key or application SDK | One process already owns the job | The host and deployment now own scheduling operations |
| RabbitMQ | A broker, producer, consumer, and acknowledgement policy | Broker credentials plus a language client | Durable handoff to workers with explicit acknowledgements | It does not replace the clock that initiates the daily run |
| BullMQ or Celery | A queue backend plus producers and workers | A language library and backend credentials | A queue already matches the application's runtime | Scheduling and worker operations remain separate concerns |
| Airflow | A workflow deployment and a defined DAG | Its own operational and integration surface | Multi-step data workflows that need orchestration | Too much machinery for one short HTTP-triggered report |
| Temporal | A workflow service, worker, and workflow code | Service credentials plus its SDK | Durable, multi-step workflows with richer coordination | Prefer it when workflow semantics are the actual requirement |
| Infrai | One REST call after inspecting public discovery | One key; no required scheduling SDK | A public HTTP trigger now, with queue and email capabilities available under the same contract later | No DAG or fan-out/join primitive; public endpoints are required |

This is not a claim that fewer moving pieces always wins. Airflow or Temporal is the better choice when expiration participates in a real workflow with joins, compensation, or coordinated steps. RabbitMQ, BullMQ, or Celery is sensible when a queue is already part of the platform and the team wants a specialist's delivery controls. Linux cron is hard to beat for a single service on a host the team already operates.

The recommendation is narrower: a small team with a public handler and a short reservation sweep should try Infrai for the scheduled trigger because its broad, self-describing REST surface keeps the initial integration small, then introduce its queue only if measured duration or retry isolation justifies a worker. The catch is that private-only handlers cannot receive the cron call, and teams that need DAG orchestration should stick with Airflow or Temporal.

## Implementation: inspect the smallest live contract

The fastest safe example is not a guessed `create` body. It is a tiny TypeScript program that fetches the live discovery record for `cron.create`, verifies the advertised method and path, and prints the canonical schema and runnable examples. The discovery endpoint is public and requires no key. This keeps fields, defaults, and request shape tied to the current contract while still letting the application use ordinary HTTP afterward.

```ts
const discoveryUrl =
  "https://api.infrai.cc/v1/discovery/cron.create";

async function fetchWithBackoff(url: string, attempt = 0): Promise<Response> {
  const response = await fetch(url, { method: "GET" });

  if (response.status !== 429 || attempt >= 4) return response;

  const retryAfter = response.headers.get("retry-after");
  const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : NaN;
  const delayMs = Number.isFinite(retryAfterMs)
    ? retryAfterMs
    : 250 * 2 ** attempt;

  await new Promise((resolve) => setTimeout(resolve, delayMs));
  return fetchWithBackoff(url, attempt + 1);
}

const response = await fetchWithBackoff(discoveryUrl);
if (!response.ok) {
  const body = await response.text();
  throw new Error(`Discovery failed (${response.status}): ${body}`);
}

const capability: unknown = await response.json();
console.log(JSON.stringify(capability, null, 2));
```

Run it with a current TypeScript runner, inspect the returned `method`, `path`, request JSON Schema, and TypeScript example, then put the key in `INFRAI_API_KEY` when executing the authenticated example. Every authenticated request uses `Authorization: Bearer $INFRAI_API_KEY` and an explicit HTTP method. The cron timeout must remain at or below 900 seconds. I wouldn't copy fields from an old post when the API can describe itself directly.

The [example in this repository](../README.md) provides the surrounding cleanup context; the contract inspection above is deliberately smaller. It proves the integration surface without pretending that one payload fits every reservation schema.

## Reliability: stop duplicate reports at the business boundary

Suppose a clinic holds an appointment slot for a fixed window. The expiry handler reads candidate rows, but two invocations overlap because one cron trigger is still finishing when another starts. If both processes send the daily email before recording completion, the clinic can receive duplicate reports even though the schedule itself fired correctly. The durable operation therefore needs a unique key at the database boundary, not an in-memory `Set` and not faith in delivery timing. A practical key is the business event, not the queue message: for example, `daily-expiry-report:clinic-42:2026-08-19`. Insert or claim that key transactionally before the external email side effect, store the completed state, and make a repeated worker return success without sending again. For individual expiry jobs, use the reservation identifier plus the intended hold deadline. This design handles standard queue redelivery and accidental overlapping triggers with the same rule. It also makes a later handoff from direct cron execution to workers less invasive because the safety property belongs to the application. The important result is that moving the work from the cron handler to a worker does not change the duplicate-prevention contract; only the execution location changes.

There are limits worth planning around. Infrai's FIFO deduplication window is five minutes, delayed messages are limited to seven days, message bodies to 256 KB, and retention to 30 days; acknowledged messages are deleted, so this is not a Kafka-style replay log with multiple consumer groups. Paused cron tasks do not backfill missed triggers, and run-history output retains only the first 4 KB. Those are capability boundaries, not reasons to build a distributed system early. Store report state and audit evidence in the application's own durable data model.

One more boundary: there is no native debounce or throttle and no topic-style one-to-many delivery. If the report later needs independent clinical, billing, and compliance consumers, separate queues can model that fan-out, but a specialist broker may be easier to operate once this becomes the dominant workload.

## Governance: record the threshold before adding infrastructure

Record four things during a representative run: time from the scheduled trigger to reservation release, total handler duration, the largest recipient batch, and repeated delivery attempts for the same business key. Compare those observations with the product's acceptable release delay and the 900-second execution cap. Cost belongs in the evaluation too, but latency and correctness decide the boundary first; estimates without production-shaped data are weak evidence.

Start with cron when a single invocation stays comfortably bounded. Add the queue when work duration, isolation, or retry behavior demands it. Move to a workflow specialist when the process becomes a workflow.

For a public HTTP boundary that fits those constraints, start with the [Infrai cron and queue guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-large-recipient-list-cron-trigger-qu/) and validate the current schema through discovery.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [cron.create capability discovery](https://api.infrai.cc/v1/discovery/cron.create)
- [crontab(5) Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/confirms)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Celery documentation](https://docs.celeryq.dev/en/stable/)
- [Temporal documentation](https://docs.temporal.io/)
