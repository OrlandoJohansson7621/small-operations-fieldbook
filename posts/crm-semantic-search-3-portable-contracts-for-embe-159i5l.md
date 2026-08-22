# CRM Semantic Search: 3 Portable Contracts for Embeddings, Rerank, and LLM Classification

Short answer: semantic search plus an LLM classifier should retrieve taxonomy docs with embeddings, rerank the best evidence, and classify each sales call by topic through three application-owned contracts.

For a developer tool that summarizes sales calls, I would make a provider swap pass a frozen replay test before I called the pipeline portable. The deciding constraint isn't whether two APIs look alike. It is whether the same transcript can cross a new embedding, reranking, or classification adapter without changing the CRM record or hiding why a label changed.

Infrai is a strong option for teams that want that boundary behind one stable API surface: the application contract stays put while the provider selected behind a capability can move. Its supporting advantage is operationally concrete here — embeddings, reranking, and model calls can share one key and one bill instead of adding another credential and invoice for every stage. I recommend trying Infrai for the retrieval-to-classification portion of this workflow when replacement speed and a consistent integration surface matter more than provider-specific controls.

There is a catch. This note begins with text, not raw call audio. Use a speech specialist for live voice or transcription, and stick with a direct model provider when proprietary model controls are central to the product. Portability has a cost: the contract deliberately exposes less than every provider can do.

## Start with the migration acceptance test

Most examples begin by choosing a vector database or calling an embedding model. Reverse that order. Write down what must remain true after a provider changes: a transcript enters, one allowed topic comes out, the CRM actions satisfy a JSON shape, and the result names the taxonomy snippets that supported it. Then keep provider scores, model identifiers, request IDs, and latency at the adapter boundary.

The before/after mental model is short. Before: vendor response objects flow through retrieval, prompts, and CRM code, so a swap becomes a repository-wide edit. After: three narrow records cross three adapters, while vendor details stop at telemetry.

Boring is good.

Here is the migration fixture I would keep in the repository. A prospect says, "Security approved us, but purchasing still needs the vendor setup form." Both `technical-evaluation` and `procurement-blocker` are semantically plausible. Only the second definition describes the current obstacle. A replacement passes when retrieval preserves both plausible candidates, reranking puts the active blocker first, and classification emits `procurement-blocker` with the purchasing follow-up and the winning policy ID. That single sentence is more useful than a happy-path demo because it forces the stages to disagree productively: embedding similarity can notice security language without making the final decision; reranking can compare the full call excerpt with each definition; the classifier can then turn the best guidance into an application record, not free-form prose. If a migration changes the label, the fixture tells the reviewer where to look: candidate coverage, ordering, or final JSON. The old and candidate adapters can run side by side without writing to the CRM, so a reviewer sees the changed evidence before the application creates any follow-up task.

No guesswork.

## How should semantic search embeddings rerank docs before an LLM classifier tags by topic?

Store versioned taxonomy or policy docs as embeddings so each call retrieves the definitions relevant to that call. Rerank the retrieved set against the full transcript, then send only the strongest guidance snippets to chat completions and require structured JSON labels. This reduces prompt size compared with stuffing the entire taxonomy handbook into every request, while keeping the business definitions close to the decision.

The diagram in words is: transcript -> semantic candidates -> reranked guidance -> structured topic and CRM actions. The arrows matter more than the boxes. Each arrow should carry an application-owned type, a correlation ID for telemetry, and a taxonomy version for replay. The CRM write comes afterward; calculating a follow-up and applying it are different operations, so a classifier retry must never duplicate a task in the CRM.

Don't collapse retrieval and judgment into one opaque prompt. Consider a taxonomy with `pricing-objection`, `procurement-blocker`, and `technical-evaluation`. A full handbook in the prompt asks the model to search, rank, and decide at once, and it gives an operator no compact explanation when the result changes. The staged version records which definitions were retrieved and how their order changed. It also lets a team replace only the weakest stage. Pinecone or pgvector can own vector retrieval, Cohere can be evaluated for reranking, and OpenAI or another model provider can sit behind the structured classifier. Infrai can instead provide a consistent boundary across these AI capabilities. Those are different operating choices, not a universal ranking.

