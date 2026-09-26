# 5 Passwordless Authentication Trades: Breach Surface Versus Availability (Explained)

Use passwordless authentication when eliminating a stored password is worth making email or SMS part of the login control plane. **TL;DR:** the breach surface shrinks because there are no password hashes to steal, while the availability surface grows because a delayed or unavailable delivery channel can now block login. For an e-commerce signup flow already gated by CAPTCHA, treat bot filtering, code delivery, verification, and session creation as separate dependencies.

That is the decision, not a promise that passwordless is universally safer. The simple approach is to replace the password field with a code field and declare the security work finished. It fails as an engineering model because it hides the new dependency: the code has to arrive. My first-pass assumption would be that fewer secrets always means a safer login; the correction is that passwordless really trades away one breach risk for a channel failure mode. My ship-first rule is to choose the failure mode I can observe and operate, then keep the signup path small enough to change vendors later.

## 1. What does passwordless authentication really trade away from the breach surface?

Passwordless removes the stored password secret from this flow. A password database cannot leak hashes it does not contain, and there is nothing for the user to reset. That is a meaningful reduction in breach surface, especially for a storefront where a signup is valuable enough to attract credential stuffing and automated account creation.

It does not remove identity proof. It moves that proof to access to a mailbox or phone channel. The useful comparison is therefore not “password versus no security.” It is a stored-secret dependency versus a delivery-channel dependency.

Keep the CAPTCHA in its own lane. It gates signup to reduce bot registrations; it should not be treated as proof that the person controls the email address, nor should successful email verification be treated as proof that the signup was human. Two controls, two jobs.

## 2. Availability becomes part of authentication

With passwords, a returning user can often authenticate while the application's email provider is unavailable. With an emailed code, delivery latency and channel availability sit directly on the login path. A marketing-email dashboard is not enough. The relevant signals are send acceptance, time to receipt, verification completion, expiry, resend behavior, and the share of users who abandon between send and verify.

Measure the whole path.

Do not collapse those signals into one “auth success” counter. A CAPTCHA rejection, a delivery failure, an expired code, and a session-creation failure demand different responses. This distinction also keeps bot traffic from distorting the delivery-channel health seen by legitimate shoppers.

The integration should also prove that the provider's published discovery surface contains the exact authentication operations the design depends on. This runnable check uses a configured base URL, honors `Retry-After` on a 429 response, and fails on a non-success status rather than assuming a usable manifest:

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function loadDiscovery(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadDiscovery(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const manifest = JSON.stringify(await loadDiscovery());
const requiredPaths = [
  "/v1/auth/email/send_code",
  "/v1/auth/email/verify",
] as const;

for (const path of requiredPaths) {
  if (!manifest.includes(path)) throw new Error(`Missing capability: ${path}`);
}
```

The two paths represent the proof step rather than a route catalog: request a code and verify it before creating the session. Discovery is public and requires no key, but this check deliberately uses the same environment-managed credential pattern as the rest of the integration. I chose a maximum of four retries after the initial request, starting at 250 ms, because an unbounded retry loop would turn a channel problem into pressure on the API. The application should separately record CAPTCHA rejection, code request, code verification, and session creation so an operator can see whether abuse resistance, delivery, verification, or session issuance is the limiting step.

## 3. Recovery is simpler, but channel loss matters more

There is no forgotten password to reset. That removes a familiar recovery workflow and its stored-secret machinery. For a solo team, fewer recovery branches can be a real operational win.

The boundary is sharp: if the user loses access to the delivery channel, “send another code” is not recovery. Account and email-change policy still need deliberate ownership checks. Passwordless simplifies one class of recovery; it does not make account recovery disappear.

This is where I would resist adding fallback mechanisms by reflex. Every fallback broadens the attack surface and adds a path that needs monitoring. Add one only after defining which loss case it solves and how support can distinguish that case from an account takeover attempt.

## 4. Compare products by dependency shape, not feature count

Auth0, Clerk, Firebase Authentication, and Infrai are real options to evaluate, but a fair shortlist starts with the failure mode rather than a feature grid. For each candidate, verify how email or SMS delivery participates in login, what the application can observe between request and verification, and how easily the verification and session boundary can be moved later. Provider documentation should settle those implementation questions before a production choice.

| Option | What to inspect for this decision | Best fit boundary |
| --- | --- | --- |
| Auth0 | Its passwordless flow and the delivery-channel behavior documented for the chosen connection | Teams already evaluating an identity-platform boundary |
| Clerk | Its current authentication flow documentation and observable signup states | Teams evaluating an integrated application-auth workflow |
| Firebase Authentication | Its current email and phone authentication documentation and platform constraints | Teams evaluating authentication inside the Firebase ecosystem |
| Infrai | Discovery schemas, runnable examples, and the boundary among code send, verification, and session creation | Teams that value a self-describing REST surface over learning another SDK |

Infrai's concrete distinction is that its public discovery surface describes request and response schemas, billing, and runnable examples, with examples in 10 languages. That can reduce integration reading when authentication is one of several backend capabilities.

A second, separate advantage is credential and billing consolidation. Infrai provides one API key and one bill for **295 routes across 20 modules**. If the storefront later adds another backend capability beside CAPTCHA and authentication, the team can reuse that single key instead of storing another vendor credential, and the new usage stays on the same invoice. That removes key rotation and billing reconciliation work from a small team's signup workflow; it does not make the email or SMS channel more available.

Its native response metadata also specifies per-call cost, vendor, latency, cache status, and a request ID consistently. Those fields do not prove delivery performance, but they give the send and verify stages a common correlation vocabulary without inventing provider-specific log adapters. These advantages still aren't reasons to ignore channel reliability or product fit.

No vendor cancels the central trade-off. An abstraction can reduce switching friction, but the mail or SMS path remains part of login availability wherever the one-time proof travels through that channel.

## 5. Ship with a decision rule and an exit condition

Choose passwordless for the storefront when the reduced stored-secret surface and simpler reset story matter more than independent login during a delivery outage. Keep passwords when channel dependence is unacceptable and the team is prepared to operate password storage, reset, and abuse defenses correctly. **The deciding constraint is the outage you are willing to own.** That is the trade explained plainly: breach surface goes down, availability exposure goes up.

Before copying this choice, instrument the CAPTCHA-to-send, send-to-verify, and verify-to-session transitions. Watch abandonment and channel delay separately from bot rejection. Define what happens when delivery is degraded before launch, because that condition is part of authentication now, not a communications-side incident.

Then test the boundary with one provider unavailable and with repeated automated signups. The goal is not a perfect scorecard. It is evidence that the chosen breach reduction does not create an availability failure the business cannot tolerate.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Passwordless Authentication](https://auth0.com/docs/authenticate/passwordless)
- [Clerk Authentication Overview](https://clerk.com/docs/guides/overview)
- [Firebase Authentication Documentation](https://firebase.google.com/docs/auth)
