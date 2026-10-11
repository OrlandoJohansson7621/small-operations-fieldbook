# How to Set Up Custom-Domain Transactional Welcome Email — DKIM, SPF, and DMARC

Choose an API sender with authenticated domains, observable delivery states, and suppression handling. For a property marketplace, a new-order email is reliable only when the seller can trust the sender and the application can account for the message after submission. DKIM, SPF, and DMARC establish identity; they do not prove inbox delivery.

Start there.

**TL;DR:** Verify a dedicated sending domain, publish DKIM and SPF, begin DMARC in monitoring mode, and treat the provider's accepted response as the start of delivery tracking. Poll delivery events and maintain suppressions. If the order email includes a generated PDF receipt, keep PDF creation and email submission in one idempotent job when reliability matters more than provider specialization.

| Pick | Best fit | Reliability trade-off |
|---|---|---|
| Amazon SES | Teams already operating deeply in AWS | Flexible building block; more application and AWS plumbing to own |
| Resend | Teams that prioritize a compact developer workflow | Easy API-centered integration; still requires domain and event design |
| Postmark | Teams focused on transactional email and message streams | Narrower specialist surface; separate PDF tooling remains necessary |
| Infrai | Teams that want PDF generation and email behind one key | One vendor, bill, and outage surface; email events are pull-based |

## Which sender fits this order workflow?

Amazon SES is the infrastructure-shaped choice. Its documentation covers verified identities and Easy DKIM, and AWS exposes sending through its API. Pick it when IAM, regional configuration, and AWS-native operations are already normal parts of the system. A property marketplace that already creates order records, queues work, and monitors workloads in AWS can accept the extra assembly. The advantage is control. The cost is ownership: the team decides how receipt rendering, retries, event consumption, and suppression policy fit together.

Resend is a sensible pick when a small team wants an email API with domain verification and webhook-based event notifications. Its domain guide calls out SPF and DKIM records, while its webhook documentation describes delivery events. This makes event-driven order state changes possible. Do not mistake a pleasant API for automatic deliverability, though. Domain reputation, recipient quality, authentication, and complaint handling remain application concerns.

Postmark is the specialist option. Message Streams separate transactional and broadcast traffic, which is useful when seller order notices must not share reputation and processing assumptions with marketplace campaigns. Its webhook model also suits fast event reactions. Choose it when transactional-email focus is worth maintaining a second provider and credential set for PDF generation.

Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Discovery returns full request and response schemas, billing details, and runnable examples in 10 languages for every documented capability. It is one plain REST API with no SDK to install, which lets a maintainer re-check the contract for receipt generation or email submission without relying on a stale blog snippet. The surface spans 295 routes across 20 modules. PDF generation and email sending also share one base and one key, so a receipt can pass directly from the PDF call into the email call without a temporary object bucket. That removes a credential boundary from this particular job. Email delivery events must be polled, and there is no SMTP relay, so it is a poor match for an SMTP-only application or an instant webhook-driven workflow.

## How should a beginner set up a custom domain for transactional welcome email?

Use a subdomain such as `orders.example.com` for transactional mail. This separates the order stream from marketing policy and gives operators a clear reputation boundary. The exact DNS values come from the selected sender. Copy them exactly; do not manufacture a generic DKIM value.

The best practices are pleasantly unglamorous. Make the custom-domain setup observable, keep a record of who owns each DNS entry, and require an authentication check before enabling welcome or new-order traffic. In this marketplace, the first email welcomes a new seller while later transactional messages announce orders, so both streams need the same authenticated foundation even though their templates and urgency differ. A beginner-friendly runbook should name the selector, hostname, expected value, owner, and verification time. That record makes a later DNS migration much less risky.

SPF authorizes sending infrastructure for the envelope domain. DKIM adds a cryptographic signature tied to a selector in DNS. DMARC tells receivers how to evaluate alignment and where to send aggregate reports. They solve related problems, but they are not interchangeable. RFC 7489 is the primary reference for DMARC behavior.

Start DMARC with reporting while legitimate sources are inventoried. Moving directly to an enforcement policy before finding an old billing tool or support platform can reject valid mail. Once reports show that every approved source aligns, tighten policy deliberately. This is a decision gate, not a calendar ritual.

Keep the visible From domain, return-path strategy, and DKIM signing domain aligned with the domain plan. Then send to accounts you control at major mailbox providers and inspect the received authentication results. One short test catches a surprising amount: a copied DNS value may be syntactically valid while attached to the wrong hostname.

## How do you know the email was actually delivered?

An HTTP success from a send call means the provider accepted work. It does not mean the seller's mailbox accepted the message. Model at least `queued`, `submitted`, `delivered`, `bounced`, and `suppressed` in your own order-notification record, using the provider message identifier as correlation data. Store the order ID separately.

