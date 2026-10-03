# Secure Logistics Seller Password Reset: Hashed Email Tokens, Expiry, and Abuse Limits

A secure password reset is an application security workflow with email at its final boundary. For a logistics marketplace, the practical goal is to restore a seller's access before a new order stalls, while retaining evidence that each recovery link expired, worked once, and was not issued without abuse controls. **Short answer:** generate 32 random bytes, store only a SHA-256 digest with a short expiry, consume it atomically, and make every public response indistinguishable. Use an email provider only to deliver the link.

The audit trail matters more than the provider logo. Record request acceptance, a pseudonymous account reference, token expiry, consumption, delivery identifier, and delivery state. Never record the raw token or password. This division is useful during an incident review: the database proves what the application authorized, while provider events show what happened after handoff.

Keep that boundary hard.

## How should Node.js build a secure password reset flow?

Start with four state transitions: requested, issued, consumed, and expired. A request may be accepted even when no account exists; that is how the endpoint avoids confirming which seller emails are registered. Rate limits belong at both the IP and account-reference levels, with thresholds chosen from the marketplace's own traffic and risk model rather than copied from a tutorial.

The reset record should contain a token digest, user ID, creation time, expiry time, consumed time, and a request correlation ID. Store the digest under a unique constraint. On redemption, hash the presented token and perform one database transaction that locks the matching row, rejects expired or consumed rows, updates the password hash, marks the token consumed, and invalidates the user's other outstanding reset records. Commit before reporting success. That transaction is the single-use guarantee; deleting a row in a later callback is not.

Keep the URL sparse. It needs an opaque token and a fixed application route, not the seller's email, order number, carrier data, or any other sensitive identifier. Set a short expiry according to the actual support and threat model, and make the page exchange the token before loading third-party analytics that could receive a referrer.

No identity leaks.

## A runnable application boundary

The following TypeScript keeps security state in the application and treats delivery as a replaceable adapter. The repository methods deliberately express the required atomic operations rather than hiding them behind a provider callback. The send request uses a client-supplied idempotency key, reports real response bodies, and backs off on 429 responses.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type ResetRepository = {
  save(input: { userId: string; digest: string; expiresAt: Date }): Promise<void>;
  consumeAndChangePassword(input: {
    digest: string;
    passwordHash: string;
    now: Date;
  }): Promise<boolean>;
};

const apiBase = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const appOrigin = process.env.APP_ORIGIN;
if (!apiBase || !apiKey || !appOrigin) {
  throw new Error("Missing API_BASE_URL, INFRAI_API_KEY, or APP_ORIGIN");
}

const digest = (token: string) =>
  createHash("sha256").update(token).digest("hex");

async function post(
  path: string,
  body: unknown,
  idempotencyKey: string
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(body)
    });
    if (response.ok) return response.json();

    const detail = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`${path} failed (${response.status}): ${detail}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Retry budget exhausted");
}

export async function issueReset(
  repo: ResetRepository,
  user: { id: string; email: string },
  emailRequest: (resetUrl: string) => Record<string, unknown>
): Promise<void> {
  const token = randomBytes(32).toString("base64url");
  const expiresAt = new Date(Date.now() + 15 * 60_000);
  await repo.save({ userId: user.id, digest: digest(token), expiresAt });

  const resetUrl = new URL("/account/reset", appOrigin);
  resetUrl.searchParams.set("token", token);
  await post("/email/send", emailRequest(resetUrl.toString()), randomUUID());
}

export async function redeemReset(
  repo: ResetRepository,
  token: string,
  passwordHash: string
): Promise<boolean> {
  return repo.consumeAndChangePassword({
    digest: digest(token),
    passwordHash,
    now: new Date()
  });
}
```

`emailRequest` is injected because the verified provider schema, not guessed field names, should define the request body. Validate that object against the discovery schema during integration. The 15-minute value is an explicit product decision in this example, not a universal security standard. Change it only with a documented reason.

Retries are dangerous here.

The public request handler should return the same status, body shape, and broadly similar timing for known and unknown addresses. Internally, it can skip token creation for a missing account. Do not reveal that branch to the caller.

## Delivery choice is a compliance choice

