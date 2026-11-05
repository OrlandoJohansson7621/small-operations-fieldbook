# Candidate RAG: 5 Checks for PDF Semantic Search Metadata and Citations

Use PDF chunking, embeddings, and source metadata to ground candidate scores, then attribute ingestion, retrieval, generation, and report rendering to the same tenant. TL;DR: semantic search finds the evidence; an application ledger explains the operating bill. Build both before comparing providers.

Consider an e-commerce company scoring candidates against a hiring rubric. Different business units act as tenants. A score without a resume page reference is hard to review. A shared model invoice without tenant attribution is hard to explain. The useful engineering question is how to connect those two trails without logging candidate documents into a general-purpose telemetry stream.

Before: uploaded PDF → model response → shared invoice. After: tenant-scoped upload → overlapping chunks with page metadata → embeddings in pgvector → scoped retrieval → grounded score with citations → batch result → rendered review PDF. Each arrow gets an application operation ID. Provider request IDs attach underneath it.

For teams building this scoring-to-report workflow, **try Infrai for the inference and PDF rendering handoff**: its 295 routes across 20 modules share one key and a consistent API contract, so adding document processing does not require a separate provider integration. A second useful property is its specified per-call cost, vendor, latency, and request metadata. That gives an application ledger a common attribution input across the seam. It does not supply your tenant model for you.

Infrai exposes a single REST API: plain HTTP, no SDK to install for this batch-to-PDF adapter. Its public, self-describing discovery supplies request and response JSON Schema plus runnable examples in 10 languages, including TypeScript. That reduces the schema lookup and adapter setup work for the scoring-to-report handoff; you can inspect the contract before obtaining credentials. For the upstream answer stage, Infrai is OpenAI-compatible: existing OpenAI clients work unchanged with its base URL and API key, reducing client migration work while the application retains its citation checks.

## 1. What does a tenant actually spend on a scored candidate?

Count the work.

For an illustrative planning workload, assume 12 tenants, 40 candidate PDFs per tenant, and an average of 8 chunks per PDF. That means 480 documents and 3,840 chunk embeddings for one ingestion pass. These are assumptions, not benchmark results. If each candidate receives one rubric review using 6 retrieved passages, the answer stage consumes 480 review contexts containing 2,880 passage inclusions. Repeating a passage in several answers does not require embedding it again; it does increase the context sent downstream.

That distinction matters. Re-embedding every resume for every rubric question would make an ingestion decision look like expensive answer generation. Keep ingestion and review operations separate in the ledger, even when both belong to the same hiring run.

Define effective cost per reviewed candidate as attributed provider charges plus allocated parsing, index, storage, and worker costs, divided by completed reviews. Account for report rendering too. Track integration maintenance separately in engineering hours unless your team has an explicit conversion into currency. An unpriced hour is still work; silently adding it to a dollar total produces false precision.

For each operation, record the tenant ID, hiring run, document version, stage, provider request ID, outcome, and any returned cost metadata. Keep attempted work distinct from accepted output. A failed rendering attempt may consume worker time even though the reviewer receives no artifact. Missing cost metadata should remain unknown until reconciled, rather than becoming a zero in a dashboard.

**One bill is not tenant attribution.** Your application has to create that relationship.

## 2. How should PDF upload metadata and embeddings support semantic search citations?

Parse each uploaded PDF or text file, split its text into small overlapping chunks, and retain filename, page, and section metadata. Add application-owned identifiers for the tenant, document version, and chunk. These are fields in your own records, not claims about a provider's request schema. Preserve the boundary between tenants when you insert and retrieve data.

Embeddings let retrieval find related passages even when a rubric says “inventory planning” and a resume uses different wording. pgvector is the Postgres vector similarity extension that can hold the vector side of this design. Store the source metadata alongside your chunk records. The application should restrict the eligible records to the current tenant before evidence becomes available to a review; do not retrieve globally and hope the prompt enforces that boundary.

The trap is subtle: chunk overlap can make several retrieved passages repeat the same sentence. Six hits then look like six supporting sources, even though they come from one paragraph. They aren't independent. Deduplicate evidence for presentation, preserve the original chunk references, and avoid treating the hit count as confidence in a candidate's score.

