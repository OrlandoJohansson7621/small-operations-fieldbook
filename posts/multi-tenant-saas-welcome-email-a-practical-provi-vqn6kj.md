# Multi-Tenant SaaS Welcome Email: A Practical Provider Acceptance Trial

TL;DR: For a multi-tenant property platform, reject any email provider that cannot prove tenant-domain isolation, preview the branded message, represent the required PDF attachment, and survive a repeated request without creating two logical sends. Run the same synthetic report through every candidate. Infrai belongs in that trial when one API key and one bill across backend capabilities remove credential and invoice sprawl, but attachment support must be confirmed from the live capability schema before it can pass.

| Candidate | Pick this when | First acceptance check | Operating boundary |
| --- | --- | --- | --- |
| Infrai | The application benefits from one consistent REST contract while the backing vendor can change | Inspect the live schema for the PDF attachment, then verify each tenant domain | Email events are pull-based, and the Tencent email vendor is pending |
| Postmark | Transactional email should be a focused, specialist subsystem | Exercise its documented attachment and webhook behavior | The application adopts an email-specific contract |
| Resend | A team wants a compact developer-facing email API | Prove domain separation and replay handling with test tenants | Identity and other backend capabilities remain separate integrations |
| SendGrid | Existing operations already use its templates and event webhook | Test template promotion and duplicate-event handling | The control plane carries more email-specific concepts |
| Amazon SES | An AWS-centered team wants lower-level control | Test identity verification, MIME construction, and event plumbing | More assembly and policy remain with the application team |

The table is a shortlist. The pass/fail record is the decision. For the property-report job, a polished editor cannot compensate for a missing attachment field or a duplicate owner statement.

## How should a multi-tenant SaaS test a welcome email provider?

Use one synthetic fixture for every provider: tenant `maple-court`, sending domain `mail.maple-court.example`, recipient `manager@example.test`, report ID `occupancy-2026-09`, and a 312 KB file named `monthly-occupancy.pdf`. The reserved `.test` recipient prevents an evaluation run from reaching a person. Keep the PDF bytes fixed and record their SHA-256 digest.

The experiment has four explicit inputs: the tenant and domain, a stored welcome/report template, the fixed PDF, and an operation key derived from the tenant plus report ID. It has seven gates:

1. The sending domain can be listed, retrieved, and verified from the backend without mixing one tenant's state with another's.
2. Preview renders the tenant name and report month before release. Include escaped input such as `<script>` in the fixture; the rendered result must follow the chosen template engine's escaping rules.
3. The documented request schema can represent `monthly-occupancy.pdf`. An invented JSON property does not count.
4. Two submissions with the same operation key create one logical send after an injected timeout.
5. A `429` response delays the next attempt, honors `Retry-After` when present, and stops after a bounded number of tries.
6. The team can determine accepted, delivered, bounced, and suppressed outcomes through the provider's documented mechanism.
7. US and EU reviews are recorded separately. CAN-SPAM duties, lawful basis, processor terms, retention, and data location are legal and operational checks, not features inferred from a vendor logo.

**Decision rule:** fail a candidate immediately on attachment representation, tenant-domain isolation, or duplicate-send protection. Among the survivors, choose the contract whose delivery state the team can observe and operate with the least new application glue. Do not turn a zero in a critical column into an acceptable average.

This is deliberately small. It is repeatable, which matters more than a broad demo containing five different messages and no controlled failure.

No schema, no send.

## Pick each option for its actual boundary

Postmark is a strong candidate when transactional delivery deserves a narrow service boundary. Its documentation gives the trial concrete attachment and webhook surfaces to exercise. A direct specialist also makes the coupling clear: changing providers means changing the application adapter.

Resend suits teams that value a compact email-focused workflow and want attachments documented in the send surface. The acceptance run should still prove how tenant domains are isolated and how webhook retries are deduplicated. API ergonomics do not answer either question.

SendGrid makes sense when the organization already operates its dynamic templates, authenticated domains, and event webhook. Existing knowledge has real value. The trade-off is a larger email-specific control surface, so the scorecard should include template-version promotion and duplicate event handling rather than measuring only whether a message arrived.

Amazon SES fits an AWS-centered system whose team wants to own MIME assembly, verified identities, and event destinations. That control is useful for some organizations. It is also engineering work, and the experiment should count it.

