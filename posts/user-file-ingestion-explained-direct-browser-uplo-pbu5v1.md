# User File Ingestion Explained: Direct Browser Upload to Object Storage or Server Relay

Short answer: for marketplace training documents, use a presigned browser upload when the application can verify the object before publishing it; use a Node.js relay when policy requires inspection before the object reaches durable storage. In both designs, keep retention authority in application metadata and treat storage lifecycle rules as an independently testable deletion backstop.

This is a governance decision disguised as a plumbing decision. Sending bytes directly from React to S3-compatible object storage can simplify delivery, while sending them through a server gives that server a chance to inspect the stream. Neither path, by itself, proves who authorized a document, which retention class applied, or whether deletion happened on schedule. Those facts need durable records.

For a small team, the least complex design is the one with one explicit policy owner and two boring data paths.

## What should a React browser, Node.js server, and object storage each control?

The browser should select the document, report its proposed size and media type, transfer bytes, and display progress. It should not choose a trusted tenant prefix, a retention deadline, or the final authorization state. Browser fields are claims, not evidence.

The Node.js service should authenticate the marketplace operator, select a server-generated object key, assign the retention class, and create an upload record before any transfer begins. With direct upload, it signs one narrowly scoped request and later verifies stored metadata. With a relay, it applies the same policy while streaming the request onward. The object store holds bytes and applies lifecycle configuration; the application database explains why those bytes exist.

That separation matters when a seller replaces a training guide halfway through a review cycle. Suppose the database contains revision `rev_7`, tenant `market_42`, state `authorized`, an expected byte length, and retention class `review-30d`. A direct transfer can finish while the tab closes before the completion call. The object then exists, but it isn't eligible for model training because the application has not verified and promoted it. A reconciler can inspect stale authorization records, compare the stored object with the expected metadata, and either finalize the matching revision or remove an unclaimed object. It must not infer policy from the original filename or reset the retention clock merely because a client retried. This longer path is deliberate: it turns an ambiguous half-finished interaction into evidence that an operator can audit.

Keep access private.

## Build the policy record before moving bytes

The following TypeScript focuses on the contract rather than a vendor SDK. The adapter exposes signing and metadata operations; the application owns authorization, state transitions, and deletion intent. Policy values are inputs because a useful upload expiry or retention duration cannot be derived from an architecture diagram. I'm not sure any fixed expiry works for every marketplace: observed mobile transfer times and retry rates should settle it.

```ts
type RetentionClass = "review-30d" | "contractual";
type UploadState = "authorized" | "ready" | "rejected";

type UploadIntent = {
  displayName: string;
  contentType: "application/pdf" | "text/plain";
  byteLength: number;
  retentionClass: RetentionClass;
};

type Actor = { tenantId: string; userId: string };

interface StorageAdapter {
  signPut(input: {
    key: string;
    contentType: string;
    expiresInSeconds: number;
  }): Promise<{ url: string; headers: Record<string, string> }>;
  head(key: string): Promise<{
    byteLength: number;
    contentType: string;
    entityTag?: string;
  }>;
}

async function authorizeArtifact(
  actor: Actor,
  intent: UploadIntent,
  policy: { maxBytes: number; uploadTtlSeconds: number },
  storage: StorageAdapter,
) {
  if (intent.byteLength < 1 || intent.byteLength > policy.maxBytes) {
    throw new Response("Document exceeds the application policy", { status: 413 });
  }

  const artifactId = crypto.randomUUID();
  const key = `tenants/${actor.tenantId}/training/${artifactId}`;

  await artifacts.insert({
    artifactId,
    tenantId: actor.tenantId,
    authorizedBy: actor.userId,
    displayName: intent.displayName,
    contentType: intent.contentType,
    expectedBytes: intent.byteLength,
    retentionClass: intent.retentionClass,
    key,
    state: "authorized" as UploadState,
  });

  const signed = await storage.signPut({
    key,
    contentType: intent.contentType,
    expiresInSeconds: policy.uploadTtlSeconds,
  });

  return { artifactId, uploadUrl: signed.url, uploadHeaders: signed.headers };
}

async function transferFromBrowser(
  file: File,
  authorization: {
    artifactId: string;
    uploadUrl: string;
    uploadHeaders: Record<string, string>;
  },
) {
  const uploaded = await fetch(authorization.uploadUrl, {
    method: "PUT",
    headers: authorization.uploadHeaders,
    body: file,
  });

  if (!uploaded.ok) {
    throw new Error(`Transfer failed with status ${uploaded.status}`);
  }

  return fetch(`/api/training-artifacts/${authorization.artifactId}/verify`, {
    method: "POST",
    credentials: "same-origin",
  });
}

async function verifyArtifact(actor: Actor, artifactId: string, storage: StorageAdapter) {
  const artifact = await artifacts.findAuthorized(artifactId, actor.tenantId);
  if (!artifact) {
    throw new Response("Artifact is not authorized", { status: 409 });
  }

  const stored = await storage.head(artifact.key);
  if (
    stored.byteLength !== artifact.expectedBytes ||
    stored.contentType !== artifact.contentType
  ) {
    throw new Response("Stored metadata differs from the authorization", {
      status: 422,
    });
  }

  await artifacts.markReady(artifactId, stored.entityTag);
}
```

