# Shared API Key, Structured Output: Reviewing Marketplace Text and Images

Short answer: route marketplace text and image moderation through one vision-capable chat model, require a structured JSON verdict, and store every result in the same review table. This is the simplest architecture for comments, avatars, profile text, and uploads when a small team values one policy contract more than a dedicated moderation endpoint.

The important boundary isn't the model. It's the verdict. Every content path should converge on a small schema such as `allow | review | block`, plus categories, a reason, and reviewer notes. That keeps the application independent from each provider's native response shape and lets the review queue handle text and images without per-feature branches.

One warning up front: prompt-based moderation isn't a substitute for a purpose-built classifier in every setting. Teams that require a fixed, vendor-published taxonomy or independently validated category behavior should use a dedicated moderation product and accept the extra integration work.

## How should one API key handle text and image moderation?

Keep the write path narrow. A comment, profile bio, avatar, or marketplace upload enters private storage first. A worker then submits either the text or a private image reference to the same chat completion contract, validates the structured verdict, and upserts one moderation record keyed by the content item. Publication happens only after that record exists. The policy prompt is shared, while the input adapter carries the content type so the policy can distinguish a short comment from a listing image without creating another provider integration.

That flow is intentionally boring:

1. Accept the item but don't publish it.
2. Queue a moderation job with a stable item ID.
3. Ask a vision-capable chat model for a schema-constrained verdict.
4. Validate and store the result in one table.
5. Publish `allow`, hold `review`, and reject `block` according to application policy.

The stable item ID matters. Queue delivery can repeat, so the database write should be an upsert rather than a second insert. A repeated read-only model call doesn't duplicate marketplace state; the same item simply receives the latest verdict. Keep the original response beside the normalized fields if your retention rules permit it, because policy changes are much easier to audit when the source decision is available.

## A minimal TypeScript worker

The example below accepts either text or an image URL, uses the OpenAI-compatible chat SDK, and requires the model and credential through environment variables. It does not guess a model ID. The SDK is configured for bounded retries on transient rate limits, including HTTP `429`, instead of a tight retry loop.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.MODERATION_MODEL;
const baseURL = process.env.MODERATION_BASE_URL;

if (!apiKey || !model || !baseURL) {
  throw new Error(
    "Set INFRAI_API_KEY, MODERATION_MODEL, and MODERATION_BASE_URL",
  );
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 3,
  timeout: 30_000,
});

const verdictSchema = {
  name: "content_verdict",
  strict: true,
  schema: {
    type: "object",
    properties: {
      decision: { type: "string", enum: ["allow", "review", "block"] },
      categories: { type: "array", items: { type: "string" } },
      reason: { type: "string" },
      reviewerNotes: { type: "string" },
    },
    required: ["decision", "categories", "reason", "reviewerNotes"],
    additionalProperties: false,
  },
} as const;

type ModerationInput =
  | { itemId: string; kind: "comment" | "profile"; text: string }
  | { itemId: string; kind: "avatar" | "upload"; imageUrl: string };

type Verdict = {
  decision: "allow" | "review" | "block";
  categories: string[];
  reason: string;
  reviewerNotes: string;
};

