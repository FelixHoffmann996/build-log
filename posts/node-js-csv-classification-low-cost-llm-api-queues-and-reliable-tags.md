# Node.js CSV Classification: Low-Cost LLM API Queues and Reliable Tags

Bottom line: for a cheap LLM text classification API workflow in Node.js, I put CSV rows through a bounded queue, send small multi-row requests, validate every returned label, and checkpoint results as they finish. A provider-managed asynchronous batch can be attractive for a large, non-urgent file, but it isn't my default because the portability and recovery story matters more to me than shaving one more line from the bill.

**The unit of reliability is a row, not a request.** I want each row to have a stable ID, a constrained label set, and an independent retry record. That design lets me change the model or API without rebuilding the whole job, and it keeps one malformed response from poisoning an afternoon's run.

My practical rule is plain: start with an application-side queue when the file is small enough to finish during a normal deployment window; consider an asynchronous batch job when turnaround can be measured in hours and the input is already frozen. Either way, measure accepted rows, rejected rows, tokens, latency, and label drift on a hand-checked sample before calling the pipeline done.

## How should a Node.js LLM API batch job classify and tag bulk CSV rows?

The data flow is CSV file to parser, parser to normalized row, row to durable work item, work item to classifier, and validated label to an append-only result file. I keep the original text beside its stable ID until the final join. The API sees only the fields needed for classification — less payload means fewer tokens and less accidental data exposure — while the local job owns ordering, retries, and reconciliation.

There are three reasonable execution shapes. None wins everywhere.

| Shape | Best fit | Main cost control | Catch |
| --- | --- | --- | --- |
| One row per request | Tiny runs and debugging | Short prompts | Request overhead and rate limits dominate sooner |
| Several rows per request | Routine bulk tagging | Shared instructions across rows | A response must be mapped and validated carefully |
| Provider-managed async batch | Large, frozen inputs with loose deadlines | Batch-oriented billing or scheduling, where offered | Provider format and lifecycle increase switching work |

I usually choose the middle row. Ten to fifty items per request is a tuning range, not a universal law; the right number depends on text length, output limits, and the failure boundary I can tolerate. Your mileage may vary. I cap concurrency separately from request size because they solve different problems: grouping reduces repeated prompt overhead, while concurrency controls pressure on the remote API.

The label contract stays boring: one ID and one label from an allowlist. No prose. I also version the taxonomy in the output. If `billing_issue` becomes `billing` next month, I need to know which rows were produced under each definition rather than silently mixing them.

## A runnable TypeScript queue before the trade-offs

This example expects `id,text` columns, a `CLASSIFIER_URL`, and an `API_KEY`. The endpoint is intentionally supplied through configuration, so the queue isn't coupled to a vendor route. It sends groups of 20 rows with concurrency 3, retries throttled requests, validates the response, and writes newline-delimited JSON checkpoints. Install `csv-parse` and run it with the TypeScript runner already used by your project.

```ts
import { createReadStream } from "node:fs";
import { appendFile } from "node:fs/promises";
import { parse } from "csv-parse";

type Row = { id: string; text: string };
type Tagged = { id: string; label: string };

const labels = new Set(["billing", "bug", "feature", "other"]);
const endpoint = process.env.CLASSIFIER_URL;
const apiKey = process.env.API_KEY;

if (!endpoint || !apiKey) throw new Error("Set CLASSIFIER_URL and API_KEY");

async function classify(rows: Row[], attempt = 0): Promise<Tagged[]> {
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      labels: [...labels],
      items: rows,
      output: { format: "json", fields: ["id", "label"] },
    }),
  });

  if (response.status === 429 && attempt < 4) {
    await new Promise((resolve) => setTimeout(resolve, 2 ** attempt * 1_000));
    return classify(rows, attempt + 1);
  }
  if (!response.ok) throw new Error(`Classification failed: ${response.status}`);

  const body = (await response.json()) as { items?: Tagged[] };
  const items = body.items ?? [];
  const expected = new Set(rows.map((row) => row.id));
  if (items.length !== rows.length) throw new Error("Result count mismatch");
  for (const item of items) {
    if (!expected.has(item.id) || !labels.has(item.label)) {
      throw new Error(`Invalid classification for ${item.id}`);
    }
  }
  return items;
}

async function runPool<T>(jobs: Array<() => Promise<T>>, width: number): Promise<T[]> {
  const results: T[] = [];
  let cursor = 0;
  async function worker() {
    while (cursor < jobs.length) {
      const index = cursor++;
      results[index] = await jobs[index]();
    }
  }
  await Promise.all(Array.from({ length: width }, () => worker()));
  return results;
}

const rows: Row[] = [];
const parser = createReadStream("input.csv").pipe(
  parse({ columns: true, skip_empty_lines: true, trim: true })
);
for await (const raw of parser) {
  if (raw.id && raw.text) rows.push({ id: String(raw.id), text: String(raw.text) });
}

const chunks: Row[][] = [];
for (let i = 0; i < rows.length; i += 20) chunks.push(rows.slice(i, i + 20));
await runPool(
  chunks.map((chunk) => async () => {
    const tagged = await classify(chunk);
    await appendFile("tagged.ndjson", tagged.map((item) => JSON.stringify(item)).join("\n") + "\n");
  }),
  3
);
```

