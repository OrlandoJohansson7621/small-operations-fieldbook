# Property Queue Template Ownership for SaaS Alert Emails and Domain Verification

| Template owner | Pick this when | Main control | Main risk |
|---|---|---|---|
| Platform team | Every property follows one support process | Central schema and release | Local routing nuance becomes a backlog |
| Queue team | Leasing, maintenance, and billing have distinct workflows | Queue-scoped templates | Shared fields drift between teams |
| Property operator | Each tenant must control wording or identity | Tenant-scoped drafts and approvals | Unreviewed content reaches production |

TL;DR: Build SaaS event alert emails around a clear template owner, then keep custom-domain DKIM verification, event contracts, and delivery telemetry at the platform boundary. For a property contact form, that usually means queue-owned content with a centrally enforced envelope. A maintenance team can change its instructions without gaining access to signing keys or rewriting retry behavior.

The diagram in words is short: contact form -> classified event -> queue template -> policy check -> authenticated sender -> delivery event. Store the immutable event and chosen template version before sending. That gives an operator something better than a vague complaint that an email "never arrived": a stable trail from form submission to transport outcome.

## How should a Node.js service build SaaS event alert emails?

Start with the consequence of a bad edit. If a platform engineer must approve a change to office hours, ownership is too centralized. If a property manager can alter the sender domain, routing keys, or required safety text, it is too loose. The useful boundary sits between content and transport.

A template is more than HTML. Treat its subject, plain-text body, HTML body, required variables, locale, queue, and revision as one reviewed artifact. The event contract stays separate. In this scenario, a `contact.received` event can carry a property ID, category, reply address, and message without knowing how maintenance phrases an after-hours response.

Do not let the template choose the recipient from arbitrary input. Resolve the destination from trusted property and queue configuration after classifying the form. Keep the visitor's address as validated data for the reply workflow, not as authority over routing. This is a design rule, not a mail-provider feature.

Keep that line firm.

## Pick the ownership model that matches the change boundary

Platform ownership fits a small operation with one support policy and infrequent edits. It minimizes variation. It also makes the platform team the editorial gate, so set an explicit review turnaround and expose template previews to the people answering the messages.

Queue ownership fits the common middle. Leasing can own leasing copy; maintenance can own escalation language. The platform still owns variable schemas, authentication state, rendering safety, and delivery telemetry. This split keeps operational decisions close to the queue while preserving one sending contract.

Tenant ownership fits properties with contractual brand or language requirements. Add draft, preview, approval, and rollback states before publication. Restrict editable fields. A tenant should be able to change a greeting and instructions without changing the authenticated `From` domain or inserting a new destination address.

The trade-off is plain: more local control produces more review surfaces. Choose it because the business boundary requires it, not because a template editor looks convenient. A tempting first design is to store one editable HTML blob per property and call the problem solved. It fails under ordinary change: the leasing queue adds a variable, maintenance still uses the old shape, a property duplicates the template, and nobody can tell which revision rendered an alert. The better unit is a published revision tied to a declared variable schema and one ownership scope. That costs more state up front. It buys a reviewable answer when a queue asks what residents actually received.

## Implement verification and rendering as explicit state

A custom sending domain should not become active merely because someone typed it into a settings page. Model the lifecycle with 3 states: `pending`, `verified`, or `disabled`. Verification reads the expected DNS record, compares the complete value, records the check time, and only then permits that domain in the sending envelope. DKIM signing belongs behind the transport interface; application code should never assemble private-key material into a template.

Here is a focused TypeScript sketch. The DNS challenge proves control for this application workflow. The transport remains responsible for producing properly authenticated mail, while the application refuses an unverified identity.

The challenge uses 24 random bytes.

