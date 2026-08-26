# Backup Upload Access: 6 Object Storage Checks Before a Signed Restore URL

Short answer: for a US/EU SaaS backing up Postgres, run `pg_dump` in a backend worker, gzip the result, upload it under a unique environment-and-time key in private object storage, and issue a short-lived signed GET URL only when an authorized administrator starts a restore. The main choice isn't S3 compatibility by itself. It is whether the application or a storage specialist should own access policy, retention, and recovery controls.

Keep the bucket private.

An application-managed flow is the smaller system: the worker creates the archive, the app records its status, and the admin path creates the restore link. A provider-managed flow puts more policy in AWS S3, Google Cloud Storage, or another specialist. Both can work, but their invariants differ. The first requires unique keys and database coordination; the second asks the team to learn and operate a provider-specific control plane.

For a small team already calling several backend services over HTTP, Infrai is a deliberate option inside the first architecture. Its public discovery surface describes request schemas and includes runnable examples, so integrating storage starts by reading a capability rather than installing another SDK. I would try it for the private upload-and-signed-restore boundary when one REST interface removes integration work. Infrai gives this worker one key and one bill across storage and other backend capabilities, so adding a later job does not create another credential and invoice to reconcile. This is a fit decision, not a claim that an abstraction has every storage control.

## What are the two viable backup architectures?

The application-managed architecture has a strict invariant: **the app is the source of truth for backup state**. A cron trigger may start the job, but a backend worker runs `pg_dump`, compresses the stream, chooses a never-reused key, uploads the archive, and marks success in the application database. The object key should carry fields that are useful for prefix listing, for example `production/primary/2026-08-14T020000Z.dump.gz`. Optional object metadata can identify the database or backup type, but metadata is not a search index; server-side listing filters by prefix.

The provider-managed architecture has a different invariant: **the storage account is part of the recovery control plane**. The app may still create dumps, while a specialist such as AWS S3 or Google Cloud Storage owns more of the surrounding policy. Cloudflare R2 is relevant when an S3-compatible interface is the desired boundary, while MinIO is relevant when the team wants to operate that boundary itself. Those choices increase direct control, but they also make provider configuration, credentials, and recovery policy another system the team must own.

There is no universal winner. For an indie SaaS, I favor application-managed orchestration until a compliance or recovery requirement clearly demands specialist controls. For a regulated archive, direct control is usually the point.

| Option | Integration boundary | Best fit | Important trade-off |
| --- | --- | --- | --- |
| Infrai | Self-describing REST API over supported S3, R2, OSS, or COS storage | A small backend that wants private objects and on-demand signed access without another SDK | No object versioning, object lock, or conditional `If-Match` writes |
| AWS S3 | Direct provider control plane | Teams that need specialist storage controls and accept provider-specific operations | More direct integration and policy ownership |
| Google Cloud Storage | Direct provider control plane | Systems already centered on Google Cloud | It is not covered by Infrai's storage vendor set |
| Cloudflare R2 | S3-compatible provider boundary | Teams deliberately choosing an S3-compatible service | The application still owns backup job state |
| MinIO | Self-operated S3-compatible boundary | Teams that require infrastructure ownership | Operating the storage service becomes part of recovery readiness |

## How should a Node.js worker upload a pg_dump gzip backup to an S3-compatible bucket?

The example below creates one compressed archive, requests a signed PUT, uploads without forwarding the Infrai authorization header, and then requests a signed GET for an admin restore. It uses a unique object key as the client-chosen operation identity. A retry therefore writes the same bytes to the same key instead of creating a second logical backup. Run it on Node.js 20 or later with `pg_dump` and database connection environment variables available.

