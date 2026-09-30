# TypeScript Rerank API for Small Job Board Result Sets (Worth the Cost)

**TL;DR:** Add a reranking stage only when a labeled test set shows that it fixes consequential ordering errors inside the small candidate set your first retriever already finds. For job-board search, that means measuring whether the stage moves genuinely suitable jobs above merely keyword-similar ones. For property-management deduplication, the same rule asks whether it promotes the true duplicate record before an operator reviews the queue. If reranking does not change a downstream decision, its extra request, tokens, and latency buy nothing.

Start with 20 candidates, rerank the top 10, and keep the feature behind a flag. Those are deployment parameters, not universal thresholds. The evidence should decide whether they stay.

## When does a small result set still need reranking?

Set size is the wrong first question. Candidate ambiguity matters more. A lexical or embedding retriever can return only eight plausible records and still order them badly because the signals needed for the final distinction were compressed or normalized away.

The main limitation is structural: a reranker only reorders what it receives. It doesn't repair missing candidates, stale records, or incorrect hard filters. The trade-off is equally concrete. More context can improve the final comparison, but scoring more documents increases processing work and can extend response time.

Consider two maintenance records: "water under sink, unit 4B" and "kitchen cabinet leak, apartment 4-B." A first-stage retriever should catch the overlap. The harder judgment is whether they describe the same event rather than two repairs at the same address. Dates, unit normalization, work-order status, and a terse free-text note all matter together. Job-board search has an equivalent failure: a posting can share a title and skills with the query while differing on location, seniority, or employment type.

Reranking is useful when the first stage has adequate recall but weak ordering. It cannot recover a true match that never entered the candidate set. Retrieval-augmented generation uses the same broad separation between retrieval and downstream use of retrieved material; the original RAG paper describes combining parametric and non-parametric memory rather than treating retrieval as the final answer.

This creates a clean decision rule: diagnose misses before buying another scoring stage. If relevant items are absent from the candidates, improve query construction, filters, normalization, or the first-stage index. If they are present but buried, test reranking.

## Put the decision behind one TypeScript boundary

The data flow is deliberately plain. Normalize the query and hard constraints, retrieve a wider candidate set, apply deterministic filters, optionally rerank a narrow slice, and return the final ordering with enough metadata to explain which path ran. One interface keeps the experiment reversible and prevents the rest of the application from depending on a particular model or service.

The example below is runnable TypeScript. Its local scorer is intentionally modest: it demonstrates the contract, timeout behavior, stable fallback, and instrumentation point without pretending that token overlap is a semantic model.

```ts
type Candidate = {
  id: string;
  text: string;
  retrievalScore: number;
};

type RankedCandidate = Candidate & {
  rerankScore?: number;
  rankSource: "retrieval" | "rerank";
};

type Reranker = (
  query: string,
  candidates: readonly Candidate[],
  signal: AbortSignal,
) => Promise<readonly number[]>;

const words = (value: string): Set<string> =>
  new Set(value.toLowerCase().match(/[a-z0-9]+/g) ?? []);

const localReranker: Reranker = async (query, candidates) => {
  const queryWords = words(query);
  return candidates.map((candidate) => {
    const candidateWords = words(candidate.text);
    const overlap = [...queryWords].filter((word) => candidateWords.has(word));
    return overlap.length / Math.max(queryWords.size, 1);
  });
};

async function rankCandidates(
  query: string,
  candidates: readonly Candidate[],
  rerank: Reranker,
  options: { enabled: boolean; limit: number; timeoutMs: number },
): Promise<readonly RankedCandidate[]> {
  const baseline = [...candidates]
    .sort((a, b) => b.retrievalScore - a.retrievalScore)
    .map((candidate) => ({ ...candidate, rankSource: "retrieval" as const }));

  if (!options.enabled || baseline.length < 2) return baseline;

  const head = baseline.slice(0, options.limit);
  const tail = baseline.slice(options.limit);
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), options.timeoutMs);

  try {
    const scores = await rerank(query, head, controller.signal);
    if (scores.length !== head.length || scores.some((score) => !Number.isFinite(score))) {
      return baseline;
    }

    const reranked = head
      .map((candidate, index) => ({
        ...candidate,
        rerankScore: scores[index],
        rankSource: "rerank" as const,
      }))
      .sort((a, b) =>
        (b.rerankScore - a.rerankScore) ||
        (b.retrievalScore - a.retrievalScore) ||
        a.id.localeCompare(b.id),
      );

    return [...reranked, ...tail];
  } catch {
    return baseline;
  } finally {
    clearTimeout(timer);
  }
}

const records: Candidate[] = [
  { id: "wo-184", text: "Unit 4B kitchen cabinet leak reported Monday", retrievalScore: 0.78 },
  { id: "wo-219", text: "Water under sink in apartment 4-B on Monday", retrievalScore: 0.75 },
  { id: "wo-090", text: "Unit 4B bathroom faucet replacement", retrievalScore: 0.71 },
];

const result = await rankCandidates(
  "duplicate: kitchen sink leak at unit 4B Monday",
  records,
  localReranker,
  { enabled: true, limit: 3, timeoutMs: 150 },
);

console.log(result.map(({ id, rankSource }) => ({ id, rankSource })));
```

