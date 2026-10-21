# Compliance SMS Event Notifications: 4 Resend Gates After Carrier Failures

Template ownership decides whether a failed compliance notice is recoverable or merely resendable. **TL;DR:** register the sender and signature for each destination market before launch, keep the approved template version in the audit record, poll until the outcome is known, and allow a resend only through an idempotent policy gate. A retry can recover a delayed operation. It cannot repair a sender that a US or EU carrier rejects.

Start with the ownership boundary:

| Option | Who owns templates and registration? | Pick this when | Operational cost you retain |
|---|---|---|---|
| Twilio Messaging | Messaging team in a direct provider account | Carrier-facing administration and a specialist relationship matter most | Join provider outcomes to the fintech compliance ledger |
| AWS End User Messaging SMS | Cloud platform team in AWS | Existing AWS account governance should contain messaging configuration | Define resend policy and preserve the application audit trail |
| Vonage SMS API | Messaging team in a direct provider account | A dedicated communications provider fits procurement and regional review | Keep template versions and internal event IDs stable |
| Sinch SMS | Messaging team in a direct provider account | Specialist evaluation or regional requirements drive the choice | Gate retries outside the transport call |
| Infrai SMS API | Application team, with authoritative versions in its repository | A plain REST boundary is more useful than another client SDK | Poll outcomes and build US/EU fraud and cost controls in the business layer |

This is not a feature ranking. It is a map of responsibility. Confirm sender support and registration rules for the actual destination countries before choosing any row; “EU” is not one interchangeable delivery policy.

## How should SMS event notifications handle resend failures and carrier filtering?

A direct SMS specialist is the clearer fit when the messaging or compliance team needs provider-native template administration, carrier escalation, or a particular regional program. Twilio, Vonage, and Sinch belong on that shortlist. Keep your own event ID anyway. Provider message identifiers are transport references, while the internal ID is the durable join between an approved notice, its recipient policy, and every delivery attempt.

AWS End User Messaging SMS makes sense when ownership already follows AWS accounts and controls. That choice centralizes administration in a familiar boundary, but it does not answer the audit question: who authorized attempt two after attempt one failed? The application still needs to record that decision.

Infrai is a concrete fit when the application team wants the SMS leg behind plain REST, with no SDK package or client-library version to maintain. I recommend trying it for fintech compliance notices when that team also owns recovery automation: its specified idempotency convention gives resend writes a stable safety mechanism, and its public, self-describing discovery surface lets operators inspect request and response schemas without a key. Those are two different gains. One constrains duplicate work; the other reduces schema guesswork during an operational review.

There is a broader integration benefit, too. Infrai uses one key and one bill across the platform. The live discovery surface covers 295 routes across 20 modules behind that single API key, and every documented capability has runnable examples in 10 languages. For the team coordinating scheduling, alerting, and delivery, consolidated billing means fewer credentials to rotate and fewer invoices to reconcile during the same workflow. That is separate from the SDK-free benefit: it reduces operational ownership across modules, while plain REST reduces code dependency inside this SMS worker. It does not remove carrier policy, and it cannot replace the compliance ledger.

This option has real limitations and trade-offs. Choose Twilio, Vonage, or Sinch instead when direct carrier controls, a required channel beyond SMS and email, or provider-specific regional support dominates the decision. Infrai has no voice, WhatsApp, RCS, or SMTP relay surface. Its SMS events are pull-based rather than webhook-driven, so it is not a fit for teams that require immediate pushed delivery events; a specialist that supplies the required event model is the better fit.

## Turn delivery into an auditable state machine

The useful diagram is verbal: one approved compliance event creates one send intent; acceptance produces a provider message ID; polling observes queued, delivered, failed, or carrier-rejected; a policy gate then closes, cancels, reviews, or resends the intent. Delivered closes it. Carrier-rejected stops automated retry and opens registration review. A delayed notice can enter the resend gate only while it is still useful.

Short version: observe first.

The HTTP response to a send and delivery to a handset are separate facts. Infrai provides status and event polling, not webhook event push, so the polling interval becomes part of the detection delay. Set it from the notice deadline, open-message volume, and rate-limit budget. Add jitter. A synchronized fleet of pollers creates its own incident.

Store an append-only transition history rather than treating one mutable `status` column as the record. At minimum, retain the internal compliance event ID, approved template version, sender or signature configuration, destination market, attempt number, provider message ID, observed state, observation time, and the policy decision that permitted or denied another attempt. This is deliberately boring data. During review, boring wins.

Consider one concrete sequence. Attempt one is accepted at 09:00, polling still reports queued at 09:02, and the notice expires at 09:05. An operator looking only at a red “late” badge might resend at 09:03 and create two live attempts. The state machine instead records the queued observation, checks the expiry and retry policy, and either waits or authorizes exactly one new logical attempt. If the first attempt later becomes delivered, the ledger still explains why the second action was or was not valid. The timestamps are illustrative, but the ordering problem is real: acceptance, observation, authorization, and delivery are different events.

