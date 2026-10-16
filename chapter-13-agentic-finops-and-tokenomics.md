# Chapter 13: Agentic FinOps & Tokenomics

[← Previous: Chapter 12 — Agentic Security, Identity & Sandboxing](./chapter-12-agentic-security-identity-and-sandboxing.md) | [Book Index](./README.md) | [Next: Chapter 14 — Multimodal, Computer-Use & Streaming Agents →](./chapter-14-multimodal-computer-use-and-streaming-agents.md)

---

A single-turn chatbot might consume ~1,000 tokens per request. An unoptimized multi-agent workflow with 4 sub-agents, 60 bound tool schemas, and 8 reasoning turns can easily burn well over 100,000 tokens on a single customer ticket. Every turn re-sends the system prompt, every tool schema, and the ever-growing history. The example isn't contrived. Anthropic reported (*"How we built our multi-agent research system"*, June 2025) that its agents used roughly 4× more tokens than chat interactions, and its multi-agent research system roughly 15× more.

This chapter is about cost, latency, and throughput engineering for agents. In practice that means breaking an agent's spend down into a **Token Bill of Materials (TBOM)**, driving down **Cost per Validated Outcome (CpO)** without giving up accuracy, and building inference pipelines that stay fast under load.

TBOM and CpO are this book's working terms, not industry-standard acronyms. The ideas behind them (cost decomposition, and cost per successful task) are ordinary FinOps practice.

---

## 13.1 Beginner: what an agent run actually costs

### Price per million tokens is a vanity metric
Picking a model because its API price per 1M tokens is 30% lower often raises your total bill. If the weaker model needs 4 extra retry turns, calls redundant tools, or fails validation 25% of the time, the savings disappear.

The number that matters is **Cost per Validated Outcome ($\text{CpO}$)**. The numerator includes the cost of every run, failed ones included. The denominator counts only the runs that pass verification:

$$\text{CpO} = \frac{\sum_{\text{all runs}} \left(\text{Uncached Input} + \text{Cache Writes} + \text{Cache Reads} + \text{Thinking/Output} + \text{Tool/Search API}\right)}{\text{Count of Runs Passing Trajectory + Outcome Verification}}$$

### The six parts of a token bill
Every agent invocation has six distinct token cost components:

```mermaid
flowchart LR
    subgraph TBOM["The 6 Components of an Agentic Token Bill of Materials (TBOM)"]
        T1["1. Static System Prefix<br/>(System Prompt + Core Rules)"]
        T2["2. Tool & Skill Schemas<br/>(Re-sent on every one of K turns)"]
        T3["3. Retrieved RAG / Memory<br/>(Chunks injected per step)"]
        T4["4. Cumulative Turn History<br/>(Re-sent each turn: O(K^2) total tokens if uncompacted)"]
        T5["5. Reasoning / Thinking Tokens<br/>(Billed at the output-token rate)"]
        T6["6. Retry / Critic Loops<br/>(Failed attempts & multi-agent hops)"]
    end
```

Two of these hide multipliers that surprise people.

> [!WARNING]
> Thinking tokens are billed as output tokens. All major providers do this: OpenAI reports them as `reasoning_tokens` inside output usage, Gemini reports `thoughts_token_count`, and Claude bills extended thinking as output tokens (for models that return *summarized* thinking, you still pay for the full thinking tokens generated). Output tokens commonly cost ~4×–8× more than input tokens. A thinking budget of 8,000 tokens per step, if fully used across a 6-step ReAct loop, bills 48,000 output tokens, which can easily dominate the dollar cost. A budget is a cap, not a target, so measure actual usage.

The second multiplier is long-context surcharges. Some models charge a higher per-token rate once a prompt crosses a size threshold. Gemini 2.5 Pro, for example, prices prompts above 200K tokens at a higher tier, as did the 1M-context beta of Claude Sonnet 4/4.5. Check your provider's price sheet, because unbounded history can quietly push every call into the expensive tier.

