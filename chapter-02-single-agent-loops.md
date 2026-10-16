# Chapter 2 (Beginner): Single-Agent Orchestration Patterns

[⬅️ Previous: Chapter 1 — Foundations](./chapter-01-foundations-and-mental-models.md) | [📚 Book Index](./README.md) | [Next: Chapter 3 — Deterministic Multi-Agent Workflows ➡️](./chapter-03-deterministic-multi-agent-workflows.md)

---

At the beginner level of agentic orchestration, a **single agent** operates inside a control loop equipped with tools and working memory. Understanding why and where these single-agent loops succeed—and where they break—is essential before designing multi-agent topologies.

> [!NOTE]
> Patterns in this chapter are numbered **1.x** because they are the **Level 1** (single-agent) patterns of the book's pattern catalog. Level 2–4 patterns appear in Chapters 3–5 (see the numbering table in [Chapter 1](./chapter-01-foundations-and-mental-models.md), §1.4).

---

## Pattern 1.1: ReAct (Reason + Act Loop)
*The atomic unit of autonomous agency (Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models", 2022; ICLR 2023).*

### Architecture
The LLM operates in a continuous **Thought $\rightarrow$ Action $\rightarrow$ Observation** cycle. In each iteration, it examines the conversation history, reasons about what information is missing, emits one or more tool calls, and inspects the returned tool observations (including error messages) until it decides the task is complete.

```mermaid
flowchart TD
    User["User Query"] --> LLM["LLM (Thought / Reasoning)"]
    LLM --> Decision{"Tool Calls Emitted?"}
    Decision -- "Yes (Action)" --> Exec["Execute Tool(s) in Environment"]
    Exec --> ErrCheck{"Tool Raised Exception?"}
    ErrCheck -- "No" --> Obs["Format Result as Observation"]
    ErrCheck -- "Yes (Self-Healing)" --> ErrObs["Format Traceback/Error as Observation<br/>(Do NOT crash the loop)"]
    Obs --> LLM
    ErrObs --> LLM
    Decision -- "No (Task Complete)" --> Output["Final Response"]
```

### Why This Architecture?
* **Why it exists:** Many problems cannot be solved in a single retrieval pass because **Step $t+1$ depends on the runtime output of Step $t$** (e.g., you don't know which error log stream to query until you first check which deployment occurred yesterday).
* **Self-Healing via Error-as-Observation:** A crucial production detail often omitted in toy tutorials is **exception feedback**. When an LLM passes invalid arguments to a tool or writes a SQL query with a syntax error, the runtime must catch the exception and return `"ToolError: column 'user_id' does not exist; available columns are [id, account_id]"` as a tool observation so the LLM can self-correct on the next turn. Make error messages **actionable** (what was wrong, what is valid)—a bare stack trace wastes tokens and teaches the model little.
* **Trade-offs & Failure Modes:**
  1. **Local Myopia ("Lost in the Weeds"):** Because ReAct only plans one step ahead, a complex 300-line stack trace in Step 2 can derail the agent from answering the second half of the user's prompt.
  2. **Quadratic Context Bloat ($O(K^2)$):** Every raw tool output stays in the message history across all subsequent turns (Chapter 1, Law 2). Cap or summarize large observations before appending them.
  3. **Runaway Loops:** Without a hard step/token budget, an agent can repeat the same failing call indefinitely. Always enforce `max_steps` in code, never in the prompt alone.

### ReAct in 2026: Native Tool Calling & Reasoning Models
The original ReAct paper was a *prompting* technique: the model wrote `Thought:` / `Action:` / `Observation:` text that the harness parsed with regexes. Modern implementations look different in three important ways:
* **Native tool calling replaces text parsing.** Models emit structured tool-call objects validated against a JSON Schema. With **strict / constrained-decoding modes** (offered by the major APIs), malformed or wrongly typed arguments become rare—error-as-observation still matters for *semantic* and *runtime* failures (missing rows, permission denied, timeouts).
* **The "Thought" is often internal.** Reasoning models think before (and, with *interleaved thinking*, between) tool calls. Prompting them to "reason step by step" is redundant and can hurt; instead, state the goal, constraints, and stop criteria. **Pass reasoning state back unchanged** each turn—e.g., Anthropic thinking blocks or Gemini thought signatures must be returned with the assistant message, or the API may reject the request or the model loses its train of thought. Append the full assistant message, not just its text.
* **Parallel tool calls.** A single model turn can request several independent tool calls (e.g., fetch deployments *and* error logs). Execute independent calls concurrently and return one result per `tool_call_id`.

### Production-Grade Framework-Agnostic Pseudocode
```python
from typing import Any, Callable

MAX_OBS_CHARS = 8_000  # Cap each observation to contain the O(K^2) context tax

def react_agent(
    user_query: str,
    tools: dict[str, Callable[..., Any]],
    model=llm,                # Injected so callers can choose a fast/cheap model
    max_steps: int = 10,
) -> str:
    messages = [
        # Goal + constraints + stop criteria. (Reasoning models think internally;
        # an explicit "think step by step" instruction is unnecessary for them.)
        {"role": "system", "content": "Resolve the user's request. Verify facts with tools. "
                                      "Stop and answer once every part of the request is addressed."},
        {"role": "user", "content": user_query},
    ]
    for step in range(max_steps):
        response = model.generate(messages, tools=[tool_schema(t) for t in tools.values()])
        # Append the FULL assistant message (text + tool calls + any reasoning/thought
        # signature blocks) so the next turn has consistent state.
        messages.append(response.message)

        if not response.tool_calls:  # LLM decided the goal is complete
            return response.text

        for call in response.tool_calls:  # Independent calls may also be run concurrently
            try:
                if call.name not in tools:
                    raise ValueError(f"Unknown tool '{call.name}'. Available: {sorted(tools)}")
                result = tools[call.name](**call.args)  # call.args: parsed JSON arguments
                observation = str(result)
            except Exception as exc:
                # CRITICAL: Feed tool errors back as observations so the LLM can self-correct
                observation = f"ToolExecutionError ({type(exc).__name__}): {exc}"

            if len(observation) > MAX_OBS_CHARS:
                observation = observation[:MAX_OBS_CHARS] + "\n...[truncated; refine your query]"
            messages.append({"role": "tool", "tool_call_id": call.id, "content": observation})

    raise RuntimeError(f"Guardrail triggered: ReAct loop exceeded {max_steps} iterations.")
```

### Runtime Execution Trace (Running Scenario)
1. **User:** *"Why did my `checkout-service` start throwing 503 errors after yesterday's deployment, and can I get an SLA credit?"*
2. **Turn 1 — Thought:** *"I need to check recent deployments for `checkout-service`."*
   * **Action:** `get_deployments(service="checkout-service", days=2)`
   * **Observation:** `Deploy #842 at 2026-10-15 14:00 UTC (DB connection pool reduced from 50 -> 5)`.
3. **Turn 2 — Thought:** *"DB pool dropped to 5. Let me check 503 error logs around 14:00 UTC."*
   * **Action:** `query_logs(service="checkout-service", status_code="503")`
   * **Observation:** `ToolExecutionError (TypeError): status_code must be int, got str`. *(Self-healing triggered! Strict schema validation would usually prevent this particular error; the recovery mechanism is the same for runtime errors it can't catch.)*
4. **Turn 3 — Thought:** *"Let me fix the argument type to integer `503`."*
   * **Action:** `query_logs(service="checkout-service", status_code=503)`
   * **Observation:** `ConnectionPoolExhausted from 14:02 to 14:44 UTC (42 minutes total downtime)`.
5. **Turn 4 — Thought:** *"Now I must check if 42 minutes qualifies for an SLA credit."*
   * **Action:** `check_sla_policy(downtime_minutes=42)`
   * **Observation:** `SLA credit requires >45 minutes of monthly downtime.`
6. **Turn 5 — Thought:** *"I have verified both answers."* $\rightarrow$ **Emits Final Answer** (root cause: the pool-size change in Deploy #842; 42 minutes is below the 45-minute threshold, so no automatic credit).

### Distinct Industry Examples
* **Text-to-SQL Data Analyst:** Lists tables $\rightarrow$ inspects schema $\rightarrow$ runs SQL $\rightarrow$ gets syntax error observation $\rightarrow$ fixes SQL $\rightarrow$ returns answer.
* **IT Helpdesk Access Bot:** Checks user's Okta groups $\rightarrow$ checks manager approval status $\rightarrow$ provisions access role (a write action that should sit behind a human-approval gate—see Chapter 17).

---

## Pattern 1.2: Plan-and-Execute (Planner–Executor–Replanner)
*Decoupling strategic multi-step planning from tactical step execution. Inspired by Plan-and-Solve prompting (Wang et al., ACL 2023) and ReWOO (Xu et al., 2023), and popularized as the LangChain/LangGraph "plan-and-execute" agent.*

### Architecture
Instead of interleaving global planning and noisy tool execution inside one prompt, an explicit **Planner LLM** first generates a structured checklist of steps. A **Step Executor** (a scoped ReAct loop) executes one step at a time in an isolated context window, returning **only a concise summary** to a **Replanner**, which either advances the checklist or dynamically revises remaining steps.

```mermaid
flowchart TD
    Goal["User Goal"] --> Planner["Planner LLM<br/>(Frontier Reasoning Model)"]
    Planner -- "Initial Plan: [Step 1, Step 2, Step 3]" --> Executor["Step Executor<br/>(Fast/Cheap Model + Scoped ReAct Loop)"]
    Executor -- "Concise Step Summary ONLY<br/>(50KB raw tool logs discarded!)" --> Replanner["Replanner LLM<br/>(Frontier Reasoning Model)"]
    Replanner --> Check{"Goal Complete?"}
    Check -- "No (Keep or Rewrite Remaining Steps)" --> Executor
    Check -- "Yes" --> Final["Final Synthesized Answer"]
```

### Why This Architecture?
* **Why use it over ReAct:**
  1. **Reduces Myopia:** The explicit plan acts as an external cognitive anchor across 10+ step horizons.
  2. **Breaks the $O(K^2)$ Context Tax:** If Step 1 requires inspecting 50,000 tokens of raw logs, those logs live and die inside Step 1's isolated `Executor` scope. The Replanner and Step 2 only receive a 150-token structured summary.
  3. **Model Tiering (Cost & Latency Optimization):** Use a frontier reasoning model (e.g., a Gemini Pro- or Claude Opus-class model) for the Planner/Replanner (called only a few times), and a fast, low-cost model (e.g., a Flash- or Haiku-class model) for the Step Executor's inner tool calls.
* **Trade-offs & Failure Modes:**
  * **Upfront Planning Latency:** Must wait for the initial plan before executing the first tool call, and every step adds a Replanner call.
  * **Stale Plan Trap:** If you implement "Plan-once-and-Execute-blindly" without a **Replanner**, an unexpected discovery in Step 1 makes Steps 2–5 useless.
  * **Lossy Summaries:** The executor may summarize away the one detail the Replanner needed. Ask for **structured** step results (key findings, evidence IDs, open questions) rather than free prose.

> [!TIP]
> **Modern variants.**
> * **Plan-as-a-Tool (to-do lists):** Many 2025–2026 coding and research agents keep a single strong ReAct loop but give it a `write_todos`-style tool to create and update an explicit checklist in context (Claude Code's to-do/task tools and LangChain's Deep Agents follow this approach). You get the "cognitive anchor" without a separate planner, though not the context isolation.
> * **Parallel DAG plans:** ReWOO-style planners emit all tool calls up front with placeholders (`#E1`, `#E2`), and LLMCompiler (Kim et al., 2023) schedules independent steps in parallel—useful when steps don't depend on each other's observations.

### Production-Grade Framework-Agnostic Pseudocode
```python
from typing import Optional
from pydantic import BaseModel

class PlanState(BaseModel):
    is_complete: bool
    updated_remaining_steps: list[str]
    final_response: Optional[str] = None

def plan_and_execute(user_goal: str, tools: dict, max_steps: int = 8) -> str:
    plan: list[str] = frontier_llm.generate_structured(
        f"Create a concise step-by-step plan for: {user_goal}", schema=list[str]
    )
    completed_steps: list[tuple[str, str]] = []

    for _ in range(max_steps):  # Hard cap on executed steps (plans can grow on replanning)
        if not plan:
            break
        current_step = plan.pop(0)

        # Isolated context: fast model runs ReAct; raw tool logs never leak out
        try:
            step_summary = react_agent(
                user_query=f"Global Goal: {user_goal}\nPrior Findings: {completed_steps}\n"
                           f"Execute ONLY this step and summarize findings: {current_step}",
                tools=tools,
                model=fast_llm,
            )
        except RuntimeError as exc:  # Step hit its own guardrail: let the Replanner adapt
            step_summary = f"STEP FAILED: {exc}"
        completed_steps.append((current_step, step_summary))

        # Replanner adapts remaining steps based on actual runtime findings
        decision: PlanState = frontier_llm.generate_structured(
            f"Goal: {user_goal}\nCompleted: {completed_steps}\nRemaining Plan: {plan}",
            schema=PlanState,
        )
        if decision.is_complete and decision.final_response:
            return decision.final_response
        plan = decision.updated_remaining_steps

    return frontier_llm.generate(f"Synthesize final answer for {user_goal} from {completed_steps}")
```

### Runtime Execution Trace (Running Scenario)
*To show replanning, this trace uses an alternate version of the incident in which the customer's premise turns out to be wrong.*
1. **Planner:** Creates 3-step plan: `[1. Check recent deploys for checkout-service, 2. Correlate 503 error timestamps with deploy, 3. Check SLA credit eligibility]`.
2. **Step Executor (Step 1):** Queries deployment API $\rightarrow$ *Surprise Runtime Discovery:* There was **no deployment** to `checkout-service` yesterday!
3. **Replanner:** Sees Step 1 disproved the user's premise. It **dynamically rewrites Step 2** from *"Correlate 503 timestamps with deploy"* to *"Check upstream dependency health (`auth-service` and `payment-gateway`) for outages yesterday."*
4. **Step Executor (Step 2 & 3):** Discovers `auth-service` had a 60-minute outage causing `checkout-service` 503s $\rightarrow$ confirms 60m > 45m threshold $\rightarrow$ approves SLA credit.

---

## Pattern 1.3: Self-Refinement, Episodic Reflexion & Tree-of-Thoughts
*Trading extra test-time compute for higher reliability and reasoning depth.*

### Critic's Distinction: Three Patterns Often Confused as "Reflection"
Many blogs lump all self-correction into a single box. In architectural practice, there are **three distinct test-time compute patterns**:

| Sub-Pattern | Mechanism | Memory Used | Best Used When... |
| :--- | :--- | :--- | :--- |
| **1. Intrinsic Self-Refinement** *(Self-Refine, Madaan et al., NeurIPS 2023)* | Generate draft $\rightarrow$ Critique against rubric $\rightarrow$ Rewrite within a **single episode**. | Working Memory only. | Formatting, tone polishing, or multi-constraint document drafting. |
| **2. Grounded Reflexion** *(Reflexion, Shinn et al., NeurIPS 2023)* | Execute trial $\rightarrow$ Evaluate with an **external signal** (unit tests, compiler, SQL `EXPLAIN`, environment reward) $\rightarrow$ Convert failure into a verbal lesson stored in **Episodic Memory** across retries. | Working + Episodic Memory. | Coding agents, SQL generation, schema extraction, and environment tasks with pass/fail signals. |
| **3. Branching Search: Best-of-$N$ & Tree-of-Thoughts (ToT)** *(ToT: Yao et al., NeurIPS 2023)* | **Best-of-$N$:** sample $N$ complete candidates in parallel and pick one with a verifier/reward model (or majority vote, as in self-consistency). **ToT:** expand $B$ candidate "thoughts" at each step, score partial branches with a heuristic/verifier, and prune or backtrack (BFS/DFS). | Candidate set / Tree State in Working Memory. | Complex algorithmic puzzles, architectural trade-off exploration, or high-variance code synthesis. |

> [!NOTE]
> The original Reflexion paper's evaluators ranged from exact-match and heuristics to LLM judges and **self-generated** unit tests. This book's "Grounded Reflexion" deliberately prioritizes **trustworthy external verifiers** first, because the lesson is only as good as the failure signal behind it.

> [!TIP]
> **Reasoning models have internalized much of this search.** Thinking models are trained to explore, self-check, and backtrack inside their own reasoning, so explicit ToT scaffolding is rarely worth its cost in production today. What still pays off is (a) raising the model's **thinking budget/effort** for hard steps, and (b) **Best-of-$N$ with an external verifier** (tests, a checker, a reward model), which adds information the model cannot generate by thinking alone.

```mermaid
flowchart LR
    Task["Input Task"] --> Actor["Actor / Generator LLM"]
    Actor -- "Candidate Output" --> DetCheck["1. Deterministic Verifiers<br/>(Compiler, Pytest, Pydantic, Regex)"]
    DetCheck --> DetPass{"Deterministic Pass?"}
    DetPass -- "No (Traceback / Error)" --> Reflector["Reflector LLM<br/>(Synthesizes Verbal Lesson)"]
    DetPass -- "Yes" --> RubricCheck["2. LLM Rubric Evaluator"]
    RubricCheck -- "Fail" --> Reflector
    Reflector -- "Appends Lesson to Episodic Buffer" --> Actor
    RubricCheck -- "Pass" --> Final["Verified Output"]
```

> [!WARNING]
> **The Ungrounded Reflection Anti-Pattern:** Asking an LLM *"Review your answer above; are you sure it is correct?"* **without** an external ground-truth verifier (test runner, schema validator, citation checker) or explicit rubric frequently makes things worse. Huang et al. (*"Large Language Models Cannot Self-Correct Reasoning Yet"*, ICLR 2024) found that intrinsic self-correction on reasoning tasks often **degrades** accuracy, and Laban et al.'s *FlipFlop* experiment (2023) reported models changing their answer in roughly 46% of cases when simply challenged with "Are you sure?", with accuracy dropping on average. This **sycophantic flip-flopping** often turns correct answers into incorrect ones. Always run deterministic verifiers first!

### Production-Grade Framework-Agnostic Pseudocode (Grounded Reflexion)
```python
from typing import Callable, Optional
from pydantic import BaseModel

class RubricVerdict(BaseModel):
    passed: bool
    failed_items: list[str]
    concise_lesson_for_next_trial: str

def grounded_reflexion_agent(
    task: str,
    deterministic_validators: list[Callable[[str], Optional[str]]],  # Return error msg or None
    rubric: list[str],
    max_trials: int = 3,
) -> str:
    episodic_lessons: list[str] = []
    draft = ""

    for trial in range(max_trials):
        draft = actor_llm.generate(
            f"Task: {task}\nLessons from prior failed trials (DO NOT REPEAT): {episodic_lessons}"
        )
        # Stage 1: Cheap, deterministic checks first (Regex, AST, Schema, Unit Tests)
        det_errors = [err for check in deterministic_validators if (err := check(draft))]
        if det_errors:
            lesson = reflector_llm.generate(
                f"Task: {task}\nDraft: {draft}\nFailed checks: {det_errors}\n"
                "State the root cause and one concrete rule to avoid it next time:"
            )
            episodic_lessons.append(lesson)
            continue

        # Stage 2: Rubric-grounded LLM verification (ideally a different model/prompt than the actor)
        verdict = reflector_llm.generate_structured(
            f"Task: {task}\nDraft: {draft}\nStrictly verify each rubric item: {rubric}",
            schema=RubricVerdict,
        )
        if verdict.passed:
            return draft
        episodic_lessons.append(verdict.concise_lesson_for_next_trial)

    # Never silently ship an unverified draft: escalate (human review, fallback answer, or error)
    return escalate_to_human(task=task, last_draft=draft, lessons=episodic_lessons)
```

### Runtime Execution Trace (Running Scenario)
1. **Trial 1 Draft:** *"Deploy #842 on internal host `10.128.4.19` caused 42m of 503s. We have credited \$200 to your account."*
2. **Stage 1 Deterministic Validator (`check_no_internal_ips`):**
   * ❌ Fails immediately in $0.2\text{ms}$: `"Found RFC1918 private IP 10.128.4.19 in external customer response."`
   * **Episodic Lesson Recorded:** *"Never include internal `10.x.x.x` host IPs in customer-facing text."*
3. **Trial 2 Draft:** Removes the IP, but still claims a \$200 SLA credit was issued.
4. **Stage 2 Rubric Evaluator:**
   * ❌ Fails Rubric Item #2 (*"Must match SLA policy tool output"*): `42m < 45m threshold (ineligible)`.
   * **Episodic Lesson Recorded:** *"42m downtime is below the 45m SLA threshold; explain policy accurately instead of promising an automatic credit."*
5. **Trial 3 Draft:** Passes both Stage 1 deterministic validators and Stage 2 rubric check. $\rightarrow$ **Returned.**

> [!TIP]
> Whenever a rubric item can be checked in code (here, comparing the stated credit against the `check_sla_policy` tool output), promote it to a Stage 1 deterministic validator. LLM rubric checks are for what code genuinely cannot judge.

---

## Chapter 2 Summary Comparison

| Single-Agent Pattern | Control Flow Owner | Context Growth | Latency & Token Cost (rough, vs. one LLM call) | Ideal Use Case | Primary Failure Mode |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1.1 ReAct Loop** | Single LLM (1 step ahead) | $O(K^2)$ (All tool logs stay in window) | $1\times\text{–}5\times$ | Short, dynamic 2–5 step tool explorations. | Local myopia; context window pollution on long horizons. |
| **1.2 Plan-and-Execute** | Frontier Planner + Scoped Executor | $O(K)$ (Raw step logs discarded) | $3\times\text{–}8\times$ | Multi-part 5–15 step goals with large intermediate tool outputs. | High upfront latency; stale plans if replanner is omitted; lossy step summaries. |
| **1.3 Grounded Reflexion** | Actor $\rightarrow$ Deterministic + Rubric Verifier | $O(\text{trials})$ | $2\times\text{–}4\times$ | Code/SQL generation, structured extraction, high-compliance responses. | Sycophantic flip-flopping if run *without* deterministic/rubric grounding. |

---

[⬅️ Previous: Chapter 1 — Foundations](./chapter-01-foundations-and-mental-models.md) | [📚 Book Index](./README.md) | [Next: Chapter 3 — Deterministic Multi-Agent Workflows ➡️](./chapter-03-deterministic-multi-agent-workflows.md)
