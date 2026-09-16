# Transactional Email API: Node.js Direct Send or Outbox for SaaS Welcome Emails

Short answer: for a SaaS transactional email API handling welcome emails, choose a Node.js outbox setup when an order receipt must survive a provider timeout, deploy, or regional hiccup. A direct send is fine for a low-stakes confirmation where the payment workflow can safely retry. The boundary is delivery reliability, not the number of template fields.

After payment settles, commit the order and an email intent in one database transaction. A worker renders the versioned template, submits the message, and records the provider response. That separation keeps a provider outage from turning a successful charge into a failed checkout response.

## Should a SaaS welcome email API use direct send or an outbox?

The direct path has less code, but a timeout, restart, or duplicate webhook can produce a missing or duplicate receipt. The queue path adds a table and worker, making delivery an explicit state machine: pending, sending, sent, or permanently failed. I prefer that state for receipts because support can answer “what happened?” from records instead of logs.

It fails.

Keep it boring.

That extra machinery is a limitation. A tiny internal tool may accept direct-send loss during an outage; a regulated SaaS billing system should not.

```ts
type ReceiptJob = { id: string; orderId: string; recipient: string; templateVersion: string; attempts: number };
interface MailTransport { send(input: { to: string; template: string; idempotencyKey: string }): Promise<{ accepted: boolean; messageId?: string }> }
async function deliver(job: ReceiptJob, mail: MailTransport): Promise<"sent" | "retry" | "failed"> {
  const result = await mail.send({ to: job.recipient, template: `order-receipt-${job.templateVersion}`, idempotencyKey: `receipt:${job.orderId}` });
  if (result.accepted) return "sent";
  if (job.attempts < 5) return "retry";
  return "failed";
}
```

Store the idempotency key and provider message ID with the job; do not assume every transport enforces idempotency. Claim rows with a lease so workers cannot send the same job concurrently. Exponential backoff with jitter handles transient responses, while permanent mailbox rejection moves to review after the retry budget.

## How do custom domains and templates affect delivery?

A custom From domain is an authentication project. Publish SPF and DKIM records, then deploy DMARC in monitoring mode before tightening policy. RFC 7489 defines alignment between the visible From domain and authenticated mail. Verify alignment in real US and EU recipient mailboxes.

Pin a template version in the outbox row and render plain text and HTML. Include order number, amount, currency, and settlement timestamp from the committed record, never a later catalog lookup. A template editor without preview fixtures is a release risk.

SendGrid exposes dynamic templates and event webhooks; Mailgun emphasizes domain and routing controls; Postmark separates transactional streams from broadcast traffic. These are capability differences, not a ranking. Compare webhook retention, EU data-region choices, suppression behavior, and export format with your runbook.

| Approach | Setup surface | Best fit | Main limitation |
| --- | --- | --- | --- |
| Direct API call | Node.js REST or SDK | Informational welcome email | Timeout can hide acceptance |
| Database outbox | SQL table plus worker | Financial receipt | More operational code |
| Managed queue | Queue client plus worker | Multiple regions | Queue semantics vary |

Track settlement-to-enqueue, enqueue-to-acceptance, acceptance-to-delivery, and delivery-to-complaint clocks. Redact message bodies and payment details. Alert on pending age and deferred delivery, not only worker crashes.

Test provider timeout after acceptance, duplicate webhook, expired DNS authentication, template failure, and a full queue during deployment. Each case needs a replayable state transition. A dead-letter job should contain identifiers sufficient for a human to requeue it safely.

For a Node.js 20 service, I keep the worker process boring: one lease query, one render call, one transport call, and one state update. That narrow loop makes a five-attempt retry policy auditable, and it leaves room to swap an SDK for REST when a region or data-residency requirement changes.

Use direct send when a receipt is informational and the payment flow has an independent retry mechanism. Use an outbox when the receipt is part of the financial record, retries must be auditable, or US/EU routing and custom-domain authentication are contractual requirements.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/
- https://postmarkapp.com/developer
