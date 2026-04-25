# LayerInfinite Integration Patterns for Modern AI Agents

As AI agents evolve from simple chat wrappers into sovereign systems, the way developers build them has fractured into three distinct architectures. 

How a developer integrates **LayerInfinite (LI)** depends entirely on where their agent sits on the autonomy spectrum. This document outlines the rigorous, production-grade integration strategies for each architectural pattern.

---

## Glossary of Core Concepts
Before diving into patterns, it is critical to define the terminology used in modern agent telemetry:
- **Decision Boundary:** The exact moment in the code where the orchestrator decides *what to do next*. LayerInfinite should be integrated right at this execution boundary.
- **Context Extraction:** LayerInfinite's ability to automatically read the arguments passed to your functions (like capturing `"npm install"` from `run_bash_command("npm install")`) to help diagnose why generic tools fail in the real world.
- **Action Normalization:** The practice of converting highly generic, messy commands (like a raw bash string) into categorical families (like `bash:install` or `bash:test`) so the routing engine has a clean, repeatable action to score.
- **Soft Routing (Guardrails):** Using LayerInfinite's probability scores not to force the agent's hand, but to simply advise, veto, or penalize highly failure-prone actions before the agent commits to them.

---

## Pattern 1: Bounded Tool-Calling (Narrow Agents)
**Frameworks:** LangChain, Vercel AI SDK, Custom Orchestrators  
**Use Cases:** Automated DevOps, Customer Support Triage, Financial Reconciliations  

### The Architecture
Narrow agents operate in highly constrained environments. Instead of having free rein over a terminal, the agent is given a specific, bounded list of "Tools" (Python functions) it can use. 

### How Developers Integrate LayerInfinite
In this paradigm, the "Action" is the tool itself. Developers integrate LI primarily to leverage **Assist** or **Auto** routing modes. Because the tools are specific, the historical success rate of a tool directly dictates whether the agent orchestrator should invoke it again.

```python
# The developer explicitly decorates each highly-specific tool. 
# The Python function name is automatically used as the action name.
@li.action("database_latency_spike")
def restart_postgres_service():
    # Execute infrastructure code...
    return True

@li.action("database_latency_spike")
def scale_read_replicas():
    # Execute infrastructure code...
    return True
```

**The Operational Value:** LayerInfinite does not intercept the LLM; rather, it sits at the orchestration middleware layer to **shape tool selection**. When a latency spike occurs, LI advises the execution layer: *"Scaling read replicas has a 94% success rate for this issue, whereas restarting Postgres only succeeds 12% of the time."*

---

## Pattern 2: Open-Ended Sovereign Loops
**Frameworks:** AutoGen, SWE-agent, OpenDevin, LangGraph  
**Use Cases:** Autonomous Software Engineering, Web Research, Data Mining  

### The Architecture
Sovereign coding and research agents typically abandon large libraries of narrow tools in favor of a few **low-level primitives**: shell, editor, browser, and search. They operate in continuous observation/thought/action loops.

### How Developers Integrate LayerInfinite
Naïvely decorating a `run_bash_command` primitive is ineffective, as running `pytest` is vastly different from `rm -rf /`. Instead, developers use LayerInfinite's **Action Normalization** and **Context Extraction** to derive learnable routing units.

```python
# The orchestrator derives a normalized action key from the raw command family
def execute_normalized_bash(cmd_string: str, session_goal: str):
    if "npm install" in cmd_string:
        return run_bash_install(cmd_string=cmd_string, session_goal=session_goal)
    else:
        return run_bash_generic(cmd_string=cmd_string, session_goal=session_goal)

# Developers use normalized wrapper functions so LayerInfinite can route them
@li.action(task="react_bug_fix", name="bash:install")
def run_bash_install(cmd_string: str, session_goal: str):
    # LI automatically extracts `cmd_string` as raw_context for deeper diagnosis
    result = subprocess.run(cmd_string, shell=True, capture_output=True)
    return result.returncode == 0

@li.action(task="react_bug_fix", name="bash:generic")
def run_bash_generic(cmd_string: str, session_goal: str):
    result = subprocess.run(cmd_string, shell=True, capture_output=True)
    return result.returncode == 0
```

