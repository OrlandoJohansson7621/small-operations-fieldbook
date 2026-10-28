# Auditable Email and SMS Event Notifications for Settled Payment Receipts

The least complex choice is a provider-neutral notification service with an immutable event record, a rendered message snapshot, and a delivery result for every receipt. Pick the channel after the payment is settled. Keep the payment event and the delivery attempt separate. That gives compliance reviewers an answer they can inspect instead of a vague claim that an API call succeeded.

| Option | Pick this when | Evidence you must retain | Main trade-off |
| --- | --- | --- | --- |
| Direct email API plus SMS API | Email is the default and SMS is a fallback or a separate alert path | Provider request ID, message hash, timestamps, status callbacks, template version | Two operational surfaces and two callback models |
| Cloud email service plus a messaging provider | Receipt volume and regional delivery needs differ by channel | The same event ledger fields across both adapters | More adapter code, but clearer channel ownership |
| Self-hosted mail transfer plus an SMS API | Your team operates mail infrastructure and accepts the delivery burden | SMTP transaction logs, signed event records, carrier response data | Maximum control, with more abuse, reputation, and on-call work |

## How should I compare email and SMS API providers for event notifications?

“Best” is too blunt for a settled-payment receipt. Measure the evidence chain first: can the system prove which payment event produced which content, when the send was requested, which channel accepted it, and what happened afterward? A provider with a polished send endpoint can still be a poor fit if the callback history is hard to export or if the message body cannot be tied to a template revision.

A useful comparison sheet has one row per candidate and these columns: delivery channels, status-event detail, retention and export controls, regional processing options, suppression behavior, authentication support, and failure semantics. Add a column for ownership. Someone must own template changes, callback verification, and replay decisions.

My decision rule is concrete: if a candidate cannot expose a stable message identifier and an exportable status history, it cannot be the system of record for a compliance-sensitive receipt. Choose the next candidate, even when its send API looks easier.

Three common products illustrate why the columns matter, without creating a universal ranking. Twilio exposes messaging APIs and a SendGrid Email API, so one organization can evaluate both channels under related account controls; its boundary is that channel-specific delivery semantics still need separate handling. Amazon SES is an email service, not an SMS service, so it can be a focused email option while another system owns text delivery. Mailgun documents an email-focused sending and events model, which can suit a team that wants email evidence without treating email and SMS as one identical transport. These are differences in scope, not scores.

The compliance question is narrower than “did the request return 200?” It is “can we reconstruct the decision and the communication?”

## How do you build the evidence chain?

Use a small event ledger before calling an external API. The ledger record should contain a stable payment ID, settlement timestamp, recipient policy decision, channel, destination fingerprint, template ID and version, content hash, attempt ID, provider name, provider message ID when available, and normalized status. Store the rendered body separately when policy permits it; otherwise retain the exact approved input plus its hash and a controlled rendering reference.

The flow is easy to say and easy to get subtly wrong:

settled payment -> notification intent -> rendered snapshot -> send attempt -> provider acknowledgement -> signed status event -> final audit record

Do not let a retry create a second payment receipt by accident. Give the intent an idempotency key derived from the payment ID, receipt type, recipient, and template version. Give each attempt its own ID. A timeout means “unknown,” not “failed,” until a reconciliation job checks the provider event stream or an export. This distinction prevents an operator from replaying a message that was accepted just because the first HTTP response was lost.

Here is the adapter boundary I use for a reviewable implementation. The interface says what the application needs; each provider adapter translates its own response and callback vocabulary.

```ts
type Channel = "email" | "sms";
type DeliveryState = "accepted" | "unknown" | "failed";

type ReceiptIntent = {
  paymentId: string;
  attemptId: string;
  channel: Channel;
  recipient: string;
  templateVersion: string;
  contentHash: string;
  idempotencyKey: string;
};

type SendResult = {
  state: DeliveryState;
  providerMessageId?: string;
  providerRequestId?: string;
  observedAt: string;
};

interface NotificationAdapter {
  send(intent: ReceiptIntent): Promise<SendResult>;
}

async function sendReceipt(
  adapter: NotificationAdapter,
  intent: ReceiptIntent,
  record: (event: object) => Promise<void>,
): Promise<SendResult> {
  await record({ type: "send_requested", intent });

  try {
    const result = await adapter.send(intent);
    await record({ type: "send_observed", intent, result });
    return result;
  } catch (error) {
    await record({
      type: "send_unknown",
      intent,
      observedAt: new Date().toISOString(),
    });
    throw error;
  }
}
```

The catch path deliberately records uncertainty before rethrowing. A queue can retry it, but the retry worker must consult the intent ledger first. That is the small piece many otherwise tidy notification systems omit.

For email, build unsubscribe handling into the message policy. RFC 8058 defines the one-click unsubscribe mechanism for list email using HTTPS POST, and it requires the relevant message headers to be present in the message. A transactional receipt may have a different legal treatment from promotional mail, but the classification should be explicit and tested rather than inferred from the subject line.

For SMS, record consent source, jurisdictional policy, sender identity, and opt-out state. Keep the receipt text short enough for the selected message format, but do not trim away the transaction identifier that lets support match it to the ledger. A fallback SMS should be a new attempt linked to the same intent, not a silent mutation of the email attempt.

## Which operational details change the decision?

Callbacks are evidence, not truth by themselves. Verify their signature or authenticated transport, reject stale or duplicate events safely, and normalize them into an internal state machine. Keep the original payload alongside the normalized state when retention rules allow it. That makes a later dispute explainable: the normalized state is useful for queries, while the original event preserves what the external system actually said.

Test the ugly cases. Send after a worker restart. Receive the same callback twice. Receive an accepted callback after a timeout. Change a template between intent creation and retry. Lose the callback and recover it through reconciliation. A good test asserts the resulting ledger, not only the number of outbound requests.

Deployment deserves the same discipline. Roll out a new template version behind an explicit policy, sample rendered messages before broad release, and alert on a missing callback stream separately from delivery failure. Track queue age, acceptance rate by channel, callback lag, unknown outcomes, and duplicate-intent attempts. These metrics answer different questions; combining them into one delivery percentage hides the exact failure that a reviewer will ask about.

## Limits of the comparison

No provider can establish that a person read a receipt. A successful API acknowledgement usually describes acceptance by a service, not final inbox or handset visibility. Carrier filtering, mailbox rules, blocked destinations, and downstream outages remain outside the application’s direct control.

The architecture also does not remove policy work. Retention periods, access controls, regional data handling, consent, and the legal classification of a receipt need an owner. Choose the provider mix that produces inspectable evidence for those obligations, then validate the result with a real replay and reconciliation exercise. The decision is sound when an auditor can follow one settled payment through intent, content, attempt, callback, and final state without trusting an undocumented manual step.

## Further reading

- RFC 8058, One-Click Unsubscribe: https://datatracker.ietf.org/doc/html/rfc8058
- Twilio Messaging API overview: https://www.twilio.com/docs/messaging/api
- Twilio SendGrid Mail Send API: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon Simple Email Service API reference: https://docs.aws.amazon.com/ses/latest/APIReference/Welcome.html
- Mailgun Events API: https://documentation.mailgun.com/docs/mailgun/api-reference/openapi-final/events
