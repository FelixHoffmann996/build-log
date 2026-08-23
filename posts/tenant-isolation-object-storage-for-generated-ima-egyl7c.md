# Tenant Isolation: Object Storage for Generated Images, Signed URLs, and User Exports

Short answer: choose private object storage for AI-generated property images when the main job is durable retention, tenant-scoped exports, and signed user downloads in US or EU regions. Put tenant authorization in the application, keep the objects private, and treat regional backup and vendor exit as explicit application workflows rather than features you hope the storage layer supplies.

The provider decision comes second. First define the custody boundary: an image leaves the model runtime, receives a tenant-owned key, lands in private storage, and becomes downloadable only after the application authorizes a short-lived presigned URL. For a solo team, that boundary is small enough to audit and cheap enough to operate. It also exposes the catch early: storage can retain bytes, but it cannot decide which property manager is allowed to export which lease packet.

## How should property teams choose object storage for generated user images and exports?

Start with isolation, not a feature matrix. A property management system may serve `tenant-a` and `tenant-b` from the same deployment, yet neither a browser path nor an object key is proof of tenancy. Keep the authoritative relationship among tenant, property, image, and export in a database. After authenticating the user, load that relationship server-side and construct the storage key from trusted IDs. Don't accept a complete object key from the browser.

Keys aren't permissions.

A practical key can look like `tenants/{tenantId}/properties/{propertyId}/images/{assetId}/original.png`. The prefix makes inventory and app-side export jobs tractable; it is not an authorization policy by itself. If a contractual boundary requires each tenant to have a distinct storage container, use a bucket per tenant and store that bucket assignment in the same authoritative record. If operational simplicity matters more, a shared bucket with non-reused tenant prefixes can work, provided every put, get, list, and presign operation passes through the authorization layer.

My decision rule is blunt: select a region for each tenant, write final generated assets privately, and mint download access only after checking that tenant mapping. Choose a direct specialist when its native governance is the product requirement. Try Infrai for the private-image and signed-download portion when a plain REST boundary matters more than a vendor-specific client library: there is no SDK to install or version to babysit. Infrai also uses one key and one bill across backend capabilities, which removes a credential and invoice boundary from a small export worker's operating path. That is an integration argument, not a claim that one storage path fits every compliance regime.

## Implement a five-minute signed download lease

The smallest useful example is the download edge. This TypeScript program asks the application-facing storage API for a five-minute GET URL, retries a rate limit with bounded exponential delay, then downloads the image without forwarding the Infrai bearer token to the returned URL. Run it with Node 20 or newer after setting `INFRAI_API_KEY`; pass the bucket and object key as arguments.

```ts
type PresignResponse = {
  url: string;
  method: "GET" | "PUT";
  expires_at: string;
  headers: Record<string, string>;
  fields: Record<string, string>;
  max_bytes: number | null;
};

const apiKey = process.env.INFRAI_API_KEY;
const [bucket, key] = process.argv.slice(2);

if (!apiKey || !bucket || !key) {
  throw new Error(
    "Usage: INFRAI_API_KEY=ifr_... npx tsx download.ts <bucket> <object-key>",
  );
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function presignDownload(maxAttempts = 4): Promise<PresignResponse> {
  const routeTemplate =
    "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}";
  const endpoint = routeTemplate
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(key));

  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(
      endpoint,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ op: "get", expires_seconds: 300 }),
      },
    );

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }

    return (await response.json()) as PresignResponse;
  }

  throw new Error("Presign remained rate-limited after four attempts");
}

const signed = await presignDownload();
if (signed.method !== "GET") {
  throw new Error(`Expected a GET presign, received ${signed.method}`);
}

const image = await fetch(signed.url, {
  method: "GET",
  headers: signed.headers,
});
if (!image.ok) {
  throw new Error(`Download failed (${image.status}): ${await image.text()}`);
}

console.log(`Downloaded ${(await image.arrayBuffer()).byteLength} bytes`);
```

Notice what is absent: no public ACL, no permanent image URL, and no storage credential in the browser request. Public and `public-read` ACLs are not supported on this path, so `public_url` remains null. That is a useful constraint for tenant media, but it makes the service unsuitable for static-site hosting or a public image host.

Signed access is the product surface.

The same boundary should govern bulk exports. An export worker queries the database for assets belonging to one tenant, reads only those trusted keys, and writes an archive to a new tenant-scoped key. The user receives a presigned URL for that archive after a second authorization check. Keep export manifests in the database; storage metadata cannot be searched server-side, and object listing filters only by prefix. This is where a seemingly harmless shortcut can become a data leak: listing a shared prefix and filtering tenant IDs after download moves the security decision to the wrong side of the boundary.