### Three latency metrics
1. **TTFT (Time to First Token)** is queueing time plus prefill compute. Prefill scales with the *uncached* input length, so a cache hit on a long prefix skips most of that work.
2. **TPOT (Time Per Output Token)**, also called ITL (Inter-Token Latency), is dominated by memory-bandwidth-bound autoregressive decode. It scales with the model's *active* parameter count (for Mixture-of-Experts models, active rather than total parameters), KV-cache size, and server load.
3. **TCT (Task Completion Time)** is the end-to-end wall-clock latency across all $K$ serial LLM calls and external tool executions:
   $$\text{TCT} = \sum_{k=1}^{K} \left(\text{TTFT}_k + N_{\text{out}, k} \cdot \text{TPOT}_k + L_{\text{tool}, k}\right)$$
   Note that $N_{\text{out},k}$ includes thinking tokens, so hidden reasoning adds directly to wall-clock time. The formula assumes strictly serial steps. With parallel tool calls or parallel sub-agents, each parallel group contributes the latency of its *slowest* branch (a `max`, not a sum). That's why fan-out is a latency lever as well as a cost risk.

---

## 13.2 Intermediate: six levers for cutting cost

Used together, the six architectural levers below can cut CpO substantially (teams frequently report reductions well above 50%), and they often improve end-to-end latency at the same time. Treat the ranges in the table as illustrative orders of magnitude, not guarantees. Measure them on your own traces.

| Lever | How it works | Illustrative CpO impact | Latency impact |
| :--- | :--- | :---: | :---: |
| 1. Prefix (prompt) caching | Keep `[Tool Schemas + System Prompt + Static Docs]` byte-identical at the *front* of every request and keep history append-only; never inject timestamps/UUIDs or re-order JSON keys in the prefix. | Cached input typically billed at ~10%–50% of the base input price (provider/model dependent) | Large TTFT reduction on long prefixes |
| 2. Model tiering and routing | Route simple/FAQ traffic to small, fast models (mini/Haiku/Flash-class, or small open-weight Llama/Qwen/Mistral models); use frontier models *only* for Planner/Judge nodes and small models for Step Executor loops. | Often the single largest saving on total spend | Typically faster TPOT on small models |
| 3. Adaptive thinking budgets | Use the lowest reasoning setting that passes your evals for classification/tool routing, and reserve high settings for complex planning. Parameter names differ by provider: OpenAI `reasoning_effort` (`minimal`/`low`/`medium`/`high`, `none` on some models), Claude extended thinking `budget_tokens` (minimum 1,024) or disable it, Gemini 2.5 `thinking_budget` (`0` disables thinking on Flash), Gemini 3 `thinking_level` (`"low"`/`"high"`, more levels on some models), and open-weight hybrid reasoners such as Qwen3 expose an `enable_thinking` switch in their chat template. | Large savings on output tokens | Saves seconds per turn |
| 4. Progressive tool and skill disclosure | Lazy-load MCP tool schemas (Chapter 9) and `SKILL.md` playbooks (Chapter 8) instead of binding 80 schemas upfront, *in a cache-friendly way* (see the note below). | Large savings on schema tokens | Faster prefill |
| 5. Sub-agent context scoping and masking | Offload noisy log/search steps to isolated sub-agents and mask tool observations $>3$ turns old (Chapter 11), turning $O(K^2)$ cumulative tokens into roughly $O(K)$. | Large savings on long loops | Prevents late-turn slowdowns |
| 6. Semantic and deterministic caching | Cache deterministic tool lookups and high-frequency canonical FAQ answers (keyed by exact match or embedding similarity) in an in-memory store (Redis/Valkey, self-hosted or managed, e.g., Amazon ElastiCache, Azure Managed Redis, Google Memorystore). | Near-100% of LLM cost saved on hits (an embedding lookup still costs a little) | Milliseconds on hits |

