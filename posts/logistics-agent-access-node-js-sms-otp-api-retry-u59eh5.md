# Logistics Agent Access: Node.js SMS OTP API Retry Boundaries

TL;DR: For a logistics support console, the simplest SMS passcode API is the one that owns code generation, expiry, one-time use, and attempt limits behind two small operations: start and check. Keep queue routing in your application, wrap the provider behind a narrow TypeScript interface, and reject repeated requests before they reach that interface. This reduces integration work without pretending SMS is a high-assurance authenticator.

A contact form makes this boundary concrete. A shipper submits a tracking issue, the application routes it to billing, pickup, customs, or delivery support, and an agent signs in before reading the case. The routing decision and the login challenge have different failure modes. Do not bury both inside a messaging callback or let a delivery response grant access.

The hard part is small.

## How should a Node.js SaaS login retry an SMS OTP API?

Count the security state your application must own, not the number of lines in a quickstart. An API that merely sends an SMS leaves you responsible for generating a code, storing a verifier, expiring it, limiting guesses, preventing reuse, and deciding what a resend does to the previous code. A managed verification API can own more of that state, but the application must still control eligibility, abuse limits, sessions, audit events, and the mapping from an authenticated agent to a support queue.

For this use case, I would score integration effort with five questions:

1. Does one call begin a challenge and one call check it, without exposing the code to application logs?
2. Are expiry, single use, and failed-attempt limits part of the verification boundary?
3. Can the adapter distinguish a rejected request, a temporary failure, and an unknown outcome?
4. Can regional policy be selected without spreading country rules through route handlers?
5. Can the implementation be replaced without changing sessions or queue-routing code?

NIST treats authentication through the public switched telephone network as a restricted out-of-band method. That is the right design posture: SMS can be a practical recovery or lower-assurance factor, but it should not silently become proof for sensitive administrative actions. The application should offer a stronger authenticator where account risk requires one.

This choice has a real limitation. SMS OTP is not suitable as the only factor for administrators who can export customer data, change payout details, or reset other agents. Use a phishing-resistant authenticator for those roles; keep SMS for the lower-risk support access the policy permits. The trade-off is extra enrollment work in return for a stronger authentication boundary.

No delivery API removes that trade-off.

## Put the boundary in code before comparing APIs

The adapter below is deliberately boring. It shows the contract that matters to the application and a small in-memory implementation for local testing. Production can bind the same interface to a managed verification service; the route handler does not need provider-specific status names or callback payloads.

```ts
import { createHmac, randomInt, randomUUID, timingSafeEqual } from "node:crypto";

type Region = "US" | "EU";
type Queue = "billing" | "pickup" | "customs" | "delivery";
type StartResult =
  | { kind: "started"; challengeId: string }
  | { kind: "limited"; retryAfterMs: number };
type CheckResult =
  | { kind: "approved" }
  | { kind: "denied" }
  | { kind: "expired" };

interface PasscodeGateway {
  start(input: { phoneE164: string; region: Region }): Promise<StartResult>;
  check(input: { challengeId: string; code: string }): Promise<CheckResult>;
}

type Challenge = {
  digest: Buffer;
  expiresAt: number;
  attemptsLeft: number;
  used: boolean;
};

class LocalPasscodeGateway implements PasscodeGateway {
  private readonly challenges = new Map<string, Challenge>();
  private readonly lastStart = new Map<string, number>();

  constructor(private readonly secret: string) {}

  async start(input: { phoneE164: string; region: Region }): Promise<StartResult> {
    const now = Date.now();
    const prior = this.lastStart.get(input.phoneE164) ?? 0;
    const retryAfterMs = 30_000 - (now - prior);
    if (retryAfterMs > 0) return { kind: "limited", retryAfterMs };

    const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
    const challengeId = randomUUID();
    this.challenges.set(challengeId, {
      digest: this.digest(challengeId, code),
      expiresAt: now + 5 * 60_000,
      attemptsLeft: 5,
      used: false,
    });
    this.lastStart.set(input.phoneE164, now);

    // A test harness captures this value; production sends it inside the adapter.
    process.stdout.write(`local code for ${challengeId}: ${code}\n`);
    return { kind: "started", challengeId };
  }

  async check(input: { challengeId: string; code: string }): Promise<CheckResult> {
    const challenge = this.challenges.get(input.challengeId);
    if (!challenge || challenge.used || challenge.attemptsLeft <= 0) {
      return { kind: "denied" };
    }
    if (Date.now() >= challenge.expiresAt) return { kind: "expired" };

    challenge.attemptsLeft -= 1;
    const candidate = this.digest(input.challengeId, input.code);
    if (!timingSafeEqual(candidate, challenge.digest)) return { kind: "denied" };

    challenge.used = true;
    return { kind: "approved" };
  }

  private digest(challengeId: string, code: string): Buffer {
    return createHmac("sha256", this.secret)
      .update(`${challengeId}:${code}`)
      .digest();
  }
}

function routeQueue(subject: string): Queue {
  const text = subject.toLowerCase();
  if (text.includes("invoice")) return "billing";
  if (text.includes("pickup")) return "pickup";
  if (text.includes("customs")) return "customs";
  return "delivery";
}
```

