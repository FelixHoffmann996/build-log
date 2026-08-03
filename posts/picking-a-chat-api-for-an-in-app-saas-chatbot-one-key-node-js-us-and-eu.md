# Picking a chat API for an in-app SaaS chatbot: one key, Node.js, US and EU

**Short answer:** for an in-app SaaS chatbot, pick an OpenAI-compatible chat endpoint you can reach with one key from a plain Node.js route, and treat the region as per-customer configuration instead of an architecture decision. The API you can walk away from cheaply beats the API that benchmarks best this quarter.

I've shipped this twice. The first time badly.

That first version had a vendor SDK imported directly into my Express handlers, the model id hardcoded in three places, and streaming delegated to the SDK's own abstraction. It held up for about five months. Then I wanted a small model for the 80% of questions that amount to "where is my invoice", something stronger for the rest, and an EU-hosted option for two customers whose procurement people actually read the data processing addendum — and every one of those changes reached into application code. That rewrite cost me a week I hadn't budgeted, which is why the rest of this note is organized around swap cost rather than around model quality.

## What should an in-app SaaS chatbot API give you on day one?

Three properties, and raw model quality isn't one of them.

The first is a request shape with more than one implementation behind it. A messages array in, a choice or a stream of deltas out, `POST /v1/chat/completions` as the path — that shape has become the de facto wire format, spoken by hosted gateways, self-hosted proxies and local runtimes alike. When your backend speaks it, swapping providers is an environment variable and a redeploy. When your backend speaks a proprietary SDK's object model, swapping providers is a sprint.

The second is one credential covering everything the assistant reaches for. A support chatbot rarely stays pure chat; within a quarter it wants embeddings for a retrieval index, and maybe transcription for voice notes. Three separate signups means three key rotations, three invoices, and three vendors to chase when latency spikes.

The third is region as a setting, not a fork. More on that below.

## The Node.js route, from user message to streamed reply

The data flow stays boring on purpose. Your React widget posts to your own endpoint, your endpoint attaches the tenant's context and the model id it has been configured with, the upstream call streams tokens back, and you pipe those bytes straight through to the browser as server-sent events while recording usage on the way out. The model provider never sees your session cookies and never learns your user ids. Here's the entire server side.

```ts
import express from "express";

const app = express();
app.use(express.json());

// One credential, one base URL, both from env — a provider swap is a deploy, not a diff.
const BASE_URL = process.env.CHAT_BASE_URL ?? "https://gateway.internal.example/v1";
const API_KEY = process.env.CHAT_API_KEY!;
const MODEL = process.env.CHAT_MODEL!;

app.post("/api/chat", async (req, res) => {
  const { messages } = req.body as { messages: { role: string; content: string }[] };

  const upstream = await fetch(`${BASE_URL}/chat/completions`, {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${API_KEY}` },
    body: JSON.stringify({ model: MODEL, messages, stream: true }),
  });

  // Surface throttling to the caller. Never let a retry wrapper eat this silently.
  if (upstream.status === 429) {
    const retryAfter = upstream.headers.get("retry-after") ?? "5";
    res.status(429).set("retry-after", retryAfter).json({ error: "rate_limited", retryAfter });
    return;
  }
  if (!upstream.ok || !upstream.body) {
    res.status(502).json({ error: "upstream_unavailable" });
    return;
  }

  res.setHeader("content-type", "text/event-stream");
  res.setHeader("cache-control", "no-cache");
  await upstream.body.pipeTo(
    new WritableStream({
      write: (chunk) => void res.write(chunk),
      close: () => res.end(),
    }),
  );
});

