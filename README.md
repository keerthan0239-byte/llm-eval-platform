# LLM Evaluation & Observability Platform

A platform to test and monitor LLM applications**. It runs the same questions through multiple models (**GPT-4o, Claude, Llama), scores every answer for **accuracy, hallucinations, latency and cost, and exposes live metrics to Prometheus + Grafana dashboards with alerts. Production apps can also stream their LLM calls into it for real-time observability.

## Why
Teams shipping LLM features need to answer: *Which model is most accurate for our use case? Is it making things up? How fast and how expensive is it? Did a prompt or model change make things worse?* This platform answers those with automated, repeatable evaluation runs.

## Features
- Multi-provider evaluation** – OpenAI (GPT-4o), Anthropic (Claude), and any OpenAI-compatible server for Llama (Ollama, Groq, vLLM), configured in `models.yaml`.
- Automated scoring pipeline
  - Accuracy– an LLM-as-judge compares each answer to a reference answer, plus fuzzy text similarity.
  - Hallucination detection – the judge flags claims not supported by the given context; a rule-based check flags numbers that don't appear in the context.
  - Latency (average and p95), *token usage* and *cost* per model.
- Leaderboard– sort models by accuracy, hallucination rate, latency or cost.
- Production tracing – any app can `POST /traces` after each LLM call (see `examples/send_trace.py`).
- Observability – Prometheus metrics, a pre-built Grafana dashboard, and alert rules for high hallucination rate, low accuracy, slow responses and error spikes.
- Offline mode – a built-in `mock` model and heuristic judge so it runs and tests with no API keys.
- Deployment – Docker Compose (API + Prometheus + Grafana), Kubernetes manifests with a nightly evaluation CronJob, GitHub Actions CI.

## Architecture
```
            ┌──────────── models.yaml ────────────┐
Dataset ──▶ Runner ──▶ GPT-4o │ Claude │ Llama ──▶ Judge (LLM + heuristics)
 (JSONL)      │                                         │
              └──────▶ SQLite (runs, results, traces) ◀─┘
                              │
 Your LLM app ──POST /traces──┤
                              ▼
                  FastAPI /prometheus ──▶ Prometheus ──▶ Grafana dashboards + alerts
```

## Project structure
```
app/providers.py   one interface for OpenAI, Anthropic, Llama (OpenAI-compatible) and a mock model
app/metrics.py     prompts, LLM-judge parsing, fuzzy accuracy, hallucination heuristics
app/runner.py      evaluation pipeline: concurrent calls, scoring, summary statistics
app/storage.py     SQLite storage for runs, per-question results and traces
app/telemetry.py   Prometheus counters, histograms and gauges
app/main.py        REST API
app/cli.py         run evaluations from the terminal
datasets/          evaluation sets in JSONL (question, context, reference)
monitoring/        Prometheus config, alert rules, Grafana dashboard
k8s/               Deployment, Service, PVC and nightly eval CronJob
```

## Quick start (no API keys needed)
```bash
python -m venv venv && source venv/bin/activate     # Windows: venv\Scripts\activate
pip install -r requirements.txt
python -m app.cli --models mock --no-judge
```

## Compare real models
1. Copy `.env.example` to `.env` and add your OpenAI and Anthropic keys.
2. For Llama, install [Ollama](https://ollama.com) and run `ollama pull llama3.1` (free, runs locally).
3. Run:
```bash
python -m app.cli --dataset customer_support --models gpt-4o claude llama
```
Example output format:
```
Model           Accuracy   Halluc.    Avg ms    p95 ms      Cost $  Errors
gpt-4o             ...        ...        ...       ...         ...       0
```

## Full stack with dashboards
```bash
docker compose up --build
```
- API docs: http://localhost:8000/docs
- Grafana: http://localhost:3000 (dashboard "LLM Evaluation & Observability", login admin/admin)
- Prometheus alerts: http://localhost:9090/alerts

Start a run from the API:
```bash
curl -X POST localhost:8000/runs -H "Content-Type: application/json" \
  -d '{"dataset": "customer_support", "models": ["gpt-4o", "claude", "llama"]}'
curl localhost:8000/runs/<run_id>/leaderboard
```

## API
| Method | Path | Description |
|---|---|---|
| POST | `/runs` | Start an evaluation run (runs in the background) |
| GET | `/runs`, `/runs/{id}` | List runs / run summary (`?include_results=true` for every answer) |
| GET | `/runs/{id}/leaderboard` | Models ranked by `sort_by` (accuracy, hallucination_rate, avg_latency_ms, total_cost_usd) |
| POST | `/traces` | Report a production LLM call |
| GET | `/traces/summary` | Call count, average latency, total cost, success rate |
| GET | `/models`, `/datasets` | Available models and datasets |
| GET | `/prometheus` | Prometheus metrics |

## Add your own dataset
Create `datasets/<name>.jsonl`, one JSON object per line:
```json
{"id": "q1", "question": "...", "context": "facts the model is allowed to use", "reference": "the correct answer"}
```
Include questions whose answer is *not* in the context – they test whether a model admits it doesn't know instead of hallucinating.

## Deploy on Kubernetes (e.g. AWS EKS)
```bash
docker build -t llm-eval-platform:latest .
kubectl create secret generic llm-eval-secrets \
  --from-literal=OPENAI_API_KEY=sk-... --from-literal=ANTHROPIC_API_KEY=sk-ant-...
kubectl apply -f k8s/deployment.yaml
```
The CronJob re-runs the evaluation every night so quality regressions trigger Prometheus alerts.

## Tests
```bash
pytest -q
```
Tests use the mock model and an in-memory database, so they never call paid APIs.

## Design decisions
- LLM-as-judge + heuristics: judges catch meaning, heuristics are free and catch obvious errors like invented numbers; if the judge fails, scoring falls back to heuristics.
- temperature = 0 for reproducible comparisons.
- Concurrency limit (semaphore) keeps runs fast without hitting provider rate limits.
- Prices in config, not code, because they change often.
- SQLitekeeps setup simple; swap for PostgreSQL to run multiple replicas.

## Future improvements
- Web UI for side-by-side answer comparison
- Prompt version tracking and A/B tests
- RAG-specific metrics (context recall, answer relevance)
- Slack/email notifications from Alertmanager
