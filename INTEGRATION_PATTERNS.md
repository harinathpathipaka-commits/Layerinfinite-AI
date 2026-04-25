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

### Python Implementation
```python
import os
from layerinfinite import Layerinfinite

li = Layerinfinite(api_key=os.getenv("LI_API_KEY"), agent_id="support-agent", mode="assist")

# 1. Decorate your existing tools (The function name becomes the action name)
@li.action("refund_issue")
def issue_full_refund(user_id: str):
    return True

@li.action("refund_issue")
def issue_partial_credit(user_id: str):
    return True

# 2. Inject into your Agent Execution Loop
# Depending on the mode you selected, drop ONE of these lines into your agent's main loop:

# For ASSIST mode (Soft Routing):
suggestion = li.suggest(task=current_issue)
# -> Append `suggestion.action_name` to your agent's LLM prompt

# For AUTO mode (Deterministic Execution):
# -> REPLACE your existing manual tool execution with this line:
result = li.run(task=current_issue, user_id=customer_id)
# -> `result` contains whatever the chosen tool returns (e.g., True)
# -> Pass this result back into your agent's context so it knows what happened

# For RECOMMEND mode (Passive Analytics):
# -> KEEP your existing manual tool execution exactly as it is:
issue_full_refund(user_id=customer_id) 
# -> LayerInfinite silently logs the outcome in the background
```

### TypeScript Implementation
```typescript
import { Layerinfinite } from 'layerinfinite-sdk';

const li = new Layerinfinite({ apiKey: process.env.LI_API_KEY!, agentId: 'support-agent', mode: 'assist' });

// 1. Decorate your existing tools
const issueFullRefund = li.action('refund_issue', 'issue_full_refund', async (userId: string) => { return true; });
const issuePartialCredit = li.action('refund_issue', 'issue_partial_credit', async (userId: string) => { return true; });

// 2. Inject into your Agent Execution Loop
// Depending on the mode you selected, drop ONE of these lines into your agent's main loop:

// For ASSIST mode:
const suggestion = await li.suggest(currentIssue);
// -> Append `suggestion?.actionName` to your agent's LLM prompt

// For AUTO mode:
// -> REPLACE your existing manual tool execution with this line:
const result = await li.run(currentIssue, customerId);
// -> `result` contains whatever the chosen tool returns
// -> Pass this result back into your agent's context so it knows what happened

// For RECOMMEND mode:
// -> KEEP your existing manual tool execution exactly as it is:
await issueFullRefund(customerId); 
// -> LayerInfinite silently logs the outcome in the background
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

### Python Implementation
```python
import subprocess
from layerinfinite import Layerinfinite

li = Layerinfinite(api_key="...", agent_id="swe-agent", mode="recommend")

# 1. Normalize your generic primitives into specific action families
@li.action("react_bug")
def bash_install(cmd_string: str):
    return subprocess.run(cmd_string, shell=True).returncode == 0

@li.action("react_bug")
def bash_generic(cmd_string: str):
    return subprocess.run(cmd_string, shell=True).returncode == 0

# 2. Inject into your Agent Execution Loop
# Depending on the mode you selected, drop ONE of these lines into your agent's main loop:

# For RECOMMEND mode (Observe & Circuit Break):
# -> KEEP your existing manual tool execution exactly as it is:
bash_install(cmd_string=current_command)
stats = li.observe(task=current_issue) 
# -> Use `stats.success_rate` to halt the agent if it gets stuck in an infinite failure loop

# For ASSIST mode (Soft Guardrails):
suggestion = li.suggest(task=current_issue)
# -> Tell your LLM: "LI warns that the `suggestion.action_name` approach works best here."

# For AUTO mode (Forced Recovery):
# -> REPLACE your existing manual tool execution with this line:
result = li.run(task=current_issue, cmd_string=current_command)
# -> `result` contains the tool's return value (e.g., terminal output)
# -> Pass this result back into your agent's context so it can read the output
```

### TypeScript Implementation
```typescript
import { Layerinfinite } from 'layerinfinite-sdk';
import { execSync } from 'child_process';