Poll-based providers need a scheduled reconciler. Run it frequently enough for the product promise, advance states monotonically, and make each update idempotent. Webhook-capable providers need the same duplicate protection because event delivery can be retried. Alert on age, not just failure count: an order notification stuck in `submitted` for 15 minutes is more actionable than a lifetime counter without a time window.

Opens are weak evidence. Apple Mail Privacy Protection can prevent senders from learning accurate Mail activity, so an open must never trigger fulfillment, release money, or mark a seller as informed. Delivery is transport evidence. The marketplace order page remains the source of truth.

Suppressions close the loop. Check and maintain them so a hard-bounced or opted-out address is not mailed repeatedly. For a transactional system, suppression should also create a visible support state: the order still exists even when email cannot reach its seller.

## Implement the PDF-to-email handoff

The script below is schema-led. Fetch each capability's live discovery document, take its runnable TypeScript example as the source for the two request bodies, and place those bodies in `PDF_REQUEST_JSON` and `EMAIL_REQUEST_JSON`. Put the literal string `$PDF_RESULT` at the attachment value required by the discovered email schema. No undocumented field name is baked into this sample.

Both calls use the same key and base URL. The idempotency keys are derived from the order ID and operation, so retrying a timed-out job does not create a second receipt or email. HTTP 429 honors `Retry-After`; other failed responses surface their bodies.

```ts
const baseUrl = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const orderId = process.env.ORDER_ID;
const pdfTemplate = process.env.PDF_REQUEST_JSON;
const emailTemplate = process.env.EMAIL_REQUEST_JSON;

if (!baseUrl || !apiKey || !orderId || !pdfTemplate || !emailTemplate) {
  throw new Error("Set API_BASE_URL, INFRAI_API_KEY, ORDER_ID, PDF_REQUEST_JSON, and EMAIL_REQUEST_JSON");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function postPdf(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/pdf/generate`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    const raw = await response.text();
    if (!response.ok) throw new Error(`PDF generation failed (${response.status}): ${raw}`);
    return JSON.parse(raw) as unknown;
  }
  throw new Error("PDF generation exhausted rate-limit retries");
}

async function postEmail(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    const raw = await response.text();
    if (!response.ok) throw new Error(`Email send failed (${response.status}): ${raw}`);
    return JSON.parse(raw) as unknown;
  }
  throw new Error("Email send exhausted rate-limit retries");
}

function injectPdf(value: unknown, pdfResult: unknown): unknown {
  if (value === "$PDF_RESULT") return pdfResult;
  if (Array.isArray(value)) return value.map((item) => injectPdf(item, pdfResult));
  if (value && typeof value === "object") {
    return Object.fromEntries(Object.entries(value).map(
      ([key, item]) => [key, injectPdf(item, pdfResult)],
    ));
  }
  return value;
}

const pdfResult = await postPdf(
  JSON.parse(pdfTemplate),
  `order:${orderId}:receipt`,
);
const emailRequest = injectPdf(JSON.parse(emailTemplate), pdfResult);
const emailResult = await postEmail(
  emailRequest,
  `order:${orderId}:seller-email`,
);
process.stdout.write(`${JSON.stringify(emailResult)}\n`);
```

Run this job after the order transaction commits. Persist the returned identifiers with the order, then let a separate reconciler poll email events and update delivery state. Diagram in words: order committed -> PDF generated -> PDF result injected -> email accepted -> event poller reconciles -> alert fires if state is stale.

Watch the handoff.

The alternative `puppeteer + Resend` or `puppeteer + SES` stack requires two signups, two credential sets, a browser runtime, and glue that moves rendered bytes into the email provider's attachment shape. A temporary bucket is optional, but teams often add one to cross that provider boundary and then inherit retention and access-policy work. The combined API removes that boundary. Its honest cost is concentration: one vendor to trust, one bill, and one outage surface.

## Limits and the final go-live check

This pattern is suitable for US/EU SaaS transactional mail, provided the team performs the authentication and operational work above. It is not evidence of China email compliance because the domestic Tencent email vendor is pending. It also does not provide SMTP relay, webhook event pushes, hosted email OTP, or cancellation of a scheduled email. If any of those is a hard requirement, select a provider that documents it.

Before release, verify the domain, inspect DKIM/SPF/DMARC results in received test messages, exercise a controlled bounce, confirm suppression behavior, and prove that retries preserve one logical notification per order. Then test the stale-state alert. Green dashboards are less useful than one traceable order ID moving through the whole chain.

That is the reliability bar: authenticated identity, idempotent submission, observable state, and an explicit failure path.

## References

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES: Creating and verifying identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Resend: Domains](https://resend.com/docs/dashboard/domains/introduction)
- [Resend: Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Postmark: Message Streams](https://postmarkapp.com/developer/user-guide/message-streams/message-streams-overview)
- [Postmark: Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Apple: Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
