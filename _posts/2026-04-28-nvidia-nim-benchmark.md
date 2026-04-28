---
layout: post
title: "I benchmarked 21 NVIDIA NIM free-tier models on real production AI-SRE workloads. Here's the architecture that emerged."
description: "5 wins, 0 losses for an unexpected model. 38% of the catalog returns 404. And the highest-impact change wasn't a model swap."
date: 2026-04-28
categories: [ai-infrastructure, llm-benchmark, sre]
tags: [nvidia-nim, llm, sre, openclaw, devstral, mistral-nemotron, qwen3-coder]
---


*5 wins, 0 losses for an unexpected model. 38% of the catalog returns 404. And the highest-impact change wasn't a model swap.*

---

For six months I've been running an AI-SRE pipeline in production at a fintech — automating Sentry triage, support-ticket classification, RCA generation, and Graylog query suggestion. The default analysis model was Mistral Nemotron on NVIDIA NIM's free tier, chosen after a 12-model benchmark in April. It works fine on synthetic prompts.

This week I tested it on real tickets from our ~6,000-entry resolved-issue knowledge base. The results changed the architecture.

> **TL;DR:** Devstral-2-123b — Mistral's "dev-focused" model — beat Mistral Nemotron 5-0 across diverse services with 71% vs 54% accuracy. Nemotron-3-Super-120b won on the hardest cross-service tickets. Qwen3-Coder-480b produced the cleanest Elixir code review of any free-tier model. 38% of NIM's catalog returns 404. And the single highest-impact change wasn't a model swap — it was prompt engineering.

This article walks through 7 passes of testing across 21 model probes, the failure modes that matter for production, and the 4-tier architecture that emerged.

