# Modern SaaS App Logging: A Decision Guide for Managed Search and Custom Ingestion APIs

**Short answer:** For a modern SaaS app, treat a Loggly alternative such as managed log search and a custom log ingestion API as different operating models, then choose the smallest one that can answer an incident question within your latency, retention, privacy, and staffing limits.

| Path | Pick this when | The trade-off to test |
| --- | --- | --- |
| Managed log search (Papertrail, Loggly, or Better Stack) | You need usable search quickly and have limited platform time | Export, retention, access control, and ingest limits remain vendor contracts |
| Custom log ingestion API | You need a stable event contract, routing control, or a strong data-location requirement | Your team owns buffering, retries, schema changes, storage, and on-call recovery |
| A split path | Diagnostic logs need search, while audit records need a separate durable store | Two delivery policies must stay visibly separate |

The product names are useful comparison labels, not a ranking. The durable decision is about failure behavior. A search screen that cannot show delivery lag is an incident risk, regardless of who hosts it.

## How should a modern SaaS app compare managed logging with a custom ingestion API?

Start with four questions from the Google SRE monitoring model: what is the request latency, traffic, error rate, and saturation? Logs explain individual events; metrics should carry the aggregate signal that wakes the on-call engineer. If a logging choice cannot show its own queue age, rejected events, and storage use, it is not observable enough to trust during an outage.

Then write three incident queries before signing up or building anything: find every event for one request ID, find all failed operations for one deployment, and show the gap between emitted and accepted events. A managed service can shorten the path from stdout to search. A custom API can make the envelope and routing rules yours. Neither removes the need to test loss, delay, and access boundaries.

Keep the contract boring. One newline-delimited JSON event should have a timestamp, severity, service, environment, deployment version, operation, request ID, outcome, and duration. User context belongs in an optional opaque identifier, not an email address. Cardinality, retention, and redaction rules should be reviewed like a database schema.

That's the first gate.

## Where do managed search tools fit, and where do they stop?

Managed search is a good fit when the team wants a short implementation path, shared dashboards, and someone else to operate the ingest tier. Papertrail, Loggly, and Better Stack can be placed in the evaluation set without assuming they have identical query languages, alert semantics, retention policies, or export behavior. Ask each one for the same evidence: documented limits, delivery acknowledgement, delayed-ingest visibility, role controls, deletion behavior, and a usable export format.

The catch is the contract you do not control. A hosted search product may be unsuitable when records must stay in a particular region, when a legal hold requires an independent copy, or when an outage must not block a release workflow. It is also a poor fit if the bill or retention model cannot be bounded from your traffic envelope. Stick with a managed path when the team can accept those boundaries and can rehearse an exit; the time saved on upgrades is real operational capacity.

Do not make price the primary argument. Compare the full work budget: collector maintenance, query tuning, storage recovery, access reviews, and the engineer who handles a dropped batch at 02:00. A low per-event figure is meaningless if the system loses the evidence needed to diagnose a failed payment.

## What does a safe custom log ingestion API actually own?

A custom endpoint is a small distributed system. The application emits locally and returns to its request path; a collector batches events, applies size and redaction rules, and sends them to the API. The API authenticates, validates the schema version, acknowledges only what it has durably accepted, and exposes counters for accepted, rejected, sampled, and dropped events. Storage and search then have their own retention and access policies.

The queue is the center of the design. Give it a finite byte and age budget. Retry transient failures with jitter and a ceiling. On permanent validation failures, count and quarantine the event rather than retrying forever. A 429 is a signal to slow down, not permission to grow memory without bound. I would test a 429, a slow destination, and a process restart in CI; I'm not sure any design is ready until those three cases produce visible metrics and a bounded impact on request latency.

Make the test concrete. Start a worker with a 10 MB queue, send a burst that fills it, and make the destination acknowledge nothing for 30 seconds. The application should keep its request budget, the queue should report age before it reports drops, and the runbook should identify which events are sacrificed first. Kill the worker after it writes a batch but before the acknowledgement, restart it, and verify that the batch ID prevents a duplicate. Finally, return a permanent 400 for one malformed event and confirm that the worker quarantines that event while later batches continue. This one sequence exercises backpressure, crash recovery, idempotency, and schema enforcement together; a dashboard that only says “ingest healthy” will miss at least one of them.

Here is a TypeScript sender with an explicit timeout and bounded retry count. It is intentionally generic: the endpoint can be a self-hosted collector or a managed ingestion URL, while the application keeps the same event contract.

```ts
type LogEvent = {
  timestamp: string;
  level: "debug" | "info" | "warn" | "error";
  service: string;
  environment: string;
  version: string;
  operation: string;
  request_id: string;
  outcome: "ok" | "error";
  duration_ms: number;
  user_id?: string;
};

export async function sendBatch(
  endpoint: string,
  events: LogEvent[],
  token: string,
): Promise<void> {
  const body = JSON.stringify({ schema_version: 1, events });
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 2_000);
    try {
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "content-type": "application/json", authorization: `Bearer ${token}` },
        body,
        signal: controller.signal,
      });
      if (response.ok) return;
      if (response.status < 500 && response.status !== 429) {
        throw new Error(`permanent ingest rejection: ${response.status}`);
      }
    } finally {
      clearTimeout(timeout);
    }
    await new Promise((resolve) => setTimeout(resolve, 100 * 2 ** attempt));
  }
  throw new Error("ingest retry budget exhausted");
}
```

The sender is not the queue. Put it behind a bounded worker so a slow destination cannot hold request handlers open. Include a batch ID and schema version, and make the receiver idempotent on that ID. When a field changes, accept both versions during rollout, update queries, then remove the old version after its usage has drained.

## Which failure tests separate a useful logger from a noisy one?

Run a test matrix against every serious option. Saturate the application until the queue reaches its byte limit. Rate-limit the destination. Drop the process after local write but before acknowledgement. Replay the same batch. For each case, record request latency, queue age, event loss by reason, duplicate rate, and time-to-find by request ID. GitHub Actions can run these checks as a repeatable workflow, with secrets supplied through its normal secret controls rather than committed to the repository.

The result should be a short runbook: how to pause optional debug volume, how to preserve error and audit-class events, how to export a representative time window, and who owns recovery. A green application dashboard does not prove that its evidence arrived.

One more boundary matters. Diagnostic logs are not an audit ledger. If a transaction needs acknowledged, immutable retention, send a separately governed record to a durable store. Sampling can protect search capacity, but it is a bad policy for the only record of a security decision. Your mileage may vary with regulatory scope and incident response practice; document the assumption instead of hiding it in a logger default.

The best choice is the one that survives the drill: an engineer receives a request ID and can retrieve the operation, outcome, deployment, and correlated error without guessing which layer dropped the event. Managed search reduces platform work. A custom API increases control. The decision is sound only when its limits are explicit and tested.

## Sources

- https://sre.google/sre-book/monitoring-distributed-systems/
- https://docs.github.com/en/actions
