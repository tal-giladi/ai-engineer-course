# Lesson 6.1 — Tool Calling: The Protocol Beneath Every Agent

**Module 6: The Agent Loop from Scratch** · ~12 min
**Objective:** M6-LO1 — Implement tool calling end to end — schema, dispatch, result formatting, parallel calls — against the raw API.

---

## Introduction

"Agentic AI" is the most heavily marketed term in this field, and underneath the marketing is a mechanism you can hold in your head in about five minutes. This lesson is that mechanism.

Getting it demystified matters for two reasons. Practically, you cannot debug what you think is magic. And in interviews, the difference between a candidate who says "we used an agent framework" and one who can describe the message sequence, explain where their code sits in it, and name the failure modes, is the difference between a screen and an offer.

The mechanism is simpler than the reputation. What is *not* simple — and what this lesson spends most of its time on — is designing tools the model actually uses correctly.

---

## Key Concept 1: The round trip

Tool calling is a structured output whose schema describes a function signature, plus a convention for returning the result.

Five steps:

**1. You declare tools** as JSON Schema alongside your messages.

```python
tools = [{
    "type": "function",
    "function": {
        "name": "search_docs",
        "description": "Search the product documentation. Returns "
                       "relevant excerpts with their source paths.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string",
                          "description": "Natural-language search query."},
                "section": {"type": "string",
                            "enum": ["api", "guides", "changelog"],
                            "description": "Optional: restrict to a section."},
            },
            "required": ["query"],
        },
    },
}]
```

**2. The model responds with a tool call** rather than text — a name and a JSON argument object.

**3. Your code executes it.** This is the step people get wrong conceptually. **The model does not run anything.** It emits a request. Your code decides whether to honour it, validates the arguments, calls the function, and handles errors. Every safety property in Lesson 6.5 lives in this step.

**4. You append a tool-result message** to the conversation, tagged with the call's ID.

**5. The model continues**, now with the result in context. It may answer, or call another tool.

That is the whole protocol. An agent is this in a loop.

---

## Key Concept 2: Tool arguments are untrusted structured input

The arguments arrive as a JSON object the model composed. Everything from Module 2 applies:

```python
class SearchDocsArgs(BaseModel):
    query: str = Field(max_length=500)
    section: Literal["api", "guides", "changelog"] | None = None

def dispatch(call: ToolCall):
    handler, arg_model = REGISTRY[call.name]
    args = arg_model.model_validate(call.arguments)   # raises on invalid
    return handler(args)
```

Enums, length limits, and range constraints in the schema prevent whole classes of bad call at generation time when the backend enforces schemas (Lesson 2.2), and validation catches the rest. In Lesson 6.5 and again in Module 11, this same validation becomes a security boundary — because a tool argument is the payload an injected instruction would use.

---

## Key Concept 3: Tool descriptions are prompt real estate

Here is where the actual engineering is, and it is consistently underestimated.

The model chooses a tool based almost entirely on its name, description, and parameter descriptions. Those strings are part of your prompt. They deserve the same care — and the same versioning from Lesson 2.4.

**Write descriptions that say when to use it, not just what it does.**

```diff
- "Searches documents."
+ "Search the product documentation for factual information about
+  features, APIs, and configuration. Use this before answering any
+  question about how the product works. Does not cover pricing or
+  account-specific data."
```

That "does not cover" clause is doing as much work as the rest.

**Fewer, well-separated tools beat many overlapping ones.** If two tools have overlapping purposes, the model will pick wrong a meaningful fraction of the time — and you will not be able to tell whether the failure was selection or execution. Three or four sharply-distinguished tools outperform ten fuzzy ones.

**Constrain arguments with enums and ranges.** Every free-text argument is a place for the model to be creative when you wanted determinism.

**Name things the way you would explain them.** `search_documentation` beats `query_index_v2`.

---

## Key Concept 4: Result formatting decides whether the agent survives

This is the failure mode that kills first agents, and it is invisible until it happens.

A tool returns the raw JSON from an internal API: 40,000 tokens of nested objects, most of it irrelevant. That result goes into the conversation. It stays there for every subsequent turn. By step three you have exhausted the context window (Lesson 1.3) and the agent fails in a way that looks nothing like "my tool returned too much data."

Rules:

- **Return what the model needs, not what the API returned.** Summarize, project, filter.
- **Cap the size.** A hard token limit per tool result, with explicit truncation the model can see: `[... 47 more results truncated ...]`.
- **Externalize large payloads.** Return a handle — `result_id: "r_8891", summary: "...", 312 rows"` — and provide a second tool to fetch specific parts. Lesson 6.3 develops this.
- **Format errors for a reader.** `"Search failed: index unavailable. Retry may succeed."` is actionable; a Python traceback is 400 tokens of noise.