**Full per-model raw outputs, per-ticket grading rubrics, and methodology data:** [pranavj17.github.io/2026/04/28/nvidia-nim-benchmark](https://pranavj17.github.io/2026/04/28/nvidia-nim-benchmark/)

---

## The system under test

The pipeline I'm benchmarking against runs every 3 minutes during business hours:

```
[Slack/Sentry alert] OR [support ticket created]
        ↓
  bash dispatch (system crontab)
        ↓
  Routing model: gemma4:latest on local Ollama (Mac Mini M4 Pro)
        ↓
  Bash pre-fetch: Metabase + Sentry + Graylog + GitLab APIs
        ↓
  Analysis model: Claude Sonnet 4.6 (subscription, 1-turn)
        ↓
  Output: ticket comment + Slack post + (rare) auto-fix MR
```

This processes about 50 tickets/day across 7 internal services. The architecture pre-fetches all evidence in bash before passing a single 1-turn call to Claude — chosen because Claude was the strongest analyst and we wanted to minimize subscription budget consumption (15 calls per 5 hours).

**The question I was answering: could free NIM models replace any of these tiers without quality loss?**

---

## Methodology

I designed seven passes of increasing rigor, each addressing a specific architectural question:

1. **Synthetic latency** — which models are fast enough?
2. **Real KB simple** — do they get production tickets right?
3. **Real KB complex** — what happens on hard cross-service problems?
4. **Tool calling** — which support agentic workflows?
5. **Frontier + specialized** — what about the largest and the niche models?
6. **Coding-specific** — which produces production-ready code?
7. **Routing comparison** — does swapping the local routing model help?

Each pass used representative real workloads, not synthetic benchmarks. Where ground truth existed (closed tickets with human-resolved RCA), I scored model output against it on a points rubric. All tests ran from the same Mac Mini against the same NIM endpoint with the same auth key.

---

## Pass 1 — Synthetic latency baseline

A single SRE-style prompt with ~140 input tokens, max_tokens=1024, run against 8 candidate models.

The top four returned in under 4 seconds:

- **mistralai/devstral-2-123b-instruct-2512** — 2.7s, 134 tokens
- **meta/llama-4-maverick-17b-128e-instruct** — 2.9s, 186 tokens
- **mistralai/mistral-nemotron** — 3.3s, 150 tokens (production baseline)
- **qwen/qwen3-coder-480b-a35b-instruct** — 3.5s, 142 tokens

The slow tier:

- **qwen/qwen3-next-80b-a3b-instruct** — 11.0s
- **nvidia/nemotron-3-super-120b-a12b** — 12.5s, 415 tokens (reasoning-heavy)

**The first red flag:** two models in the catalog timed out completely at 180 seconds — `mistralai/mistral-medium-3-instruct` and `deepseek-ai/deepseek-v4-flash`. This was the early signal that catalog presence ≠ usable inference.

Synthetic prompts couldn't distinguish the top four. I needed real tickets to see real differences.

---

## Pass 2 — Real KB tickets, head-to-head (Devstral vs Nemotron)

I pulled 5 closed tickets from our knowledge base, picked for service diversity:

- **Ticket A** (portfolio service): Mutual fund misclassified as external after broker-registration change
- **Ticket B** (auth service): Mobile-number login redirects to signup (orphan account)
- **Ticket C** (member + onboarding): Account stuck "initiated" — workflow not triggered after document upload
- **Ticket D** (CRM + notifications): Survey segment targeting bug
- **Ticket E** (CRM + WMS sync): Mapping silently dropped for disabled accounts

Each model received only the ticket description — no schema, no root cause, no fix. The ask: (1) likely root cause in 2–3 sentences, (2) recommended fix, (3) one SQL or investigation step. Then I graded each output against the actual ground truth on a 9–12 point rubric.

### The result: Devstral 5–0

> **Across 48 graded points: Devstral 71% vs Mistral Nemotron 54% — a 17 percentage-point gap.**

Devstral won every single ticket. Latency cost: about 18% slower per call (3.8s avg vs 4.5s avg) for 17 percentage points more accuracy.

A 5–0 sweep is unusual. What was Nemotron actually doing wrong?

### Failure mode 1 — Confidently hallucinating database columns

On the portfolio-service ticket, Nemotron's verification SQL referenced a column `is_external` that doesn't exist in our schema, and joined against a fictitious `fund_master.fund_family` table that also doesn't exist:

```sql
-- Nemotron's verification SQL:
SELECT COUNT(*) FROM portfolio_db.investments
WHERE folio_number IN (...) AND is_external = true
AND fund_code IN (
  SELECT fund_code FROM portfolio_db.fund_master
  WHERE fund_family IN (...)
);
```

Devstral's version was closer to real schema — it confused a column *value* for a column name, but stayed in the realm of plausible:

```sql
-- Devstral's verification SQL:
SELECT folio_number, scheme_name, classification_flag, broker_code
FROM portfolio_db.investments
WHERE folio_number = '...'
  AND classification_flag = 'external';
```

A human running Devstral's query finds the discrepancy in seconds and adjusts. Running Nemotron's query fails completely against an entirely fictional table.

> **For autonomous workflows where output gets piped to a SQL executor, this is the difference between "useful triage" and "broken pipeline."**

### Failure mode 2 — SQL filters that hide the bug

On the auth login-redirect ticket, the ground truth was that a customer's mobile number was associated with an *orphan duplicate auth user* — by definition not active.

Nemotron's investigation SQL:

```sql
WHERE mobile = '...' AND status = 'active';
```

The `status = 'active'` filter is the bug. This query returns zero results and leads the human investigator down the wrong path.

Devstral's version:

```sql
WHERE mobile = '...' OR mobile LIKE '%...';
```

No status filter. Would surface the orphan. Would also catch country-code formatting variants.

### Failure mode 3 — Vagueness vs spontaneous architectural insight

On the CRM/WMS sync ticket, both models received only that the ticket described "RM mapping doesn't reflect in CRM despite re-mapping" with the disabled-account symptom and "no errors in Graylog."

Nemotron suggested generic remediation:

> "Manually trigger a sync for the affected client and monitor CRM logs for any hidden validation errors or pipeline failures."

Devstral named specific architectural elements without being told the architecture:

> "Run a manual sync for the affected client via the CRM integration tool with verbose logging enabled, then inspect the response and any intermediate service logs (e.g., **Kafka, ETL jobs**) for hidden errors or mismatched data formats."

Our actual architecture *does* use Kafka for cross-service events. The model inferred it from "WMS → CRM sync with silent drops" — that's the kind of intuition senior SREs develop over years.

---

## Pass 3 — Complex multi-service scenario

A harder ticket: an account-split operation propagated email update to auth, CRM, WMS, and folio main records — but **not** to the externally-registered folio email used by the withdrawal pipeline. Result: withdrawal validation fails on stale email.

Five models tested on a 15-point rubric across trigger identification, gap location, immediate fix, long-term fix, and investigation quality.

The winner this time wasn't Devstral:

- **nvidia/nemotron-3-super-120b-a12b** — **14/15 (93%)** at 24.6s. Cleanest analysis. Best long-term fix.
- **mistralai/devstral-2-123b** — 13/15 (87%) at 7.6s. Direct hit on ground truth. Best speed/quality balance.
- **minimaxai/minimax-m2.7** — 12.5/15 at 88.5s. Strong analysis, but latency unusable for cron.
- **mistralai/mistral-nemotron** — 11/15 (73%) at 10.2s. Captures sync gap but doesn't pinpoint specific data flow.
- **qwen/qwen3-next-80b-a3b-thinking** — **2.5/15 (failed)** at 16s. Thought aloud for 1500 tokens, ran out of budget before producing the structured answer.

### Three findings on complex scenarios

> **Reasoning models DO win on hard problems.** The slow tier earns its latency.

But the latency cost is steep. 24.6s vs 7.6s. For 30-min cron triage, Devstral still wins on practical grounds. Nemotron-3-Super is viable only as tier-2 escalation for the 1–2 hardest tickets per day.

**Thinking models need bigger token budgets.** Qwen3-thinking emitted 1500 tokens of visible reasoning and ran out before producing the structured answer. If you use thinking models, set max_tokens >= 4096 minimum.

**MiniMax M2.7 is unusable on cron at 88s** — but interestingly, when delegated to tool-calling instead of analytical mode, the same model returned in 2.9s.

---

## Pass 4 — Tool calling support

OpenAI-compatible tool calling on free NIM. Three tool definitions: `query_graylog`, `query_metabase`, `search_kb`. The prompt: "investigate this Sentry alert."

Seven of eight models worked correctly. Best two chained two tool calls in a single response:

- **mistralai/devstral-2-123b** — 2 chained calls in 1.5s (`search_kb` + `query_graylog`)
- **minimaxai/minimax-m2.7** — 2 chained calls in 2.9s

Single-call but solid:

- **qwen/qwen3-coder-480b** — 1.5s, sophisticated query syntax
- **mistralai/mistral-nemotron** — 2.6s
- **meta/llama-3.3-70b-instruct** — 2.1s
- **nvidia/nemotron-3-super-120b** — 2.4s

And one slow:

- **nvidia/llama-3.3-nemotron-super-49b-v1.5** — 13.4s, 381 reasoning tokens before the call

### The Llama-4-Maverick gotcha

> **Llama-4-Maverick on NIM is silently broken for tool calling.**

The model *appears* to attempt a tool call by emitting JSON in the `content` field:

```json
{
  "type": "function",
  "name": "query_graylog",
  "parameters": {"stream_id": "...", "query": "..."}
}
```

But this is in the `content` field, not the proper `tool_calls` structure. Code that does:

```python
response.choices[0].message.tool_calls  # → []
```

silently gets an empty list. The model "knows" it should call but doesn't follow the OpenAI tool-calling protocol correctly.

This is the kind of bug that takes weeks to diagnose in production — looks like the model's lazy or confused, when actually the protocol is broken. Don't use Llama-4-Maverick on NIM for tool-calling pipelines without a JSON parser fallback.

### Why Devstral's chained calls matter

Without being asked, Devstral called both `search_kb` AND `query_graylog` in the same response. That's how a senior SRE actually thinks: "let me check past tickets, AND look at logs." Most models call one tool, wait for the result, then call the next. Devstral predicting both upfront is a meaningful agent-loop quality.

---

## Pass 5 — Frontier and specialized models

Six untested candidates. The results:

**Worked but not worth it:**
- **mistralai/mistral-large-3-675b** at 9.7s — 2x slower than Devstral, similar quality
- **deepseek-ai/deepseek-v4-pro** at 55.7s — strong analysis but unusable cron latency

**Worked and useful:**
- **sarvamai/sarvam-m** at 29.4s — translated mixed Hindi/Tamil/English correctly. Strong domain fit.
- **nvidia/nv-embedcode-7b-v1** at 1.3s — code-specific embeddings, 4096-dim, semantically correct clustering. Production-ready.

**Listed but not actually deployed on free tier:**
- **writer/palmyra-fin-70b-32k** — returns 404 "Function not found for account"
- **nvidia/gliner-pii** — no working endpoint discovered (NIM container only, not free-tier hosted)

### The Indian-language finding

The customer base writes tickets mixing English with Hindi, Tamil, Bengali, Marathi. Generalist models often misclassify these. Sarvam-m correctly translated and classified a representative ticket:

> "Sir mera SIP ka amount change nahi ho raha hai... வருகிற மாதம் தான் debit ஆகணும்..."
> → "Customer cannot update SIP amount; debit must reflect for upcoming month."

29-second latency is acceptable for the small subset of tickets that actually need it. Production wrapper must strip `</think>` tags — Sarvam-m emits visible reasoning.

---

## Pass 6 — Coding deep-dive

A representative production-shaped Elixir module with four bugs (nil safety, idempotency, query efficiency, error handling). Eight code-relevant models tested.

The module under review:

```elixir
defmodule MyApp.KycService do
  alias MyApp.{Repo, User, KycSubmission}
  import Ecto.Query

  def submit_kyc(user_id, kyc_data) do
    user = Repo.get(User, user_id)
    address = user.address.line1

    submission = %KycSubmission{
      user_id: user_id,
      pan: kyc_data["pan"],
      address: address,
      status: "pending"
    }

    Repo.insert(submission)
  end

  def check_kyc_status(user_id) do
    submissions = Repo.all(from k in KycSubmission, where: k.user_id == ^user_id)
    submissions |> Enum.map(fn s -> {s.id, s.status} end) |> List.last()
  end
end
```

The four bugs to find: nil crash on `user.address.line1`, no idempotency, inefficient query, no error handling on `Repo.insert`.

### Results

**Winner — qwen/qwen3-coder-480b-a35b-instruct** scored 14/15 in 22.2 seconds. Cleanest idiomatic Elixir of any model tested.

The kind of code only senior Elixir engineers write:

```elixir
def submit_kyc(user_id, kyc_data) when is_integer(user_id) and is_map(kyc_data) do
  case Repo.get(User, user_id) do
    nil -> {:error, :user_not_found}
    user ->
      with {:ok, address} <- extract_address(user),
           {:ok, pan} <- validate_pan(kyc_data),
           {:ok, submission} <- find_or_create_submission(user_id, pan, address) do
        {:ok, submission}
      else
        error -> error
      end
  end
end

defp extract_address(%{address: %{line1: line1}}) when is_binary(line1), do: {:ok, line1}
defp extract_address(_), do: {:error, :invalid_address}
```

Pattern matching with guards. `with` chain for railway-oriented programming. defp helpers for separation of concerns. Consistent `{:ok, _}` / `{:error, _}` tuples.

Other results:
- **mistralai/devstral-2-123b** — 11/15 at 9.8s. Solid, but a subtle bug in its own fix.
- **qwen/qwen2.5-coder-32b** — 9/15 at 105s. Pattern-matched on a string key for an Ecto struct (atom keys) — would fail at runtime.
- **mistralai/mistral-nemotron** — 7/15 at 11.3s. Found 8 bugs but introduced 3 new ones in the fix.

**Four other code-specialist models — codestral-22b, granite-34b-code, codellama-70b, codegemma-7b — all returned 404. Not actually deployed on free tier.**

### Mistral Nemotron's dangerous code mistakes

Nemotron found *more* bugs than any other model — but its fix introduced three new ones:

```elixir
unless Map.has_key?(kyc_data, "pan") and user.address.line1 do
  raise "Missing required KYC data"
end

existing = Repo.one?(from k in KycSubmission, ...)  # Repo.one? doesn't exist

%KycSubmission{
  status: :pending  # atom...
}

# But the existing-check filtered:
where: k.status == "pending"  # ...string. Always misses.
```

`Repo.one?` doesn't exist in Ecto. `user.address.line1` still crashes on nil even inside the `unless`. The status field is an atom but the existing-check filters by string.

> **For autonomous code-generation workflows, this is the worst possible failure mode: confident, plausible, broken.**

A reviewer skimming would approve. CI catches it eventually but burns time and budget.

---

## Pass 7 — Routing model

Routing is the first step — classifying tickets into service / type / context — and runs hundreds of times a day. Latency matters more than for any other tier. Currently I use `gemma4:latest` on local Ollama. Tested against `upstage/solar-10.7b-instruct` (NIM):

- **ollama/gemma4:latest** (current production) — 5.5s, 8/15 (53% accuracy), local + unlimited
- **upstage/solar-10.7b-instruct** — 8.0s, 6/15 (40% accuracy), NIM with 40 RPM cap

Gemma4 wins on every axis. Solar offers nothing meaningful.

### The taxonomy insight

Both models scored under 60%. The failure pattern was the same: **neither knows the service taxonomy.** Pure pattern-matching gets you 50% accuracy. Adding ~7 lines of taxonomy to the prompt would likely lift the same model to 80%+:

```
Service taxonomy (example):
- Portfolio service: investments / folios / external classification
- Onboarding service: KYC / new account workflows
- Member service: profiles / email / account state
- Auth service: login / OTP / mobile / session
- Notifications service: email / SMS / push dispatch
- CRM: relationship-manager mapping / CRM-WMS sync
- Order service: orders / withdrawals / redemptions
```

> **Same model. Same latency. ~30 percentage point accuracy lift. Free.**

---

## The phantom-catalog finding

After 7 passes covering 21 distinct model probes, eight models in NVIDIA's `/v1/models` listing were not actually inferentially available on free tier:

- **mistralai/mistral-medium-3-instruct** — TIMEOUT 180s
- **deepseek-ai/deepseek-v4-flash** — TIMEOUT 180s
- **writer/palmyra-fin-70b-32k** — 404 "Function not found for account"
- **nvidia/gliner-pii** — no working endpoint discovered
- **mistralai/codestral-22b-instruct-v0.1** — 404
- **ibm/granite-34b-code-instruct** — 404
- **meta/codellama-70b** — 404
- **google/codegemma-7b** — 404

**8 of 21 = 38% catalog attrition.**

This is a much bigger gap than "occasional phantom listings." It's structural. The `/v1/models` endpoint behaves like a "previously available or potentially available" list, not "currently deployed."

> **Production fallback chains MUST probe each model before relying on it.** Don't trust catalog presence.

---

## The synthesis: a 4-tier production architecture

After all 7 passes, the architecture that emerged from the data:

- **Router** (ticket → service+type) → `ollama/gemma4:latest` (local, current) plus taxonomy in prompt. Local, free, unlimited.
- **Triage analyst** (default, ~80% of tickets) → `mistralai/devstral-2-123b-instruct-2512` (NIM, free). 5-0 vs Nemotron, 4.5s, supports tool calling.
- **Hard-ticket analyst** (cross-service, complex) → `nvidia/nemotron-3-super-120b-a12b` (NIM, free). 14/15 on the complex scenario. 24.6s acceptable for tier-2 escalation.
- **Code generator** (auto-fix MR) → `qwen/qwen3-coder-480b-a35b-instruct` (NIM, free). Best Elixir code review at 14/15.
- **Embeddings** (KB / RAG) → `nvidia/nv-embedcode-7b-v1` (NIM, free). Code-trained, 4096-dim.
- **Indian-language pre-router** → `sarvamai/sarvam-m` (NIM, free, conditional branch).
- **Tool-calling agent (v3 architecture)** → `mistralai/devstral-2-123b` (NIM, free). Chains 2 tool calls in 1.5s.

> **All free except local M4 Pro electricity. Total monthly inference cost: $0.**

Compare to running an equivalent stack on Anthropic+OpenAI APIs: $200–1000/month for similar throughput.

### What gets deprecated

- **Mistral Nemotron** — hallucinates DB columns and SQL functions. Replace with Devstral immediately.
- **Llama-4-Maverick on NIM** — broken tool-calling protocol. Avoid in agent loops.
- **Phantom listings** (codestral, granite-code, codellama, codegemma, palmyra-fin, gliner-pii) — design fallback chains around the 13 confirmed-working models, not the 21 catalog entries.

---

## The meta-finding: prompt engineering > model selection

Three of the seven passes pointed at the same conclusion:

1. **Routing accuracy** — 53% with no taxonomy → ~80% with 7-line taxonomy added
2. **SQL accuracy** — every model hallucinates columns; service-context files solve this for everyone
3. **Indian-language tickets** — even sarvam-m benefits from one-shot examples in prompt

> **Adding domain context to existing prompts almost always beats swapping models.** It's faster to ship, costs nothing in latency or money, and works the same way regardless of which model is behind the API.

The Q1 2026 LLM landscape is shifting underneath production systems weekly — DeepSeek V4, Llama 4, MiniMax M2.7, Mistral Large 3, Nemotron 3 family all dropped in the last 90 days. **Engineering teams that re-benchmark when they have a problem are 60+ days out of date.** A quarterly re-bench takes a Sunday.

---

## Caveats — what would strengthen this

**5-ticket benchmarks are small.** Adding 5 more tickets per pass would harden the conclusions, especially the 5-0 sweep.

**Self-graded scoring has bias.** A blind grade by another engineer (or a method that doesn't know which model produced which output) would be more rigorous.

**No schema context in prompts.** Adding actual service schema files would lift everyone's accuracy. The relative gaps might compress, widen, or stay — that's worth measuring.

**Latency variance under load.** Devstral hit a 27-second outlier on one test. Free-tier latency can spike. Production stability needs more validation before swapping.

---

## What I'm doing next

1. Adding service taxonomy to the routing prompt — measuring the 53% → 80%+ lift
2. Side-by-side test of `nv-embedcode-7b-v1` vs `nomic-embed-text` on the actual KB
3. Documenting the 4-tier architecture in handover notes
4. Prototyping a tool-calling agent loop — replacing bash pre-fetch with `Devstral with tools → multi-turn loop`

If you've benchmarked LLMs against real production tickets, or built free-tier-only AI infrastructure, I'd love to compare notes — especially on latency variance under sustained load and on prompt-engineering vs model-selection trade-offs.

---

*Full per-ticket grading rubric, methodology data, and per-model raw outputs at* [*pranavj17.github.io/2026/04/28/nvidia-nim-benchmark*](https://pranavj17.github.io/2026/04/28/nvidia-nim-benchmark/) *— rendered tables, browsable structure.*
