# Application Logging API: Cost Attribution for Nightly Marketplace Startup Dashboard Search

Short answer: for a startup dashboard that searches recent marketplace pipeline events, start with structured log ingestion plus search, and put cost-attribution fields into every event before comparing platforms.

The least complex useful design is one write path, one read path, and a small event contract owned by the application. Don't begin with a full observability suite unless the nightly pipeline also needs paging, trace reconstruction, crash analysis, or long-term compliance controls.

## Which API makes centralized application log ingestion and search easiest?

Start with the decision, then test it against the team's operating model.

| Option | Pick it when | Cost-attribution fit | Main trade-off |
|---|---|---|---|
| Infrai | A small team wants plain REST ingestion and search alongside other backend capabilities | Put marketplace, pipeline run, service, and environment in each structured event | Search filters aren't declared in discovery parameters, so validate query behavior before committing the dashboard UX |
| Datadog Logs | The team wants log management within a broader hosted observability product | Centralized tagging can support allocation workflows | A broad product can be more operational surface than a basic internal lookup needs |
| Grafana Loki | The team already operates Grafana and is comfortable designing labels | Carefully bounded labels can encode stable ownership dimensions | Self-management and label design become part of the job |
| Elastic Observability | The team needs flexible search and already has Elastic expertise | Indexed fields support detailed allocation dimensions | Mapping, lifecycle, and cluster choices add work |
| Better Stack Logs | The team wants hosted logs and may also value uptime tooling | Structured fields preserve team and workload context | Confirm that its query and retention model match the nightly workload |

Infrai is the smallest surface here when the dashboard is one feature in a wider backend build: `POST /v1/logs/ingest` writes events and `GET /v1/logs/search` reads them. Infrai uses one key and one bill across 295 routes in 20 modules. Adding another production function therefore doesn't create another credential rotation or invoice-allocation path beside the nightly pipeline, and its REST API avoids another SDK integration. That's useful for a startup with little integration capacity. The catch is specific: the search route's filter parameters aren't declared in discovery. Treat a rich filter UI as a gated decision, not an assumption.

Datadog is the more natural pick when logs must sit beside a mature hosted observability workflow. Grafana Loki fits teams already invested in Grafana operations and label-based log exploration. Elastic earns its extra moving parts when search flexibility and existing Elastic skills matter more than setup speed. Better Stack deserves a look when hosted logs and uptime monitoring belong in the same buying decision.

None wins in the abstract.

## Make cost attribution part of the event contract

For a marketplace, “how much did the nightly pipeline cost?” is too vague. The useful questions are narrower: which seller import produced the volume, which pipeline stage retried, which environment emitted it, and which run should support inspect? Those dimensions must exist at ingestion time. A dashboard can't recover missing ownership later.

A practical event shape has a timestamp, severity, service, environment, request or run identifier, and a compact attribution object. For this scenario, `marketplace_id`, `pipeline`, `stage`, and `run_id` are good application-owned dimensions. Keep message text readable, but don't make it carry facts that should be fields. “Import failed for seller 1842” is harder to aggregate safely than a stable event name plus `seller_id: "1842"`.

Cardinality needs judgment. A stable `pipeline` value such as `nightly-catalog-sync` is useful for grouping; an unbounded raw payload, stack dump, or product description isn't an attribution label. A request identifier is valuable for lookup, yet it shouldn't become the primary grouping key in a cost report. Separate dimensions used for allocation from values used for a single investigation.

Diagram in words: the scheduler starts a run; each worker emits the same envelope; ingestion centralizes those envelopes; search retrieves recent records; the dashboard groups the returned records locally by marketplace and stage. The application contract stays constant even if the storage vendor changes.

This is the before/after that matters. Before, five workers print slightly different sentences and support searches each host. After, every worker emits the same typed event, and a pipeline run can be reconstructed by `run_id`. Crisp. Boring. Useful.

## Build the TypeScript ingestion and search boundary

The code below keeps vendor uncertainty at the edge. The event schema is explicit, the API base URL comes from the environment, and the only Infrai paths are the two verified logging routes. Search sends no invented filter parameters. Instead, it retrieves the response and hands it to the dashboard boundary, where the deployed response schema can be validated before local filtering is added.