It's deliberately a narrow adapter. In production I add an idempotency key if the chosen API supports one, but I don't pretend that header is portable. The NDJSON output is the checkpoint: on restart, I load completed IDs into a set and omit them when building `chunks`.

## What makes cheap classification expensive in practice?

Tokens are visible; rework isn't. The first cost leak I look for is an oversized prompt repeated for every row. A compact taxonomy with crisp boundaries usually does more for spend and consistency than chasing a nominally cheaper model. I record input size and output size per chunk, then compare cost per accepted row rather than cost per request. Rejected, duplicated, or unmappable results count against the denominator.

Retries lie.

I hit 317 consecutive `401` responses on a support export because of one config footgun. I had copied an auth value with the literal `Bearer ` prefix into the environment variable, while the code added the same prefix again. The generic “check credentials” message sent me toward the wrong account before I printed the header shape and saw the duplicated scheme. I checked permissions, regenerated the secret, and reran the same doomed request before looking at what my process had actually assembled. Now the secret stores only the token, configuration validation runs before the CSV is opened, and logs show the auth scheme but never the credential. Small footgun, wasted morning. Quality has its own bill too. I keep a fixed evaluation slice with examples near each category boundary and score it whenever the prompt, taxonomy, or model changes. I'm not sure why teams so often sample only obvious rows; those rows produce comforting numbers and miss the disagreements that matter. For multilabel work, I score each label independently and inspect false positives because a plausible extra tag can pass casual review.

Then there is latency. A queue with width 3 is predictable but may be too slow; width 30 may amplify throttling and turn retries into a synchronized wave. I add jitter to retry delays in a real worker and lower concurrency when the recent throttle rate crosses an operational threshold chosen from my own measurements. No magic value.

Privacy can outweigh every token calculation. Don't send raw customer fields that the classifier doesn't need. Redact or replace identifiers before enqueueing, define a retention window for inputs and outputs, and make the destination region part of configuration review. Cheap is irrelevant if a data-handling requirement rules out the service.

## When should the architecture change?

The application-side queue is not suitable when a dataset is huge, immutable, and allowed to finish well after the initiating process exits; in that case, a provider-managed asynchronous batch or a durable self-hosted worker can be the cleaner choice. Stick with one-row calls when you're still debugging a taxonomy or when each item needs a different instruction. Choose a conventional non-LLM classifier when labels are stable, training data is adequate, and deterministic local inference matters more than open-ended language understanding.

The catch with multi-row prompts is correlated failure. One truncated or structurally invalid response can force a retry for several otherwise good items. Smaller chunks reduce that blast radius but repeat more instructions. I tune against real text-length percentiles — not an average that hides the long tail — and place exceptionally large rows into their own requests.

Deployment is where I try to stay unromantic. The worker gets a concurrency limit, a request timeout, bounded retries, and a dead-letter file containing IDs plus sanitized failure metadata. Logs carry job ID, chunk ID, attempt, duration, response status, and counts, but never full source text. A dashboard needs throughput, throttle rate, validation failures, estimated input/output usage, and queue age. Alerts should describe action: reduce concurrency, inspect taxonomy drift, or replay a named dead-letter set.

Ship the canary.

Before I ship, I dry-run the parser without network calls, confirm IDs are unique, check that every label has boundary examples, and process a tiny canary file end to end. Then I compare the output row count with the accepted input count, manually review a stratified sample, and archive the prompt and taxonomy version beside the result. I also rehearse restart behavior by stopping the worker mid-file. This is dull work — exactly the kind I want around an API bill and a customer dataset.

## References

- Cohere, “Rerank overview”: https://docs.cohere.com/docs/rerank-overview
- OpenAI, “Whisper”: https://github.com/openai/whisper