> [!IMPORTANT]
> The levers interact, and dynamic tool loading can destroy your prefix cache. Tool definitions usually sit at the very front of the prompt, so adding or removing tools mid-session invalidates the cache for everything after them. The Manus team (*"Context Engineering for AI Agents: Lessons from Building Manus"*, 2025) called the KV-cache hit rate the single most important metric for a production agent and recommended "mask, don't remove": keep the tool list stable and restrict which tools are callable via tool-choice constraints or logit masking. Provider features help here too. Anthropic's Tool Search Tool (Nov 2025) lets you mark tools with `defer_loading: true`, so discovered tools are appended later in the context instead of rewriting the cached prefix.

Semantic caches have their own failure mode: they can return confidently wrong answers. An embedding-similarity hit for *"How do I cancel my order?"* may get served for *"How do I cancel my subscription?"*. Use conservative similarity thresholds and set TTLs. Scope cache keys by tenant and user permissions so personalized answers are never shared across users, and keep anything stateful or time-sensitive out of the cache entirely.

### How prompt caching works in practice
Prefix matching is exact. Caches match the longest identical prefix, and one changed byte invalidates everything after it. Order content from most static to most dynamic: tools → system prompt → static documents → conversation history → latest user turn.

Minimum sizes and TTLs apply. Short prompts aren't cached (OpenAI and many Claude models, for example, require ~1,024+ tokens), and entries expire after minutes to hours depending on provider and settings.

Pricing models differ by provider:
* OpenAI caches automatically. The cached-input discount ranges from 50% to 90% depending on the model.
* Anthropic uses explicit `cache_control` breakpoints. Cache *writes* cost more than base input (1.25× for the 5-minute TTL, 2× for the 1-hour TTL); cache *reads* cost 0.1×.
* Gemini has implicit caching on by default for recent models. Explicit context caches carry an additional per-hour storage charge.
* Cloud model platforms (Amazon Bedrock, Azure OpenAI / AI Foundry, Google Vertex AI) offer prompt caching for supported models, with pricing that can differ from the model vendor's own API.
* Self-hosted open-source inference servers reuse KV cache across requests that share a prefix (vLLM automatic prefix caching, SGLang RadixAttention). You get no per-token discount, but you do get the TTFT and GPU-throughput benefit.

Measure your hit rate from usage metadata: OpenAI `usage.prompt_tokens_details.cached_tokens`, Anthropic `cache_read_input_tokens` / `cache_creation_input_tokens`, and Gemini `usage_metadata.cached_content_token_count`.

Put together, a cost-aware request path looks something like this:

```mermaid
flowchart TD
    Req["Incoming Request"] --> Cache{"1. Exact / Semantic<br/>Cache Hit?"}
    Cache -- "Yes (0 LLM Tokens)" --> FastOut["Return Cached Answer (milliseconds)"]
    Cache -- "No" --> Router["2. Fast Router (Small Model, minimal thinking)"]
    Router -- "Simple Tier (e.g., 65%)" --> Small["Single-Pass Small-Model Agent<br/>(Prefix-Cached + Scoped Tools)"]
    Router -- "Complex Tier (e.g., 35%)" --> Plan["3. Frontier Planner (medium thinking)"]
    Plan --> Workers["4. Parallel Small-Model Workers<br/>(Deferred Tool Loading + Observation Masking)"]
    Workers --> Out["Verified Deliverable"]
```

---

## 13.3 Advanced: budget enforcement, capacity planning, and cost telemetry

