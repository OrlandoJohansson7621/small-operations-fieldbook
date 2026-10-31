# Marketplace Seller Alerts: Node.js Multi-Channel Email Fallback to SMS Explained

A marketplace seller should receive a new-order notification through the least disruptive channel first, then get a clearly bounded SMS fallback when the email path cannot reach a terminal state. The practical design is a small state machine: record the order event, send email, poll its status, and promote to SMS only after a defined deadline. Keep every transition in an audit record. That is what makes delivery behavior explainable during a payment dispute or an on-call handoff.

Evidence matters.

| Path | Pick this when | Main trade-off |
| --- | --- | --- |
| Email, then status polling | The seller has a stable email address and the sender can expose delivery state | Lower interruption, but the workflow must wait for evidence |
| Email and SMS in parallel | The order is time-sensitive enough to justify duplicate messages | Faster awareness, with more duplicate and opt-out risk |
| Email, then queued SMS after a deadline | Email is the default and SMS is an escalation channel | Requires a durable timer and an idempotency key |

Choose deliberately.

## How should multi-channel event notifications handle email fallback to SMS?

For a marketplace seller, choose the third row when the business rule is “email first, SMS only if email does not become deliverable soon enough.” Do not treat an accepted submission as proof that a person saw the message. An email provider may accept a request while delivery, deferral, bounce processing, or mailbox filtering remains unresolved.

The deadline is a product decision, not a magic number. A team might start with 10 minutes for a normal order and review that choice against seller response expectations, mailbox-provider behavior, and SMS consent. The code should read the deadline from configuration so the policy can change without rewriting the event handler.

There is a useful diagram in words: order event -> durable notification record -> email request -> polled delivery state -> either close or enqueue SMS -> final audit state. Every arrow should be repeatable. Every terminal state should explain why the next action was or was not taken.

## What should polling actually prove?

Polling is a measurement loop, not a second send attempt. Store the provider-independent facts you need: channel, attempt identifier, last observed status, next poll time, and the policy version. Map external statuses into a small internal vocabulary such as `pending`, `delivered`, `failed`, and `unknown`. Keep `unknown` distinct from `failed`; a timeout can mean “we lack evidence,” not “the message was rejected.”

The first implementation I would review is the one that uses `setTimeout` inside an HTTP request. It works in a local demo and disappears when the process restarts. A queue or scheduled worker owns the next poll instead. That gives the poll a durable place to resume and lets the order handler return quickly.

No magic number.
Use an idempotency key based on the order and notification purpose, not on the worker attempt. A retry of `order-1842:seller-alert` must find the existing email attempt rather than create a second alert. The SMS promotion gets its own key, `order-1842:seller-alert:sms`, and is permitted only once.

## A compact Node.js workflow

The adapters below are deliberately generic. Their job is to hide transport-specific names behind the state machine, so the policy can be tested without a live messaging service.

```ts
type DeliveryState = "pending" | "delivered" | "failed" | "unknown";

type Notification = {
  orderId: string;
  sellerId: string;
  email: string;
  phone: string;
  createdAt: number;
  emailAttemptId?: string;
  emailState: DeliveryState;
  smsQueued: boolean;
};

type EmailAdapter = {
  send(input: {
    idempotencyKey: string;
    to: string;
    template: string;
    data: Record<string, string>;
  }): Promise<{ attemptId: string }>;
  status(attemptId: string): Promise<DeliveryState>;
};

type SmsQueue = { 
  enqueue(input: { idempotencyKey: string; to: string; body: string }): Promise<void>;
};

async function advanceNotification(
  record: Notification,
  email: EmailAdapter,
  sms: SmsQueue,
  now = Date.now(),
): Promise<Notification> {
  if (!record.emailAttemptId) {
    const sent = await email.send({
      idempotencyKey: `${record.orderId}:seller-alert`,
      to: record.email,
      template: "new-order",
      data: { orderId: record.orderId },
    });
    return { ...record, emailAttemptId: sent.attemptId, emailState: "pending" };
  }

  const state = await email.status(record.emailAttemptId);
  const deadlineMs = 10 * 60 * 1000;
  const expired = now - record.createdAt >= deadlineMs;

  if (state === "delivered" || state === "failed" || !expired || record.smsQueued) {
    return { ...record, emailState: state };
  }

  await sms.enqueue({
    idempotencyKey: `${record.orderId}:seller-alert:sms`,
    to: record.phone,
    body: `New order ${record.orderId} is ready for review.`,
  });
  return { ...record, emailState: state, smsQueued: true };
}
```

The worker persists the returned record in the same store that owns the notification key. In production, claim a record before advancing it, and use a compare-and-set update when writing the result. Two workers can otherwise observe `smsQueued: false` at the same time. The state machine is small; the concurrency boundary is where the hard bugs live.

A Mustache template can keep seller-facing text separate from orchestration code. Escape interpolated values by default, and pass an order identifier as data rather than concatenating untrusted seller or customer text into a template. The template syntax is intentionally limited, which is useful for predictable message rendering.

## How do you observe the fallback without creating noise?

Emit one structured event for each transition, with `order_id`, `seller_id`, `channel`, `attempt_id`, `state`, `policy_version`, and a correlation identifier. Metrics should answer three operational questions: how many email attempts reach `delivered`, how many cross the fallback deadline, and how many SMS jobs are actually queued. An alert on every individual timeout will wake someone for normal variation; an alert on a sudden change in the fallback rate is more useful.

Log the reason for promotion, too. `email_status_unknown_after_deadline` tells an investigator more than `sms_sent`. Avoid putting email addresses or phone numbers in ordinary logs. Keep the audit record access-controlled and retain only what the notification policy requires.

Email sender identity and recipient expectations matter before the code ships. Sender guidance from Yahoo highlights authentication, reputation, complaint handling, and list hygiene; those concerns affect delivery evidence even when the orchestration is correct. A status poll cannot repair a poorly governed sending identity.

Test the transitions as data, not as sleep-based integration tests. Feed the worker sequences such as `pending -> delivered`, `pending -> unknown -> deadline`, and `pending -> failed`. Run the same sequence twice and assert that the second run does not send another email or enqueue another SMS. Then test a crash after enqueue but before persistence; the idempotency key must make the retry harmless.

Limits to make explicit.

This workflow cannot prove that a seller read an email or SMS. It can prove only what the transport reports and what your worker recorded. Polling also adds delay and traffic, so use bounded backoff and stop at a terminal state. If the business requires immediate awareness, parallel delivery may be the honest policy, with duplicate suppression and consent rules designed up front.

A practical polling schedule might be 30 seconds, then 2 minutes, then 5 minutes, with the deadline checked on every run. Those intervals are policy choices, not guarantees from a transport. Record them with the policy version so a later review can distinguish a changed schedule from a changed delivery system.

The clean boundary is simple: email remains the primary attempt, polling supplies evidence, and SMS is a policy-driven escalation. Keep the records durable, the states small, and the decision visible in metrics. That makes a new-order notification easier to operate than a chain of opaque retries.

## Further reading

- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/
