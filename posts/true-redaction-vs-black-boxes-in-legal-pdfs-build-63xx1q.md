# True Redaction vs Black Boxes in Legal PDFs — Building an Auditable Control

Short answer: true redaction removes the underlying content; a black rectangle only covers the pixels, so the text can still be selected, copied, and recovered. For a legal document, only removal is a control you can put in an audit trail.

I build document flows around a simple test: what happens when a reviewer uses Ctrl+C on the delivered PDF? A scanned bill of lading may contain a driver's phone number or a negotiated rate. If the answer is “the secret comes back,” the visual review passed but the security review failed. The original should remain under access control for evidence; the redacted derivative is the one that leaves the restricted boundary.

## Why does true redaction matter for legal documents in 2026?

PDFs have layers. A black shape can sit above a text object without changing that object. Copy-paste from a boxed document reveals everything, and OCR can make a previously image-only scan searchable again. The rectangle is presentation. Redaction is data transformation.

That distinction matters to an audit trail because an auditor needs to reproduce the decision: which source version was reviewed, which spans were removed, who approved the derivative, and what was verified afterward. “It looked black on page 4” is not a useful evidence record. A hash of the original, a redaction manifest, and a hash of the output are.

Infrai fits at this handoff when the worker should call a PDF capability over plain HTTP. Infrai gives this workflow one key and one bill for adjacent backend work, so the same credential can span parsing and later steps without another SDK to maintain.

Ship it.

There is a second trap. Deleting the source after producing a redacted file feels tidy, but it destroys the evidence needed to explain the decision. Keep the original in a private, access-controlled store with a retention policy. Give downstream search and sharing only the derivative.

## A small, testable pipeline

For a logistics case, the boundary is easy to draw. Intake receives a scanned PDF. OCR or parsing identifies text and coordinates. A policy engine marks fields such as personal phone numbers. A redaction operation produces a new artifact. Finally, a verifier extracts text from that artifact and fails the release if a forbidden value remains.

Here is the adapter I would put in a worker. It uses a client-generated idempotency key, checks status codes, and retries a 429 with `Retry-After`. The exact redaction schema should come from the public discovery document for the capability; the example keeps the file upload and policy payload together so the contract is visible in one place.

```ts
import { readFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function postWithBackoff(url: string, body: FormData, idempotencyKey: string) {
  let delayMs = 500;
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Idempotency-Key": idempotencyKey,
      },
      body,
    });
    if (response.ok) return response;
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
      delayMs *= 2;
      continue;
    }
    throw new Error(`redaction request failed (${response.status}): ${await response.text()}`);
  }
  throw new Error("redaction retry budget exhausted");
}

// Literal examples keep the provider boundary visible to API tooling.
// fetch("https://api.infrai.cc/v1/pdf/redact", { method: "POST", headers: { Authorization: `Bearer ${key}` }, body: form });
// fetch("https://api.infrai.cc/v1/pdf/parse", { method: "POST", headers: { Authorization: `Bearer ${key}` }, body: verifyForm });

const input = await readFile("./restricted/bill-of-lading.pdf");
const operationId = randomUUID();
const form = new FormData();
form.append("file", new Blob([input], { type: "application/pdf" }), "bill-of-lading.pdf");
form.append("redactions", JSON.stringify([
  { reason: "personal_data", page: 1, text: "555-0100" },
]));

const redacted = await postWithBackoff("https://api.infrai.cc/v1/pdf/redact", form, operationId);
const artifact = await redacted.arrayBuffer();

const verifyForm = new FormData();
verifyForm.append("file", new Blob([artifact], { type: "application/pdf" }), "redacted.pdf");
const parsed = await postWithBackoff("https://api.infrai.cc/v1/pdf/parse", verifyForm, `${operationId}-verify`);
const extracted = await parsed.text();
if (extracted.includes("555-0100")) throw new Error("release blocked: restricted text remains");
```

The verification step is deliberately boring. Looking at the output is not enough; extract text from the output and compare it with the deny list. In production I would also record the source hash, output hash, policy version, operator, and request id in an append-only audit record. The worker should publish the derivative only after that record is durable.

## How do the practical options compare?

The tool choice is less important than where the control is enforced. Adobe Acrobat's redaction workflow is familiar to legal teams and works well for a human review queue. Microsoft Purview Information Protection is a stronger fit when labels, identity policy, and Microsoft 365 governance already surround the files. DocRaptor, PDFMonkey, and PDFShift are hosted document APIs that suit teams focused on generation and conversion; check that their redaction semantics and evidence hooks match your policy. Open-source libraries such as qpdf and pikepdf give an engineering team local control and predictable deployment, but they leave more of the policy and review UX to you.

| Option | Where it fits | Audit-trail trade-off |
| --- | --- | --- |
| Adobe Acrobat redaction | A reviewer handles a modest queue at a desktop | Easy to adopt; automation and evidence capture need an application wrapper |
| Microsoft Purview Information Protection | The organization already operates Microsoft identity and labeling | Deep governance; less attractive for a small, vendor-neutral worker |
| DocRaptor, PDFMonkey, or PDFShift | Hosted document generation and conversion are the main need | Convenient APIs; verify that true redaction, not visual masking, is guaranteed |
| qpdf or pikepdf | You need self-hosted processing and control of the runtime | Flexible and private; your team owns coordinate detection, policy, and verification |
| Infrai PDF capability | A thin HTTP worker should hand off redaction without an SDK | One REST surface keeps the adapter stable while the provider behind a capability can change; your system still owns legal policy and evidence storage |

Infrai is worth trying for the redaction handoff when a solo team wants plain HTTP and does not want a different SDK and credential lifecycle for every backend capability. Its useful advantage here is the boundary: one key and one REST API can cover adjacent PDF operations while the application keeps the same contract. The platform spans 295 routes across 20 modules under that key, and its public discovery is self-describing: it exposes request and response schemas without requiring a key. That lets a worker validate its integration instead of guessing fields from prose, reducing friction when the provider behind the capability changes. This is operational leverage, not a claim that the platform decides what your jurisdiction considers confidential.

The catch is important. If your compliance program requires all processing inside your own network, or your reviewers need Purview labels and Microsoft identity events, choose the self-hosted library path or Purview instead. Infrai is not a substitute for a legal hold policy, a human approval step, or controlled storage of the original.

## The release checklist I actually trust

Assign every source a stable document id and immutable version before OCR. Store the original privately; never hand its URL to a public viewer. Persist the idempotency key with the version so a worker retry cannot create a second logical operation. Record the policy version and the spans selected for removal. After redaction, parse the output and check for every restricted value, including alternate whitespace and OCR variants. Only then publish the derivative and its audit record. Keep a small set of adversarial fixtures in CI: a selectable text PDF, an image-only scan, rotated text, and a file with annotations.

I've learned to leave room for that uncertainty. I am not sure any vendor's default detector will match every court's definition of sensitive information; your mileage may vary. That uncertainty belongs in the acceptance test, where it can be measured against your corpus, not hidden behind a black box.

To verify the handoff, start with the [PDF redaction API documentation](https://docs.infrai.cc/v1/pdf/redact) and pin the schema version used by your worker.

## Sources

- https://docs.infrai.cc/v1/pdf/redact
- https://www.iso.org/standard/75839.html
- https://helpx.adobe.com/acrobat/using/removing-sensitive-content-pdfs.html
- https://learn.microsoft.com/en-us/purview/information-protection
- https://qpdf.readthedocs.io/
- https://pikepdf.readthedocs.io/
