# Backend Exception API Evidence: Searchable Groups Without Heartbeat Monitoring

TL;DR: For Node.js and Python cron jobs or workers, start with a small backend exception API that accepts one structured event per failed run, computes a stable fingerprint, and lets operators search by run, tenant, model call, and deployment. That is the least complex option that can reconstruct a failed fintech AI agent loop. Without a separate schedule signal, though, exception tracking cannot prove that a job ran at all.

| Pick this collection boundary | Pick it when | Reconstruction strength | Main trade-off |
|---|---|---|---|
| Direct exception API | A job can make one bounded call after failure | Fast path from stack to run context | Reporting can fail with the worker |
| Structured logs plus grouping | Logs already leave every runtime reliably | Exceptions and nearby steps share one stream | Parser changes can split groups |
| Local buffer or queue | Losing evidence during an outage is unacceptable | Retains ordered evidence across exporter trouble | More state, retries, and duplicate handling |

The practical default is the first row. Keep the event contract portable and the grouping logic visible. Move to buffering only when retained failure evidence is worth another stateful component. Price matters, but only after evidence quality, delivery behavior, and searchability.

## What should a backend exception tracking API capture for cron jobs?

An agent loop can fail after a model response, during a policy check, or while posting a ledger-side result. A stack trace answers where the program threw. It does not answer which run was affected, what consumed 18.4 seconds, or which external step accounted for most usage. Incident reconstruction needs a timeline key and enough bounded dimensions to join the pieces.

Use a generated `run_id` across the job. Add `job_name`, `runtime`, `deployment`, and a low-cardinality `stage`. Record durations and usage as numbers instead of hiding them in a message. Keep prompts, model output, credentials, account numbers, and raw payment data out of the payload. Useful evidence and excessive disclosure are separated by a thin line.

That boundary matters.

A compact event needs identity, execution context, measurements, and the failure itself. Identity includes `run_id`, `job_name`, and an opaque tenant reference. Execution context includes stage, attempt, runtime, and deployment. Measurements cover total loop duration, model-call duration, and normalized usage units. Failure fields contain exception type, scrubbed message, stack, and fingerprint version. The fingerprint deserves its own version. Group on stable features such as exception type, normalized top application frame, job name, and stage. Do not group on the full message when it contains IDs, amounts, timestamps, or generated text. Imagine 40 failed transfer checks whose messages differ only by transfer ID: message-based grouping creates 40 groups, while an operator needs one defect group and 40 affected runs. The opposite mistake is just as costly during reconstruction. Grouping every timeout under `TimeoutError` hides whether the delay happened during a model call, a policy check, or a ledger write. Including the bounded stage in the fingerprint keeps those failure sites apart without letting unique business values fragment the evidence.

Keep it boring.

Here is the diagram in words: scheduler starts run; run opens stages; stages emit measurements; a failure emits one scrubbed envelope; the backend groups by a versioned fingerprint; an operator searches the group, pivots to `run_id`, and reads the timeline. Each arrow carries an identifier, not prose someone must parse during an incident.

## Which collection boundary fits the job?

Choose the direct API when simplicity controls the decision. It suits a short cron job or worker whose retry policy already has a strict deadline. The reporter needs a short timeout and must never replace the original exception. Capture the failure, attempt delivery, then preserve the job's real exit behavior.

Structured logs are a serious option when stdout or a process supervisor already provides dependable transport. RFC 5424 defines a syslog message model with structured data and severity semantics, which is a useful reference for separating machine-readable fields from free-form text. The operational catch is real: grouping now depends on the log pipeline, parsing rules, and retention. Test those rules with stack shapes from both runtimes.

A local buffer or queue is for evidence that must survive a simultaneous application and network failure. It decouples completion from export and supports controlled retries. It also introduces duplicate delivery, backpressure, disk limits, poison events, and another recovery path. Require an idempotency key such as `event_id`, then test replay deliberately.

Runtime parity matters. JavaScript and Python serialize exceptions differently, and async frames can differ from synchronous ones. Normalize into one envelope while retaining the original exception type and scrubbed stack. Operators should be able to ask the same questions in either runtime: Which deployment? Which stage? Which run? Which group changed after release?

## A focused TypeScript implementation

This generic transport creates a versioned envelope, applies a hard reporting timeout, and returns control to the caller. A Python worker can implement the same JSON contract rather than imitate JavaScript exception objects.