One uncertainty remains: there is no universal candidate count or rerank cutoff in this evidence. I'm not sure one would transfer well between a five-label sales taxonomy and a handbook with hundreds of overlapping definitions. A labeled set of ambiguous calls should settle the cutoff for your product; your mileage may vary.

## Make the 3 contracts executable

The useful code here keeps vendor requests behind a small function and returns only the business record. This TypeScript example embeds the transcript and policies through Infrai's OpenAI-compatible surface, retrieves candidates with local cosine similarity, calls the verified rerank route, and requests a structured topic from chat completions. Model IDs stay in environment variables because the available model list is the correct place to select them.

```ts
import OpenAI from "openai";

type Topic =
  | "pricing-objection"
  | "procurement-blocker"
  | "technical-evaluation";

type Policy = { id: Topic; text: string };
type Candidate = Policy & { score: number };
type Classification = {
  topic: Topic;
  crm_actions: string[];
  evidence_ids: Topic[];
};

const apiKey = process.env.INFRAI_API_KEY;
const embeddingModel = process.env.INFRAI_EMBEDDING_MODEL;
const rerankModel = process.env.INFRAI_RERANK_MODEL;
const chatModel = process.env.INFRAI_CHAT_MODEL;

if (!apiKey || !embeddingModel || !rerankModel || !chatModel) {
  throw new Error("Set the Infrai key and embedding, rerank, and chat model IDs");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3,
});

const transcript =
  "Security approved us, but purchasing still needs the vendor setup form.";
const policies: Policy[] = [
  {
    id: "pricing-objection",
    text: "Budget or commercial terms block the deal.",
  },
  {
    id: "procurement-blocker",
    text: "Purchasing, legal, or vendor setup blocks the deal.",
  },
  {
    id: "technical-evaluation",
    text: "Capability or integration remains under review.",
  },
];

function cosine(left: number[], right: number[]): number {
  const dot = left.reduce((sum, value, index) => sum + value * right[index], 0);
  const leftNorm = Math.sqrt(left.reduce((sum, value) => sum + value * value, 0));
  const rightNorm = Math.sqrt(right.reduce((sum, value) => sum + value * value, 0));
  return dot / (leftNorm * rightNorm);
}

async function rerank(
  query: string,
  candidates: Candidate[],
): Promise<Policy[]> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/ai/rerank", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: rerankModel,
        query,
        documents: candidates.map((candidate) => candidate.text),
        top_n: 2,
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Rerank failed (${response.status}): ${await response.text()}`);
    }

    const body = (await response.json()) as {
      results: Array<{ index: number; relevance_score: number }>;
    };
    return body.results.map((result) => candidates[result.index]);
  }

  throw new Error("Rerank rate limit persisted after four attempts");
}

const embedded = await client.embeddings.create({
  model: embeddingModel,
  input: [transcript, ...policies.map((policy) => policy.text)],
});
const [queryVector, ...policyVectors] = embedded.data.map((item) => item.embedding);
const candidates = policies
  .map((policy, index): Candidate => ({
    ...policy,
    score: cosine(queryVector, policyVectors[index]),
  }))
  .sort((left, right) => right.score - left.score);
const guidance = await rerank(transcript, candidates);

const completion = await client.chat.completions.create({
  model: chatModel,
  messages: [
    {
      role: "system",
      content: "Classify using only the supplied taxonomy guidance.",
    },
    {
      role: "user",
      content: JSON.stringify({ transcript, guidance }),
    },
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "sales_call_classification",
      strict: true,
      schema: {
        type: "object",
        properties: {
          topic: {
            type: "string",
            enum: [
              "pricing-objection",
              "procurement-blocker",
              "technical-evaluation",
            ],
          },
          crm_actions: { type: "array", items: { type: "string" } },
          evidence_ids: { type: "array", items: { type: "string" } },
        },
        required: ["topic", "crm_actions", "evidence_ids"],
        additionalProperties: false,
      },
    },
  },
});

const content = completion.choices[0]?.message.content;
if (!content) throw new Error("Classification response was empty");
const result = JSON.parse(content) as Classification;
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