```ts
import { randomBytes, timingSafeEqual } from "node:crypto";
import { resolveTxt } from "node:dns/promises";

type DomainState = {
  domain: string;
  challenge: string;
  status: "pending" | "verified" | "disabled";
  checkedAt?: string;
};

type ContactEvent = {
  id: string;
  propertyId: string;
  category: "leasing" | "maintenance" | "billing";
  replyTo: string;
  message: string;
};

type RenderedMail = {
  from: string;
  to: string;
  replyTo: string;
  subject: string;
  text: string;
  html: string;
  headers: Record<string, string>;
};

interface MailTransport {
  send(mail: RenderedMail): Promise<{ transportId: string }>;
}

export function issueDomainChallenge(domain: string): DomainState {
  return {
    domain: domain.toLowerCase(),
    challenge: randomBytes(24).toString("base64url"),
    status: "pending"
  };
}

export async function verifyDomain(state: DomainState): Promise<DomainState> {
  const answers = await resolveTxt(`_support-domain.${state.domain}`);
  const observed = answers.map(parts => parts.join(""));
  const expected = Buffer.from(`property-support=${state.challenge}`);
  const matched = observed.some(value => {
    const candidate = Buffer.from(value);
    return candidate.length === expected.length &&
      timingSafeEqual(candidate, expected);
  });

  return {
    ...state,
    status: matched ? "verified" : "pending",
    checkedAt: new Date().toISOString()
  };
}

export async function sendContactAlert(
  event: ContactEvent,
  domain: DomainState,
  transport: MailTransport,
  resolveQueue: (propertyId: string, category: ContactEvent["category"]) => string,
  render: (event: ContactEvent) => Omit<RenderedMail, "from" | "to" | "replyTo" | "headers">
): Promise<{ transportId: string }> {
  if (domain.status !== "verified") {
    throw new Error("Sending domain is not verified");
  }

  const content = render(event);
  return transport.send({
    ...content,
    from: `support@${domain.domain}`,
    to: resolveQueue(event.propertyId, event.category),
    replyTo: event.replyTo,
    headers: {
      "X-Event-Id": event.id,
      "X-Template-Scope": event.category
    }
  });
}
```

Keep rendering deterministic: the same event plus the same published template revision should produce the same subject and bodies. Escape untrusted form content in HTML, retain a plain-text alternative, and reject missing required variables before the transport call. Preview with hostile inputs too: an empty unit number, a long unbroken message, and text containing markup are more instructive than a perfect sample submission.

Retries need an idempotency boundary. Persist a send attempt keyed by event ID and notification purpose, then hand that stable identity to the transport adapter. A process restart must not silently turn one contact form into two queue alerts. Do not infer success from the absence of an exception; record the transport acknowledgement and later delivery events as separate facts.

One event. One intent.

## Observe the route, not just the send call

Use one correlation ID from form acceptance through classification, rendering, and transport callbacks. Emit structured events for `classified`, `rendered`, `accepted`, `delivered`, `deferred`, and `failed`, with the property ID, queue, template revision, and domain state. Avoid logging the visitor's message body or full address. Support text often contains access details and other sensitive material.

Alert on outcomes a support lead understands. A sudden rise in forms routed to an `unknown` category is a classifier or taxonomy problem. A verified domain moving to a blocked sending state is a configuration problem. An increasing delay between acceptance and delivery is a transport problem. One generic "email failed" counter collapses three different owners into one noisy page.

Open tracking is a weak success signal. Apple Mail Privacy Protection can download remote content in the background, so an image request does not reliably mean a person read the alert. For this workflow, measure queue acknowledgement or first response when the business needs an engagement signal. Delivery still matters, but delivery and human attention are different events.

Sender policy belongs in deployment checks. Google's sender guidelines describe authentication expectations and additional requirements for higher-volume senders. Validate the applicable SPF, DKIM, DMARC, alignment, DNS, and encrypted-transport conditions before enabling production traffic; do not wait for a deliverability incident to discover that a domain was only visually verified in the UI. Requirements can change, so make the linked primary guidance part of the operational review.

Test the boundary in layers. Unit tests lock template variables and escaping. Integration tests exercise DNS answers, domain state, queue resolution, and transport failures. A small production probe can verify that the authenticated path still accepts a message without using a resident's contact form as the probe. Keep that probe outside support analytics.

## Limits worth keeping visible

DNS verification confirms control of a challenge at a point in time; it does not approve template content, guarantee inbox placement, or prove that a queue will answer. DKIM authentication also does not replace routing controls or human review. Keep those claims separate in the interface and in runbooks.

This design deliberately stops before prescribing a provider. The durable pieces are the ownership boundary, published template revision, explicit domain state, stable event identity, and outcome telemetry. Those survive a transport change. More important, they tell each team exactly which failure it owns.

## References

- https://support.google.com/a/answer/81126
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