The endpoint names in this example belong to the application, not to a storage provider. `signPut` and `head` are adapter contracts rather than claimed public routes. The browser must send the returned header map unchanged, because a signed request can be rejected with `403` when a signed header, method, or key no longer matches.

Don't log the upload URL. Possession of a presigned URL is temporary authority, so analytics events, error breadcrumbs, and support transcripts are poor places for it.

## Make CORS a deployment contract

CORS answers whether browser JavaScript from an origin may issue and observe a cross-origin request. It does not authorize an object operation. That distinction explains a common diagnostic split: the same signed request can work outside a browser while browser code is stopped by a failed preflight.

For direct upload, configure the bucket to allow the exact production origin, the upload method, and the request headers covered by the signature. Add development origins explicitly. If the client must read a response header such as an entity tag, expose only that header. A browser may send `OPTIONS` before the actual upload, so test preflight and transfer as separate phases. CORS configuration models are not standardized across storage implementations. Test the deployed policy instead of inferring its syntax from an API-compatibility label.

Deployment tests should cover an allowed origin, a disallowed origin, an expired signature, a changed media type, a document one byte beyond the application limit, and a repeated verification call. Capture the phase and status code, but redact query strings. “Network error” isn't enough to tell a policy denial from an interrupted transfer.

## Compare evidence, not upload convenience

Presigned upload and server relay create different evidence at different times. The useful comparison is where the organization can enforce a rule, not which diagram has fewer arrows.

| Policy question | Presigned browser upload | Node.js relay |
|---|---|---|
| Who authorizes the write? | Application signs a bounded request | Application accepts the request and writes onward |
| Can content be inspected before durable storage? | No; inspect after upload and before publication | Yes; inspect the incoming stream before the final write |
| Does application bandwidth carry the document? | No | Yes |
| Is storage CORS required? | Yes, for the browser-to-storage request | No, when the browser calls its same application origin |
| What must recovery reconcile? | Authorization row, stored object, and verification state | Incoming request, upstream write, and application state |

Direct upload fits ordinary documents when post-upload verification is acceptable and application bandwidth should stay out of the data plane. The catch is the pre-storage boundary. It is not suitable when malware scanning, data-loss prevention, transformation, or rejection must happen before durable storage. Use a relay or isolated ingestion service then. A relay is also the practical choice for clients that cannot satisfy the required cross-origin request.

A relay has its own limit: the application now carries every byte. Stream with backpressure instead of buffering whole files, enforce body limits consistently at the edge and in Node.js, and abort the upstream write when the client disconnects. A documented `413` is actionable; two hidden limits at different hops are not.

Provider choice does not remove these design duties. Amazon S3 documents presigned URLs and browser POST uploads. Cloudflare R2 documents an S3-compatible API, presigned URLs, CORS, and a compatibility matrix. Google Cloud Storage documents V4 signed URLs for XML API endpoints and its own CORS configuration. These are three implementations to test against the same adapter contract, not a ranking. Multipart behavior, checksum support, signing details, and lifecycle semantics should be verified in primary documentation before migration or deployment.

## Prove that retention is reproducible

A retention policy is reproducible when the same inputs lead to the same deletion intent and an auditor can reconstruct the decision. Record the policy identifier or version, authorization time, responsible tenant, object key, state, and the date or rule that determines deletion. Keep the original filename as escaped display metadata; it is not an access-control boundary.

The database and object store do not share a transaction. An object may be written before the application records verification, and application metadata may be removed before a lifecycle engine evaluates the object. Plan for that gap. A scheduled reconciler should examine stale authorization records, confirm whether corresponding objects exist, promote only exact metadata matches, and remove or quarantine objects that have no live authorization. Separately, compare due deletion records with storage inventory or metadata so that a delayed worker is visible. Lifecycle configuration provides a backstop, while the database supplies business context and audit history.

Use separate prefixes or buckets when retention classes map cleanly to lifecycle scope. Do not let a client choose that scope, and do not encode mutable policy only in a filename. Changes to policy need a migration plan: identify affected objects, preserve the old decision for audit, apply the new rule deliberately, and verify the resulting storage configuration.

Operationally, ship this with phase-specific metrics for authorization, preflight, transfer, verification, publication, reconciliation, and deletion. Alert on old `authorized` records and objects without live metadata. Exercise interrupted transfers and repeated verification during deployment, then run a deletion canary using a non-production artifact whose expected lifecycle is known. Review redaction so signed query strings never enter logs. That's the checklist I care about because it tests the policy, rather than merely proving that a happy-path upload reaches a bucket.

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html
- https://developers.cloudflare.com/r2/
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://cloud.google.com/storage/docs/access-control/signed-urls
- https://cloud.google.com/storage/docs/cross-origin
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
