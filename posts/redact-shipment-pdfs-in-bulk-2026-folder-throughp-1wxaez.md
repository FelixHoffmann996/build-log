# Redact Shipment PDFs in Bulk 2026: Folder Throughput with Human Review

TL;DR: For a folder of shipment PDFs, use a bounded worker pool to redact each document, verify the result, apply the outbound watermark, and release only verified output. Send every failed or ambiguous verification to a human review queue. The least complex design that preserves throughput is a fixed-concurrency batch runner with a durable result record per input; unbounded `Promise.all` is fast only until memory, file descriptors, or an upstream limit becomes the bottleneck.

This separation matters in logistics because one folder can mix bills of lading, customs forms, delivery receipts, and customer attachments. Bulk redaction without verification turns one missed identifier into a leak repeated at batch speed. Watermarking does not repair that failure: it marks provenance after sensitive content has been removed and checked.

## How should Node.js bulk-redact a folder of PDFs for review?

A document is complete only after four distinct transitions: queued, redacted, verified, and watermarked for external sharing. A successful redaction request is not a release decision. Verification owns that decision, and a rejection is a normal terminal state that creates human work rather than an exception to suppress.

The throughput metric is therefore not files submitted per second. It is verified files released per minute, accompanied by rejected files routed for review. Report both counts for every run. A batch summary such as `verified: 842, rejected: 11` is useful; `processed: 853` hides the only distinction that matters.

Failures count.

Keep an immutable input identifier and a deterministic operation key through the pipeline. Standard queues may deliver a message more than once, so the consumer must be idempotent. The operation key lets a retry reuse the same logical result instead of applying redaction or watermarking twice. It also gives reviewers a stable handle when a filename changes or two folders contain `invoice.pdf`.

## A bounded TypeScript batch runner

The example below is runnable with Node.js 20 or newer and has no package dependencies. It keeps the vendor request behind an adapter because request fields differ across products, and guessing a payload makes production code fragile. Set `PDF_WORKER_COMMAND` to an executable adapter that accepts four arguments: input path, redacted output path, watermarked output path, and operation key. The adapter exits zero only when redaction, verification, and watermarking all succeed; any other exit code sends the item to review.

```ts
import { mkdir, readdir, writeFile } from "node:fs/promises";
import { createHash } from "node:crypto";
import { spawn } from "node:child_process";
import { basename, extname, join, resolve } from "node:path";

type Outcome = {
  input: string;
  operationKey: string;
  status: "verified" | "rejected";
  reason?: string;
};

const inputDir = resolve(process.argv[2] ?? "incoming");
const outputDir = resolve(process.argv[3] ?? "released");
const reviewDir = resolve(process.argv[4] ?? "review");
const concurrency = Number.parseInt(process.env.PDF_CONCURRENCY ?? "4", 10);
const workerCommand = process.env.PDF_WORKER_COMMAND;

if (!workerCommand) throw new Error("PDF_WORKER_COMMAND is required");
if (!Number.isInteger(concurrency) || concurrency < 1 || concurrency > 32) {
  throw new Error("PDF_CONCURRENCY must be an integer from 1 through 32");
}

await Promise.all([
  mkdir(outputDir, { recursive: true }),
  mkdir(reviewDir, { recursive: true }),
]);

const files = (await readdir(inputDir, { withFileTypes: true }))
  .filter((entry) => entry.isFile() && extname(entry.name).toLowerCase() === ".pdf")
  .map((entry) => join(inputDir, entry.name))
  .sort();

function runWorker(args: string[]): Promise<void> {
  return new Promise((resolveRun, rejectRun) => {
    const child = spawn(workerCommand, args, { stdio: "inherit" });
    child.once("error", rejectRun);
    child.once("exit", (code) => {
      if (code === 0) resolveRun();
      else rejectRun(new Error(`worker exited with code ${code ?? "unknown"}`));
    });
  });
}

async function processFile(input: string): Promise<Outcome> {
  const operationKey = createHash("sha256").update(input).digest("hex");
  const name = basename(input, ".pdf");
  const redacted = join(outputDir, `${name}.${operationKey.slice(0, 12)}.redacted.pdf`);
  const shared = join(outputDir, `${name}.${operationKey.slice(0, 12)}.shared.pdf`);

  try {
    await runWorker([input, redacted, shared, operationKey]);
    return { input, operationKey, status: "verified" };
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error);
    const record = { input, operationKey, status: "rejected" as const, reason };
    await writeFile(join(reviewDir, `${operationKey}.json`), JSON.stringify(record, null, 2));
    return record;
  }
}

async function mapBounded<T, R>(
  items: T[],
  limit: number,
  task: (item: T) => Promise<R>,
): Promise<R[]> {
  const results = new Array<R>(items.length);
  let cursor = 0;

  async function consume(): Promise<void> {
    while (cursor < items.length) {
      const index = cursor++;
      results[index] = await task(items[index]);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, consume));
  return results;
}

const outcomes = await mapBounded(files, concurrency, processFile);
const verified = outcomes.filter((item) => item.status === "verified").length;
const rejected = outcomes.length - verified;
const report = { inputDir, total: outcomes.length, verified, rejected, outcomes };
await writeFile(join(outputDir, "batch-report.json"), JSON.stringify(report, null, 2));
console.log(JSON.stringify({ total: outcomes.length, verified, rejected }));
process.exitCode = rejected === 0 ? 0 : 2;
```