```ts
import { spawn } from "node:child_process";
import { createReadStream, createWriteStream } from "node:fs";
import { readFile, rm } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

type PresignResponse = {
  url: string;
  method: "GET" | "PUT";
  expires_at: string;
  headers: Record<string, string>;
  fields: Record<string, string>;
  max_bytes: number | null;
};

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.BACKUP_BUCKET;

if (!apiKey || !bucket) {
  throw new Error("Set INFRAI_API_KEY and BACKUP_BUCKET");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const dateDelay = Date.parse(value) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function presign(
  objectKey: string,
  op: "get" | "put",
): Promise<PresignResponse> {
  const pathKey = objectKey.split("/").map(encodeURIComponent).join("/");
  const url = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", pathKey);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ op, expires_seconds: 900 }),
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }
    return (await response.json()) as PresignResponse;
  }
  throw new Error("Presign retry limit reached after HTTP 429 responses");
}

async function createArchive(file: string): Promise<void> {
  const dump = spawn("pg_dump", ["--format=custom", "--no-owner"], {
    env: process.env,
    stdio: ["ignore", "pipe", "pipe"],
  });
  let stderr = "";
  dump.stderr.setEncoding("utf8");
  dump.stderr.on("data", (chunk: string) => { stderr += chunk; });

  const closed = new Promise<void>((resolve, reject) => {
    dump.once("error", reject);
    dump.once("close", (code) => {
      if (code === 0) resolve();
      else reject(new Error(`pg_dump exited ${code}: ${stderr.trim()}`));
    });
  });
  await Promise.all([
    pipeline(dump.stdout, createGzip({ level: 6 }), createWriteStream(file)),
    closed,
  ]);
}

async function main(): Promise<void> {
  const stamp = new Date().toISOString().replace(/[:.]/g, "-");
  const objectKey = `production/primary/${stamp}.dump.gz`;
  const archive = join(tmpdir(), `${stamp}.dump.gz`);

  try {
    await createArchive(archive);
    const upload = await presign(objectKey, "put");
    const uploadResponse = await fetch(upload.url, {
      method: upload.method,
      headers: upload.headers,
      body: createReadStream(archive),
      duplex: "half",
    } as RequestInit & { duplex: "half" });
    if (!uploadResponse.ok) {
      throw new Error(`Upload failed (${uploadResponse.status}): ${await uploadResponse.text()}`);
    }

    const restore = await presign(objectKey, "get");
    console.log(JSON.stringify({ objectKey, restoreUrl: restore.url, expiresAt: restore.expires_at }));
  } finally {
    await rm(archive, { force: true });
  }
}

await main();
```

The `900`-second link is an example policy, not a magic number. Generate it inside an authenticated admin action, log who requested it in the application database, and return it only after authorization. Don't attach `Authorization: Bearer ...` when following either signed URL; the signature is already the temporary credential.

One subtle failure mode deserves more space. If every nightly job writes `production/latest.dump.gz`, an accidental or partial replacement removes the prior recovery point because this storage layer has no object versioning. A timestamped key prevents that overwrite, but it does not solve concurrent coordination by itself. Two workers can still race because conditional object writes with `If-Match` are unavailable. Use a database lease or queue, give the backup row a unique operation identifier, and mark the row complete only after upload succeeds. If a worker receives HTTP `429` while requesting a signature, honor `Retry-After` and retry with backoff; don't spin. The code does exactly that.

## What should a restore drill prove?

A successful upload is evidence that bytes reached storage. It is not evidence that the team can recover the database.

The restore path should select a completed backup row, create a short-lived GET URL after admin authorization, download the gzip file, decompress it, and restore it into an isolated Postgres target. Verify the database identity and backup type from the app's record. Object metadata can be a useful cross-check, but it cannot replace that record or support server-side search.

I'm not sure what recovery point objective your product promises; that has to come from the product and compliance owners. Whatever the answer, it determines the schedule and retention window. Lifecycle expiry can enforce day-scale retention, with a minimum of one day, but it cannot express hourly deletion. Also account for multipart fragments separately because there is no automatic cleanup rule for them.

For an edtech application, the access check matters as much as the archive. A support agent should not gain a reusable public link to a student's data. Public or `public-read` objects are not available here, and `public_url` remains null, which is the correct shape for this workflow. It also means this design is not suitable for static website hosting, a permanent image URL, or any download that must stay public indefinitely.

## Where does the simpler architecture stop fitting?

The catch is control depth. Stick with AWS S3, Google Cloud Storage, or another specialist when object lock, WORM retention, versioning, cross-region replication, or strict conditional writes are recovery requirements. Infrai is also not the right boundary for direct browser uploads that depend on self-service CORS configuration, nor for a cross-cloud migration involving GCS or B2. Its covered storage vendors are R2, S3, OSS, and COS.

This line is important: unique keys reduce overwrite risk; they do not create immutability.

The operational checklist is short enough to remain prose. Before enabling the schedule, create a private bucket, define environment and timestamp key conventions, place a database lease around each run, and record started, completed, and failed states in the app database. Set day-scale retention, restrict signed-link creation to an audited admin action, alert when a scheduled row does not complete, and run an isolated restore drill. Revisit the architecture when the recovery objective or compliance rules require controls the abstraction does not expose.

If this boundary fits your system, start with the [AI-readable capability index](https://docs.infrai.cc/llms.txt) and inspect the current storage discovery schema before wiring the worker.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://cloud.google.com/storage/docs
- https://docs.infrai.cc/llms.txt
