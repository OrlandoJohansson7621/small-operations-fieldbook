# Receipt Portal SMS OTP Delivery — Observe Sender Registration and Carrier Filtering

| Pick this when | Operational choice | Evidence to keep |
| --- | --- | --- |
| Payment has settled and a receipt needs sending | Track the receipt as its own notification job | Settlement reference, receipt job ID, submission outcome |
| A customer cannot complete SMS-based 2FA login | Trace the verification attempt before offering another send | Attempt ID, sender configuration, destination country, terminal outcome if available |
| Several attempts appear stuck after submission | Investigate the affected delivery cohort | Age of unresolved attempts, registration status, documented route information |

TL;DR: For a logistics order receipt, the least complex useful design is a receipt job triggered by settled payment, with a separate trace for SMS OTP login. A missing login code and a missing receipt need different evidence. For US application-to-person traffic sent over ten-digit long codes, check sender registration; then investigate carrier filtering and the delivery path. For EU destinations, establish the specific country's requirements before applying a US explanation. Keep anti-fraud decisions visible alongside transport outcomes.

Start with evidence.

This field guide uses delivery reliability as the decision axis: can an operator explain which stage stopped progressing, without exposing a customer's code or confusing successful authentication with successful receipt submission? The proposed event names and thresholds below are design examples, not delivery guarantees or measured results.

## When should the receipt have its own job?

Pick a separate receipt job when the business event is settled payment. In this logistics scenario, a customer may need a receipt while a different part of the application asks for a login code. Define the receipt's eligibility from the payment event. Give the authentication attempt its own lifecycle, and keep authorization checks on any receipt retrieval endpoint. Separating these jobs does not make a protected receipt public.

Diagram in words: payment settles → receipt job becomes eligible → notification is submitted → available delivery evidence is recorded. Alongside it: customer requests login → anti-fraud policy decides → an SMS attempt may be submitted → the customer completes verification or the attempt expires. Connect both paths to the order through internal references. Don't collapse them into one green “message sent” flag.

An email channel is a possible receipt transport; the email service overview in References describes sending email through an API or SMTP. That is a channel capability, not proof that a particular receipt reached a recipient. Choose the transport against the receipt's requirements, and define what evidence your application will actually retain. This article's harder diagnostic problem is the SMS verification path.

The trade-off is a second state machine. In return, the team can answer a precise support question: did the payment event fail to create a receipt job, or is a separate login challenge preventing account access? An authentication resend should not create another receipt job. Use the settlement reference to enforce that application rule.

## Why can SMS OTP delivery fail after sender registration?

Pick an attempt-level trace when the symptom is “I can't log in.” Begin by separating local policy decisions, submission outcomes, transport evidence, and verification completion. A local anti-fraud denial belongs in the first category. Calling it carrier filtering sends the investigation to the wrong team. Likewise, reserve “delivered” for the meaning documented by the transport integration; record customer verification as a distinct event.

For US A2P 10DLC, the cited compliance documentation describes registration for application-to-person messaging using US ten-digit long-code numbers. Inspect the registration and sender association applicable to the attempted message. Keep that scope explicit. The documentation does not establish a single registration rule for every US sender type or for every EU country, and registration alone should not be your application's evidence of receipt.

Carrier filtering is a relevant investigation branch, but require an attributable status or support finding before assigning it as the cause. Record the original outcome alongside your normalized category. If an integration returns an ambiguous result, label it unknown and preserve the evidence needed to investigate; a tidy dashboard should not manufacture certainty.

Shared routes deserve the same discipline. Treat a shared delivery path as a hypothesis when several failures cluster, and request documented routing information where available. Country and sender configuration are useful grouping dimensions. A cluster alone cannot tell you which downstream path carried a message, whether filtering occurred, or whether registration caused the failure. That uncertainty is resolved by transport evidence, not another chart.

The EU branch needs a country. “EU SMS” is too broad a configuration label for this runbook: the US registration reference cannot substantiate the destination's sender requirements. Require current destination-specific documentation before changing a sender or route, and record that reference in the team's operational configuration.

## When should the team investigate a cohort?

Pick aggregate monitoring when individual traces exist but operators still cannot tell whether the problem is spreading. Build separate counters for local policy denials, submission failures, unresolved submitted attempts, reported delivery outcomes, and completed verifications. Use the labels your integration can support. Avoid inventing a carrier dimension when you have only a destination country.

A useful before-and-after is small: replace “SMS success: green” with “submission accepted; delivery unresolved; verification incomplete.” That view tells an operator what to inspect next without claiming to know why the handset is silent. Keep per-attempt identifiers in restricted logs, and use bounded categories for metric labels. Neither a phone number nor an order ID belongs in a metric label.