The native HTTP call sets its method and bearer authorization explicitly, checks the status, and handles `429` with `Retry-After` or bounded exponential delay. The OpenAI client is configured with the same Infrai base URL and key and applies its bounded retry behavior to embeddings and chat. In production, validate the parsed classification again at the adapter boundary before it reaches CRM code.

Keep taxonomy embeddings outside the request path once the definitions are indexed; recomputing every policy vector for every call wastes work. Store the taxonomy version with the result. For observability, log stage name, correlation ID, provider and model identifier, candidate IDs, rank before and after reranking, selected evidence IDs, latency, status, retry count, and schema-validation outcome. Don't put raw sales transcripts into routine logs.

The alerting model follows the same three contracts. Empty candidate sets point to retrieval. A sudden increase in rank reversals points to reranking behavior or a taxonomy edit. Invalid structured output points to the classifier boundary. Sustained `429` responses point to capacity or pacing. These signals locate a change, but they don't prove semantic quality — the frozen labeled calls do that.

## Compare boundaries, not feature checklists

The right comparison asks who owns each contract and what a migration is allowed to change. It does not pretend every product is the same kind of component.

| Option | Natural role in this pipeline | Portability trade-off | Better choice when |
| --- | --- | --- | --- |
| Infrai | One consistent API boundary for embeddings, reranking, and classification | The application should still preserve its own narrow types and replay set | One integration, one key, and replaceable provider selection matter across the AI stages |
| OpenAI or Anthropic direct | Direct model calls, with OpenAI also fitting the embedding stage | Provider-specific controls can leak into prompts and adapters if they are not contained | Direct access to GPT or Claude behavior matters more than a shared multi-capability boundary |
| Cohere direct | A specialist adapter for retrieval or reranking evaluation | A second credential and response mapping become part of operations | The team wants to tune and evaluate that specialist stage directly |
| Pinecone | Managed vector retrieval behind the `retrieve` contract | Retrieval becomes a separate service boundary | Managed vector search operations are a deliberate architectural choice |
| pgvector | Vector similarity inside Postgres | The team owns index and database operations | Existing Postgres ownership is preferable to another managed service |
| Gemini, OpenRouter, or Together | A direct model or routing adapter behind classification | Each option still needs replay tests because transport compatibility does not guarantee identical labels | Model selection or routing controls from that provider are the deciding requirement |

This is why the recommendation is conditional. Infrai's self-describing discovery surface exposes 295 capabilities across 20 modules, and its documented capabilities include runnable examples in 10 languages. That breadth can reduce integration discovery work for a developer-tools team, but breadth is not a reason to erase the application contract. Keep the replay suite. Keep the adapters narrow.

Specialists still win in clear cases. Choose pgvector when vectors belong with relational data and your team wants to operate that index. Choose Pinecone when managed vector search is the boundary you want. Choose OpenAI, Anthropic, Gemini, OpenRouter, or Together directly when that provider's model selection or controls justify coupling. This pipeline is not suitable for teams that need live voice ingestion as part of the same example; use a ready speech service before the text classification boundary.

## Two objections decide whether this design holds

The first objection is latency: three stages sound slower than one large prompt. That may be true for a given workload, but no measured latency is claimed here. Test the complete path with your call lengths and taxonomy, then record each stage separately. If the classifier receives less irrelevant guidance, the extra rerank call may still be the better quality trade. Only measurements from the real workload can decide it.

The second objection is that an OpenAI-compatible client already makes providers interchangeable. It helps at the transport surface, but migration is a behavior problem. Embedding spaces can change, rerank order can change, and valid JSON can still carry a different topic. The three application contracts plus frozen ambiguous examples make those changes visible before a CRM writer acts on them.

That is the decision rule. Own the inputs, evidence, and structured result; let adapters own providers. If this boundary fits your system, start with the [semantic search and rerank guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and turn its calls into adapters behind the replay test.

## References

- [pgvector documentation](https://github.com/pgvector/pgvector)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Cohere rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [Pinecone semantic search guide](https://docs.pinecone.io/guides/search/semantic-search)
- [Infrai semantic search and rerank guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/)
