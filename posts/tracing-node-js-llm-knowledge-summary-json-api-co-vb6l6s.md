# Tracing Node.js LLM Knowledge Summary JSON API Costs by Tenant

Short answer: in Node.js, generate each LLM summary as structured JSON that follows a server-validated schema, then record the accepted API call's cost metadata against the tenant that requested it.

| Option | Pick it when | Per-tenant visibility work you still own | Main limitation |
| --- | --- | --- | --- |
| OpenAI direct | OpenAI is an intentional product boundary | Join tenant context to usage and your cost ledger | Adding another provider means another adapter |
| Anthropic direct | Anthropic-specific behavior is part of the design | Normalize its response into the same tenant event | The application contract remains provider-aware |
| AWS Bedrock | AWS is already the governance and billing boundary | Preserve the product tenant through the cloud call | Cloud alignment adds value only when the team uses it |
| Self-describing multi-provider API | Routing may change while one operational contract must remain stable | Supply the tenant ID and store the returned per-call metadata | The broader platform is unnecessary for a fixed-provider application |

The decision is less about which model writes the nicest paragraph. For an edtech service answering questions over private course material, the useful test is whether an operator can connect one renderable answer to one tenant, one model call, and one cost event without putting the private source passage into a general log. Start there.

## Make accepted cost the primary operating signal

Use two linked records. The content record contains `title`, `overview`, `bullets`, `risks`, and `action_items`. The operational record contains the application's `tenant_id` plus the request, vendor, cost, latency, and cache metadata available from the accepted completion. Join them with a request identifier, but don't copy the retrieved lesson or policy text into the operational event.

The flow in words is compact: tenant request -> private retrieval -> chat completion -> JSON validation -> rendered answer. A parallel lane starts only after validation: completion metadata -> tenant ID -> cost ledger -> dashboard and alert.

This ordering matters. A string that happens to survive `JSON.parse` isn't yet application data. Missing `action_items`, an extra top-level key, or a scalar where the UI expects an array should increment a contract-rejection counter, not produce a half-empty card. Meanwhile, a `429` belongs to transport telemetry, and an accepted but unsupported answer belongs to answer-quality evaluation. Calling all three an “LLM error” creates a tidy graph that can't tell an on-call engineer what to do.

Keep the denominator honest.

## How can a Node.js LLM API turn a structured summary JSON schema into telemetry?

Use a low-cardinality event name such as `knowledge_summary.accepted`, then aggregate `cost_usd` by `tenant_id`. Keep `request_id` for tracing, not as a metric label. The summary itself should never be a metric label either. This split gives finance a tenant rollup, gives support a trace handle, and gives engineering a rejection rate without indexing private educational content in every observability system.

A crisp before/after helps. Before the boundary, a malformed response, a rate-limited request, and a questionable answer all look like one failed summary. After it, the service can answer three separate questions: did the call complete, did the JSON contract pass, and which tenant owns the accepted cost? Those are actionable signals.

I'm not sure a universal rejection-rate paging threshold is defensible. Ten requests from a pilot classroom and ten thousand requests from a district deployment have different noise profiles. Establish a baseline for each service, alert on a sustained change, and retain the raw count beside the rate so a quiet tenant doesn't create a dramatic percentage from one failure.

For pasted source material, count tokens before the chat call with `POST /v1/ai/tokens/count`. The exact request shape should come from discovery rather than a guessed tokenizer payload. Count the instructions and schema along with the source; they all consume the context budget. If the generated object misses a required field, validate on the server and allow one bounded retry with a shorter source chunk.

No infinite loops.

This runnable example uses the OpenAI-compatible client for `POST /v1/chat/completions`. It asks for one exact object shape, validates every field, retries `429` responses with exponential backoff while honoring `Retry-After`, retries one malformed candidate with a shorter source, and emits tenant cost metadata only after the summary passes admission. The API key stays in the environment.

