# WhatsApp Order-Support Agent

**A multi-tenant, production-shaped LLM support agent for a WhatsApp-first D2C platform.**

Simulates two real online stores (a skincare brand and a footwear brand) sharing one agent stack, handling order lookups and policy Q&A over a WhatsApp-style chat UI — with the guardrails, observability, and cost tracking a real deployment would need, not just a happy-path demo.

**Runs with zero API keys.** Clone it, run three commands, see it fully working in under 2 minutes.

---

## Why this project exists

Most LLM agent demos are a chat completion call with a system prompt. This one is built the way a support agent talking to real customers about real orders actually needs to work:

- It **can't hallucinate an order status or a discount code** — every factual claim is checked against the tool result that grounds it before it's allowed to reach the customer.
- It **can't leak one brand's data into another's conversation** — tenant isolation is enforced structurally, before retrieval even runs, not as an afterthought filter.
- It **knows what it costs to run** — every turn is traced with token counts and dollar cost, aggregable into cost-per-conversation numbers.
- It **catches its own regressions** — a 13-case eval suite checks intent routing, RAG grounding, and adversarial guardrail behavior automatically.
- It **fails safely** — a hard timeout and deterministic fallback mean a slow or broken model never leaves a customer hanging or gets a made-up answer.

## What it demonstrates

| Capability | Where to look |
|---|---|
| Agent architecture — tool calling, explicit node boundaries | [`app/agents/graph.py`](app/agents/graph.py) — LangGraph `StateGraph`: router → policy_qa / order_status / discount_request / escalate |
| Deciding LLM call vs. deterministic rule | `discount_request` node is a fixed business rule, **not** an LLM call — see comments in `graph.py` |
| RAG pipeline: chunking, embedding, retrieval | [`app/rag/`](app/rag/) — heading-aware chunking, pluggable embeddings (offline TF-IDF or OpenAI), hybrid scoring |
| Multi-tenant data isolation as a security property | Hard tenant filter *before* similarity search — [`vector_store.py`](app/rag/vector_store.py), regression-tested in [`tests/test_rag.py`](tests/test_rag.py) |
| Debugging retrieval failures | [`docs/retrieval_postmortem.md`](docs/retrieval_postmortem.md) — two real bugs found and root-caused while building this |
| Input/output guardrails, prompt injection defense | [`app/guardrails/`](app/guardrails/) |
| Preventing hallucinated order/discount claims | Output grounding check — [`output_guard.py`](app/guardrails/output_guard.py) |
| Deterministic fallback for model failure | [`app/agents/runner.py`](app/agents/runner.py) — timeout-wrapped graph execution |
| Tracing, evals, regression suites | [`app/observability/tracing.py`](app/observability/tracing.py), [`eval/`](eval/) |
| Cost consciousness, model-tier routing | [`eval/cost_report.py`](eval/cost_report.py), `get_llm(tier=...)` in [`app/llm/client.py`](app/llm/client.py) |

## Quick start

```bash
git clone <your-repo-url>
cd wa-support-agent

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
cp .env.example .env

python -m app.rag.build_index
uvicorn app.main:app --reload --port 8000
```

Open **http://127.0.0.1:8000** for the demo chat UI — tenant switcher, one-click sample messages (order lookup, policy question, a discount-manipulation attempt, a prompt-injection attempt).

No API key needed by default — it runs on a small deterministic mock model (`LLM_PROVIDER=mock`) wired through the *exact same* agent graph, guardrails, RAG pipeline, and tracing a real model would use. Set `LLM_PROVIDER=openai` and an `OPENAI_API_KEY` in `.env` to run it against real GPT models — no other code changes needed.

## Verify it yourself

```bash
python -m pytest tests/ -v        # 12 unit tests: guardrails + tenant isolation
python -m eval.run_eval           # 13-case eval suite: routing, RAG, adversarial cases
python -m eval.cost_report        # cost per conversation / per tenant / model-routing comparison
```

## Project structure

```
app/
  agents/graph.py        # LangGraph agent — the architectural core
  agents/runner.py        # tracing + timeout + deterministic fallback wrapper
  rag/                    # chunking, embeddings, per-tenant vector store
  guardrails/             # input (injection/PII) and output (grounding) checks
  llm/client.py           # mock / OpenAI backends behind one interface, cost accounting
  observability/tracing.py # structured JSONL trace logging
  tools/order_lookup.py    # mock order API, tenant-scoped
  main.py                 # FastAPI app

data/tenants/<brand>/     # per-tenant mock orders + policy knowledge base
eval/                     # eval set, eval runner, cost report
tests/                    # pytest suite
frontend/index.html        # WhatsApp-style demo UI
docs/
  architecture.md          # design decisions and tradeoffs, explained
  retrieval_postmortem.md   # two real retrieval bugs, found and fixed
```

