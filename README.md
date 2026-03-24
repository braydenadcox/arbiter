# Arbiter

> An intelligent inference routing engine that automatically selects the optimal LLM for every request using Thompson Sampling — balancing cost, latency, and task success across multiple providers.

---

## The Problem

AI teams blindly route every request to their best model and overpay massively. A simple question should never cost the same as a complex code refactor. No production system exists to automatically match request complexity to the right model and *learn* from outcomes over time.

Arbiter fixes this.

---

## How It Works

Arbiter sits between your application and every major LLM provider. You send one request. Arbiter decides which model handles it, scores the result, and updates its routing policy — getting smarter on every request.

```
Client → Arbiter → [Capability Filter] → [Thompson Sampling] → Provider → Score → Update Policy
```

The bandit learns. Simple requests get routed to cheap fast models. Complex requests get routed to powerful ones. You never overpay. You never underpay.

→ [View full system diagram](docs/diagrams/overview.md)

---

## Quick Start

```bash
git clone https://github.com/yourusername/arbiter
cd arbiter
cp .env.example .env        # add your provider API keys
docker compose up -d --build
```

The API is available at `http://localhost:8000`.

---

## Usage

### Send a request

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{
    "task_type": "code_exec",
    "prompt": "Write a function that reverses a string",
    "request_id": "abc-123",
    "constraints": {
      "max_cost_usd": 0.01,
      "max_latency_ms": 3000
    }
  }'