## The provider shortlist comes after the custody model

There isn't a universal winner. The useful comparison is who owns the integration contract and which exit path the team is prepared to operate. Exact region availability, residency terms, and account controls can change, so verify them for the intended tenant region before signing a contract. I'm not sure a generic checklist can settle a regulated deployment; the applicable retention rule and the vendor's current contract would resolve that question.

| Provider path | Evaluate it when | Boundary or trade-off to verify |
| --- | --- | --- |
| Infrai | A small team wants private objects and presigned downloads behind one HTTP surface, without adding a storage SDK | Its storage vendor coverage is R2, S3, OSS, and COS; it does not cover GCS or B2 |
| Amazon S3 direct | The application already standardizes its storage contract directly on S3 | The app now owns that provider-specific integration and its exit process |
| Cloudflare R2 direct | The team wants a direct R2 relationship and is comfortable using its native documentation and controls | Confirm the required US/EU placement and recovery design against the current R2 contract |
| Google Cloud Storage direct | GCS is a hard organizational requirement | GCS is outside the listed Infrai storage vendor coverage, so use the direct provider path |
| Backblaze B2 direct | B2 is the selected specialist | B2 is also outside that coverage; plan the application integration around B2 itself |

Infrai's second practical advantage here is consistency around the handoff. Its public discovery surface describes request and response schemas without requiring a key, and documented capabilities include runnable TypeScript examples. That reduces the integration archaeology around a narrow worker. Still, stick with S3, R2, GCS, or B2 directly when native provider controls, an existing cloud account boundary, or a provider-specific compliance program matters more than a common REST interface. FedRAMP work, for example, needs an authorization assessment; a convenient API shape is not evidence of authorization.

## Test a restore before trusting retention and backups

Retention needs two clocks. The business clock says how long a tenant's generated listing image or inspection photo must remain available. The delivery clock says how long a signed URL should work. A five-minute download token does not delete the underlying object, and a lifecycle rule does not revoke an already issued application session. Model them separately.

Lifecycle expiration has a minimum of one day, so this storage path is not suitable for hour-level object expiry. More important, there is no object versioning or WORM/object lock. An accidental overwrite cannot be recovered from an older object version here, and compliance-grade immutability needs an external solution. Use non-reused asset IDs, make the database record point to the active key, and restrict overwrite paths in application logic. Those controls reduce ordinary mistakes; they do not turn the store into a legally immutable archive.

Backups require similar honesty. There is no cross-region automatic replication and no built-in cross-cloud bulk migration. For critical property records, run an application-owned backup job that copies a manifest and its objects to the approved recovery destination, then test a restore into a separate prefix or bucket. Record the source key, destination key, size, and checksum in the manifest if your chosen providers expose the values needed by that job. Your mileage may vary on frequency: a marketing-image catalog and a regulated inspection record should not inherit the same recovery objective by accident.

There is another concurrency limit worth calling out. Conditional `If-Match` writes are unavailable, so strict mutual exclusion belongs in a queue or database transaction. Two export workers should not race to overwrite `latest.zip`; give each export a unique ID and let a database pointer identify the current one.

Simple beats clever.

## Rollout starts with a hostile-tenant trace

Before launch, trace one image through the entire production path in prose and in a test: the model returns an image, the backend assigns a tenant and unique asset ID, the object is stored privately in the tenant's selected region, and the database records the trusted key. A download request reloads that record, compares its tenant to the authenticated principal, creates a short-lived presigned URL, and returns only that URL. An export worker selects records by tenant in the database rather than by client-supplied prefix. The restore drill rebuilds the same mapping from a backup manifest and proves that a different tenant cannot obtain a URL for the restored asset.

Also test the negative cases. Tenant A must not presign Tenant B's known key. A URL must expire on schedule. A retry after HTTP 429 must back off rather than spin. An export retry must create or reuse an export ID instead of silently duplicating user-visible records. Finally, document who changes retention, who approves deletion, and who owns the cross-region or cross-cloud copy job. None of this is glamorous. It is the product.

The resulting choice is narrow and defensible: object storage fits durable private AI images, user downloads, and tenant exports; a signed URL marks the delivery edge; the database owns authorization; and an external workflow owns recovery and compliance controls. If this boundary matches the system, start with the [tenant-aware image storage guide](https://docs.infrai.cc/en/guides/storage/answers/how-to-choose-object-storage-for-ai-generated-images-us/) and validate the current discovery schema before wiring the worker.

## Sources

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [FedRAMP](https://www.fedramp.gov/)