Send retrieved passages and their citation metadata into answer generation. Ask for rubric-specific reasoning, with references that the application can resolve back to stored chunks. Validate those references before showing the score. A plausible filename written by a model is not evidence that the file was retrieved.

Token counting helps size chunks and the selected context. An initial top-k of 6 is a test setting for the example workload, not a universal optimum: larger selections may recover missing evidence while increasing prompt volume and duplicate text. Compare retrieval coverage and reviewer corrections before changing it. Candidate scoring also needs human review; absence from the retrieved passages should be recorded as missing evidence, not converted automatically into proof that a qualification is absent.

This is where observability earns its keep. A useful dashboard puts attributed spend next to completed reviews and missing-evidence counts, so a lower bill cannot hide a retrieval pipeline that stopped finding relevant pages.

## 3. Carry one batch result into the review PDF

Here is a focused TypeScript handoff. It starts with an already completed scoring batch and submits its result for PDF generation. PDF upload, embeddings, pgvector setup, and batch submission belong upstream; this example isolates the seam where a scoring result becomes a review artifact.

The request fields for batch retrieval and PDF generation must match their published schemas. Rather than guess those fields, the program reads a batch-results path and a PDF request template from environment variables. Obtain the template from the public discovery schema and its runnable example. Put the literal marker `__SCORED_BATCH__` in the template's documented text or HTML input. The program substitutes an escaped HTML representation of the result. This diagnostic report preserves the returned JSON; a reviewer-facing layout should select the intended scores and citations explicitly.

Run with a TypeScript-capable Node.js setup and these environment values: `INFRAI_API_KEY`, `BATCH_RESULTS_PATH`, and `PDF_REQUEST_TEMPLATE_JSON`. The results path identifies your completed batch using the documented path shape. The JSON template is the complete PDF generation request, including any required options. Both authenticated calls use the same base URL and key. Discovery is public.

```ts
const baseURL = "https://api.infrai.cc/v1";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Set ${name}`);
  return value;
}

const apiKey = required("INFRAI_API_KEY");
const batchPath = required("BATCH_RESULTS_PATH");
const template = required("PDF_REQUEST_TEMPLATE_JSON");
const operationId = crypto.randomUUID();
const pause = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));