```

### Response

```json
{
  "response": "def reverse(s): return s[::-1]",
  "selected_arm": "gpt-4.1-mini",
  "execution_arm": "gpt-4.1-mini",
  "fallback_chain": [],
  "latency_ms": 420,
  "cost_usd": 0.0002,
  "scored": true,
  "success": true,
  "policy_version": "1.0.0",
  "scoring_version": "1.0.0"
}
```

---

## Supported Task Types

| Task type | Scoring method |
|-----------|---------------|
| `code_exec` | Executes in sandboxed Docker container — binary pass/fail |
| `json_extract` | Validates against expected schema — binary pass/fail |
| `factual_qa_eval` | Compares against known evaluation dataset — binary pass/fail |

Only scorable tasks are supported in v1. Unscorable requests are processed, marked as unscored, and excluded from policy updates.

---

## Provider Arms

| Arm | Cost tier | Capability tier |
|-----|-----------|-----------------|
| GPT-4.1 | High | High |
| GPT-4.1-mini | Low | Medium |
| Claude Sonnet | Medium | High |
| Claude Haiku | Low | Medium |
| Gemini 1.5 Pro | Medium | High |
| Gemini Flash | Low | Medium |

Each arm carries a versioned config preset: temperature, max tokens, timeout, retry policy, and response format constraints.

---

## Architecture

### Core loop

Every request follows the same path:

1. **Validate** — schema check, deduplication via `request_id`
2. **Capability filter** — hard removal of arms that cannot handle the task
3. **Cost filter** — remove arms exceeding `max_cost_usd`
4. **Thompson Sampling** — sample Beta(α, β) per eligible arm, select highest
5. **Execute** — provider adapter with timeout, retry, and fallback chain
6. **Score** — binary hard signal per task type
7. **Update** — atomic `HINCRBY` on Redis α/β, full audit log to Postgres
8. **Return** — response + full decision metadata to client

→ [View core loop diagram](docs/diagrams/core-loop.md)

### Bandit policy

Arbiter uses Thompson Sampling over a Beta distribution per arm per task type. Arms with more successes accumulate higher α and are sampled higher more often. Exploration happens naturally — no epsilon tuning required.

**Cold start:** Informed priors based on public benchmark data. Arms are not initialized equally — prior reflects known model capability differences.

**Non-stationarity:** Exponential decay (λ=0.99 per request) prevents old data from dominating as models update silently over time.

**Fallback attribution:** When a fallback occurs, the selected arm is marked `unscored`. Neither the selected nor the fallback arm receives a reward update. Learning data stays clean.

→ [View bandit policy diagram](docs/diagrams/bandit.md)

### Scoring

Reward is binary success only. Cost and latency are tracked separately and never mixed into the reward signal — this prevents the policy from learning to cheap out on quality.

| Task | Hard signal |
|------|------------|
| `code_exec` | Sandboxed execution — pass if exits 0 |
| `json_extract` | Schema validation — pass if valid |
| `factual_qa_eval` | Dataset comparison — pass if matches known answer |

### State

**Redis** holds live bandit policy state (α/β per arm per task type). All updates are atomic via `HINCRBY` — no race conditions under concurrent load.

**Postgres** holds the durable request log, full decision audit trail, and all cost/latency analytics used for baseline comparisons.

→ [View state management diagram](docs/diagrams/state.md)

### Fallback and circuit breaker

Each arm defines a timeout threshold, a single retry, and a circuit breaker that suppresses the arm after N consecutive failures. A background health probe runs every 30 seconds per provider — proactively suppressing dead arms before the circuit breaker is needed.

→ [View fallback and circuit breaker diagram](docs/diagrams/fallback.md)

### Code execution sandbox

`code_exec` tasks run in a pre-warmed pool of isolated Docker containers (minimum 3 idle). Each container enforces:

- 5 second execution timeout
- 128MB memory cap
- No network access
- No filesystem writes outside `/tmp`

On return, containers are sanitized (processes killed, `/tmp` wiped, environment cleared) and returned to the pool. Sanitization failure destroys the container and spins up a replacement.

Pool exhaustion queues requests (max depth: 50). If the queue is full, the API returns `503` with a `Retry-After` header. No silent drops.

→ [View sandbox diagram](docs/diagrams/sandbox.md)

### Provider abstraction

Each provider has an isolated adapter that normalizes request format, response format, token/cost reporting, error types, and timeout behavior. Provider-specific behavior never leaks into core routing logic.

→ [View provider abstraction diagram](docs/diagrams/providers.md)

---

## Versioning

Every major component carries an explicit version field logged with every request:

| Field | What it tracks |
|-------|---------------|
| `policy_version` | Bandit state and priors |
| `scoring_version` | Scoring logic per task type |
| `capability_map_version` | Arm eligibility per task |
| `config_preset_version` | Arm timeout/retry/format configs |

A policy version bump triggers a migration that archives existing Redis state to Postgres and resets α/β to the new informed priors. No silent state corruption.

Priors are reviewed quarterly. If benchmark delta exceeds threshold, a version bump is triggered.

---

## Benchmarks

Arbiter ships with a synthetic workload generator that runs 10k–100k requests across all task types and measures:

- **Success rate over time** — learning curve as the bandit converges
- **Arm selection distribution** — how routing shifts as policy learns
- **Cost reduction vs always-strongest baseline** — cost saved without sacrificing success rate
- **Routing overhead** — p50/p95/p99 latency added by Arbiter itself

Baselines compared: always GPT-4.1, always Gemini Flash, static heuristic router, Arbiter.

---

## Project Structure

```
arbiter/
├── docs/
│   └── diagrams/          # System design diagrams
│       ├── overview.md
│       ├── core-loop.md
│       ├── bandit.md
│       ├── fallback.md
│       ├── state.md
│       ├── sandbox.md
│       └── providers.md
├── src/
│   ├── api/               # FastAPI routes, schemas, middleware
│   ├── bandit/            # Thompson Sampling, Beta distributions, decay
│   ├── providers/         # Adapter per provider
│   ├── scoring/           # Hard scoring per task type
│   ├── sandbox/           # Container pool, sanitization, execution
│   ├── fallback/          # Circuit breaker, retry, health checks
│   ├── logging/           # Decision audit logging
│   └── config/            # Versioned arm configs, capability map
├── db/
│   ├── redis/             # Policy stats schema
│   └── postgres/          # Migrations, request log schema
├── benchmarks/            # Synthetic traffic datasets per task type
├── dashboard/             # Analytics dashboard (built after core loop)
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Environment Variables

```env
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=
REDIS_URL=redis://localhost:6379
DATABASE_URL=postgresql://user:password@localhost:5432/arbiter
REDACT_PROMPTS=true
LOG_DISABLE=false
SANDBOX_POOL_SIZE=3
SANDBOX_QUEUE_MAX=50
```

---

## Definition of Done (v1)

- Client can send a supported request with explicit task type
- Eligible arms are filtered correctly by capability and cost
- Bandit selects correctly via Thompson Sampling
- Real provider adapters execute with timeout and retry
- Fallback works and logs cleanly without corrupting policy
- Hard scoring runs correctly per task type
- Reward updates persist atomically to Redis
- Redis and Postgres remain consistent under concurrent load
- Cost, latency, and success metrics are queryable
- Baseline comparisons are visible in dashboard
- Router overhead stays under 50ms p95

---

## Known Limitations

- Only three task types supported in v1. Extension pattern is defined — new task types implement the `TaskType` interface.
- Benchmark evaluation set is fixed size. Results may not generalize to all prompt distributions.
- Informed priors are based on public benchmark data at time of release. Model performance drifts — priors are reviewed quarterly.
- Prompt content is redacted in logs by default. Full prompt logging can be enabled via `REDACT_PROMPTS=false` for development only.

---

## License

MIT
