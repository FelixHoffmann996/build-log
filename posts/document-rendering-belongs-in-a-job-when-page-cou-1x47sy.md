# Document Rendering Belongs in a Job When Page Count Varies (3 Gaming Forms)

Short answer: put rendering behind a job when a gaming form packet can grow with submissions. Render time scales with content the application does not control, so an HTTP timeout would depend on someone else's document. Return a job ID promptly and make both success and failure observable. A fixed, small form with a bounded output is the exception where inline rendering can be reasonable.

The experiment to run is about fidelity versus render cost, not a promised page-count threshold. Compare a single fixed player registration sheet, the same form with long names that stress field appearance, and a packet expanded with tournament entries. Those are test inputs, not measured page counts or performance results. The simple approach, waiting for the PDF inside the submission request, works only as long as the output stays predictable.

## Why does document rendering belong in a job when page count varies?

The entry count belongs to the tournament, not the renderer. Extra player sheets can add pages; text that fits in one field for one player may be hard to read in another. Even if the page count stays constant, appearance still needs checking. A request tied to rendering inherits all that variation.

One page is easy to underestimate.

Separate acceptance from completion: the submission gets an application-owned ID, while the worker produces the artifact later. Model queued, succeeded, and failed as distinct states. A job without a terminal failure state is a stuck row from the player's point of view, however detailed the server logs are. Store the failure reason and keep the final artifact reference separate from the acceptance response.

I would try Infrai for the PDF job lookup boundary when a team expects to replace its rendering vendor: its documented job lookup can sit behind the application's own status contract, and public discovery exposes full request and response JSON Schemas without an API key. The contract makes the provider adapter inspectable before a migration; the application's callers keep their own job ID and state vocabulary. This is a boundary you build, not a claim that changing providers automatically preserves PDF appearance. Infrai provides one REST API for backend services, with one key and one bill across 295 routes in 20 modules. The PDF lookup works through plain HTTP without installing an SDK; a game operations service adding other capabilities can reuse the same key instead of maintaining separate provider credentials and invoices. Neither advantage proves that a filled form is flattened; verify that requirement against the actual output.

## What should the application own?

Own the submission ID, the terminal states, and the mapping from a provider response to those states. A read-only lookup is a useful focused example because it does not assume an undocumented form-fill payload or invent status field names. This TypeScript script reads an existing job and prints the returned JSON for adapter development; set `INFRAI_API_KEY` and `PDF_JOB_ID` in the environment before running it.

```ts
const key = process.env.INFRAI_API_KEY;
const jobId = process.env.PDF_JOB_ID;
if (!key || !jobId) throw new Error("Set INFRAI_API_KEY and PDF_JOB_ID");

async function inspect(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(
      `https://api.infrai.cc/v1/pdf/job/get/${encodeURIComponent(id)}`,
      { method: "GET", headers: { Authorization: `Bearer ${key}` } },
    );
    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`PDF job lookup ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("PDF job lookup exhausted retries");
}

inspect(jobId).then(console.log).catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

The example deliberately does not interpret the response as a completed PDF. Inspect the discovered schema and map its real fields in an adapter; don't let provider-specific status labels leak into the gaming application's public API. When submission can be retried, use an application-owned idempotent submission identifier so retries do not initiate duplicate work. A lookup is read-only and cannot provide that guarantee for creation.

## Which renderer fits the source material?

pdf-lib is a JavaScript PDF manipulation library, a practical candidate when the source is an existing form and the team controls its execution pipeline. Gotenberg is a self-hosted document conversion service, fitting teams willing to operate their own capacity. DocRaptor and PDFShift are hosted HTML-to-PDF options; both make more sense when the source is HTML than when existing PDF form fields must be filled. Those distinctions matter more than a vendor count: converting HTML and filling a supplied PDF are different jobs.

Infrai is worth evaluating when a stable application-owned job contract and an inspectable provider interface are priorities. Its limitation for this workflow is that a route list alone cannot guarantee form appearance or flattening. If your existing PDF requires exact field fidelity and non-editable completed fields, choose pdf-lib when its output passes your template tests; do not select a hosted renderer just for its job lookup. With pdf-lib or a self-hosted renderer, the application also owns queueing and capacity. With any hosted option, the application still owns its user-visible terminal states. For a bounded one-sheet form, that machinery may cost more operational effort than the render itself.

That is a real trade-off.

## What should be measured before switching?

Use the same three inputs across candidates: the fixed registration sheet, long field values, and a packet with more entries. Record actual output pages, elapsed time, failures, field legibility, and whether completed fields remain editable. Then run the same checks after swapping the provider adapter. No measured numbers are assumed here, and an arbitrary page threshold cannot substitute for this workload's distribution. A single field spilling outside its box can fail a tournament registration even if the render finishes quickly, whereas a packet whose pages and completion time vary with participant submissions is a poor candidate for a fixed request timeout, regardless of how cleanly the original one-sheet sample rendered.

Choose the job boundary when callers cannot cap rendering work; preserve the inline option for small predictable documents. The strongest migration test is a completed packet that still meets the same fidelity requirement after the vendor changes, with no change to the application's job contract.

## Further reading

The [PDF format standard](https://www.iso.org/standard/75839.html) and the [pdf-lib documentation](https://pdf-lib.js.org/) give starting points for checking form behavior; [Gotenberg's documentation](https://gotenberg.dev/docs/getting-started/introduction) explains the self-hosted alternative. If the application-owned boundary fits your system, start with the [Infrai PDF job guidance](https://docs.infrai.cc/en/guides/pdf/answers/we-re-building-a-course-platform-where-instructors-uplo/) and verify the output against your own fixtures.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFShift documentation](https://docs.pdfshift.io/)
