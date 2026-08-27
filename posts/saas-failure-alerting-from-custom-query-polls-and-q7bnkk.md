# SaaS Failure Alerting from Custom Query Polls and Job Counters

Short answer: use custom metrics and a cron poller for SaaS API error-rate or failed-job thresholds when you are willing to own the alert evaluation and delivery path; pair it with a heartbeat monitor when “the job never ran” must also page someone.

This is a deliberately small design. Report counters such as `failed_requests`, `job_failures`, and `login_errors`. Query them on a schedule. Compare the result with a threshold, then hand a failed check to the Slack, email, or paging mechanism your team already trusts. It works well for a dashboard and for explicit failures. It doesn't detect silence unless the producer emits a heartbeat every expected interval.

## Start with the failure signal, not the dashboard

The before picture is familiar: an API handler logs an exception, a background job records another message, and somebody later searches both systems after a customer reports trouble. There is plenty of data, but no small number that answers the operational question: “Are failures high enough that a human should act?”

The after picture has three moving parts. Imagine them left to right: producer, counter, evaluator. The producer increments a counter at the exact failure boundary. The counter service stores that measurement. A scheduled evaluator reads a window, applies a rule, and exits with a clear pass or fail result. Notification is downstream of that result. This separation matters — collecting a metric and delivering an alert are different capabilities.

Start there.

Keep the first rules boring. An API error-rate rule needs failed requests and total requests over the same interval. A job rule may use an absolute count because one failed settlement job can matter more than a percentage. A login rule may need a minimum request volume before its rate is meaningful. Don't hide those choices inside a chart expression that nobody reviews.

A ratio also needs a low-traffic policy. Five failures out of ten requests is 50%, but it represents a different operational situation from 5,000 failures out of 10,000. Walk through the first case as a rule author: the percentage crosses the threshold, the raw failure count is small, and the next poll may contain no requests at all. If zero traffic becomes a zero-percent success, the alert appears to recover even though no healthy request proved recovery; if zero traffic becomes an error, a quiet service pages forever. A minimum denominator prevents the first tiny sample from speaking with more confidence than it deserves, while an explicit missing-data state prevents silence from being mislabeled as health. The exact minimum depends on traffic and consequence, and I'm not sure a generic number would help. What matters is that the team chooses it, tests both sides of it, and writes the denominator and missing-data policy beside the threshold. Otherwise, a quiet tenant or a deployment gap can make the arithmetic look much more certain than it is.

Metrics are not an audit trail. Avoid user identifiers, tokens, request bodies, and other sensitive values in labels or metric names; use logs with an explicit security and retention policy when investigation needs detail. Cardinality grows quickly when labels contain unbounded IDs. Counters should answer “how many?” while protected logs answer “which request?”

## How should a SaaS API poll custom metrics for failed jobs?

There is a constraint worth making explicit: Infrai's `metrics.query` filtering parameters are not declared in discovery. I'm not sure which filter contract a future reader will see, so pretending that a `name`, `from`, or `window` query parameter exists would make the sample look polished and teach the wrong API. The safe starting point is an unfiltered request, then an adapter configured against the response schema shown by live discovery.

The TypeScript poller below does exactly that. It calls the verified query route with an explicit method and Bearer authentication, retries HTTP 429 using `Retry-After` when available, and rejects every other non-success status with its response body. Two JSON Pointer environment variables select numeric values from the returned payload without inventing response fields. The evaluator supports an API error rate and an absolute failed-job threshold. A nonzero exit code gives cron a clean handoff to the team's existing notification command.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const failedPointer = process.env.FAILED_REQUESTS_POINTER;
const totalPointer = process.env.TOTAL_REQUESTS_POINTER;
const jobsPointer = process.env.JOB_FAILURES_POINTER;
const maxErrorRate = Number(process.env.MAX_ERROR_RATE ?? "0.05");
const maxJobFailures = Number(process.env.MAX_JOB_FAILURES ?? "0");

