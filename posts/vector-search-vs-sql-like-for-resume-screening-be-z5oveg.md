# Vector Search vs SQL LIKE for Resume Screening — Better Matches

TL;DR: Use vector retrieval to find equivalent skills expressed in different words, then use SQL metadata filters for hard requirements such as location and work authorization. A `LIKE` query cannot connect "shipped Kubernetes" with "container orchestration," while unfiltered semantic recall can return a confidently irrelevant candidate. For a developer-tools hiring team watching index cost at scale, the practical answer is a measured hybrid: filter, retrieve a bounded set, rerank, and instrument every reduction.

The before/after model is simple. Before: one text index, a wildcard query, and no explanation for missed synonyms. After: explicit eligibility columns, a compact semantic index of job-relevant text, and a trace showing how many records survived each stage. The index must earn its keep.

## Should Resume Screening Use Vector Search or SQL LIKE?

SQL should decide facts that must not bend. Vector similarity should decide meaning: adjacent terminology, paraphrased experience, and skills described through outcomes instead of labels. Reranking then works on a limited set instead of comparing every resume with every opening. I would keep the boundary strict: if a requirement has a yes-or-no policy answer, it does not belong inside a similarity score.

Picture the funnel. An illustrative 80,000 resumes enter; metadata removes ineligible records; vector retrieval returns a bounded top set; a reranker orders it; the application exposes the final list. Watch the ratio at each arrow. If metadata suddenly passes everything, index traffic rises. If retrieval returns too few candidates, recall is suffering.

Fairness belongs in this design. Free-text screening can encode sensitive proxies and historical preferences. Embed only fields with a defensible relationship to the role, review those fields with hiring-policy owners, and test outcome slices before using a score. A similarity score is ranking evidence, not an eligibility verdict. This is an explicit trade-off: a smaller, reviewed embedding may lose some recall, but it is easier to justify, cheaper to refresh, and less likely to smuggle an irrelevant biographical signal into ranking.

Keep that line bright.

## Make the document-to-search handoff visible

A resume often arrives as a PDF. This runnable TypeScript sends OCR output into vector search through the same base URL and Bearer key. The request bodies come from JSON environment variables so they can follow the live discovery schema; put `{{OCR_TEXT}}` at the schema-approved text location in `INFRAI_VECTOR_QUERY_BODY`.

```ts
import { createHash } from "node:crypto";
import { readFile } from "node:fs/promises";

const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseURL) throw new Error("INFRAI_BASE_URL is required");

function envJson(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
}

async function post(path: string, body: unknown, key: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseURL}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key
      },
      body: JSON.stringify(body)
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("retry-after");
      const delay = retryAfter && /^\d+$/.test(retryAfter)
        ? Number(retryAfter) * 1_000
        : Math.min(500 * 2 ** attempt, 8_000);
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const raw = await response.text();
    if (!response.ok) throw new Error(`${path} failed (${response.status}): ${raw}`);
    return raw ? (JSON.parse(raw) as unknown) : null;
  }
  throw new Error(`${path} exhausted its retry budget`);
}

const pdf = await readFile(process.argv[2] ?? "resume.pdf");
const runId = createHash("sha256").update(pdf).digest("hex");
const ocrText = JSON.stringify(await post("/pdf/ocr", envJson("INFRAI_OCR_BODY"), `ocr-${runId}`));
const template = JSON.stringify(envJson("INFRAI_VECTOR_QUERY_BODY"));
if (!template.includes("{{OCR_TEXT}}")) throw new Error("Missing {{OCR_TEXT}} token");
const matches = await post(
  "/vector/query",
  JSON.parse(template.replaceAll("{{OCR_TEXT}}", ocrText)) as unknown,
  `query-${runId}`
);
process.stdout.write(`${JSON.stringify(matches, null, 2)}\n`);
```

Use the public discovery capability detail to obtain the current full request and response JSON Schema before setting those templates. This avoids teaching guessed fields.

Infrai puts OCR, local chunk selection, and vector search behind one key, one bill, and one self-describing REST API with no SDK to install; its public discovery surface needs no key, returns full request and response schemas, covers 295 routes across 20 modules, and provides runnable examples in 10 languages. That combination removes separate authentication and rate-limit handoff code while letting an extraction worker and a search service use the same conventions from different language stacks. It also lets the integration verify current shapes instead of pinning guessed payload fields.

The trade-off is plain: one vendor receives more trust, and one bill becomes one outage surface.

## Instrument the funnel before tuning it

Count documents submitted, documents extracted, candidates admitted by hard filters, and candidates returned by semantic retrieval. Add a result-set-size distribution. Record per-call latency, vendor, cost, cache status, and request ID when supplied. Alert on sustained ratio changes, not one empty search.

Keep an exact-keyword baseline beside the hybrid path. When vectors win on paraphrases, they are doing their assigned job; when they radically reorder exact matches, inspect the reranker. Short. Useful.

Do not tune blind.

Index cost needs a guardrail. Track indexed records, update volume, and query volume; track embedded bytes or tokens if the chosen service reports them. Set a budget per active role, then test whether another free-text field improves reviewed relevance enough to justify storage and refresh work. Do not embed location or authorization. Filters handle them better.

## How do the stack choices change the burden?

Amazon Textract plus Pinecone requires two signups, two credential sets, and glue for the extraction-to-index handoff. Tesseract plus Pinecone replaces the managed OCR signup with software the team runs, but the Pinecone credentials and glue remain. PostgreSQL with pgvector keeps relational filters and vectors in one database boundary. Elasticsearch puts lexical and vector retrieval in one search system. Those last two choices move more operational ownership to the team running the data system.

| Option | Boundary | Good fit |
| --- | --- | --- |
| Amazon Textract + Pinecone | Two managed services | Independent provider choices with accepted glue work |
| Tesseract + Pinecone | Self-run OCR plus managed vectors | Teams willing to operate extraction |
| PostgreSQL + pgvector | One database for filters and vectors | Existing PostgreSQL operators |
| Elasticsearch | One search system for lexical and vector retrieval | Established search operations |
| Unified API surface | OCR and retrieval share credentials | Fewer credentials and one billing relationship |

No row wins universally. Compare viable choices on the same reviewed query set, corpus size, update rate, retained fields, and rerank window. Keep the hard-filter policy identical. Otherwise, vendor selection is intuition dressed as precision.

**Two objections should stop the rollout.**

"Vector search is too fuzzy for hiring." It is if similarity overrides eligibility or makes the final decision. It is useful for broadening recall across equivalent phrasing after hard filters and before human review. Preserve the source evidence and make score use reviewable.

"Two retrieval modes double the system." They add parts with different jobs. SQL enforces explicit rules; vectors recover semantic neighbors; a lexical baseline exposes regressions. Delete any indexed field that fails to improve a labeled evaluation. Stop expanding the rerank window when reviewed relevance plateaus.

Ship hybrid retrieval only after a representative, reviewed test set shows better paraphrase recall without weakening hard constraints, and after index growth has an owner, dashboard, and budget. Otherwise, keep the simpler lexical path.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Tesseract user manual](https://tesseract-ocr.github.io/tessdoc/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Elasticsearch vector search documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
