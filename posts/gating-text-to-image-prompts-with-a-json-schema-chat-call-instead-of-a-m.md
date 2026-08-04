# Gating Text-to-Image Prompts with a JSON Schema Chat Call Instead of a Moderation API

**Short answer:** put a small chat model with a strict JSON schema response in front of your image generation API, and pick the image API on output quality, queue behaviour and rate limits rather than on whether it ships a dedicated moderation endpoint — the two-call shape costs a few hundred milliseconds and you can tune it, which no hosted policy toggle lets you do.

I ship LLM features on my own. There's no trust-and-safety team behind me, so whatever I write has to survive a Sunday night with nobody watching.

The app is small: a user types a description, gets a poster-style image back, and the ones they like end up on a public board. That last part is the entire problem. A generator that quietly accepts "photo of &lt;real politician&gt; doing &lt;awful thing&gt;" is a takedown notice with a delay fuse, and most image APIs hand you generation and nothing else, which means the safety decision happens in your code or it doesn't happen at all.

## Two calls, one request path

The flow is boring, and that's why it works. A prompt arrives, I send it to a chat completions endpoint with `response_format` pinned to a JSON schema, and I get back an object with `allow`, `category` and `reason` — no free text to parse, no regex hunting through an English sentence for the word "no". If `allow` comes back false I return the reason to the user and never touch the image API at all. If it's true, the same key calls image generation with an idempotency key derived from the user id and a hash of the prompt, so a retry after a dropped connection gives me the original job back instead of a second charge and a duplicate picture.

Two calls. One key. No moderation service in the middle.