For an illustrative alert policy, flag submitted attempts that remain unresolved for 120 seconds. Treat 120 seconds as a review threshold your team chose, not a carrier deadline, OTP lifetime, or promise to the customer. Review the oldest unresolved attempt together with the count. Otherwise, a small pocket of stuck attempts can disappear inside a healthy-looking aggregate.

There is a trade-off here: narrower destination and sender cohorts make troubleshooting easier, but sparse cohorts provide less context. Include counts and timestamps in the alert. A low verification completion count is an invitation to inspect the journey; by itself it does not prove an SMS transport failure.

## Implement one honest alert before adding retries

The following TypeScript example evaluates a snapshot of attempt state. It deliberately emits an investigation signal for an unresolved submission. It does not convert missing evidence into a delivery failure. The sample identifiers are fictional, and the threshold is illustrative.

```ts
type Attempt = {
  attemptId: string;
  purpose: "login_otp";
  destinationCountry: string;
  senderConfig: string;
  state: "policy_denied" | "submitted" | "delivered" | "failed";
  submittedAtMs?: number;
};

type Signal = {
  event: "sms_delivery_unresolved";
  attemptId: string;
  destinationCountry: string;
  senderConfig: string;
  ageSeconds: number;
};

function unresolvedSignal(
  attempt: Attempt,
  nowMs: number,
  reviewAfterMs = 120_000,
): Signal | undefined {
  if (attempt.state !== "submitted") return undefined;
  if (attempt.submittedAtMs === undefined) return undefined;
  const ageMs = nowMs - attempt.submittedAtMs;
  if (ageMs < reviewAfterMs) return undefined;
  return {
    event: "sms_delivery_unresolved",
    attemptId: attempt.attemptId,
    destinationCountry: attempt.destinationCountry,
    senderConfig: attempt.senderConfig,
    ageSeconds: Math.floor(ageMs / 1_000),
  };
}

const sample: Attempt = {
  attemptId: "attempt_logistics_0042",
  purpose: "login_otp",
  destinationCountry: "US",
  senderConfig: "receipt_portal_login",
  state: "submitted",
  submittedAtMs: 1_000,
};

const signal = unresolvedSignal(sample, 121_000);
if (signal) console.log(JSON.stringify(signal));
```

At the example's boundary, the signal reports an age of 120 seconds. A policy-denied attempt produces no unresolved-delivery signal because nothing was submitted. A delivered or failed attempt also produces none. Those are useful test cases because they catch a misleading alert, not just a broken timestamp calculation. Add cases for a submission timestamp of zero, a future timestamp, and a missing timestamp before adapting this function to a production worker.

The snapshot comes from storage your application maintains. Persist attempt state, validate incoming status events against the integration's documented authenticity mechanism, and define how duplicate or late events affect the snapshot. The function does not implement those boundaries. Nor does it deduplicate alerts: if a scheduler evaluates the same unresolved attempt repeatedly, use an alert key and a chosen repeat policy in the surrounding system. Consider a synthetic test in which an attempt first crosses the review threshold, then receives a terminal outcome, then receives the same outcome again. Require the first evaluation to open one investigation, the terminal update to resolve it, and the duplicate to leave that resolution intact. Keep the original event timestamps and ingestion timestamps available to the investigator. For conflicting outcomes, define an explicit review state rather than letting arrival order silently determine the answer. Then replay the sequence after a worker restart. The alert should still point to the same attempt, and none of these authentication events should create a second settled-payment receipt job.

Keep the code out of telemetry.

Here “code” means the customer's OTP: omit it, the message body, and the full phone number from this signal. The internal attempt ID is enough to connect the alert to a restricted investigation record. Keep payment details out as well.

Before deployment, replay synthetic accepted, denied, delayed, and terminal outcomes into a test environment. Check that an operator can navigate from an unresolved alert to the attempt record, sender configuration, and original transport evidence. Roll out the monitoring with a defined owner and a review window before making it trigger automated retries. Observability that nobody can act on is another queue.

## Limits that should stay visible

This approach explains where evidence stops. It cannot prove handset receipt from an accepted submission, infer an undisclosed shared route, or turn a US registration reference into EU-wide guidance. An explicit unknown outcome is a useful operational result.

Retries need their own decision. Specify a resend budget, challenge-expiry behavior, and anti-fraud checks before automation; otherwise the response to an unresolved attempt can be another unresolved attempt. Evaluate operational effort and message volume alongside reliability, without making unit price the acceptance criterion. Keep the settled-payment receipt job independent throughout that investigation.

## Further reading

References:

- US A2P 10DLC compliance documentation. Scope: registration for US application-to-person messaging over ten-digit long codes. https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Email service overview. Scope: email sending through an API or SMTP; useful background for the separate receipt channel. https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