The fallback is part of the design, not an apology. A timeout or malformed score array preserves the original retrieval order. Stable tie-breaking by retrieval score and record ID also makes repeated runs inspectable; otherwise equal rerank scores can cause confusing movement in review queues.

In production, replace `localReranker` with an adapter for a cross-encoder, a hosted scoring endpoint, or an application-specific model. Keep the function signature. The `AbortSignal` interface is part of the DOM standard and is available in modern Node.js runtimes, so cancellation does not need a vendor-specific abstraction.

## Measure changed decisions, not attractive scores

Build a labeled set from the decisions the system must support. For a property manager, each query can identify one known duplicate, a group of related-but-distinct repairs, and obvious negatives. For a job board, labels should preserve hard constraints such as remote eligibility and seniority instead of rewarding title similarity alone. Split templates, buildings, employers, or time periods across evaluation partitions where leakage would make near copies too easy.

Then compare the baseline and reranked lists at the actual review depth. Recall at the first-stage cutoff answers whether the correct item entered the room. Metrics such as mean reciprocal rank or normalized discounted cumulative gain describe ordering, but the operational metric should mirror the workflow: true duplicate in the first three records, or qualified job in the first page. Record the absolute number of corrected queries and newly damaged queries as well. An average can hide a painful regression on rare constraints.

Use a small decision table before exposing live traffic:

| Observation | Likely action | Reason |
|---|---|---|
| Relevant record is absent from retrieved candidates | Fix first-stage retrieval | Reranking cannot score an item it never receives |
| Relevant record is present but often below review depth | Trial reranking | Ordering is the failing component |
| Ordering improves, but user decisions do not | Keep the baseline | Score movement has no demonstrated value |
| Decisions improve, but the latency budget is missed | Shrink the rerank slice or run it selectively | Quality and response time are both product constraints |

Do not tune the cutoff on the same examples used for the final report. Even a modest collection of hand-labeled cases can be overfit by repeated threshold changes. Freeze a holdout, version the labels, and keep the baseline output beside every reranked output so reviewers can identify which stage introduced a mistake.

## The cost is a request path, not a price row

For a small candidate set, raw usage can look trivial. The relevant cost is broader: another network hop or local model execution, serialized query-document text, retries, observability, evaluation upkeep, and the tail latency added before the user sees results. Variable vendor pricing isn't a durable architectural argument. Measure units that remain meaningful when a provider or model changes: rerank calls per search, documents per call, input bytes or tokens, timeout rate, cache hit rate, and latency percentiles.

Selective execution usually has more leverage than shaving a fraction from every call. Skip reranking when there is one candidate, when deterministic filters leave an obvious exact identifier match, or when a cached query and corpus version already has a result. Trigger it for ambiguous cases: close first-stage scores, several candidates sharing the same normalized address, or searches whose constraints require comparing multiple fields together.

Be careful with the trigger. A score-gap threshold is meaningful only for the retriever and corpus on which it was calibrated. Log the baseline scores, trigger reason, candidate count, model version, corpus version, elapsed time, and whether the final top results changed. Avoid logging unrestricted resumes, tenant notes, or maintenance descriptions; store identifiers and derived measurements unless raw text is explicitly required and governed.

Short path, hard evidence.

## Ship the experiment with an exit condition

Roll out in shadow mode first: calculate the alternative order without showing it. Once the evaluation pipeline confirms that production-shaped inputs match the offline pattern, expose a small traffic slice and compare downstream decisions. A feature flag should select the scorer and the rerank limit independently. That makes rollback immediate and prevents a model change from being tangled with a candidate-count change.

Before enabling the path, verify that filtering happens before reranking, request cancellation reaches the adapter, failures retain the baseline, ties are deterministic, and metrics distinguish retrieval misses from ordering misses. Watch p50 and p95 latency separately, along with the fraction of searches that invoke the stage and the fraction whose visible order changes. Review harmed examples, not only aggregate gains. Finally, write an exit condition: remove or disable the stage if the holdout and live decision metric fail to clear the agreed improvement while staying inside the response-time budget.

The answer for small result sets is conditional. Reranking earns its place when relevant candidates already exist, their order causes observable mistakes, and the corrected decisions justify the extra path. Otherwise, keep the simpler retrieval pipeline and spend the engineering time on recall, labels, or hard filters.

## References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- MDN, `AbortSignal`: https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal
- Node.js, global `AbortController`: https://nodejs.org/api/globals.html#class-abortcontroller
- Normalized Discounted Cumulative Gain, original paper record: https://dl.acm.org/doi/10.1145/582415.582418