---

## Key Concept 5: Parallel calls

Models can emit several tool calls in one turn. Executing them concurrently is a real latency win when the calls are independent.

Two hazards:

**Order dependence.** If call B depends on call A's result, the model should not have emitted them together — but it sometimes does. Execute concurrently only where your tools are genuinely independent; mark tools that must be serialized.

**Amplified side effects.** Three parallel writes are three writes. Parallelism multiplies the blast radius from Lesson 6.5, and rate limits apply per tool, not per turn.

---

## Worked Example 1: Descriptions and selection accuracy

One tool, three declarations, 40 queries requiring correct selection among four tools:

| Declaration | Selection accuracy |
|---|---|
| `search` / *"Searches."* | 52% |
| `search_docs` / *"Search the product documentation for factual information about features, APIs, and configuration."* | 81% |
| Same + *"Use before answering any question about how the product works. Does not cover pricing or account data."* + enum-constrained `section` | **94%** |

Forty-two points, no model change, no code change. The failures at 52% were mostly the model using `search` for pricing questions, which a different tool handled.

## Worked Example 2: Result formatting and context survival

The `search_docs` tool, two formats, measured over a 10-step agent run:

| Format | Tokens per call | Context at step 5 | Run outcome |
|---|---|---|---|
| Raw retriever output (full chunks, metadata, scores) | ~6,200 | 41,000 | context exhausted at step 7 |
| Projected (top 3 excerpts, source paths, capped at 1,200 tokens) | ~1,100 | 9,400 | completed at step 8 |

Same tool, same queries. One version cannot finish the task.

---

## Practice

**1. Wrap your RAG pipeline as a tool.** `search_docs`, with a Pydantic argument model, an enum-constrained section filter, and a projected result format capped at ~1,200 tokens.

**2. Add two more tools** that are genuinely different: a calculator and a clock work well, because "what is 15% of the enterprise tier limit?" and "is this changelog entry recent?" are questions retrieval cannot answer.

**3. Build the dispatcher.** A registry mapping tool name to (handler, argument model), with validation before execution and reader-friendly error formatting.

**4. Measure selection accuracy.** Write 30–40 queries where the correct tool is unambiguous but not trivial. Run and score. Include queries where the correct answer is *no tool* — a general question the model should answer directly — because over-eager tool use is a real failure mode.

**5. Improve one description and re-measure.** Add a "use this when" clause and a "does not cover" clause to your weakest tool. Report the delta.

**6. Measure result size.** Log tokens per tool result. If any tool can exceed 2,000 tokens, cap it and add explicit truncation. Then run one 10-step conversation and plot context growth — you want to see the shape before Lesson 6.3 asks you to control it.

**7. Version your tool definitions** in `spine/prompts/tools/`, and stamp the version into `CallRecord`. They are prompts.

---

## Visual Aid

*A message-sequence diagram with four vertical lifelines: User, Your Code, Model, Tool. User → Your Code: "question." Your Code → Model: "messages + tool schemas." Model → Your Code: "tool_call(search_docs, {query: ...})" — with a heavy horizontal bar drawn across Your Code's lifeline at this point, labeled in bold: "YOUR CODE DECIDES. Validate → authorize → execute. The model ran nothing." Your Code → Tool → Your Code: "result." Your Code → Model: "tool_result (capped at 1,200 tokens)." Model → Your Code: "final answer." A dashed loop bracket encloses steps 2–5 with the annotation "an agent is this, in a loop → Lesson 6.2."*

---

## Key Takeaways

- **Tool calling is a structured output plus a return convention.** The model requests; your code validates, authorizes, and executes. Nothing else runs anything.
- **Tool arguments are untrusted structured input.** Validate with Pydantic, constrain with enums and ranges — this is the same boundary that becomes a security control in Module 11.
- **Descriptions are prompt real estate and drive selection accuracy** by tens of points. Say when to use the tool and when not to; prefer few sharply-separated tools over many fuzzy ones.
- **Result formatting decides whether the agent survives a long task.** Project, cap, externalize large payloads, and format errors for a reader.
- **Parallel calls buy latency and multiply side effects.** Only for genuinely independent tools.

---

## Next

One tool call is a request and a response — ordinary integration work. An agent is what happens when you put that exchange in a loop and let the model decide when to stop.

**Lesson 6.2** writes that loop by hand, in about forty lines, and then interrogates every control decision those forty lines quietly make — including the one where the model and your code disagree about whether the task is finished.
