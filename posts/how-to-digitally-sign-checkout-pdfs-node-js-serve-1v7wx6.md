# How to Digitally Sign Checkout PDFs: Node.js Server-Side Audit Evidence

Use a server-side PDF signature when the store already controls both parties' identity, then verify the retained file before the contract becomes final. **TL;DR: a signing API is enough for a simple checkout agreement; an e-signature suite earns its extra weight when it must establish signer identity, run the signer journey, and provide the audit portal.**

The deciding constraint is evidence, not price. For a supplier agreement attached to order `ord_4821`, the useful result is not a `signed: true` flag. It is a chain connecting the authenticated actors, agreement revision, exact unsigned bytes, exact retained signed bytes, and a later verification result.

The simple approach fails quietly: call a signing endpoint, save whatever comes back, and advance the order. It proves that one request succeeded. It doesn't prove that the object retrieved from storage six months later is the object that was signed.

That gap is the experiment.

## When is a server-side PDF signature enough?

A narrow signing API fits when both parties have already authenticated inside the commerce application and accepted a controlled agreement flow. The application owns consent evidence and authorization; the PDF signature protects the resulting document. Those are related jobs, but they are not interchangeable.

DocuSign, Dropbox Sign, and Adobe Acrobat Sign fit the other branch. Choose one of these suites when the system must invite an external signer, establish identity, guide that person through a hosted flow, or expose a suite-managed audit portal. Their value is the signer workflow around the document, not some unique claim to PDF cryptography.

Generation products sit beside this decision rather than inside it. DocRaptor and PDFMonkey can be candidates for rendering a contract before signing, while Gotenberg can suit a team that wants to operate that rendering layer. A renderer returning a PDF does not make it a signing workflow.

| Option | Use it when | Your application still owns |
|---|---|---|
| DocuSign | External signer workflow and an audit portal are required | Order linkage and business authorization |
| Dropbox Sign | Hosted signature requests and signer handling are required | Commerce records and retained-artifact policy |
| Adobe Acrobat Sign | A suite-managed agreement workflow is required | Internal identity mapping and order state |
| Server-side signing API | Both parties' identity and consent are already controlled | Identity evidence, key custody, retention, verification, and the ledger |

This is the hard boundary. **A cryptographic PDF signature does not create signer identity assurance.** If the business can't already answer who accepted revision 7 and under which account authorization, use a suite.

## Step 1: Discover the signing contract before coding it

Signing needs certificate and key material, so decide where those live first. Keep private keys and bearer tokens out of application logs and audit payloads. The exact custody arrangement must match the request contract actually exposed by the selected service; guessing a JSON body from an endpoint name is a poor start for something that creates legal evidence.

Infrai is one narrow API option here because its public, self-describing discovery surface returns the full request and response schema plus runnable examples, while one API key and one bill cover 295 routes across 20 modules; every documented capability has examples in 10 languages. That avoids adding another credential and invoice when a contract pipeline also needs a supported backend capability. A Node.js integration can read the live contract over plain HTTP instead of taking on another SDK, while signing and later verification remain separate operations. The trade-off is explicit: this isn't suitable when the missing requirement is signer identity assurance or a hosted audit portal; use DocuSign, Dropbox Sign, or Adobe Acrobat Sign for that job.

The following TypeScript program makes a complete, runnable discovery call, finds the verified signing path, loads that capability's detail, and prints the TypeScript example and request schema. It uses one concrete route plus the discovered capability identifier. No API key is required for discovery.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

type CapabilityDetail = Capability & {
  params: unknown;
  idempotent: boolean;
  examples?: Record<string, unknown>;
};