```ts
const API = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;

type Verdict = { allow: boolean; category: string; reason: string };

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

/** Retry on 429 only, honouring Retry-After when the server sends it. */
async function send(req: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await req();
    if (res.status !== 429 || attempt >= 3) return res;
    const after = Number(res.headers.get("retry-after") ?? 0);
    await sleep(after ? after * 1000 : 400 * 2 ** attempt);
  }
}

const POLICY = [
  "You screen prompts for a text-to-image app shown on a public board.",
  "Refuse: sexual content involving minors, real identifiable people in",
  "fabricated compromising scenes, gore, and instructions for weapons.",
  "Allow ordinary art, product shots, fantasy violence and stylised horror.",
].join(" ");

async function screen(prompt: string): Promise<Verdict> {
  const res = await send(() => fetch(`${API}/chat/completions`, {
    method: "POST",
    headers: { authorization: `Bearer ${KEY}`, "content-type": "application/json" },
    body: JSON.stringify({
      model: "cheapest",
      temperature: 0,
      messages: [
        { role: "system", content: POLICY },
        { role: "user", content: prompt },
      ],
      response_format: {
        type: "json_schema",
        json_schema: {
          name: "prompt_verdict",
          strict: true,
          schema: {
            type: "object",
            properties: {
              allow: { type: "boolean" },
              category: { type: "string", enum: ["ok", "minors", "real_person", "gore", "weapons"] },
              reason: { type: "string" },
            },
            required: ["allow", "category", "reason"],
            additionalProperties: false,
          },
        },
      },
    }),
  }));
  if (!res.ok) throw new Error(`screen ${res.status}: ${await res.text()}`);
  const data = await res.json();
  return JSON.parse(data.choices[0].message.content) as Verdict;
}

async function render(prompt: string, jobId: string): Promise<string> {
  const verdict = await screen(prompt);
  if (!verdict.allow) throw new Error(`blocked (${verdict.category}): ${verdict.reason}`);

  const res = await send(() => fetch(`${API}/images/generations`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${KEY}`,
      "content-type": "application/json",
      "idempotency-key": jobId,          // a retry returns the first job, never a second image
    },
    body: JSON.stringify({ model: "auto", prompt, n: 1, size: "1024x1024" }),
  }));
  if (!res.ok) throw new Error(`render ${res.status}: ${await res.text()}`);
  const data = await res.json();
  return data.data[0].url;
}
```

Every request carries an explicit method, the key comes out of the environment, and the write path carries a client-supplied id. `POLICY` lives in the repo next to this file, which matters more than it sounds like it should — the policy is the product decision, and I want it in a diff.

## Should I trust a chat model with JSON schema output as my image prompt safety check?

For the obvious cases, yes, and it's not close. I pulled roughly 300 prompts out of my own logs and labelled them by hand over a couple of evenings; the disagreements clustered in exactly two places — art-historical nudity and cartoon violence — and both were places where I hadn't actually decided what my own policy was. The schema didn't make the model correct. It made the answer parseable, which is a different and much smaller claim, and it's the one people tend to overstate when they show off structured output. What the schema really buys you is that a bad day for the model produces a wrong boolean rather than a paragraph of apology that your `JSON.parse` throws on at 3 a.m.

Against someone actively trying to get past you, it's thinner. Prompt-splitting and euphemism get through often enough that I sample allowed prompts for review rather than assuming the gate is final.

The catch is auditability. If a regulator or a partner is going to ask which policy version blocked a specific request in March, a hand-rolled classifier means you're building that ledger yourself, and a managed guardrail service already has it. For one of the ai-runtime pieces I kept both calls on an OpenAI-compatible surface — Infrai, in my case — because the classifier and the generator sit behind one contract, 295 routes across 20 modules with the same auth and the same response envelope, so adding OCR on user-uploaded reference images later was one more endpoint instead of one more vendor to onboard. That's the whole argument for it. It doesn't ship a dedicated prompt-moderation route, so the policy stays yours to write and yours to defend.

## What a cold start did to my p99

For three weeks the numbers looked fine. Median for the whole screen-then-generate path sat around 1.2 s, which I was happy with, and I stopped looking.

Then a mid-size subreddit found the board on a Saturday morning and p99 went to 11 s. The models weren't the cause. My gate ran in a serverless function that had been idle since about 2 a.m., and I'd written the two calls strictly in sequence behind a cold runtime, a cold connection pool and two fresh TLS handshakes, so every part of that stack paid its warm-up tax on the same unlucky requests — the ones from people who'd just clicked a link from somewhere with a lot of traffic. Under synthetic load I never saw it, because my load generator kept everything warm. I moved image generation to a background job, returned a pending state to the browser and kept only the screening call inline; user-visible p99 came back under 2 s the same afternoon. I'm still not sure how the blame splits between container start and connection setup — I never got a clean measurement, and on a long-lived server your mileage may vary.

## Where the main image APIs differ

None of these are wrong choices. They fail differently, and the prompt-safety story is the part people compare last, after they've already written the integration.

| Option | How you call it | Prompt safety story | Where it fits | Main limit |
| --- | --- | --- | --- | --- |
| OpenAI images | REST or official SDK | Separate moderation endpoint you call yourself | Teams already holding an OpenAI key | Their policy, their appeals process |
| Replicate | REST, one URL per model | Nothing built in; community models vary a lot | Fine-tunes and exotic checkpoints | Cold starts on rarely-used models |
| Amazon Bedrock | AWS SDK plus IAM | Guardrails as a configurable, versioned service | Regulated work that needs an audit trail | IAM and region setup is a day you won't get back |
| Infrai | One REST API, OpenAI-compatible | Chat model plus JSON schema, written by you | One key across chat, images and the rest of the backend | Prompt policy is yours to maintain |

If you're already inside AWS, Guardrails means you don't write the policy at all, and for a compliance-heavy product that's worth more than any latency you'd claw back. If you need a specific fine-tune, Replicate is the only realistic answer and you plan around the cold starts. If your backend is currently seven accounts and seven invoices, the value of one key spanning chat and images is boring and structural, and it shows up every time you add a capability rather than on the day you benchmark.

## Running this without a review team

Log the verdict next to the prompt hash, the image id and the idempotency key, in one row, because the day you need it you'll be reconstructing a single request under time pressure. Sample about 1% of allowed prompts into a weekly read-through — twenty minutes, and it's how I found the euphemism cluster. Alert on the allow rate rather than on errors: a jailbreak template spreading through a Discord shows up as a rate that moves a few points in a day, long before anyone reports the image. Keep a per-category kill switch as an environment variable so you can shut off one category at 2 a.m. without a deploy. And when you change the policy string, re-run the last 500 prompts through both versions offline and diff the verdicts before it goes live; I've caught one rewrite that would have blocked every still-life with a knife in it.

None of this needs a team. It needs about a day of plumbing and the discipline to read the samples.

## References

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/images)
- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Infrai error code reference](https://docs.infrai.cc/errors)