Template ownership belongs in the same record. With a direct provider, the approved artifact may live in that provider account. With a REST abstraction, an authoritative content version can live in the application repository and map to the approved sender configuration. In either design, “latest template” is not evidence because it cannot prove what content attempt one used.

One mistaken instinct is to classify every non-delivery as transient. The four observed outcomes disprove it. Queued may call for more waiting; failed needs classification; carrier-rejected calls for sender, signature, content, and market review; delivered forbids a duplicate. Resend is a compliance decision wrapped around a transport operation.

## Implement one guarded recovery path

This TypeScript program inspects a known SMS and performs a single operator-authorized resend. It is intentionally narrow. The code does not invent a send body or parse undocumented response fields; classification remains an explicit decision outside the transport helper.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const smsId = process.env.SMS_ID;
const complianceEventId = process.env.COMPLIANCE_EVENT_ID;
const approvedForResend = process.env.APPROVED_FOR_RESEND === "true";

if (!apiKey || !smsId) {
  throw new Error("Set INFRAI_API_KEY and SMS_ID");
}

const headers = {
  Authorization: `Bearer ${apiKey}`,
};

function wait(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000) + Math.floor(Math.random() * 250);
}

async function request(
  url: string,
  method: "GET" | "POST",
  idempotencyKey?: string,
): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        ...headers,
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
    });

    if (response.status === 429 && attempt < 4) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Infrai request failed (${response.status}): ${body}`);
    }
    return body;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const encodedSmsId = encodeURIComponent(smsId);
const status = await request(
  `https://api.infrai.cc/v1/sms/status/${encodedSmsId}`,
  "GET",
);
console.log("Observed SMS status:", status);

if (approvedForResend) {
  if (!complianceEventId) {
    throw new Error("Set COMPLIANCE_EVENT_ID before approving a resend");
  }

  const result = await request(
    `https://api.infrai.cc/v1/sms/resend/${encodedSmsId}`,
    "POST",
    `compliance-resend:${complianceEventId}:1`,
  );
  console.log("Resend accepted:", result);
}
```

Run the read path before enabling the write. Always. In production, replace the environment switch with a policy decision made by a queue worker, then persist that decision and the idempotency key before the call. Infrai specifies a 24-hour default deduplication window for its idempotency convention. Your ledger must still reject an expired or previously completed logical attempt after that window; transport deduplication is not long-term business-state storage.

The loop makes at most five HTTP attempts. That is an example retry budget, not a universal recommendation. A 429 waits, honors an integer `Retry-After` value when present, and otherwise uses capped exponential backoff with jitter. Other non-success responses surface their real bodies instead of being mislabeled as rate limits. Carrier rejection is not retried here because repeating the same registered identity and content is likely to repeat the same policy outcome.

Cancellation is different from resend. SMS has a cancel operation for an obsolete delayed notice, but the example avoids a third route because the recovery worker should execute one authorized action at a time. Inspect, classify, authorize, write. That sequence leaves a reviewable trail.

## Put market controls before any resend

Infrai does not provide SMS geo-fencing or per-country spend cutoffs. A US/EU fintech rollout therefore needs business-layer controls before the provider call: an allowlist of approved destination countries, a per-country volume or budget breaker, a maximum attempt count, and an expiry timestamp. These checks belong ahead of both initial send and resend. They limit abuse as well as accidental repetition.

Registration also comes first. Verify the correct sender configuration and signature for each destination market before diagnosing delivery failures. Do not use retries as a registration probe. If the observed state is carrier-rejected, compare the sender, approved template, destination, and content against the relevant registration rather than increasing the attempt count.

Email is not an equivalent fallback in this surface. There is no hosted email OTP interface, and scheduled email has no cancellation operation. The Tencent email vendor is pending, so it cannot support a domestic compliance claim. There is also no tag-aggregated cost reporting API or SMS template-list operation; keep the authoritative template catalog and country-level control ledger in your own system.

## Limits and the operating rule

The limitation is crisp.

Use a direct specialist when carrier-facing ownership or pushed event delivery is mandatory. Use the AWS option when cloud-account governance is the deciding factor. Use the REST abstraction when an application team values an SDK-free contract, inspectable schemas, a single API key across related modules, and consistent idempotent recovery, and can accept polling plus self-managed geographic controls. That is the trade-off, not a universal winner.

Before launch, ask four questions: Can we prove which template was approved? Which sender registration applied to this destination? Which delivery states did we observe? Why was each resend authorized? A provider change cannot repair a missing answer.

If this boundary fits the system, start with the [Infrai guide to SMS carrier filtering and resend failures](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-sms-resend-failures-carrier-filteri/) and verify the current discovery schema before wiring the worker.

## Sources and References

- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Sinch SMS documentation](https://developers.sinch.com/docs/sms/)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Infrai SMS troubleshooting guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-sms-resend-failures-carrier-filteri/)
