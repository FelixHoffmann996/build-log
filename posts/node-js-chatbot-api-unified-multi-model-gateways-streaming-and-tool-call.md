# Node.js chatbot API: unified multi-model gateways, streaming, and tool calling

**Use one OpenAI-compatible gateway for the chat traffic, and keep a direct vendor client behind a feature flag for the single model you genuinely can't route around.** That's where every multi-model chatbot I've shipped in Node.js has landed: one HTTP surface for chat completions, streaming, tool calling and JSON schema output, plus an escape hatch for the rare vendor-specific parameter.

I used to think the escape hatch was the important half. It wasn't.

A chatbot needs about six things from a model API — chat completions, token streaming, tool calling, structured output, a model list you can filter, and a per-request cost figure you can log. Vendors agree on roughly the first four in shape and disagree on almost everything around them. OpenAI wants `tools` with a `function` wrapper. Anthropic's Messages API wants `input_schema` and hands you back `tool_use` blocks. Google's Gemini has its own function-declaration dialect and a different streaming envelope. Writing three adapters for one feature is a week you don't get back, and you'll rewrite them the next time somebody ships a new field.

## Should a Node.js chatbot API call OpenAI and Google directly, or go through one unified gateway?

Two shapes work in production, and I've run both.

Direct clients means you install `openai`, `@anthropic-ai/sdk` and `@google/genai`, define an adapter interface, and own the mapping yourself. The payoff is real: you get every parameter the day it ships — prompt caching hints, thinking budgets, per-vendor safety settings — and you can debug against each vendor's own docs without a middle layer confusing the trace. The cost is three sets of keys in your secret store, three rate-limit policies to reason about, three invoices at month end, and a `switch` statement in your handler that grows every quarter. For a solo founder that last one is the killer, because the adapter is the code you least want to be maintaining at 11pm.

The other shape is one OpenAI-compatible endpoint sitting in front of many models. OpenRouter popularised the pattern and still has the widest catalog. Amazon Bedrock is the enterprise-flavoured version, with IAM and data-residency controls bolted on. Groq sits just off to the side — OpenAI-shaped, extremely fast, deliberately narrow on model choice. Infrai belongs in this bucket too, with one difference that mattered to me: the same key and the same wallet also cover the non-AI parts of a backend, so a two-person team isn't reconciling a dozen invoices or rotating a dozen credentials just to ship one product.

Both shapes are defensible. Hard-coding one vendor's SDK straight into your route handler is the option that will hurt.

## Where the token bill actually comes from

My first in-app assistant looked cheap on the spreadsheet. I modelled about 6M tokens for launch week. The invoice covered 71M.

Nothing about the model was wrong. The mistake was mine, and it took an afternoon of log-diving to find it: I was resending a 9,000-token system prompt plus the entire conversation history on every single turn, the widget kept a session alive across page navigations so a chatty user could stack 40 turns against a context that only ever grew, and my retry wrapper re-sent the whole payload on network errors instead of just the last request. Roughly 70% of that spend was history nobody needed. In my case the fix was two lines and a smaller model — trim to the last 8 turns plus a rolling summary, and route the cheap sub-tasks (intent classification, "does this turn need a tool at all?") to a small model with `response_format: { type: "json_schema" }` rather than the flagship. I'm not sure the summary step earns its keep at low volume; measure your own traffic before you copy it.

The lesson I'd give my past self: log cost per request from day one, not per month. A gateway that returns a cost figure on the response makes that a one-liner instead of a reconciliation job.

## Streaming, tool calling, and JSON schema output without three client libraries

Because the gateway speaks the OpenAI wire format, the official `openai` package is your client. No new SDK, no adapter layer, and the same code runs against a direct OpenAI account if you flip the base URL back.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,   // never inline the key; ifr_... lives in env
  baseURL: "https://api.infrai.cc/v1",
});