**The Operational Value:** This approach unlocks a powerful, three-tiered product story:
1. **Raw primitive logging** for general observability.
2. **Context extraction** for deep diagnosis (extracting noisy, real-world arguments into `raw_context`).
3. **Action normalization** for learnable routing. 

While **Recommend** is the default starting mode here, sovereign agents leverage LI for soft routing and guardrails—allowing the orchestrator to automatically veto or penalize highly failure-prone command families (like `bash:install`) dynamically.

---

## Pattern 3: Multi-Agent Orchestration Graphs
**Frameworks:** CrewAI, LangGraph, AutoGen Multi-Agent  
**Use Cases:** Complex Enterprise Workflows (e.g., Researcher Node ➡️ Writer Node ➡️ QA Node)  

### The Architecture
Modern multi-agent systems are fundamentally orchestration graphs with conditional routing and specialist roles. The primary "Action" a coordinator node takes is **delegating work to another specialized sub-agent**.

### How Developers Integrate LayerInfinite
Developers integrate LayerInfinite at the **decision boundaries between agents** to continuously evaluate handoff optimization.

```python
# The coordinator agent decides to delegate work to the QA Agent
@li.action("content_verification")
def delegate_to_qa_agent(draft_content):
    qa_result = qa_agent.execute(draft_content)
    if not qa_result.passed:
        raise Exception("QA Agent failed to verify the content.")
    return qa_result

# Or it decides to delegate to the Fact-Checker Agent
@li.action("content_verification")
def delegate_to_fact_checker(draft_content):
    return fact_checker_agent.execute(draft_content)
```

**The Operational Value:** LayerInfinite provides real-time **delegation scoring and sub-agent trust metrics**. If the `qa_agent` node begins consistently failing on specific data types, LayerInfinite automatically drops its trust score, enabling the graph to conditionally route the workflow to the `fact_checker_agent` instead.

---

## Practical Implementation: The "How-To" FAQ

To move from architectural theory to actual code, here is how LayerInfinite behaves practically in your environment:

### 1. Where exactly do I initialize LI?
Initialize the SDK globally, as early as possible in your agent's lifecycle, before declaring any tools or orchestrators.
```python
from layerinfinite import Layerinfinite
li = Layerinfinite(api_key="...", agent_id="my-agent-1", mode="recommend")
```

### 2. What does @li.action accept?
The decorator accepts `task` (required) and `name` (optional). Both must be static strings. If `name` is omitted, the python function's name is used automatically as the action key. If you need dynamic task routing at runtime, you can either create normalized wrapper functions or use `li.log_outcome()` directly.

### 3. What gets logged automatically?
When a decorated function runs, LI silently logs:
- **Success/Failure:** If the function returns normally, it counts as a success. If it throws an `Exception`, it counts as a failure.
- **Latency:** Execution time measured in milliseconds.
- **Context:** The `raw_context` (a scrubbed dictionary of the function's arguments, automatically omitting PII, tokens, and massive strings).

### 4. What happens on failure or retry?
LayerInfinite is designed to never crash your agent. If the network drops or the LayerInfinite API is unreachable, the SDK writes your failed logs to a local durable queue and retries them silently in a background thread. Your agent's main execution loop is completely isolated from LI telemetry timeouts.

### 5. How do modes differ in code behavior?
- **Recommend:** `li.run()` simply executes the function. Outcomes are logged passively. The agent maintains total autonomy over tool selection.
- **Assist:** The developer explicitly calls `li.suggest(task)` to fetch a hint based on historical probabilities, and appends it to the LLM's prompt.
- **Auto:** The developer calls `li.run(task)` and LayerInfinite *takes over*, deterministically picking the highest-probability tool and executing it on behalf of the agent.

---

> **"LayerInfinite should be integrated at the decision boundary the agent actually controls: tool choice for narrow agents, context-rich primitive execution for sovereign agents, and handoff choice for multi-agent systems."**
