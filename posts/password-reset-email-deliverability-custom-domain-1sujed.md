# Password Reset Email Deliverability: Custom Domain, DKIM, SPF, DMARC, Sender Warming

Short answer: give password-reset mail its own sending subdomain, authenticate it with DKIM and SPF, publish a monitoring-first DMARC policy, then warm the sender with small, observed batches. Keep transactional traffic separate from marketing traffic, and make suppression a first-class data path. The goal is a reset link that arrives quickly without teaching mailbox providers to distrust every message from your domain.

## The decision table: which control owns the risk?

| Control | Pick this when | What to measure |
| --- | --- | --- |
| Dedicated subdomain | A fintech contact form and password resets share a parent domain | Inbox placement and complaint rate by stream |
| DKIM signing | You need receiving servers to verify message integrity and domain alignment | Authentication results and selector rotation |
| SPF authorization | More than one system sends mail for the domain | SPF pass rate and DNS lookup budget |
| DMARC monitoring | You are still discovering every legitimate sender | Aggregate reports before enforcement |
| Suppression list | A recipient hard-bounced, complained, or opted out | Permanent suppression reason and timestamp |

This table is an ownership map. Your template team owns wording and link expiry; platform engineering owns DNS and signing; operations owns the suppression decision. That separation matters during an incident. A developer should not “fix” a bounce by retrying it forever.

## How should a Node.js password reset email use DKIM, SPF, and DMARC?

Treat the message as a small, observable transaction. The request handler creates a single-use token, stores only a hash with an expiry, and hands an idempotent job to a mail worker. The worker renders the template, sets a stable `From` address on the dedicated subdomain, and records the provider response ID. Never put the raw token in logs.

Here is the boundary in TypeScript. The mail transport can be an SMTP relay or an HTTP adapter; the application does not need to know which one.

```ts
type ResetJob = {
  to: string;
  token: string;
  expiresAt: Date;
};

export async function enqueueReset(job: ResetJob) {
  const message = {
    from: "Security Team <no-reply@notify.example.com>",
    to: job.to,
    subject: "Reset your password",
    text: `Use this link before ${job.expiresAt.toISOString()}.`,
    headers: { "List-Unsubscribe": "<mailto:abuse@example.com>" }
  };

  return mailQueue.add("password-reset", message, {
    jobId: `reset:${hashRecipient(job.to)}:${job.expiresAt.getTime()}`
  });
}
```

DKIM signs selected headers and the body with a private key; publish the matching public key at the selector record described by RFC 6376. SPF lists authorized sending hosts in DNS. DMARC evaluates alignment between those authenticated identities and the visible `From` domain. These are complementary checks, not three interchangeable switches.

Start DMARC with `p=none` and inspect aggregate reports. Move to quarantine or reject only after every legitimate sender is aligned. I have seen teams jump straight to enforcement, then spend a morning explaining why a support vendor's receipts vanished. The failure was configuration ownership, not the reset template.

## Warming and suppression: what should happen on day one?

Warming is a feedback loop, not a calendar ritual. Send a small cohort of real reset requests, watch delivery, deferrals, bounces, and complaints, and increase volume only when those signals remain healthy. A password-reset stream is naturally bursty, so document a cap and a queue policy for a login attack. Never generate artificial engagement to “train” a sender.

Suppression is stricter. A hard bounce or complaint should create a durable record keyed by the normalized recipient and stream. Check it before enqueueing, and check again in the worker because users can click twice while a job is in flight. Soft deferrals may be retried with exponential backoff; permanent failures should not.

Three words: measure the queue.

Useful telemetry joins the reset request ID to the mail job ID, provider response, SMTP status class, and eventual webhook event. Alert on a change in bounce or complaint rate, authentication failures, queue age, and the percentage of tokens redeemed after delivery. Do not treat an HTTP 202 from a relay as inbox placement.

## Testing, limits, and the handoff to production

Use a staging domain and mailbox matrix. Verify the rendered MIME, link expiry, plain-text fallback, and localization. Send signed messages to test inboxes, then inspect `Authentication-Results` for DKIM, SPF, and DMARC outcomes. Include retries and duplicate submissions in integration tests; a reset flow that sends twice under a timeout feels like a security bug to a customer.

Keep DNS changes in version control with an owner and rollback note. Rotate DKIM selectors without deleting the old public key until messages signed with it have aged out. Your mileage may vary across mailbox providers, and I’m not sure any single seed-inbox score predicts a real user's placement, so use production telemetry as the deciding evidence. Before launch, run a rehearsal that submits a reset from the contact form, follows the queue record through the worker, captures the relay response, waits for the webhook, and confirms that the suppression lookup and token-expiry paths produce the expected events; repeat it with a duplicate click and a forced timeout, because those are the cases that turn a healthy-looking dashboard into a customer-facing delay.

The catch is that this design is not suitable when the business requires one shared reputation for newsletters and security mail; use separate streams and ownership instead. DMARC reports are delayed and sampled, so they cannot prove that every reset arrived. SPF has a finite DNS lookup budget, and forwarding can break SPF even when DKIM survives. A self-hosted MTA gives control but transfers reputation management and on-call work to your team; a relay reduces that operational surface while adding a dependency and its own event semantics. Stick with a managed relay when your team cannot staff reputation monitoring, and choose self-hosting only when control justifies that on-call burden.

Measure first.

The practical rule is simple: choose the smallest sending system your team can observe, authenticate, and suppress correctly. Template ownership is part of deliverability, not a cosmetic afterthought.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://postmarkapp.com/guides/transactional-email-best-practices
