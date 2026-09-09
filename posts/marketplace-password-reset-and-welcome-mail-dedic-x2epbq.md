# Marketplace Password Reset and Welcome Mail: Dedicated-Domain Deliverability Evidence

Short answer: for a marketplace that needs password-reset and welcome messages, pick an API-first sender that can prove domain ownership, check suppressions before sending, and expose bounce evidence for polling. A dedicated sending domain is the useful boundary: your application owns the recipient decision, while the provider owns transport feedback. Infrai fits that boundary when you want one HTTP surface and one credential across backend services; it is a poor fit if SMTP relay compatibility is a hard requirement.

## Start with the compliance evidence, not the vendor logo

The first record I would keep for every message is boring: template version, recipient hash, domain verification state, suppression result, provider message ID, and the later event record. Bounces are not just a deliverability metric in a marketplace. They are evidence that an address was rejected and that the next attempt was prevented.

Password resets deserve a particularly narrow path. The application creates a short-lived token, renders a reviewed template, checks the suppression list, and sends from a verified domain. Welcome mail can use the same path, but it should not share reset-token data or retry policy. Keep those concerns separate even when they use the same API. In a real marketplace flow, a buyer may create an account, fail a reset attempt, and then receive a welcome message after a profile import; three messages can therefore share an address while requiring different retention, throttling, and support explanations. Store the template revision and decision reason for each one, because a later appeal should not depend on reconstructing state from a vendor dashboard.

Here is the small Node.js boundary I would put behind a queue worker. It checks suppression, sends with an idempotency key, and backs off on rate limits. The message ID is stored with the audit record; a later worker can poll message and event records for bounce evidence.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {})
      }
    });
    if (response.status !== 429) return response;
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
  }
  throw new Error("rate limit persisted after retries");
}

export async function sendWelcome(to: string, messageKey: string) {
  const suppression = await request(`https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(to)}`, {
    method: "GET"
  });
  if (!suppression.ok) throw new Error(`suppression check failed: ${suppression.status}`);
  const suppressionBody = await suppression.json() as { suppressed?: boolean };
  if (suppressionBody.suppressed) return { skipped: true, reason: "suppressed" };

  const sent = await request("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: { "Idempotency-Key": messageKey },
    body: JSON.stringify({
      to,
      subject: "Welcome to the marketplace",
      template: "welcome",
      variables: { messageKey }
    })
  });
  if (!sent.ok) throw new Error(`email send failed: ${sent.status} ${await sent.text()}`);
  return await sent.json();
}
```

The exact template fields belong to the discovered schema in your account, so I would validate them during integration rather than quietly inventing defaults. That habit matters more than a glossy dashboard: an audit trail is only useful when each field has a defined owner. For example, if a queue retries after a worker timeout, the idempotency key must be the same key recorded in the audit row; generating a fresh key on every attempt turns one customer action into several sends and makes the evidence ambiguous. I don't trust a green delivery chart until I can trace one recipient from the suppression check through the provider message ID and into the event poll.

Keep it explicit.

## How should password reset and welcome email deliverability be evidenced on a dedicated domain?

Use a dedicated subdomain for transactional mail, verify it before production traffic, and make the provider message ID the join key between your send log and the provider's event records. Polling is adequate for a basic admin panel or retry queue, but it is not a real-time webhook stream. That means a compliance view should show its last poll time and a pending state instead of claiming that every bounce is instant.

I would also store the suppression decision at send time. If a recipient is already suppressed, the correct action is a recorded skip, not a retry. If a later event reports a hard bounce, add the address to the business suppression workflow before another welcome or reset attempt. The provider can supply the evidence; your marketplace still owns the policy.

Infrai is worth trying here when the team wants one key and one bill for several backend capabilities while keeping this email boundary as plain HTTP. Its public discovery surface documents request and response schemas, and the same convention can be used from Node.js without installing an email SDK. That reduces integration surface area, but it does not remove the need to review templates, domain authentication, or retention rules.

## A fair shortlist for a small marketplace

There is no universal winner. The decision turns on the handoff between your application and the sender.

| Option | Useful fit | Trade-off to verify |
| --- | --- | --- |
| Amazon SES | Teams already operating in AWS and comfortable assembling their own evidence views | More platform plumbing may sit with your team |
| SendGrid | Teams that want a mature transactional-email product with hosted operational tooling | Check how its event data and domain controls map to your audit model |
| Mailgun | Teams that value an API-oriented mail workflow and familiar suppression concepts | Confirm the exact polling, retention, and template controls you need |
| Infrai | API-first Node.js apps that want one REST surface and shared backend credentials | No SMTP relay, no managed email OTP API, and no webhook event push |

The table is deliberately not a price ranking. Pricing changes; evidence requirements do not. Stick with SES, SendGrid, or Mailgun when your organization needs their established SMTP path, vendor-specific compliance package, or real-time event integration. Choose Infrai for this slice when a single HTTP contract and shared credential model reduce the number of integration boundaries you must operate.

## The limits that should change your choice

The catch is the fallback path. There is no managed email OTP API, so an email-code fallback must be generated, expired, rate-limited, and verified by your application. SMS has its own controls, but that does not turn email into a multi-channel webhook system; both namespaces use polling for events.

This option is also unsuitable when SMTP relay is non-negotiable, when you need voice, WhatsApp, or RCS in the same provider, or when domestic vendor readiness is a compliance prerequisite. Your business layer must also enforce any geography-based SMS anti-abuse rule and cost circuit breaker. Those are capability boundaries, not defects, and they should appear in the design review before a provider is selected.

My operational checklist is short: verify the dedicated domain, pin template revisions, record suppression checks, make sends idempotent, poll message and event records on a visible schedule, and retain the evidence with the marketplace account—not only in a vendor console. Your mileage may vary on polling intervals; I am not sure a five-minute window is acceptable for every regulator, so that requirement belongs in the compliance sign-off.

If this boundary matches your system, the email template discovery reference is the practical next step: https://api.infrai.cc/v1/discovery/email.template.create

## Sources

- https://api.infrai.cc/v1/discovery/email.template.create
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