### 1. Enforcing per-request token and dollar budgets in code
In a multi-agent hierarchy, a bug in a child worker's loop can quietly burn \$15 of tokens on a single user request unless something shared puts a ceiling on it. That's the job of a `TokenBudgetEnforcer`. Create one per user request and share it with the root agent and every sub-agent through request-scoped state (for example, a LangGraph state field, the OpenAI Agents SDK run context, ADK's `InvocationContext`/session state, or a Python `contextvar`). Check it before each call, assuming the worst case, and record actual usage after each call:

```python
import threading
from dataclasses import dataclass, field


class BudgetExceededError(RuntimeError):
    pass


@dataclass
class TokenBudgetEnforcer:
    """One instance per user request, shared by the root agent and all sub-agents."""

    max_budget_usd: float = 0.25
    max_total_tokens: int = 80_000
    spent_usd: float = 0.0
    spent_tokens: int = 0
    _lock: threading.Lock = field(default_factory=threading.Lock, repr=False)

    def _check(self, extra_usd: float = 0.0, extra_tokens: int = 0) -> None:
        if (self.spent_usd + extra_usd > self.max_budget_usd
                or self.spent_tokens + extra_tokens > self.max_total_tokens):
            raise BudgetExceededError(
                f"Circuit breaker tripped: ${self.spent_usd:.4f} / {self.spent_tokens} tokens spent "
                f"(caps: ${self.max_budget_usd:.2f} / {self.max_total_tokens} tokens)."
            )

    def preflight(self, est_input_tokens: int, max_output_tokens: int,
                  price_in_1m: float, price_out_1m: float) -> None:
        """Call BEFORE each LLM call. Pessimistic: assumes no cache hits and full max_output_tokens."""
        worst_usd = est_input_tokens / 1e6 * price_in_1m + max_output_tokens / 1e6 * price_out_1m
        with self._lock:
            self._check(worst_usd, est_input_tokens + max_output_tokens)

    def record_llm_usage(
        self,
        uncached_in: int,
        cache_read_in: int,
        cache_write_in: int,
        out_and_thinking: int,
        price_in_1m: float,
        price_cache_read_1m: float,
        price_cache_write_1m: float,
        price_out_1m: float,
    ) -> None:
        """Call AFTER each LLM call with the provider's reported usage."""
        call_tokens = uncached_in + cache_read_in + cache_write_in + out_and_thinking
        call_usd = (
            uncached_in / 1e6 * price_in_1m
            + cache_read_in / 1e6 * price_cache_read_1m
            + cache_write_in / 1e6 * price_cache_write_1m
            + out_and_thinking / 1e6 * price_out_1m
        )
        with self._lock:  # parallel workers may share this tracker
            self.spent_tokens += call_tokens
            self.spent_usd += call_usd
            self._check()
```

Pass the *remaining* budget down as each call's `max_output_tokens` so that no single call can overshoot. Be aware that under heavy fan-out, concurrent preflights can each pass before any usage is recorded. If you need a strict guarantee, reserve the worst-case amount atomically in `preflight` and reconcile it in `record_llm_usage`. I'd also keep your framework's built-in loop caps on as a second line of defense: LangGraph's `recursion_limit`, OpenAI Agents SDK `Runner.run(..., max_turns=...)`, CrewAI's per-agent `max_iter`, or ADK `RunConfig(max_llm_calls=...)`.

### 2. Capacity planning: pay-as-you-go vs. provisioned throughput
Multi-agent patterns like Parallelization / Sectioning (Pattern 2.2, Chapter 3) and Orchestrator–Workers (Pattern 3.1, Chapter 4) produce spiky micro-bursts. One user request might fan out into 8 concurrent LLM calls in the same instant.

Standard pay-as-you-go (PAYGO) endpoints enforce Tokens-Per-Minute (TPM) and Requests-Per-Minute (RPM) quotas, and shared capacity can return `HTTP 429` under load. Tail latency spikes when 8 workers all back off at once. At a minimum, bound fan-out with a concurrency semaphore and retry with jittered exponential backoff that honors `retry-after`.

Beyond that, most teams end up with a hybrid capacity setup in three layers:

1. **Provisioned throughput for base load.** Reserve fixed capacity sized to your steady-state load (say, P50–P75 concurrency). The products include Azure OpenAI PTUs (Provisioned Throughput Units), Amazon Bedrock Provisioned Throughput, and Google Vertex AI Provisioned Throughput (purchased in GSUs). If you self-host open-weight models (e.g., vLLM or TGI on reserved GPU nodes), the equivalent is a committed-use or reserved GPU pool. Reserved capacity removes most rate-limit throttling and makes latency more predictable. Keep in mind that it reserves throughput; it does not hard-guarantee a TTFT.
2. **Spillover and service tiers for peaks.** Decide what happens to traffic above your reserved capacity: overflow to PAYGO, or reject it so you can queue or degrade gracefully. Each provider handles this differently. Azure OpenAI can spill over from a provisioned deployment to a standard deployment of the same model. On Google Vertex AI, traffic above Provisioned Throughput spills over to PAYGO by default, and the per-request `X-Vertex-AI-LLM-Request-Type` header (`dedicated` / `shared`) forces PT-only (returns 429 when exhausted) or PAYGO-only. On OpenAI, the `service_tier` parameter selects `priority` (premium price, lower and more consistent latency), `default`, or `flex` (Batch-rate pricing, slower, may return 429).
3. **Batch for async work.** Run non-urgent background jobs (memory consolidation, nightly reports, offline evals) on Batch APIs. OpenAI Batch, Anthropic Message Batches, Amazon Bedrock batch inference, Azure OpenAI Batch, and Gemini/Vertex AI batch prediction are discounted by roughly 50% in exchange for asynchronous completion, typically within 24 hours.

### 3. Cost telemetry and attribution
You can't optimize spend you can't attribute. Emit token usage on every LLM span (Chapter 10) using the OpenTelemetry GenAI semantic conventions, and export it to whatever tracing backend you run (Langfuse, Arize Phoenix, Datadog, Grafana, or your cloud's native tracing service). These conventions are still evolving, so pin the version you depend on.

On each call, record `gen_ai.request.model`, `gen_ai.usage.input_tokens`, and `gen_ai.usage.output_tokens`, plus cache attributes such as `gen_ai.usage.cache_read.input_tokens` where your instrumentation supports them. Then tag spans with business dimensions like `tenant_id`, `feature`, `agent_name`, and `route_tier`, so cost rolls up per customer, per sub-agent, and per workflow instead of only per API key.

For dashboards, track CpO per workflow, cache hit rate (cached ÷ total input tokens), the thinking-token share of output, retries per validated success, and P95/P99 cost per request. Set an alert on sudden drops in cache hit rate. It's usually the first sign someone added a timestamp to the system prompt.

---

## Chapter 13 summary

| Level | Capability | What it optimizes | Key takeaway |
| :--- | :--- | :--- | :--- |
| Beginner | TBOM decomposition and Cost per Validated Outcome (CpO) | $\text{CpO} = \frac{\text{Total Spend}}{\text{Verified Successes}}$ | Optimize for *Cost per Validated Outcome*, not raw price per 1M tokens; track thinking tokens as output cost; remember that parallel branches cost tokens but save wall-clock time. |
| Intermediate | The six cost levers (prefix cache, model tiering, thinking budgets, lazy schemas) | Large CpO reductions and lower TTFT | Keep prefixes byte-identical for cache hits (and load tools in a cache-friendly way), tier models (frontier Planner + small Executor), and cap thinking budgets. |
| Advanced | `TokenBudgetEnforcer` circuit breakers, hybrid provisioned + PAYGO capacity, OTel cost telemetry | Tail latency ($P99$), runaway-loop prevention, cost attribution | Enforce pre-flight + post-call dollar/token caps in shared request-scoped state, size Provisioned Throughput for base load, push async work to Flex/Batch tiers (~50% off), and attribute every token via OTel GenAI spans. |

---

[← Previous: Chapter 12 — Agentic Security, Identity & Sandboxing](./chapter-12-agentic-security-identity-and-sandboxing.md) | [Book Index](./README.md) | [Next: Chapter 14 — Multimodal, Computer-Use & Streaming Agents →](./chapter-14-multimodal-computer-use-and-streaming-agents.md)
