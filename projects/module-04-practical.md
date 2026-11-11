# Module 4 practical — Chunking a codebase for retrieval

This practical takes the chunking experiments from the module and moves them from prose to code. You index a real open-source codebase, compare three chunking strategies with everything else held constant, and explain in your own words why code behaves differently from documentation when you split it.

## Task

Index a mid-sized open-source codebase and measure retrieval recall across three chunking strategies:

1. Fixed-size chunks.
2. Function-boundary-aware chunks.
3. Function-boundary chunks with a contextual header carrying the file path and class name.

Keep the embedding model and the vector store the same for all three, and check chunk sizes, header tokens included, against your embedding model's maximum sequence length.

## Deliverable

`exercises/m4-code-retrieval/` with the recall table and a paragraph on why code chunking differs from prose chunking.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Controlled comparison | Embedding model and vector store held constant across the three strategies |
| Per-stratum reporting | Recall reported by query type, not only aggregate |
| Header isolation | The contextual-header effect measured on its own, as a single-variable change |
| Sequence limit respected | Chunk sizes checked against the embedding model's maximum, headers included |
| Reflection | Explains what differs about code |

<details><summary>Reviewer notes (spoilers)</summary>

Target reflection: code chunking differs because function and class boundaries are strong semantic boundaries that fixed-size splitting destroys; identifiers are exact tokens that favor BM25; and a function detached from its file path and class name is far less interpretable than a paragraph detached from its heading, so contextual headers matter more for code, not less.

</details>
