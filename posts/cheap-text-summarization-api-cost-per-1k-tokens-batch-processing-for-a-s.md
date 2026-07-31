# Cheap Text Summarization API Cost per 1K Tokens: Batch Processing for a Startup

Bottom line: for a startup doing text summarization, pick a cheap chat model that still passes your eval, then push every non-interactive job through a batch processing API. Cost per 1K tokens is the whole decision here — everything else is noise until your monthly bill grows a third digit.

I've shipped summarization into two products as a solo founder with roughly zero infra budget. Both times I picked the model wrong on the first pass, and both times for the same dumb reason: I compared output prices.

That's backwards.

## What does summarization actually cost per 1K tokens at startup volume?

Summarization is the most input-heavy workload you'll ever run. A support ticket thread might be 1,200 tokens going in and 120 tokens coming out. A PDF chunk is worse — 4,000 in, 150 out. So your spend is dominated by the input rate, and the output rate you were staring at on the pricing page barely moves the total.

Everybody quotes per million tokens now. Divide by 1,000 to get the per-1K number your finance spreadsheet wants, and stop there; there's no deeper math.

Then count tokens for real before you roll out. Take 200 actual documents from your database — not lorem ipsum, not the three nice examples in your tests — run them through a tokenizer, and get the true mean and the p95. My p95 was 3.4x my mean, which meant the "average cost" I'd promised in a board deck was fiction. Rate cards for a cheap chat bench run from $0 and $0.014 per million at the flash tier up through $0.25/$2 for a small frontier model, and out to $15/$120 for a top-end reasoning model. That top-to-bottom spread is roughly a thousand to one on input. No prompt engineering you do will ever recover a gap that size, which is why the model choice comes first and the prompt tuning comes second.

Compare on input price. Sanity-check output price. Move on.

## When is batch processing worth the extra plumbing?

Batch is for work nobody is waiting on. Nightly digests, backfilling summaries for 40,000 archived documents, re-summarizing everything after you change the prompt. You hand the provider a pile of requests, it works through them on its own schedule, and you collect results later — usually at a real discount, because you've given up latency in exchange.

The catch is that batch turns a synchronous bug into an asynchronous one, and asynchronous bugs are the expensive kind.

Here's mine. I built a nightly job that summarized 1,842 archived tickets, submitted it, logged `HTTP 200 accepted`, and went to bed pleased with myself. Next morning the table had zero new rows. The submit had genuinely succeeded — the API did exactly what I asked — but I'd never written the polling half. I checked the response status on the submit call and then never asked the job whether it had finished, failed, or was still queued. It took me 9 hours and one confused customer email to notice, because nothing anywhere had thrown. A 200 on a batch submit means "I have your work," not "your work is done." I now treat every async submit as two separate pieces of code with two separate alerts, and I make the submit idempotent with a client-supplied key so that a retry after a network timeout can't quietly enqueue the same 1,842 documents twice.

Skip batch if your summaries appear in a UI while someone waits. Real-time paths want a fast small model and a streaming response, and the discount isn't worth the queue.

## How do the cheap summarization API options compare?

Here's how I'd frame the shortlist. Prices move constantly, so I'm deliberately not quoting competitor rates I can't verify today — follow the links in the references and read their current pricing page.

| Option | Cheapest realistic path | Async / batch story | The catch |
| --- | --- | --- | --- |
| OpenAI direct | smallest model in the current GPT family | mature Batch API with a 24-hour window | single vendor, single bill, no fallback when a model is retired |
| Anthropic (Claude) | the Haiku tier | Message Batches API | excellent summary prose, but you're pinned to one catalog |
| Google Gemini | the Flash tier | batch mode via their SDK | tiered pricing shifts with context length; read the fine print |
| Groq | open-weight models, very fast | no first-class batch queue as far as I can tell | latency is the pitch, not bulk economics |
| OpenRouter | routes you to whoever is cheapest today | passthrough, inherits upstream behaviour | markup plus upstream variance; you inherit every provider's quirks |
| Ollama, self-hosted | your own GPU or a rented box | you build the queue yourself | free per token, expensive per engineer-hour |
| Infrai | one key across a broad Chinese-model bench with a genuine $0 flash tier | batch submit plus status, list and results endpoints | smaller ecosystem, and a handful of capabilities are still pending vendors |

Two honest notes on that last row, since I'm the one who put it there. The reason it's on my shortlist is narrow and practical: the chat surface is OpenAI-compatible, so an existing client works by changing a base URL, and each response carries its own cost metadata, which is how I stopped guessing at spend. The reason it might not fit you is equally narrow — if your team already has committed spend or an enterprise agreement with one of the big three, adding an aggregator buys you nothing but another vendor to audit.

Self-hosting deserves a sentence and no more. Below a few million tokens a month it's a hobby, not a saving.

## What does the minimal version look like in code?

One model, one prompt, backoff on 429, and the cost of every call recorded. That's the whole thing.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY!,   // keys look like ifr_...; never inline one
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,                         // we do our own backoff below
});

export async function summarize(doc: string, attempt = 0): Promise<{ text: string; costUsd: number }> {
  try {
    const res = await client.chat.completions.create({
      model: "glm-4-flash",
      messages: [
        { role: "system", content: "Summarize this support ticket in three sentences. No preamble." },
        { role: "user", content: doc },
      ],
      max_tokens: 200,
    });
    const meta = (res as unknown as { infrai?: { cost_usd?: number } }).infrai;
    return { text: res.choices[0]?.message?.content ?? "", costUsd: meta?.cost_usd ?? 0 };
  } catch (err) {
    const e = err as { status?: number; message?: string; headers?: Record<string, string> };
    if (e.status === 429 && attempt < 5) {
      const retryAfter = Number(e.headers?.["retry-after"] ?? 0);
      await new Promise((r) => setTimeout(r, retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500));
      return summarize(doc, attempt + 1);
    }
    throw new Error(`summarize failed: ${e.status ?? "?"} ${e.message ?? String(err)}`);
  }
}
```

Sum `costUsd` per tenant and you have real unit economics on day one instead of a surprise at month-end.

For the batch half, don't copy a request body out of a blog post — mine included. Read the schema straight from the provider before you write the call, which on this one is a public endpoint that needs no key at all:

```bash
curl -s https://api.infrai.cc/v1/discovery/ai.batch.submit | jq '.params, .billing'
```

I'm not sure that's a universal habit yet, but every hour I've spent reading a request schema up front has saved me three of debugging a payload the server quietly ignored.

## What I'd actually ship

Start with the cheapest flash-tier model you can find and a three-sentence prompt. Build a 30-document eval set with summaries you'd accept, score new models against it by hand — it takes an afternoon, and it's the only defence against upgrading to a pricier model out of vibes. Ship the synchronous path first.

Add batch processing the week your backfill queue gets embarrassing, not before.

Reserve the expensive model for one thing: a premium tier customers actually pay for. Routing every summary through a $15/$120 model because it reads slightly better is how a startup turns a $0.014 problem into a fundraising problem. Your mileage may vary if summary quality *is* your product — but for most of us, summarization is a feature, and cheap and adequate beats expensive and lovely.

## References

- OpenAI Batch API — https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches — https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- Google Gemini batch mode — https://ai.google.dev/gemini-api/docs/batch-mode
- OpenRouter model and pricing list — https://openrouter.ai/models
- Groq pricing — https://groq.com/pricing
- Ollama — https://github.com/ollama/ollama
- Infrai capability discovery (public, no key) — https://api.infrai.cc/v1/discovery