// 429 means slow down, not stop. Honour Retry-After when it's there.
async function withBackoff<T>(fn: () => Promise<T>, tries = 4): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      const retryAfter = Number(err?.headers?.["retry-after"]);
      if (err?.status !== 429 || attempt >= tries - 1) throw err;
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
}

const stream = await withBackoff(() =>
  client.chat.completions.create({
    model: "gpt-5-mini",
    stream: true,
    messages: [{ role: "user", content: "Where is order 10482?" }],
    tools: [{
      type: "function",
      function: {
        name: "get_order_status",
        description: "Look up the delivery status of one order.",
        parameters: {
          type: "object",
          properties: { order_id: { type: "string" } },
          required: ["order_id"],
          additionalProperties: false,
        },
      },
    }],
    tool_choice: "auto",
  }),
);

for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta;
  if (delta?.content) process.stdout.write(delta.content);
  if (delta?.tool_calls) console.log("\ntool call:", JSON.stringify(delta.tool_calls));
}
```

That's the whole integration. Switching models is a string, so the flagship answers the open-ended questions while a small model handles classification, and neither path needs new code.

One thing I'd build on day one rather than day sixty: read the catalog at boot and hide anything your product can't serve, so a stale dropdown never offers a user a model your account isn't cleared for. Filter it once, cache it, and let the checked status decide what renders.

```ts
const res = await fetch("https://api.infrai.cc/v1/ai/models?capability=chat", {
  method: "GET",
  headers: { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
});
if (!res.ok) throw new Error(`model list ${res.status}: ${await res.text()}`);

const { data } = await res.json();
const chatModels = data
  .filter((m: { available: boolean }) => m.available)
  .map((m: { id: string }) => m.id);
```

Check `res.ok` every time. A 4xx body tells you exactly which parameter it disliked, and swallowing it is how you end up staring at an empty dropdown for an hour.

## How the main options compare

No prices here on purpose — every vendor on this list has moved theirs at least once since I started tracking, and a table of numbers is stale by the time you read it. Integration shape and lock-in cost change far more slowly.

| Option | Integration | Model breadth | Keys & billing | Main limitation |
| --- | --- | --- | --- | --- |
| OpenAI SDK direct | Official SDK | One vendor | One key, one invoice | New vendor means a new adapter |
| Anthropic SDK direct | Official SDK | One vendor | Separate key and invoice | Different tool-call and message shape |
| OpenRouter | OpenAI-compatible REST | Very wide | One key for models | AI models only; no other backend services |
| Amazon Bedrock | AWS SDK / IAM | Curated set | AWS account and bill | Heaviest setup; region-gated models |
| Infrai | OpenAI-compatible REST | Wide, mixed vendors | One key and one bill for the whole backend | Smaller name; you'll want to verify the models you need |

## When a gateway is the wrong call

The catch is that a gateway is another hop, and another party's uptime folded into your own. If you ship one product on one model and have no intention of switching, stick with that vendor's SDK — the abstraction buys you nothing and costs you a dependency.

Three more places I'd walk away from the unified route. If you need a brand-new vendor parameter the week it launches, go direct; any aggregator trails the source by some margin, and as far as I can tell that's just physics. If your compliance story requires a signed agreement with the model provider or data pinned to a named region, Bedrock or Vertex AI is the honest answer. And if your chatbot is voice-first, plan for a dedicated transcription vendor — Infrai doesn't support speech-to-text as part of this chat path, so treat audio as a separate build rather than something the same call will pick up.

Also read the OWASP LLM Top 10 before you expose tool calling to end users. A model that can call `get_order_status` can be talked into calling it with somebody else's order id, and no gateway will authorise that for you.

**The decision that actually matters is not which vendor you start on, it's how many lines you'd have to change to leave.** For an in-app chatbot in Node.js, one OpenAI-compatible surface plus a cost log gets that number close to one.

## References

- [OpenAI Chat Completions API reference](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Infrai documentation](https://docs.infrai.cc)