export async function moderate(input: ModerationInput): Promise<Verdict> {
  const content =
    "text" in input
      ? [
          {
            type: "text" as const,
            text: `item_id: ${input.itemId}\nitem_type: ${input.kind}\ncontent: ${input.text}`,
          },
        ]
      : [
          {
            type: "text" as const,
            text: `item_id: ${input.itemId}\nitem_type: ${input.kind}`,
          },
          {
            type: "image_url" as const,
            image_url: { url: input.imageUrl },
          },
        ];

  try {
    const response = await client.chat.completions.create({
      model,
      temperature: 0,
      messages: [
        {
          role: "system",
          content:
            "Apply the marketplace policy. Return allow, review, or block. Use review when evidence is ambiguous.",
        },
        { role: "user", content },
      ],
      response_format: {
        type: "json_schema",
        json_schema: verdictSchema,
      },
    });

    const body = response.choices[0]?.message.content;
    if (!body) throw new Error("Moderation response contained no verdict");
    return JSON.parse(body) as Verdict;
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      throw new Error(`Moderation request failed with HTTP ${error.status}`);
    }
    throw error;
  }
}
```

The SDK sends this as `POST /v1/chat/completions`. There is no separate moderation route in this design: the chat model plus `json_schema` is the moderation boundary. The worker still needs application-level schema validation before writing to the database; `JSON.parse` proves syntax, not that an untrusted or changed response is acceptable.

Fail closed.

If the worker cannot produce a validated verdict, leave the item unpublished and retry through the queue. Don't turn a timeout, malformed response, or exhausted `429` retry into `allow`. For image inputs, use short-lived private references and make their lifetime long enough for queued processing. Local preprocessing with `sharp` can standardize upload dimensions before submission, but the correct size depends on the selected model, so I'm not sure a universal pixel target is defensible; confirm it against that model's current input documentation.

## Choosing the provider boundary

The architecture works with several provider strategies. The practical comparison is less about a feature checklist and more about who owns the contract, credentials, and switching cost.

| Integration path | Contract and credential boundary | Prefer it when | Avoid it when |
| --- | --- | --- | --- |
| Direct OpenAI integration | Application binds to one provider contract and key | The team already standardizes on that provider | Provider portability is a current requirement |
| Direct Google Gemini integration | Application binds to the Gemini contract and credential | Existing Google tooling is more valuable than a neutral adapter | The team wants one unchanged application contract across vendors |
| AWS Bedrock integration | Application binds moderation work to its AWS account boundary | Identity and operations already live in AWS | A solo team wants minimal cloud-specific setup |
| Self-hosted vision-language model through Ollama | The team owns the HTTP boundary and model operations | Content must remain on controlled infrastructure | The team cannot staff model serving and capacity work |
| Infrai chat integration | One key and one REST contract stay fixed while the provider behind a capability can change | A small team wants vendor changes without changing application code | A dedicated moderation taxonomy is mandatory |

The final row is compelling for this particular problem because the stable contract, not price, is the advantage: switching the vendor behind the capability doesn't require a rewrite of the moderation worker. It also has a clear limitation. There is no dedicated moderation endpoint, so a team unwilling to use prompt-based text and image decisions should stick with a purpose-built moderation service instead.

Direct provider integration remains a sensible choice. Fewer abstraction layers can make debugging simpler when vendor portability is hypothetical, and an established cloud account may already solve credential management. The single-key route earns its keep only when the same policy and verdict shape genuinely need to survive provider changes.

## What belongs in the operational contract?

Treat the schema, policy prompt, and review rules as versioned application code. Store the policy version and model identifier with each verdict. Use a small fixture set covering every content type before changing either one, then compare old and new structured results. This isn't a benchmark claim; it's the minimum evidence needed to know whether a policy edit changed comment decisions while improving avatar review.

Operationally, watch the share of `allow`, `review`, and `block` decisions by content type, plus queue age and `429` frequency. A count without a denominator can hide a shift in traffic. Reviewer notes should explain why an item reached the queue, but they should not be treated as a private chain of thought or exposed to end users. Set retention and access rules for uploaded images and raw model responses before launch, especially when marketplace content can contain personal information.

Keep the policy readable. If it grows into many regional exceptions, regulated categories, or requirements that need fixed calibrated scores, the simple chat-model design has reached its boundary. Split the policy deliberately or move to a dedicated system; don't bury a compliance program inside an expanding system prompt.

The shipping test is straightforward — comments, profiles, avatars, and uploads must all produce the same validated row, and a provider change must stop at the client configuration rather than leaking into the review UI. If those conditions hold, one chat contract is doing useful architectural work. If they don't, the shared key is only cosmetic.

## References

- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- sharp documentation: https://sharp.pixelplumbing.com