const li = new Layerinfinite({ apiKey: '...', agentId: 'swe-agent', mode: 'recommend' });

// 1. Normalize your generic primitives into specific action families
const bashInstall = li.action('react_bug', 'bash_install', (cmdString: string) => {
    execSync(cmdString); return true;
});

const bashGeneric = li.action('react_bug', 'bash_generic', (cmdString: string) => {
    execSync(cmdString); return true;
});

// 2. Inject into your Agent Execution Loop
// Depending on the mode you selected, drop ONE of these lines into your agent's main loop:

// For RECOMMEND mode:
// -> KEEP your existing manual tool execution exactly as it is:
bashInstall(currentCommand);
const stats = await li.observe(currentIssue);
// -> Use `stats?.successRate` to halt the agent if it gets stuck

// For ASSIST mode:
const suggestion = await li.suggest(currentIssue);
// -> Tell your LLM: "LI warns that the `suggestion?.actionName` approach works best here."

// For AUTO mode:
// -> REPLACE your existing manual tool execution with this line:
const result = await li.run(currentIssue, currentCommand);
// -> `result` contains the tool's return value (e.g., terminal output)
// -> Pass this result back into your agent's context so it can read the output
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

### Python Implementation
```python
from layerinfinite import Layerinfinite

li = Layerinfinite(api_key="...", agent_id="coordinator", mode="auto")

# 1. Decorate your Handoff Functions
@li.action("content_verification")
def delegate_to_qa(draft_content: str):
    return True # Replace with actual qa_agent.execute()

@li.action("content_verification")
def delegate_to_fact_checker(draft_content: str):
    return True # Replace with actual fact_checker_agent.execute()

# 2. Inject into your Coordinator Execution Layer
# Depending on the mode you selected, drop ONE of these lines into your Coordinator's routing node:

# For AUTO mode (Dynamic Trust Routing):
# -> REPLACE your existing manual tool execution with this line:
result = li.run(task=verification_task, draft_content=document_text)
# -> Coordinator instantly skips the LLM and routes to the most reliable sub-agent
# -> `result` contains whatever the sub-agent returns (e.g., True)
# -> Pass this result back to your Coordinator so it can proceed with the workflow

# For ASSIST mode (Trust Hints):
suggestion = li.suggest(task=verification_task)
# -> Pass `suggestion.action_name` to the Coordinator LLM to inform its handoff decision
            
# For RECOMMEND mode (Passive Trust Scoring):
# -> KEEP your existing manual tool execution exactly as it is:
delegate_to_qa(draft_content=document_text)
# -> LayerInfinite silently tracks sub-agent trust scores in the background.
```

### TypeScript Implementation
```typescript
import { Layerinfinite } from 'layerinfinite-sdk';

const li = new Layerinfinite({ apiKey: '...', agentId: 'coordinator', mode: 'auto' });

// 1. Decorate your Handoff Functions
const delegateToQa = li.action('content_verification', 'delegate_to_qa', async (draftContent: string) => {
    return true; 
});

const delegateToFactChecker = li.action('content_verification', 'delegate_to_fact_checker', async (draftContent: string) => {
    return true; 
});

// 2. Inject into your Coordinator Execution Layer
// Depending on the mode you selected, drop ONE of these lines into your Coordinator's routing node:

// For AUTO mode:
// -> REPLACE your existing manual tool execution with this line:
const result = await li.run(verificationTask, documentText);
// -> `result` contains whatever the sub-agent returns
// -> Pass this result back to your Coordinator so it can proceed with the workflow

// For ASSIST mode:
const suggestion = await li.suggest(verificationTask);
// -> Pass `suggestion?.actionName` to the Coordinator LLM

// For RECOMMEND mode:
// -> KEEP your existing manual tool execution exactly as it is:
await delegateToQa(documentText);
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