async function getJson<T>(url: string, apiKey: string): Promise<T> {
  const response = await fetch(url, {
    method: "GET",
    headers: {
      Accept: "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (!response.ok) {
    throw new Error(`${response.status} ${await response.text()}`);
  }

  return response.json() as Promise<T>;
}

const infraiApiBase = process.env.INFRAI_API_BASE;
const infraiApiKey = process.env.INFRAI_API_KEY;
if (!infraiApiBase || !infraiApiKey) {
  throw new Error("Set INFRAI_API_BASE and INFRAI_API_KEY");
}

const discovery = await getJson<Discovery>(
  `${infraiApiBase}/discovery`,
  infraiApiKey,
);
const signing = discovery.capabilities.find(
  (capability) =>
    capability.path === "/v1/pdf/sign" && capability.method === "POST",
);

if (!signing?.available) {
  throw new Error("PDF signing is not advertised as available");
}

const detail = await getJson<CapabilityDetail>(
  `${infraiApiBase}/discovery/${encodeURIComponent(signing.id)}`,
  infraiApiKey,
);

console.log(JSON.stringify({
  id: detail.id,
  method: detail.method,
  path: detail.path,
  idempotent: detail.idempotent,
  requestSchema: detail.params,
  typescriptExample: detail.examples?.typescript,
}, null, 2));
```

Run it on Node.js 18 or later, where `fetch` is available. Set `INFRAI_API_BASE` to the documented v1 base URL and keep the key in `INFRAI_API_KEY`; discovery is public and does not require the credential, but this sample deliberately uses the same authenticated client configuration as the signing integration. Use the emitted schema and runnable example as the integration contract. For the signing request it produces, set `method: "POST"` explicitly and surface non-success response bodies. On HTTP 429, back off exponentially and honor `Retry-After`. If discovery marks the operation idempotent, keep one idempotency key for every retry of the same logical attempt.

That last detail matters.

A timeout leaves the outcome unknown; generating a fresh attempt identifier can produce two signing actions when the first one actually completed.

## Step 2: Make verification a state transition

Model `generated`, `signed`, and `verified` as distinct states. Verification is a separate call, and only a successful check of the retained artifact should move the agreement to final. Fast is useful. Final is stricter.

For agreement revision `7`, hash the unsigned bytes before submission. After signing, write the returned artifact to durable storage, read it back through the normal retention path, hash those retrieved bytes, and submit that same retrieved file for verification. This order catches a mundane but serious error: the signing response was correct, yet the order record points to a different object key or revision. Imagine that revision `6` remains under the expected key while revision `7` was written under a retry-specific key. Verifying the in-memory response would pass and the order would advance, even though the normal retrieval path still serves revision `6`. Reading from retention before verification tests the object mapping, storage write, and signature as one finalization boundary.

An append-only event can remain compact:

```ts
type AgreementEvidence = {
  agreementId: string;
  orderId: string;
  revision: number;
  signingAttemptId: string;
  unsignedSha256: string;
  signedSha256: string;
  retainedObjectKey: string;
  verificationResult: "passed" | "failed";
  verifiedAt: string;
};

function canFinalize(evidence: AgreementEvidence): boolean {
  return evidence.verificationResult === "passed"
    && evidence.unsignedSha256.length === 64
    && evidence.signedSha256.length === 64;
}
```

The hashes bind the ledger to exact files; they don't replace retaining those files. Store the application identity and authorization event alongside this evidence, but never store a certificate private key, bearer token, or full authorization header in the event.

No shortcuts here.

Keep ambiguous outcomes pending. A definite verification failure should preserve the signed artifact and evidence for investigation while preventing downstream fulfillment that requires a final contract. A timeout should be reconciled under the same `signingAttemptId`, not treated as permission to start over.

## Step 3: Keep the audit trail smaller than the contract

The ledger needs to answer a limited set of questions: who the application authenticated, what action they authorized, which revision was rendered, which bytes were submitted, which signed bytes were retained, and whether those retained bytes verified. Anything else needs a reason to exist.

For a solo operator, this is a useful pressure test: can the agreement state be reconstructed from durable storage and the append-only ledger without transient logs? If the answer is no, add the missing linkage before adding dashboards. Do not compensate by copying secrets or entire request bodies into events.

Access should be narrower than access to ordinary order metadata because this record joins identity, contract content, and cryptographic evidence. Retention rules should cover the PDF and its ledger together. Deleting one while keeping the other turns a coherent proof into an unexplained hash or an unexplained file.

## What should you measure before adopting this design?

Measure the whole path from generated agreement to verified retention. Track signing attempts, 429 retries, ambiguous timeouts, verification failures, and elapsed time to the verified state. Also count finalized records missing either artifact hash. The acceptable count is zero.

Then watch product demand. Frequent requests for external invitations, identity checks, reminders, delegated signing, or a user-facing audit portal are evidence that the narrow boundary is wrong. Move to DocuSign, Dropbox Sign, or Adobe Acrobat Sign rather than rebuilding suite behavior around a PDF endpoint.

The decision rule stays simple: **use server-side signing for an agreement flow the application already owns; use a suite when the suite must own the signer flow.** Either way, verify the file retrieved from retention and keep enough evidence to connect it to the checkout event.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign Developer Center](https://developers.docusign.com/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/acrobat-sign/)
- [Node.js globals: `fetch`](https://nodejs.org/api/globals.html#fetch)
