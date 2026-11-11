# Course details (for registering the course in Tal's Academy)

- **Suggested slug:** `ai-engineer`
- **GitHub repo:** `tal-giladi/ai-engineer-course` (branch `main`)
- **Free or paid:** paid (suggestion; Tal decides)
- **Risk notice:** none. (Module 11 covers prompt injection and red-teaming, but only against the
  learner's own agent running locally.)
- **One-line summary:** Go from software engineer to shipping production AI systems: evals, RAG,
  agents, fine-tuning, serving and security, built as one portfolio project.
- **Who it is for:** experienced software engineers moving into AI engineering roles.
- **Format:** 12 modules, 53 lessons, each lesson with hands-on practice and a quiz; every module ends
  in a module quiz; one capstone system built across the whole course.
- **Prerequisites:** basic ML and LLM understanding (tokens, embeddings); Docker and local deployment;
  working with coding agents; system architecture and microservices; local Docker, a paid Hugging Face
  account, a paid Google Colab account (GPU for the fine-tuning modules) and API credit with at least
  one hosted model provider.

## What you will learn

- Put any model (hosted, Hugging Face or local) behind one OpenAI-compatible client, with cost and latency telemetry from day one.
- Get typed, schema-constrained output and handle failures with bounded repair and failure budgets.
- Build golden datasets, pick task-fit metrics, calibrate an LLM judge and gate changes in CI.
- Build production RAG: chunking, hybrid retrieval, reranking, grounded citations and refusal, measured with faithfulness and context metrics.
- Write an agent loop by hand, then port it to LangGraph with checkpoints, human approval and MCP.
- Decide when to fine-tune, then run LoRA/QLoRA SFT, DPO/ORPO and distillation and prove the result against a baseline.
- Serve with FastAPI and vLLM, trace agent runs, and meet a latency and cost SLO.
- Defend against prompt injection, scope tool permissions, handle PII, and present the system in interviews.