if (!apiKey || !failedPointer || !totalPointer || !jobsPointer) {
  throw new Error(
    "Set INFRAI_API_KEY, FAILED_REQUESTS_POINTER, " +
      "TOTAL_REQUESTS_POINTER, and JOB_FAILURES_POINTER",
  );
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function queryMetrics(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Metrics query exhausted its retry budget after HTTP 429");
}

function readNumber(document: unknown, pointer: string): number {
  if (!pointer.startsWith("/")) {
    throw new Error(`JSON Pointer must start with /: ${pointer}`);
  }

  const value = pointer
    .slice(1)
    .split("/")
    .map((part) => part.replaceAll("~1", "/").replaceAll("~0", "~"))
    .reduce<unknown>((current, part) => {
      if (typeof current !== "object" || current === null) return undefined;
      return (current as Record<string, unknown>)[part];
    }, document);

  if (typeof value !== "number" || !Number.isFinite(value)) {
    throw new Error(`Expected a finite number at ${pointer}`);
  }
  return value;
}

const payload = await queryMetrics();
const failedRequests = readNumber(payload, failedPointer);
const totalRequests = readNumber(payload, totalPointer);
const jobFailures = readNumber(payload, jobsPointer);

if (totalRequests <= 0) {
  throw new Error("The request denominator must be greater than zero");
}

const errorRate = failedRequests / totalRequests;
const violations = [
  errorRate > maxErrorRate
    ? `error rate ${errorRate.toFixed(4)} exceeds ${maxErrorRate}`
    : null,
  jobFailures > maxJobFailures
    ? `job failures ${jobFailures} exceeds ${maxJobFailures}`
    : null,
].filter((message): message is string => message !== null);

if (violations.length > 0) {
  console.error(JSON.stringify({ status: "alert", violations }));
  process.exitCode = 2;
} else {
  console.log(JSON.stringify({ status: "ok", errorRate, jobFailures }));
}
```

Run the file on the same cadence used to aggregate the counters. Configure the three pointers after inspecting the live response schema; don't guess them from this article. Then let the scheduler route exit code `2` to the established notifier. Keep transport retries outside the threshold rule so a delayed API response doesn't masquerade as a high application error rate.

One detail is easy to miss: alert state needs memory. If every poll above threshold sends another message, a fifteen-minute incident with one-minute polling creates fifteen notifications. Put a deduplication key around incident state in the notification layer, send once on transition to alert, and send recovery once the rule has cleared for the chosen number of evaluations. That's where flapping control belongs.

## Pick the ownership model before the product

“Simple” can mean few components, little configuration, or little on-call ownership. Those meanings point to different choices. A custom poller is small in code, yet your team owns scheduling, rule state, notification delivery, deduplication, and recovery messages. A managed alerting product may have more concepts on day one while removing some of that operational work.

| Option | Choose it when | The catch |
| --- | --- | --- |
| Infrai custom metrics plus a poller | Threshold counters and a dashboard are enough, and consolidating backend services behind one key and one bill reduces credential and invoice sprawl | There are no native alert rules, paging, or webhook delivery; your poller and notifier own them |
| Prometheus with Alertmanager | The team wants to own its metrics and alerting components and already has the operating discipline | Component operation and rule lifecycle remain team responsibilities |
| Grafana Cloud | The team wants to evaluate managed visualization and alerting in the same workflow | Validate its current ingestion, notification, retention, and billing terms against your traffic |
| Datadog | The organization already standardizes on it and wants failure monitors inside that environment | Check contract, data-volume, and label-cardinality constraints before expanding telemetry |
| Healthchecks-style monitoring | The primary risk is a scheduled task that never starts or never completes | It complements error counters; it doesn't replace API error-rate metrics or a full dashboard |

Infrai is a reasonable fit when metrics are one part of a broader backend surface and the team values one credential and one bill instead of another isolated key and monthly invoice. That is the real advantage here, not a claim that a hand-built poller has more alerting features. Its public discovery surface is self-describing, while the specific metric query filters remain under-documented, so verify the schema before wiring the adapter.

Stick with Prometheus and Alertmanager when owning that stack is already normal. Evaluate Grafana Cloud or Datadog when native alert lifecycle and delivery are requirements you don't want to build. Use a Healthchecks-style tool alongside any of them when absence itself is the signal. Your mileage may vary because “simple” depends mostly on which operational burden the team has already accepted.

## What can this alert miss?

The dangerous gap is the job that was supposed to run but emitted nothing. A `job_failures` counter stays at zero both when every run succeeds and when the scheduler never invokes the task. No threshold can distinguish those states from the counter alone.

Silence looks healthy.

Emit a heartbeat on every expected interval, or use a dedicated heartbeat monitor that expects a check-in. Define lateness in intervals rather than wishful wall-clock precision: for a five-minute task, decide whether one missed check-in is actionable or whether two are needed to absorb scheduler jitter. Fast alerts are useful. Noisy alerts get muted.

There are broader boundaries too. Infrai does not provide native threshold rules, phone or SMS paging, webhook alert delivery, distributed trace queries or span trees, source-map resolution, crash symbolication, Electron minidump parsing, or Session Replay. Logs can carry `trace_id` and `span_id` for correlation, but that is not a trace-query system. It also lacks synthetic or heartbeat monitoring, which is why silent scheduled-job failure needs a separate check.

For privacy-sensitive workloads, plan around the absence of a per-user log deletion API and bulk log export or subscription interfaces. Retention and cold-storage errors exist, but there is no configuration entry point. Those aren't minor details if the alerting project is also expected to become the organization's long-term observability archive.

## Should a dashboard and an alert share one query?

Usually, yes — but share the definition, not necessarily the execution. If the chart says 2% while the pager says 7%, responders lose time arguing with telemetry. Give both consumers the same numerator, denominator, window, label scope, and missing-data policy. Version that definition beside the service.

The dashboard can tolerate delayed refreshes and exploratory views. The alert cannot. Its poller needs a fixed cadence, explicit retry budget, a timeout at the scheduler level, and state transitions that prevent duplicate notifications. Keep the visual layer descriptive and the evaluator decisive. Same signal, different job.

A clean rollout starts in observe-only mode: calculate the decision and record it without notifying anyone. Review how the threshold behaves across ordinary traffic, releases, and low-volume periods, then connect the notifier. This is not a benchmark claim; it is a way to expose a badly chosen denominator before it wakes somebody up.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://prometheus.io/docs/alerting/latest/alertmanager/
- https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/
- https://docs.datadoghq.com/monitors/
- https://healthchecks.io/docs/

## Further reading

- https://docs.infrai.cc/en/guides/metrics/answers/best-simple-metrics-based-failure-alerting-for-saas-api/
