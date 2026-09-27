---
description: "Expose document search as an agent tool: when the agent should search, query reformulation, returning citable snippets, and verifying citations."
---

# 07 · Retrieval as a Tool

Classic retrieval-augmented generation (RAG) retrieves **once, always**: take the user's
question, fetch the top chunks, stuff them into the prompt, answer. An agent can do
something different — treat search as **a tool it chooses to call**, possibly several
times, with queries it writes itself. This lesson is about that agent side. How to
build a good index (chunking, embeddings, hybrid search, reranking) is covered in the
[RAG Mastery Path](https://sigilipelli.github.io/rag-mastery-path/); we assume you
have a search function and focus on wrapping it well.

## Always-retrieve vs. agentic retrieval

| | Always-retrieve (pipeline) | Retrieval as a tool (agent) |
|---|---|---|
| Searches per question | exactly 1 | 0 to N |
| Query | the user's words | written by the model; can be reformulated |
| Good at | direct factual questions | multi-part questions, follow-ups, "search again with different terms" |
| Cost / latency | predictable | variable |
| Failure mode | irrelevant chunks for off-topic questions | not searching when it should; searching forever |

Rule of thumb: if almost every question needs the documents and one search usually
suffices, a pipeline is simpler and more predictable. Use a search *tool* when questions
are compound, when the agent also has other tools, or when finding the right document
often takes a second query.

## Designing the search tool

- **Description says what is searchable**: "Search the HR policy handbook (2026 edition)
  — leave, expenses, remote work, equipment." This is how the model decides whether
  search is even relevant.
- **Return snippets with IDs and sources**, not whole documents: `{"id": "leave#2",
  "source": "Leave policy §2", "text": "..."}`. IDs make citation possible.
- **Cap and label results**: top 3–5, with a score or rank, and an explicit
  `{"results": []}` when nothing matches — plus a hint ("try different keywords").
- **Scope in code**: filter by the user's permissions *inside* the tool. The model must
  never be able to search documents the user can't read.

## Worked example: search, reformulate, cite, verify

The search function below is a small keyword scorer, so the example runs offline; in
practice it would call your vector or hybrid search service. The mock model's first
query uses the user's informal wording and finds nothing; it reformulates using the
policy vocabulary, then answers with citations — which code then **verifies**.

```python title="kb_agent.py"
import math
import re
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results

DOCS = {
    "leave#1": ("Leave policy §1", "Employees accrue 1.5 days of annual leave per month "
                "of service, up to 18 days per year."),
    "leave#2": ("Leave policy §2", "Unused annual leave of up to 5 days may be carried "
                "over to the next year; carry-over expires on 31 March."),
    "expenses#4": ("Expenses §4", "Home-office equipment up to 300 EUR per year is "
                   "reimbursable with receipts."),
}

def words(t):
    return re.findall(r"[a-z0-9]+", t.lower())

def score(query, text):
    """Tiny keyword relevance: summed inverse document frequency of shared words."""
    n = len(DOCS)
    q = set(words(query))
    s = 0.0
    for w in q & set(words(text)):
        df = sum(1 for _, t in DOCS.values() if w in words(t))
        s += math.log(1 + n / df)
    return s

@tool
def search_policies(query: str):
    """Search the company HR policy handbook (leave, expenses, equipment). Returns up
    to 3 snippets with ids for citation. Use policy vocabulary for best results.

    Args:
        query: Keywords, e.g. 'annual leave carry over'
    """
    ranked = sorted(((score(query, t), i) for i, (_, t) in DOCS.items()), reverse=True)
    hits = [{"id": i, "source": DOCS[i][0], "text": DOCS[i][1]}
            for s, i in ranked[:3] if s > 1.0]
    return {"results": hits} if hits else {
        "results": [], "hint": "no matches; try the handbook's terms (e.g. 'annual leave')"}

def model(messages, schemas):
    r = tool_results(messages)
    if not r:
        return call("search_policies", query="rollover of vacation")
    if not r[-1]["results"]:
        return call("search_policies", "c2", query="unused annual leave carry over")
    return answer("You can carry over up to 5 unused days of annual leave, but they "
                  "expire on 31 March [leave#2]. You accrue 1.5 days per month "
                  "[leave#1].")

res = run_agent(model, registry(search_policies),
                "Can I keep my unused holidays for next year?")
print("ANSWER:", res["answer"])

# Verify citations: every cited id must be among the snippets actually retrieved.
def unsupported(answer_text, messages):
    retrieved = {h["id"] for r in tool_results(messages) for h in r["results"]}
    cited = set(re.findall(r"\[([a-z]+#\d+)\]", answer_text))
    return sorted(cited - retrieved)

print("unsupported citations:", unsupported(res["answer"], res["messages"]) or "none")
tampered = res["answer"] + " Equipment up to 300 EUR is covered [expenses#4]."
print("tampered answer:", unsupported(tampered, res["messages"]))
```

```text
  step 1: search_policies({"query": "rollover of vacation"})
      -> {"results": [], "hint": "no matches; try the handbook's terms (e.g. 'annual leave')"}
  step 2: search_policies({"query": "unused annual leave carry over"})
      -> {"results": [{"id": "leave#2", "source": "Leave policy \u00a72", "text": "Unused annual leave of up ...
  step 3: final answer
ANSWER: You can carry over up to 5 unused days of annual leave, but they expire on 31 March [leave#2]. You accrue 1.5 days per month [leave#1].
unsupported citations: none
tampered answer: ['expenses#4']
```

The first query used the user's informal words ("rollover", "vacation"), matched
nothing useful, and the tool said so — with a hint. The second query used the handbook's
vocabulary and found both leave snippets. The answer cites them, and the verification
step confirms every citation points to a snippet the agent actually retrieved. The
tampered answer shows what the check is for: a claim cited to `expenses#4`, which was
never retrieved in this run, is flagged. In production, fail or flag answers with
unsupported citations rather than showing them to users as sourced.

## Guiding when to search

Put the policy in the system prompt and keep it concrete:

```text
Use search_policies for any question about leave, expenses or equipment, even if you
think you know the answer — company policy differs from general knowledge.
If a search returns nothing, try once more with different terms, then say you could
not find it. Cite snippet ids in square brackets after each claim.
```

"Even if you think you know" matters: models often answer policy-shaped questions from
general knowledge, which is exactly the wrong source for company rules.

## How It Actually Works

Agentic retrieval turns search into a **decision point in the loop**. The model's
choice to search is driven by how well the tool description matches the question, and
its choice of query is driven by the words in context — which is why the first query
tends to echo the user's phrasing, and why hints like "use the handbook's terms"
improve the second.

Citations work because the snippet IDs are in the context next to the text, and models
readily copy nearby identifiers. But copying an ID is not evidence that the claim came
from that snippet: models can attach a plausible-looking ID to a claim from general
knowledge. That's why citation checking needs code — at minimum, "cited IDs ⊆ retrieved
IDs"; better, a check that the claim's key terms or numbers appear in the cited snippet.

## Common mistakes

- **Returning whole documents** and flooding the context.
- **No "nothing found" signal**, so the model fabricates from general knowledge.
- **Permission filtering in the prompt** ("only use documents the user may see") instead
  of in the tool.
- **Unlimited reformulation** — cap searches per question (lesson L1-07 guards work).
- **Trusting citations without checking them.**

## Exercise

1. Change the mock's second query to `"annual leave"` and rerun. Which snippets are
   retrieved now, and does the verification still pass? Explain using the scoring
   function.
2. Strengthen verification: for each cited snippet, check that every number in the
   sentence before the citation appears in that snippet's text.
3. Add a `department` field to each document and a `user_department` known to the tool
   (not passed by the model). Filter results so users only see their department's
   documents plus company-wide ones.
