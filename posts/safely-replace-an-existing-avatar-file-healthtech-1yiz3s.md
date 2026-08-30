# Safely Replace an Existing Avatar File: Healthtech Upload Throughput Under Storage Races

Large generated reports change the storage decision: copying an existing object during a profile update burns bandwidth that should be serving authenticated customers. **Short answer:** write every avatar upload to a unique key, publish that key with a conditional database update, and retire the previous object after the commit; reserve in-place overwrite for stores where a generation or entity-tag precondition is part of the design.

This is a throughput choice, not a claim that unique names magically solve concurrency. The object store moves bytes. A small metadata row decides which bytes are current. Keeping those jobs separate matters in a healthtech service where a 180 MB generated report and a 240 KB avatar may share the same storage client, connection pool, and egress path.

The simple approach is tempting: upload every new image to `members/42/avatar.webp`. It also gives object completion order more authority than it deserves. Two tabs can begin with the same old image, finish in the opposite order, and leave the application unable to explain which user action selected the visible result.

Don't make byte arrival your commit protocol.

## How can an avatar upload safely replace an existing object storage file?

Treat replacement as publication. First create an immutable candidate such as `members/42/avatars/0195...webp`. Then validate the candidate and conditionally change the member row from its observed revision to the candidate key. A losing request never becomes visible. Once the winning database transaction commits, a cleanup job may delete the former key.

That ordering has an important consequence: the request that finishes its upload last does not automatically win. The policy can instead be “the first valid commit wins,” or the application can attach a client-issued sequence and accept only the newest intent. The policy belongs beside authenticated user state, where it can be audited, rather than inside an opaque race between object writes.

For stores that expose object generations or HTTP entity tags, an in-place write can use a precondition. RFC 9110 defines `If-Match` as a way to make a method conditional on the selected representation matching a supplied entity tag, and describes preventing the lost-update problem as a common use. Google Cloud Storage documents generation and metageneration preconditions for conditional requests. Those controls are useful when preserving one stable object name is a hard requirement.

They aren't interchangeable with a database decision. An object precondition arbitrates writes to that object; it does not atomically update a member record, an authorization policy, and an audit event in another system. For a user-facing avatar, unique candidates plus a row compare-and-swap keep the application state transition explicit.

## How does byte-amplification testing expose waste in a storage workflow?

In the healthtech workload, generated reports deserve the data plane's attention. The avatar path should perform one upload on the request path and avoid a read-copy-write cycle. Renaming an object is commonly implemented as copying bytes to a new key and deleting the source, so the application should generate the final candidate key before upload rather than treating a temporary key as free.

This gets more important as files grow. Suppose an authenticated customer requests a large generated report while a background process “renames” completed uploads. Both operations can consume the same outbound connections and storage bandwidth. The report's access check may be fast while delivery still stalls behind avoidable copying. A clean-looking key hierarchy isn't worth that contention.

Keep the hot path short:

1. Authorize the member and reject disallowed media before storage work.
2. Generate the final unique key.
3. Stream the candidate once while enforcing an application size limit.
4. Validate the stored candidate before publication.
5. Compare-and-swap the metadata row.
6. Queue deletion only after the transaction commits.

One upload. No rename.

The same separation helps report delivery. Store an immutable report object, keep its current status and access rules in transactional metadata, and stream it only after authorization. The avatar and report domains can share a storage adapter without sharing a mutable `current` object naming convention.

## Implement one upload and one metadata commit

The code below leaves transport and database choices behind narrow interfaces. Its critical behavior is the conditional metadata write. It doesn't assume that a storage `put` and a database transaction can be made atomic.

```ts
import { randomUUID } from "node:crypto";

type AvatarState = {
  objectKey: string | null;
  revision: number;
};

interface ObjectStore {
  put(key: string, body: Uint8Array, contentType: string): Promise<void>;
  remove(key: string): Promise<void>;
}

interface MemberRepository {
  readAvatar(memberId: string): Promise<AvatarState>;
  publishAvatar(
    memberId: string,
    expectedRevision: number,
    objectKey: string,
  ): Promise<boolean>;
}

interface CleanupQueue {
  enqueue(objectKey: string): Promise<void>;
}

export async function publishAvatar(
  memberId: string,
  bytes: Uint8Array,
  contentType: "image/jpeg" | "image/png" | "image/webp",
  store: ObjectStore,
  members: MemberRepository,
  cleanup: CleanupQueue,
): Promise<{ objectKey: string; revision: number }> {
  const observed = await members.readAvatar(memberId);
  const extension = contentType.substring("image/".length);
  const candidateKey = `members/${memberId}/avatars/${randomUUID()}.${extension}`;

  await store.put(candidateKey, bytes, contentType);

  const published = await members.publishAvatar(
    memberId,
    observed.revision,
    candidateKey,
  );

  if (!published) {
    await cleanup.enqueue(candidateKey);
    throw new Error("Avatar changed before publication");
  }

  if (observed.objectKey) {
    await cleanup.enqueue(observed.objectKey);
  }

  return { objectKey: candidateKey, revision: observed.revision + 1 };
}
```

`publishAvatar` must be a real compare-and-swap, for example an update constrained by both member ID and expected revision. The cleanup consumer should be idempotent because a delivered job may be retried. It should also confirm that the object identifier is no longer referenced before deletion; a delayed job must never remove a candidate that another administrative action restored.

There is an awkward failure window after `put` and before publication. If the process exits there, the candidate is orphaned. I wouldn't add a distributed transaction to close that gap. Record candidate creation in durable metadata before or alongside upload when strict accounting is required, or run a conservative sweeper that removes only old, unreferenced candidates. The retention window should be longer than the maximum plausible upload and retry duration. I'm not sure one universal window exists; measured tail latency and retry age should set it.

For HTTP-facing code, preserve the distinction between concurrency and transport errors. A failed compare-and-swap is an expected conflict and can map to `409 Conflict`. An oversized body can map to `413 Content Too Large`. Authentication and authorization failures need their normal handling before any object key is disclosed. Exact retry behavior should follow the selected store's documented semantics, especially for conditional requests.

## Govern retention and rollout with production evidence

Start with bytes, not request counts. Track upload bytes, report-download bytes, cleanup bytes, and any copy bytes separately. A single large report can matter more to capacity than hundreds of avatar changes, so a blended operation counter hides the decision axis.

Also record publication conflicts, candidate age, orphan count, cleanup lag, and references to missing objects. The useful trace joins an authenticated request ID to the candidate key and metadata revision without logging report contents, image bytes, signed URLs, or credentials. Healthtech logging deserves a narrow data budget.

The catch is operational state. Unique keys create garbage that must be inventoried and removed, and delayed deletion consumes capacity. This pattern is not suitable when regulation or policy requires storage-native write-once retention, legal hold, or an atomic condition directly on a stable object identity. Choose a storage service with the required retention and conditional-write controls in those cases, and verify them in its primary documentation.

Stick with an in-place key when downstream systems cannot tolerate changing identifiers and the store's generation or entity-tag preconditions are enforced end to end. Even then, test concurrent writes, stale preconditions, interrupted uploads, cleanup retries, and authenticated large-file delivery under a shared bandwidth limit. The final decision should come from p95 and p99 delivery latency, total bytes copied, conflict rate, and orphan age in the actual workload.

The ship-first default remains modest: immutable candidates, one metadata authority, asynchronous cleanup, and no extra byte copy on the request path. Measure it before copying the choice into every file workflow.

## Sources

- https://www.rfc-editor.org/rfc/rfc9110
- https://cloud.google.com/storage/docs