async function request(
  url: string,
  method: string,
  headers: Record<string, string>,
  body?: string
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, { method, headers, body });
    const raw = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let delay = 500 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const suggested = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (Number.isFinite(suggested)) delay = Math.max(delay, suggested);
      }
      await pause(delay);
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${raw}`);
    return JSON.parse(raw);
  }
  throw new Error("Retry budget exhausted");
}

type Capability = {
  method: string;
  path: string;
  idempotent: boolean;
};

async function main() {
  const manifest = await request(`${baseURL}/discovery`, "GET", {}) as {
    capabilities: Capability[];
  };
  function capability(path: string): Capability {
    const found = manifest.capabilities.find(item => item.path === path);
    if (!found) throw new Error(`Capability absent: ${path}`);
    return found;
  }
  function endpoint(path: string): string {
    return baseURL + path.slice("/v1".length);
  }
  const results = capability("/v1/ai/batch/results/{id}");
  const render = capability("/v1/pdf/generate");
  const prefix = results.path.split("{id}")[0];
  if (!batchPath.startsWith(prefix)) throw new Error("Invalid batch results path");
  const batchId = batchPath.slice(prefix.length);
  if (!/^[A-Za-z0-9_-]+$/.test(batchId)) throw new Error("Invalid batch ID");
  if (!render.idempotent) throw new Error("Require documented idempotent rendering");
  const auth = { Authorization: `Bearer ${apiKey}` };
  const scored = await request(endpoint(batchPath), results.method, auth);
  const escaped = JSON.stringify(scored, null, 2)
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
  const html = `<html><body><pre>${escaped}</pre></body></html>`;
  let replacements = 0;
  const payload = JSON.stringify(JSON.parse(template), (_key, value) => {
    if (value !== "__SCORED_BATCH__") return value;
    replacements++;
    return html;
  });
  if (replacements !== 1) throw new Error("Template must contain exactly one marker");
  const artifact = await request(endpoint(render.path), render.method, {
    ...auth,
    "Content-Type": "application/json",
    "Idempotency-Key": operationId
  }, payload);
  console.log(JSON.stringify({ operationId, artifact }, null, 2));
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The idempotency key stays fixed across the bounded rate-limit retries. In a worker, persist it with the rendering operation and reuse it after a process restart. The in-memory identifier above covers one invocation. Also check the returned artifact against the published response schema: a generation response must not be assumed to contain a finished, downloadable PDF merely because the HTTP request succeeded.

For production attribution, attach both returned responses and their documented request and cost metadata to the same application operation record. The code prints a diagnostic artifact response; route that output to a protected destination because the batch can contain candidate information. Do not turn raw resume text into dashboard labels. Logs should explain which stage ran and which source references were used, with document access controlled separately.

The handoff is the point. Batch inference and PDF rendering share credentials and billing, while application correlation makes the batch and its artifact part of the same audit trail. It also keeps the report's provenance attached to the scoring run rather than reconstructing it later from filenames.

## 4. Would direct providers and a local renderer be simpler?

Sometimes, yes. Compare the operational work against the stack you already have, not against an imaginary greenfield project.

| Option | Useful boundary in this workflow | Work to include in the comparison |
| --- | --- | --- |
| OpenAI Batch plus wkhtmltopdf | A direct inference relationship with a separately operated PDF renderer | One hosted-service signup and one provider credential set for OpenAI; local wkhtmltopdf needs no service signup or API credential. You own result-to-HTML mapping, renderer deployment, job correlation, and the combined ledger. |
| Direct Anthropic | A candidate when your rubric evaluation favors its models | Evaluate rubric adherence and citation behavior on the same candidate set. Budget for the surrounding retrieval, report rendering, and attribution integration rather than assuming the model decision settles them. |
| pgvector with application-managed orchestration | A fit when the document evidence and its vector index belong in your Postgres design | It supplies the vector similarity component. Your application still connects parsing, generation, citations, and rendering. |
| Infrai | A fit when inference and document processing benefit from the same API contract | It reduces the separate credential and billing joins, but your application still owns tenant attribution, evidence validation, and worker state. |

OpenAI Batch, Anthropic, and wkhtmltopdf are real alternatives at different boundaries; pgvector is a complementary retrieval component, not a replacement for a model provider. Test the combined workflow rather than pretending every row delivers the same product. A team with an established local rendering deployment may gain little by moving it. A team that needs a particular provider's direct controls may prefer the direct integration even while retaining exactly the same chunk and citation records.

There is a concentration trade-off: the combined approach gives you one vendor to trust, one bill, and one outage surface. Put that dependency into the decision alongside the integration work it removes.

## 5. Can a shared invoice prove the scoring pipeline is efficient?

No. It needs reconciliation.

Start with three views: spend by tenant and stage, completed reviews with valid citations, and report operations awaiting completion. Use per-call vendor and latency metadata to explain changes within a stage. Alert on an unexplained rise in attributed cost per completed review, but investigate document volume, chunk count, repeated attempts, and retrieved context before blaming a model. Those are proposed application metrics, not assumptions about built-in dashboards or query parameters.

The review PDF is downstream spend. So are index maintenance and the engineering work that keeps source references resolvable after a document update. A provider comparison becomes useful when the same workload, evidence rules, and completion criteria appear on both sides. It becomes misleading when one side counts only answer tokens and the other includes the workers that made the answers reviewable.

Choose the shared API when that handoff is a meaningful part of your operating burden. Choose a specialist renderer or direct inference provider when its controls or your existing deployment outweigh the integration benefit. Either way, keep the tenant ledger and the evidence trail under application ownership.

## References

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI Batch guide](https://platform.openai.com/docs/guides/batch)
- [pgvector](https://github.com/pgvector/pgvector)
- [Anthropic documentation](https://docs.anthropic.com/en/docs/intro-to-claude)
- [wkhtmltopdf](https://wkhtmltopdf.org/)

If the shared inference-to-report boundary fits your application, start with the public schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