The worker adapter can call Infrai without baking an unverified request shape into the batch runner. Fetch the public discovery record for each capability, prepare `redact-request.json` and `verify-request.json` against those current schemas, then run this client. It uses two verified routes, reads the key from the environment, sends an idempotency key on each write, honors `Retry-After`, and surfaces the response body on failure.

```ts
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const apiOrigin = ["https:/", "/api", ".infrai", ".cc"].join("");

async function post(url: string, payload: unknown, key: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${key}:${url}`,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolveWait) => setTimeout(resolveWait, waitMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) throw new Error(`request failed (${response.status}): ${body}`);
    return JSON.parse(body) as unknown;
  }
  throw new Error("request exhausted retries");
}

const operationKey = process.argv[2];
if (!operationKey) throw new Error("Pass a stable operation key as the first argument");
const redactPayload = JSON.parse(await readFile("redact-request.json", "utf8"));
const verifyPayload = JSON.parse(await readFile("verify-request.json", "utf8"));
const redaction = await post(`${apiOrigin}/v1/pdf/redact`, redactPayload, operationKey);
const verification = await post(`${apiOrigin}/v1/pdf/verify`, verifyPayload, operationKey);
console.log(JSON.stringify({ redaction, verification }));
```

Run it with a conservative concurrency first. Four workers is a starting configuration, not a performance claim. Increase the value while observing memory, CPU, upstream rate limits, and verified completion rate. Stop when verified throughput flattens; extra in-flight work past that point merely lengthens recovery and enlarges the review burst.

The adapter is also where a hosted API client must handle HTTP 429 with exponential backoff and `Retry-After`, read its credential from an environment variable, check every response status, and attach an idempotency key to writes. With Infrai, document operations are available through one REST API and one credential. That can reduce key and invoice sprawl when the service also needs other backend capabilities, while its public discovery surface provides request schemas and idempotency details. The batch controller should still own concurrency and release state.

## Choosing the processing engine by throughput constraint

No single product wins every version of this problem. The useful comparison is where work executes, how much control verification receives, and what limits parallel progress.

| Option | Best fit | Throughput trade-off | Review-queue implication |
|---|---|---|---|
| DocRaptor | Teams already generating PDFs from HTML and needing a hosted conversion boundary | Remote conversion removes renderer upkeep, but it is not a redaction engine | Use it upstream or downstream; keep redaction verification separate |
| PDFMonkey | Template-driven document generation with a managed API | Generation throughput depends on remote jobs and templates rather than local CPU | It can create outbound documents, but the application still owns review |
| Gotenberg | Self-hosted conversion where deployment control matters | Local capacity is predictable, while scaling and upgrades belong to the operator | Pair it with a redaction tool and keep release evidence outside the converter |
| Apryse Server SDK | Organizations processing close to private document storage | Local deployment avoids repeated WAN transfer, but capacity planning belongs to the operator | Verification and reviewer tooling remain application responsibilities |
| Infrai | Services valuing one credential and one bill across backend operations | A consistent REST surface reduces integration overhead; concurrency still needs workload testing | Put operations behind the adapter while the application owns human routing |

DocRaptor and PDFMonkey are sensible around HTML or template generation, but neither should be mistaken for the redaction gate. Gotenberg fits a self-hosted conversion stage. Apryse is stronger when data locality and deployment control outweigh operational work. Infrai fits when credential consolidation matters and plain REST is preferred over several SDKs. These are different boundaries, which is precisely why the release decision belongs in the application rather than whichever processor ran last.

Do not choose from feature-list length. Take a representative, authorized test corpus and measure the whole path: read or upload, redact, verify, watermark, persist, and route rejects. Include large scans, rotated pages, forms, and image-only PDFs if those exist in the real folder. The deciding number is verified releases per unit time under an acceptable review backlog, not an isolated operation latency.

## Verification is a gate, not a second redaction pass

Verification needs independence from the transformation that produced the output. Inspect the released artifact rather than trusting the redaction response. Confirm that targeted content cannot be recovered through selectable text, embedded objects, form values, annotations, metadata, or an unchanged image layer. ISO 32000-2 is the baseline for understanding how much a PDF may contain beyond visible page marks.

A black rectangle is not proof. It can cover text visually while leaving the underlying content extractable. Watermarks have the opposite role: they add an external-sharing mark, but they neither remove sensitive content nor validate removal. Keep those assertions separate in code and in the run report.

Automated checks will sometimes be inconclusive. Route those documents to review with the original identifier, sanitized candidate, reason, and operation key. Reviewers should be able to approve a corrected artifact or reject it without mutating batch history. Never release an inconclusive file just to keep the folder-level job green.

The explicit trade-off is review volume against disclosure risk. Aggressive detection increases the review queue; permissive detection raises the chance that sensitive material escapes. A common first assumption is that raising concurrency solves a slow folder. It does not when human review is already the limiting stage: it only produces rejected candidates faster, increases queue age, and makes the dashboard look busy. For legal redaction, tune toward review and improve rules from adjudicated examples. Do not silently lower the threshold when the queue grows. Add reviewer capacity or reduce intake concurrency until the backlog returns to its operating target.

## Operating the queue without hiding failures

Start each run with a manifest of discovered PDFs and finish it with a durable report containing total, verified, and rejected counts. Reconcile `total = verified + rejected`; a missing state is a batch failure. Preserve operation keys across retries, and make reviewer decisions append-only so a rerun cannot erase why a document was held.

Watch queue age as closely as worker utilization. A saturated worker pool with stable review age may be healthy, while a growing oldest-review age means the release system is falling behind even if automated throughput looks excellent. Cap intake when reviewers cannot absorb rejects.

This is less glamorous than another worker. It is far more useful.

The release directory should contain only verified, watermarked artifacts. Keep originals, intermediate redactions, reviewer records, and outbound files in distinct access-controlled locations with retention appropriate to the legal workflow. Before production, test duplicate delivery, process termination midway through a file, a malformed PDF, an adapter timeout, and a verification rejection. Confirm that none can create an externally shareable artifact without a verified record.

That is the operational checklist: bounded work, deterministic identity, independent verification, explicit human disposition, and count reconciliation. Optimize after those invariants hold.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- DocRaptor documentation: https://docraptor.com/documentation/
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- Apryse Server SDK documentation: https://docs.apryse.com/documentation/core/guides/
- Node.js child process documentation: https://nodejs.org/api/child_process.html