app.listen(3000);
```

Roughly forty lines, no SDK, and the only provider-specific string in the whole file is a base URL. That's the setup I'd defend in review. The parts worth arguing about are the two error branches: everything else here is plumbing that any competent Node.js developer can read at a glance, while those branches decide what your users experience on a bad afternoon.

## Four ways to wire it, and what each one costs

| Approach | Setup cost | Credentials | Region control | Where it hurts |
| --- | --- | --- | --- | --- |
| Direct to a single model vendor | lowest | one per vendor | whatever that vendor publishes | every model swap turns into procurement |
| Hosted multi-model gateway | low | one key, many models | limited to the gateway's footprint | you inherit someone else's routing and uptime |
| Self-hosted gateway (LiteLLM and similar) | highest | you hold every upstream key | entirely yours | you now operate a proxy at 3 a.m. |
| Local open-weights runtime (Ollama and similar) | medium | none | trivially yours | throughput and quality ceilings on modest hardware |

Nobody stays in one row forever, and the migrations between rows are the expensive part. My bias is to start in row two and keep the code honest enough that row three stays reachable, since a self-hosted router speaks the same wire format and mostly changes who gets paged. The catch is that row three is not free: an open-source gateway in your own VPC means you own its memory profile, its connection pooling and its upgrade cadence, and I've seen a two-person team lose more hours to that than they ever saved on routing.

Row one deserves more respect than it usually gets. If your compliance story needs a single named subprocessor on a signed contract, stick with a direct vendor relationship and accept the swap cost. If your product is a coding assistant where one specific frontier model is the product, the abstraction buys you nothing.

## What US and EU actually mean once you have paying customers

Two different questions hide behind that phrase, and conflating them will bite you.

The first is latency, which is mostly about where your own server sits relative to the model provider's ingress, not about where the GPU lives. A round trip from an EU-hosted Express process to a US-hosted inference endpoint added about 140 ms of pure network overhead in my measurements, invisible against a streamed first token but very visible on a non-streaming embeddings batch.

The second is data residency, which is a legal question with a technical implementation. Chapter V of the GDPR governs transfers of personal data out of the EEA, and a chatbot transcript is personal data the moment a user pastes their own email address into it. What you want from a provider is a documented regional endpoint, a data processing addendum you can actually sign, and a retention setting you control. What you want from your own code is a per-tenant region field that selects the base URL, which is exactly why I keep that value in configuration rather than in a constant.

I'm not sure any of the residency guarantees in this market are as strong as their marketing pages imply, and if you're in health or finance your counsel will want the subprocessor list before you write a line of code.

## The 429 my retry loop swallowed

Here's the incident I promised. Our chat widget had a generic retry helper wrapped around every outbound call, three attempts with exponential backoff, written months earlier for a flaky webhook and never revisited. One Tuesday a marketing push doubled traffic and the upstream started throttling us. About 1,900 of roughly 12,000 chat requests that afternoon came back 429, and the retry helper dutifully re-sent every single one of them, three times, which of course made the throttling worse. Nothing appeared in our error dashboard, because from the application's perspective those requests eventually succeeded. What our users saw was a chat widget that took 9 seconds to say anything. I spent most of an evening staring at p95 graphs before I thought to log the response status inside the retry helper — and the fix, in the end, was seven lines: stop retrying 429 in the client, honor the `retry-after` header on the server, and queue anything that doesn't need to be interactive.

That last part matters more than it sounds. Plenty of assistant work — nightly summaries, backfilling embeddings over an existing knowledge base, re-classifying old tickets — has no user waiting on it, and running it through the same interactive path is how you end up rate limited during business hours. Batch-oriented endpoints exist for exactly this, and moving that work off the interactive path is usually the single largest reduction in throttling you can make without renegotiating anything.

For operations, keep it to four habits. Log the model id, token counts and latency on every call so a cost regression shows up as a chart rather than as an invoice. Alert on 429 and 5xx rates separately, since they mean different things and have different fixes. Keep an evaluation set of thirty or so real conversations with expected answers, and run it before any model swap, because that's the only thing standing between you and a silent quality regression. And make the base URL, the key and the model id three separate environment variables — you'll be glad the day one of them has to change alone.

## References

- OpenAI Batch API guide (batch-oriented request pattern): https://platform.openai.com/docs/guides/batch
- LiteLLM, an open-source self-hosted LLM gateway: https://github.com/BerriAI/litellm
- MDN, Server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- RFC 6585, section 4 (429 Too Many Requests): https://www.rfc-editor.org/rfc/rfc6585#section-4
- MDN, Retry-After header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- GDPR Chapter V, general principle for transfers: https://gdpr-info.eu/art-44-gdpr/