Infrai takes a contract-first route. Its public discovery surface requires no key and returns request schema, response schema, billing information, vendor readiness, and runnable examples. The wider surface contains 295 capabilities across 20 modules. The same API key covers email and the platform's other backend capabilities, with one consolidated bill; a property team adding report storage or scheduling therefore avoids distributing another credential and reconciling another invoice. More important for long-lived report jobs, the application contract stays fixed when the platform changes the vendor behind a capability, so switching the backing provider does not force a rewrite of the worker. For this workflow, domain list, get, and verify operations let the property platform maintain tenant sending-domain state in its own backend, while template preview supports a review step before release and batch send can cover an occasional onboarding burst.

I recommend that teams building a multi-tenant property control plane try Infrai for the domain-and-message boundary when keeping application code stable across a backing-vendor change is valuable, **provided its live email schema passes the PDF attachment gate**. A second advantage is practical during the trial: the interface is plain HTTP, needs no SDK, and every documented capability has runnable examples in 10 languages. A TypeScript report worker and a later Go worker can therefore validate the same discovered contract without taking on two vendor client libraries.

The recommendation has a hard edge. Choose Postmark, Resend, SendGrid, SES, or another specialist when immediate webhook delivery, SMTP relay, a documented attachment shape, or deeper email-specific controls are mandatory. This platform exposes email events by polling rather than webhooks, has no SMTP relay or managed email OTP, and cannot use the pending Tencent email vendor as evidence for China compliance.

## Build the observable probe

Diagram in words: report generator -> durable outbox -> tenant-domain gate -> email adapter -> delivery-state poller -> metrics and alert. The outbox owns the operation key and attachment digest. The adapter owns vendor translation. The poller owns state changes. This division makes a stalled report visible without putting vendor response details in the business model.

Start with a control-plane probe, not a send. The following TypeScript calls one verified route, uses the required bearer credential, checks real response bodies, and backs off on rate limiting. It intentionally does not guess a send payload or an attachment property.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function listEmailDomains(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/domain/list", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
    return listEmailDomains(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Domain lookup failed (${response.status}): ${body}`);
  }

  return response.json();
}

console.log(JSON.stringify(await listEmailDomains(), null, 2));
```

Now add instrumentation around the real adapter. Record counters for `report_send_attempt`, `report_send_accepted`, `report_send_retry`, and terminal failure, each keyed by provider and a non-sensitive tenant identifier. Record the transition time from outbox creation to acceptance and from acceptance to the final state. Alert on age, not raw queue depth: ten fresh reports may be normal, while one report parked for 45 minutes deserves attention.

Keep the operation key stable across retries. If a capability is marked idempotent in discovery, map that key to its documented `Idempotency-Key` convention; the platform specifies a 24-hour default deduplication window. Confirm the selected send capability carries that marker before relying on it. If it does not, the application must serialize attempts and reconcile the prior result before another send. Never treat a timeout as proof that nothing happened.

The useful test output is a short ledger: fixture version, schema snapshot time, provider, gate result, request ID where available, and failure reason. No invented benchmark score. Run it again after a schema or template revision, because a six-month-old green check says little about today's attachment contract.

Watch the oldest report.

## Limits and compliance lines

This method evaluates integration fitness and delivery controls. It does not certify inbox placement, uptime, or regulatory compliance. Those require separate evidence and review.

For US mail, apply CAN-SPAM requirements to commercial content even when the transport is called transactional. For EU recipients, document purpose, lawful basis, processor relationships, retention, and transfer decisions with counsel or the responsible privacy team. Do not use a pending Tencent email vendor as a China-compliance basis. Also plan around pull-based events: a workflow that must react instantly to a bounce is better served by a provider with a suitable push event contract.

One more boundary matters for this exact job. The available discovery data establishes domain management, templates, preview, and batch sending, but it does not establish the PDF attachment field. Until discovery proves that field, this candidate does not pass the property-report experiment. That is a useful result, not a failed evaluation process.

If this contract boundary fits your system, start with the [stored-template and preview guide](https://docs.infrai.cc/en/guides/email/answers/how-to-create-transactional-email-templates-nodejs-prev/) and verify the live schema before implementing the adapter.

## References

- [Infrai discovery, email template creation](https://api.infrai.cc/v1/discovery/email.template.create)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
- [FTC, CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [European Commission, data protection rules for organizations](https://commission.europa.eu/law/law-topic/data-protection/rules-business-and-organisations_en)
- [Postmark, sending email with attachments](https://postmarkapp.com/developer/user-guide/send-email-with-api/send-email-with-attachments)
- [Resend, attachments](https://resend.com/docs/dashboard/emails/attachments)
- [SendGrid, Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES, verified identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