Resend offers a focused developer-facing email API and documented SDK workflow. Amazon SES fits teams already operating inside AWS and needing its identity, sending, and event ecosystem. SendGrid and Postmark are mature transactional email choices with their own event and suppression models. None of them should own token validity or one-time consumption; those controls stay in the application database.

Infrai is a reasonable option when a small team values a self-describing REST surface: public discovery returns request and response schemas plus runnable examples, so adding a capability does not require learning another SDK. Its idempotency convention also supports safer retries. The trade-off is operational concentration: one vendor, one bill, and one outage surface. Email events are pull-only, with no webhook callbacks, so a support view that needs delivery state must poll `/email/event/list`. It also has no SMTP relay or hosted email OTP interface.

| Option | Best fit | Compliance evidence boundary | Material constraint |
| --- | --- | --- | --- |
| Resend | Teams wanting a narrow transactional email API | Application ledger plus provider delivery records | Another credential and integration if PDF generation lives elsewhere |
| Amazon SES | AWS-centered systems | Application ledger plus AWS event plumbing | More cloud configuration and IAM surface |
| SendGrid | Teams using a broad email platform | Application ledger plus provider events and suppressions | Provider-specific integration and account |
| Postmark | Transactional-email-focused workloads | Application ledger plus message events | Separate tooling for document rendering |
| Infrai | Small teams combining backend capabilities behind one key | Application ledger plus polled email events | No email webhooks; broader vendor concentration |

This is not a pricing decision. It is a decision about which evidence can be produced, how quickly support can see it, and how much provider-specific glue the team accepts.

My decision rule is blunt: choose the narrow email specialist when its event model and team familiarity produce clearer evidence; choose the combined surface when removing credential and handoff boundaries matters more than real-time callbacks. For a solo builder, I would accept polling only when support can tolerate a documented delay and the application ledger remains authoritative. I would not trade atomic token consumption for provider convenience. The provider can prove acceptance, delivery progression, or suppression; only the application transaction can prove that two racing requests did not both change the password. That division also keeps a later provider migration contained. The email adapter changes, but the digest, expiry, enumeration defense, and consumption transaction do not.

## One key across the document-to-email handoff

After recovery, the same logistics marketplace may need to notify the seller of a waiting order and attach its packing slip. A combined API can generate the PDF through `/pdf/generate` and feed that result into `/email/send` with the same bearer key and base URL. The exact request and response objects should be generated from each capability's discovery schema; inventing attachment fields is worse than a little integration code. Keep order data out of the reset link, and treat the order notice as a separate authenticated workflow.

With Puppeteer plus Resend, the team manages a browser runtime, one external provider signup, one API credential set, PDF bytes, attachment encoding, retries, and correlation between two systems. Puppeteer plus SES replaces that signup with an AWS account and AWS credentials but leaves the browser and glue code. The combined route removes the temporary-bucket transfer between the renderer and mail service. It also concentrates trust and failure impact in one provider, which belongs in the architecture record.

There is a practical boundary here. The verified schemas must determine how PDF output becomes an attachment; do not assume a public URL, and do not expose the generated document through a public bucket. For a recovery email, skip the attachment entirely. For an authenticated order notice, retain only the minimum document metadata needed for support and compliance.

## Operations complete the control

Poll delivery events when support needs to distinguish an expired link from a bounced message. Because callbacks are unavailable, polling interval and freshness should be stated in the runbook; a dashboard must not imply real-time delivery evidence. Suppression checks, SPF configuration, and domain authentication improve mail handling, but they cannot prove that a reset token was single-use.

Before release, exercise the ugly paths: two concurrent redemptions, a token one millisecond past expiry, repeated requests for one address from many IPs, repeated requests from one IP across many addresses, delivery throttling, and a retry after an uncertain network result. Confirm that only one concurrent redemption commits, every send retry reuses its idempotency key, logs never contain the raw token, and support can join an application correlation ID to the delivery identifier.

One final check is easy to miss. Changing the password should revoke active sessions according to the marketplace's security policy and trigger a neutral security notice, without including the new password or the reset token. The reset is finished only when the old credential and stale recovery links can no longer open the seller account.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Node.js crypto documentation](https://nodejs.org/api/crypto.html)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