```ts
import OpenAI from "openai";

type Summary = {
  title: string;
  overview: string;
  bullets: string[];
  risks: string[];
  action_items: string[];
};

type CallMetadata = {
  cost_usd?: number;
  latency_ms?: number;
  vendor?: string;
  cache_hit?: boolean;
  request_id?: string;
};

type CompletionWithMetadata = OpenAI.Chat.Completions.ChatCompletion & {
  infrai?: CallMetadata;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseURL = process.env.OPENAI_COMPATIBLE_BASE_URL;
if (!baseURL) throw new Error("OPENAI_COMPATIBLE_BASE_URL is required");

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 0,
});

const requiredKeys = [
  "title",
  "overview",
  "bullets",
  "risks",
  "action_items",
] as const;

function parseSummary(raw: string): Summary {
  const value: unknown = JSON.parse(raw);
  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error("Summary must be a JSON object");
  }

  const record = value as Record<string, unknown>;
  const actual = Object.keys(record).sort();
  const expected = [...requiredKeys].sort();
  if (actual.join(",") !== expected.join(",")) {
    throw new Error("Summary keys do not match the contract");
  }
  if (typeof record.title !== "string" || typeof record.overview !== "string") {
    throw new Error("title and overview must be strings");
  }
  for (const key of ["bullets", "risks", "action_items"] as const) {
    if (
      !Array.isArray(record[key]) ||
      !record[key].every((item) => typeof item === "string")
    ) {
      throw new Error(`${key} must be an array of strings`);
    }
  }
  return record as Summary;
}

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function requestCandidate(source: string): Promise<CompletionWithMetadata> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    try {
      return (await client.chat.completions.create({
        model: "auto",
        messages: [
          {
            role: "system",
            content: [
              "Return JSON only with exactly these keys:",
              "title and overview as strings; bullets, risks, and action_items",
              "as arrays of strings. Do not add keys or markdown.",
            ].join(" "),
          },
          {
            role: "user",
            content: `Summarize this private course policy:\n\n${source}`,
          },
        ],
      })) as CompletionWithMetadata;
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === 2) {
        throw new Error(`Chat request failed (${error.status}): ${error.message}`);
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(error, attempt)),
      );
    }
  }
  throw new Error("Retry policy ended without a response");
}

async function summarize(tenantId: string, source: string): Promise<Summary> {
  let candidateSource = source;
  let lastValidationError = "unknown validation failure";

  for (let attempt = 0; attempt < 2; attempt += 1) {
    const completion = await requestCandidate(candidateSource);
    const raw = completion.choices[0]?.message.content;
    if (!raw) throw new Error("Chat response did not contain summary content");

    try {
      const summary = parseSummary(raw);
      process.stdout.write(
        `${JSON.stringify({
          event: "knowledge_summary.accepted",
          tenant_id: tenantId,
          ...completion.infrai,
        })}\n`,
      );
      return summary;
    } catch (error) {
      lastValidationError = error instanceof Error ? error.message : String(error);
      candidateSource = candidateSource.slice(
        0,
        Math.max(1, Math.floor(candidateSource.length / 2)),
      );
    }
  }

  throw new Error(
    `Summary validation failed after a shorter retry: ${lastValidationError}`,
  );
}

const policy = [
  "Learners may request an extension before the due date.",
  "Instructors record approved extensions in the course portal.",
  "Accommodation records must remain private.",
].join(" ");

const summary = await summarize("tenant_northstar", policy);
process.stdout.write(`${JSON.stringify(summary, null, 2)}\n`);
```

The example deliberately has two retry layers because they represent different failures. Transport retry handles `429` and waits. Contract retry changes the input once by shortening it. A missing field is not evidence that hammering the identical request five more times will help, and an unbounded retry would also blur the tenant's actual call count.

Run the emitted event through a log-to-metric pipeline or store it in a cost ledger. A dashboard can show accepted calls, total cost, and contract rejection rate by tenant; an alert should watch a sustained shift in rejection rate or rate-limit pressure. Keep answer-quality evaluation separate. Valid JSON can still contain a claim that isn't supported by the retrieved course policy.

## Apply the telemetry boundary to provider choice

OpenAI direct is the least complex option when one provider is a deliberate constraint. Anthropic direct follows the same rule: pick it when its provider-specific behavior is part of the product, then accept that your Node.js adapter owns normalization. AWS Bedrock is the stronger fit when the surrounding workload already uses AWS as its governance boundary. None of those choices removes the need to carry an edtech tenant identifier from the application request into an internal cost event.

Infrai gives the application one key and one bill across 295 routes in 20 modules. In this workflow, that means adding scheduling or notifications doesn't create another credential and invoice join beside the tenant ledger. Its public discovery surface also returns request and response JSON Schema plus runnable examples, and OpenAI-compatible calls expose consistent per-call cost, vendor, latency, cache, and request metadata. This option fits when model routing may change but the ledger shape must not.

## Read the limits before choosing the platform boundary

This pattern is not suitable when summaries must stream token by token into the UI, because admission happens only after the complete object can be parsed. It is also a poor fit for a tiny internal tool with one fixed provider and no tenant accounting; stick with the provider's direct client and a small local validator there. Choose AWS Bedrock when AWS governance is the deciding constraint, OpenAI direct when OpenAI is intentionally the product boundary, or Anthropic direct when Anthropic-specific behavior is intentional.

Schema validity is not groundedness, privacy review, or moderation. Evaluate factual support against approved knowledge-base examples, keep retrieved passages out of broad logs, and apply an explicit content-safety policy before automation consumes `action_items`. The multi-provider platform has relevant adjacent boundaries too: ASR isn't supported, real-time voice sessions are limited to the western region, there is no dedicated moderation endpoint, and image upscaling is Lanczos-only. For moderation, use a chat model with a JSON-schema fallback rather than assuming a dedicated route exists.

The final decision rule is plain: optimize for the boundary your team intends to keep. Direct providers minimize abstraction when provider choice is fixed. Bedrock fits an established AWS control plane. A self-describing multi-provider REST surface fits teams that want routing flexibility, consistent per-call metadata, and one credential boundary across backend capabilities. In every case, the Node.js service still owns the tenant join and the decision to admit model output into the product.

## References

- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.anthropic.com/
- https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- https://json-schema.org/specification
- https://www.rfc-editor.org/rfc/rfc9110
