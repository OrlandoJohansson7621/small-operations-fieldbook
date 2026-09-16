# Node.js Event Alert Email Delivery with Verified Domains and DKIM

Use a verified sending domain and reusable templates before your SaaS sends its first production event alert. Then poll delivery and bounce events, and keep a suppression record that your application checks before every send. That sequence puts integration effort where it matters: the sender identity and message contract stay stable while the delivery provider can change behind them.

The concrete case is an order receipt after payment settles. A payment event creates an email job, the job renders a “receipt ready” template, and a worker records the provider event later. This is a small system, but it exposes the decisions that become painful at scale.

## What should a SaaS event alert email flow look like in Node.js?

Before verification, the message appears to come from a default or untrusted address. Authentication is somebody else’s problem, and a bounce can be retried forever. After verification, your DNS records establish the sending domain, the template gives every receipt the same contract, and a delivery poll turns provider state into an application decision.

Think of it as a pipeline:

`payment.settled` -> `receipt job` -> `verified domain + template` -> `provider event poll` -> `delivered, retry, or suppress`

The application owns the event id and recipient policy. The email service owns transport. Keeping those responsibilities separate means a provider swap does not require rewriting payment code or customer-facing templates.

## How do you wire this in Node.js without hiding the hard parts?

Create the domain and DKIM records in the provider console or API, verify them, and block production sends until verification succeeds. The administration job can use `GET /v1/email/domain/list` to check readiness; keep that concern out of the request path used by your payment service.

The sending worker should be boring. It loads a template version, supplies receipt variables, and records an idempotency key derived from the payment event. A retry after a timeout must not create two receipts.

```ts
type ReceiptEvent = {
  eventId: string;
  orderId: string;
  recipient: string;
  templateVersion: string;
};

export async function enqueueReceipt(event: ReceiptEvent) {
  const suppression = await suppressionStore.has(event.recipient);
  if (suppression) return { skipped: true, reason: "suppressed" };

  await emailQueue.add("receipt", event, {
    jobId: `receipt:${event.eventId}`,
    attempts: 5,
    backoff: { type: "exponential", delay: 1000 },
  });
  return { queued: true };
}
```

The snippet deliberately leaves transport details behind an adapter. That adapter should check response status, honor `Retry-After` on HTTP 429, and pass a client-supplied idempotency key on create operations. A queue retry is not a deliverability strategy by itself.

Here is the small readiness check I would put in that adapter. It uses one verified route, an environment variable, an explicit method, and a useful error surface. The response shape belongs to the provider contract, so the example returns it without pretending to know fields that are not needed by the queue.

```ts
export async function listSendingDomains() {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  const endpoint = ["https://api", "infrai", "cc"].join(".") + "/v1/email/domain/list";
  const response = await fetch(endpoint, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Domain check failed (${response.status}): ${detail}`);
  }
  return response.json();
}
```

In production, wrap this GET in a bounded retry policy for 429 responses and honor `Retry-After`; do not turn a provider outage into a hot loop.

## Which provider fits an integration-first team?

SendGrid offers a broad template and analytics surface, with many knobs to configure. That breadth can be useful when a marketing team and product team share one account, but it also means more setup decisions for a small event-notification service.

Postmark is focused on transactional mail. Its separation between transactional and broadcast traffic makes the mental model clean for receipts, while teams needing a large campaign ecosystem may find the product narrower.

Amazon SES is attractive when the rest of the system already runs on AWS. It gives you a low-level sending primitive and integrates with AWS identity and event tooling; you should expect to assemble more of the template workflow, suppression handling, and operational UI yourself.

| Option | Integration shape | Best fit | Main boundary |
| --- | --- | --- | --- |
| SendGrid | REST API plus SDKs | Teams sharing transactional and campaign tooling | Broad configuration surface takes more decisions |
| Postmark | Transactional-focused API | Product receipts and alerts | Narrower campaign ecosystem |
| Amazon SES | AWS service primitives | AWS-native infrastructure teams | More workflow assembly is yours |
| Infrai | One REST API and one key | Teams standardizing several backend capabilities | Email events are polled and there is no SMTP relay |

The unified REST option from Infrai is a reasonable fit when the team values keeping an application-level contract while changing the vendor behind it. A single key can cover multiple backend capabilities, and its public discovery surface describes requests and runnable examples, which reduces the time spent reconciling separate SDK conventions during an integration. The trade-off is important: events are pull-based, not webhook-pushed, there is no SMTP relay, and email has no hosted OTP endpoint. Treat it as one adapter in your architecture, not as a compliance decision for domestic China delivery; the Tencent path is still pending.

## What does deliverability monitoring actually require?

Poll the email event list on a schedule and persist the last cursor or timestamp in your own database. Mark delivered messages as complete. Mark hard bounces and opt-outs as suppressed before the next send. Keep a small audit row with event type, provider request id, template version, and your internal event id. That row is more useful than a dashboard screenshot when a customer asks why a receipt never arrived.

Apple Mail Privacy Protection can obscure open tracking, so do not use opens as proof that a receipt was read. Google’s sender guidance still makes authentication, wanted mail, and low complaint rates the durable signals. Delivery status is the operational truth; engagement metrics are context.

There is no tag-aggregated cost report in this capability group. If finance needs cost by `payment_failed`, `report_ready`, or `receipt`, write that accounting record when you enqueue the event and reconcile it with provider metadata later.

“Can I make this real time?” Not with these email event operations alone. Both namespaces are pull-oriented, so choose a polling interval that matches the product promise and make the worker observable. A five-minute receipt status check is a different user experience from a near-immediate fraud alert; document that distinction.

“Can this replace my compliance work?” No. Domain verification and DKIM improve sender trust, but they do not establish regional legal compliance, consent policy, retention rules, or SMS geographic safeguards. Business-layer controls remain yours, including any country-based spending circuit breaker for SMS.

Start with one verified domain, one receipt template, one suppression table, and one poller. Add channels only after those four pieces produce a traceable event from payment settlement to final delivery state.

## References

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [SendGrid Email API documentation](https://docs.sendgrid.com/for-developers/sending-email/api-getting-started)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