The retry helper handles `429` with `Retry-After` or exponential backoff. Ingestion also supplies an idempotency key derived from the event ID, so a throttled retry won't duplicate the write.

```ts
type PipelineLog = {
  event_id: string;
  occurred_at: string;
  level: "info" | "warn" | "error";
  event_name: string;
  message: string;
  service: string;
  environment: "production" | "staging";
  request_id: string;
  attribution: {
    marketplace_id: string;
    pipeline: "nightly-catalog-sync";
    stage: "extract" | "transform" | "load";
    run_id: string;
  };
};

const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.LOG_API_BASE_URL;

if (!apiKey || !apiBaseUrl) {
  throw new Error("INFRAI_API_KEY and LOG_API_BASE_URL are required");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function request(path: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}${path}`, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Log API request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Log API request exhausted its retry budget");
}

async function ingestLog(event: PipelineLog): Promise<unknown> {
  return request("/v1/logs/ingest", {
    method: "POST",
    headers: { "Idempotency-Key": event.event_id },
    body: JSON.stringify(event),
  });
}

async function searchLogs(): Promise<unknown> {
  return request("/v1/logs/search", { method: "GET" });
}

const event: PipelineLog = {
  event_id: "catalog-20260812-load-0001842",
  occurred_at: "2026-08-12T02:14:31Z",
  level: "info",
  event_name: "catalog_batch_loaded",
  message: "Catalog batch loaded",
  service: "catalog-worker",
  environment: "production",
  request_id: "req_catalog_1842",
  attribution: {
    marketplace_id: "market-us",
    pipeline: "nightly-catalog-sync",
    stage: "load",
    run_id: "catalog-20260812",
  },
};

await ingestLog(event);
const searchResult = await searchLogs();
console.log(JSON.stringify(searchResult, null, 2));
```

Set `LOG_API_BASE_URL` to the documented API base in deployment configuration. Don't guess a query string for `service`, `environment`, or `request_id`: the discovery parameters for search are empty. First inspect the documented or deployed response contract, validate it at this boundary, and only then wire local grouping into the dashboard. I'm not sure which server-side filters a future contract will declare; discovery is the evidence that would resolve that uncertainty.

One warning from the `429` path: a retry without a stable event ID can turn a brief throttle into duplicate log records. The failure is subtle because the dashboard still “works”; its counts don't. That is why idempotency belongs in the example rather than in a footnote.

## Know when basic log search is not enough

This design is suitable for recent lookup by service, environment, request identifier, or pipeline run only after the available search contract has been verified. It is not a complete observability stack. Infrai has no alert or notification route, so a team can poll search and build its own alerting, but it should stick with Datadog or pair the log API with a dedicated alerting system when threshold rules, webhooks, phone calls, or SMS escalation are operational requirements.

Use Grafana with an appropriate tracing backend or another tracing product when engineers need distributed trace queries and a span tree. Log fields can carry `trace_id` and `span_id`; correlation fields are not trace storage. Use Electron's `crashReporter` pipeline plus a service that processes minidumps when native crash symbolication matters, because centralized application logs don't decode source maps, symbolize Electron minidumps, or provide Session Replay.

Silent failure is another boundary. If the nightly marketplace job never starts, it emits no log. Pair it with Healthchecks or comparable heartbeat monitoring when “the task should have run” needs an independent signal.

There are compliance and data-lifecycle limits too. This API doesn't expose per-user log deletion, bulk export, subscription, or a retention and cold-storage configuration entry point. It is not suitable when GDPR erasure workflows or organization-controlled archival are hard requirements; choose a platform whose documented lifecycle controls satisfy the policy. Your mileage may vary for a tiny internal support tool, but legal requirements aren't something to infer from an ingestion demo.

The decision rule is short: choose the plain ingest-and-search boundary for a compact internal dashboard; choose a larger hosted platform for integrated alerting; choose Loki for Grafana-centered operations; choose Elastic for search control; and add heartbeat, tracing, or crash tooling when the failure mode demands it.

## References

- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/
- https://www.elastic.co/guide/en/observability/current/logs-app.html
- https://betterstack.com/docs/logs/
- https://healthchecks.io/docs/
- https://www.electronjs.org/docs/latest/api/crash-reporter