The local adapter uses a six-digit decimal code, a five-minute validity window, five checks, one-time acceptance, and a 30-second start interval. Those are explicit application choices for the example, not universal recommendations or claims about a vendor. NIST requires generated out-of-band secrets to contain at least six decimal digits, be accepted only once, and remain valid for no more than ten minutes; it also requires rate limiting when the secret has fewer than 64 bits of entropy. The shorter window and concrete attempt budget make the policy easy to test.

There is one intentional sharp edge: printing a test code is acceptable only in a local harness. A production adapter must never emit the code, full phone number, or reusable verification data to logs. Replace that line with the delivery call inside the adapter, not inside the HTTP handler.

## Retry transport, not verification decisions

Retries are where a superficially easy integration becomes ambiguous. If a start request is rejected before work begins, the application can return a wait time and avoid another send. If the network fails after the remote system may have accepted the request, an immediate blind retry can create two messages. Give each logical start an idempotency key when the selected API supports one; otherwise, hold the challenge in a pending state and let the user request another code only after the local interval.

Use exponential backoff with jitter for temporary transport failures. Cap it at the login screen's useful lifetime. Do not retry invalid phone numbers, policy rejections, exhausted attempt budgets, or incorrect passcodes. Those are decisions, not outages.

Checking a code needs an even stricter rule: only an explicit approved result creates a session. A timeout is unknown, not success. The UI can invite another check, while the backend keeps the same challenge identifier and never decrements a separate client-side counter as if that were authoritative.

Unknown means unknown.

This separation also keeps queue routing sane. The contact form may be accepted and routed while an agent's SMS is delayed; no case should move queues because authentication was retried. Store the chosen queue with the case, then authorize the signed-in agent against that queue after verification.

## Test the states that demos skip

A useful contract test suite runs against the local adapter and every production adapter. Start with the happy path, then spend more time on boundaries: a second check of an accepted code fails; the sixth guess fails under this example policy; an expired code cannot create a session; a rapid second start returns a retry interval; and a transport timeout never becomes approval. Use a fake clock so five minutes does not turn into a slow test.

Test US and EU configuration as data, even if both initially select the same adapter. Phone numbers should enter the boundary in E.164 form, but validation alone does not prove that a number is reachable, belongs to the user, or is safe from reassignment. Verification proves possession at that moment under the limits of the channel. Keep those meanings separate in variable names and audit events.

For the contact workflow, add cases that are easy to miss: an agent can see a delivery ticket only after authentication and queue authorization; a billing keyword cannot grant billing access; changing a case's queue produces an authorization check; and form text never appears in an SMS template. The latter prevents untrusted customer content from becoming an outbound message.

## Operate the flow without collecting secrets

Measure starts, approvals, denials, expirations, limited requests, and transport failures by region and adapter. Use opaque challenge IDs in events. A keyed hash of the normalized phone number can support abuse correlation without putting the number in ordinary logs, provided key access and retention are controlled. Avoid labels containing phone numbers because metrics systems replicate labels widely and high-cardinality data becomes expensive fast.

Watch ratios rather than a single delivery counter. A rise in starts with flat approvals can indicate abuse, poor delivery, or confused users; the metric alone cannot identify which one. Sample traces around the adapter boundary, retain the outcome category and duration, and investigate with the delivery evidence available from the selected API. Do not record the passcode to make debugging easier.

Email notifications for routed cases are a separate channel. If the system sends them, domain owners can use DMARC to publish policy and request aggregate or failure reports based on SPF and DKIM alignment. DMARC does not verify an SMS login and does not replace application authorization. Keeping those controls distinct prevents a successful email-domain check from being treated as proof of an agent's identity.

Ship with a short operational checklist written into the release review: confirm that secrets and phone numbers are absent from logs; confirm that only explicit approval creates a session; exercise expiry, reuse, resend, and timeout tests; inspect regional outcome ratios; and rehearse disabling SMS for a high-risk account without blocking support-case intake. Then review the boundary after launch. If changing providers requires edits to queue routing, session creation, or form parsing, the abstraction is already leaking.

## Further reading

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