```ts
import { createHash, randomUUID } from "node:crypto";

type FailureContext = {
  runId: string;
  jobName: string;
  tenantRef: string;
  stage: "plan" | "model_call" | "policy_check" | "ledger_write";
  attempt: number;
  runtime: "nodejs" | "python";
  deployment: string;
  loopDurationMs: number;
  modelDurationMs: number;
  usageUnits: number;
};

type ExceptionEvent = FailureContext & {
  schemaVersion: 1;
  eventId: string;
  occurredAt: string;
  exceptionType: string;
  message: string;
  stack?: string;
  fingerprint: string;
  fingerprintVersion: 1;
};

function topApplicationFrame(stack = ""): string {
  return stack
    .split("\n")
    .map((line) => line.trim())
    .find((line) => line.includes("/app/")) ?? "unknown-frame";
}

function makeEvent(error: unknown, context: FailureContext): ExceptionEvent {
  const exception = error instanceof Error ? error : new Error(String(error));
  const basis = [
    exception.name,
    topApplicationFrame(exception.stack),
    context.jobName,
    context.stage,
  ].join("|");

  return {
    ...context,
    schemaVersion: 1,
    eventId: randomUUID(),
    occurredAt: new Date().toISOString(),
    exceptionType: exception.name,
    message: exception.message,
    stack: exception.stack,
    fingerprint: createHash("sha256").update(basis).digest("hex"),
    fingerprintVersion: 1,
  };
}

async function reportFailure(
  endpoint: URL,
  error: unknown,
  context: FailureContext,
): Promise<boolean> {
  try {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(makeEvent(error, context)),
      signal: AbortSignal.timeout(1_500),
    });
    return response.ok;
  } catch {
    return false;
  }
}
```

The `1_500` millisecond limit is an example policy, not a universal threshold. Set it below the worker's remaining shutdown budget and test the slow path. Returning a Boolean makes delivery measurable without throwing a second exception over the first one. If delivery failure must survive process exit, this direct design has reached its limit. Add durable buffering.

The example omits payload scrubbing to keep the transport readable. Production code should build the outbound object from an allowlist, cap payload and stack sizes, reject unexpected fields, and test nested objects. Secrets arrive in strange places. Generated text does too.

## How do you test incident reconstruction?

Start from operator questions, then work backward. Given only an error group, can someone find every attempt for one run? Can they distinguish a policy rejection from a remote timeout? Can they compare the failed deployment with the previous one? Can they see loop latency and usage without opening sensitive content? If any answer requires guessing from a message string, the schema is unfinished.

Build fixtures for at least seven cases: two transfer IDs producing the same normalized failure, identical exception types from two stages, a multiline async stack, a non-`Error` throw, a nested secret, an exporter timeout, and a replayed `event_id`. These are contract tests for failure modes created by the design, not benchmark claims.

Then run a release test that changes a source line without changing behavior. The same defect should not explode into unrelated groups merely because line numbers moved. Run another that changes the stage. Those failures should remain distinguishable. This is where a superficially simple grouping API proves whether it supports reconstruction or merely stores traces.

Google's SRE monitoring guidance describes latency, traffic, errors, and saturation as four golden signals. Exception groups cover part of the error signal here. Stage durations contribute latency evidence. Queue depth or worker concurrency may reveal saturation. No exception event covers the whole picture, so preserve join keys across the measurements the system emits.

Test search as an incident workflow. A useful acceptance check begins with a fingerprint, filters to one deployment, pivots to a run, orders attempts by time, and compares stage duration with usage. Five moves. If that requires exporting data and writing an ad hoc script, the backend misses the stated requirement for simple searchable groups.

Five moves. No archaeology.

## Limits without heartbeat monitoring

An exception tracker observes reported failures. A terminated process may report nothing. So may a scheduler that never starts the job, a disabled schedule, or a machine that loses connectivity before export. Silence is ambiguous.

Silence is not success.

If heartbeat monitoring is intentionally out of scope, write that boundary into the runbook: this system reconstructs reported exceptions; it does not attest schedule execution. Liveness can be checked through scheduler run records or another independent expected-run signal, but that is a separate control with separate retention and alert ownership. Do not manufacture a synthetic exception to blur the distinction.

The selection rule is concise. Pick the smallest collection boundary that preserves the evidence needed to replay the incident story. Demand stable grouping, cross-runtime fields, bounded delivery, redaction, and searchable run context. Add durability when losing an event has material operational impact. Treat cost as a measured dimension such as ingest volume, retention, and operator time, never as a substitute for those capabilities.

## Further reading

- Google SRE Book, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
- RFC 5424, "The Syslog Protocol": https://datatracker.ietf.org/doc/html/rfc5424
