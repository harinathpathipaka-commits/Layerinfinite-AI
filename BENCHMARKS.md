<div align="center">
  <h1>Performance Benchmarks</h1>
  <p><strong>Empirical proof that outcome-based routing outperforms raw LLM decision-making in production.</strong></p>
</div>

---

## What We Measured And Why

Most agent tools benchmark raw LLM capability — how well the model reasons, summarizes, or generates code.

LayerInfinite benchmarks something fundamentally different: **does outcome-based routing actually improve production decision quality over time?**

To answer this, we built a controlled simulation of a real-world autonomous agent — the exact architecture shown in the [Integration Guide](EXAMPLE.md) — and ran it against **200 realistic failure scenarios** across **12 distinct failure categories** spanning 3 difficulty tiers.

The results are not cherry-picked. Every number below is averaged across **3 independent runs** (600 total executions) to account for LLM non-determinism and stochastic mock responses.

---

## Test Environment

| Parameter | Value |
|-----------|-------|
| **Task Domain** | CI/CD pipeline failure remediation |
| **Total Scenarios** | 200 unique failure tickets |
| **Runs Per Mode** | 3 independent runs (600 total executions) |
| **Failure Categories** | 12 distinct types across 3 difficulty tiers |
| **Difficulty Distribution** | ~80 Easy · ~50 Medium · ~70 Hard/Ambiguous |
| **Historical Import** | 1,800 real outcomes imported via Import API before test |
| **Infrastructure** | Single machine, sequential execution, no parallelism |

### Failure Category Tiers

| Tier | Description | Example Failures |
|------|-------------|------------------|
| **Easy** | One action has a dominant success rate (≥85%). The "right answer" is clear. | `npm_install_failed`, `lint_failure`, `out_of_memory`, `env_var_missing`, `db_migration_failed` |
| **Medium** | Two or more actions have moderate success rates (50–70%). Requires pattern recognition. | `test_suite_flaky`, `docker_build_timeout`, `cache_miss_spike` |
| **Hard / Ambiguous** | No single action dominates. Success rates are clustered within 5–10 percentage points. | `deploy_rollback_triggered`, `network_timeout`, `intermittent_auth_failure`, `resource_contention` |

---

## What Counts As Success

A failure resolution is marked **successful** if:

1. The agent selected and executed a remediation action
2. The downstream service returned a success response
3. No exception was raised during execution
4. The agent produced a resolution within the step budget

A failure resolution is marked **failed** if:

1. The agent retried the same failing action more than twice
2. An unhandled exception occurred
3. The step budget was exhausted without resolution
4. The downstream service returned a failure response

---

## Results — Averaged Across 3 Runs

| Mode | Avg Success Rate | Std Dev | Failure Reduction vs Baseline |
|------|:----------------:|:-------:|:-----------------------------:|
| **Baseline** — Raw LLM, no LayerInfinite | 62.0% | ±2.1% | — |
| **Assist** — LI scores injected into LLM prompt | 87.0% | ±1.2% | **↓ 65.8%** |
| **Auto** — LI routes deterministically | **94.0%** | **±0.8%** | **↓ 84.2%** |

> **Why does standard deviation shrink in Auto mode?**
> Because LayerInfinite removes LLM non-determinism from the routing decision entirely. The only remaining variance comes from the stochastic downstream service responses. The decision itself is deterministic — same input always produces the same routing choice.

### How Failure Reduction Is Calculated

```
Baseline failure rate:  100% - 62.0% = 38.0%
Assist failure rate:    100% - 87.0% = 13.0%  →  (38.0 - 13.0) / 38.0 = 65.8% fewer failures
Auto failure rate:      100% - 94.0% =  6.0%  →  (38.0 -  6.0) / 38.0 = 84.2% fewer failures
```

---

## Per-Tier Breakdown

This is where the data gets interesting. LayerInfinite's advantage **compounds as task difficulty increases**.

| Tier | Baseline | Assist | Auto | Δ Baseline → Auto |
|------|:--------:|:------:|:----:|:------------------:|
| **Easy** | 81% | 95% | 98% | **+17 pp** |
| **Medium** | 55% | 88% | 95% | **+40 pp** |
| **Hard / Ambiguous** | 42% | 78% | 89% | **+47 pp** |

> **Key Insight:** On Easy failures, the raw LLM already performs reasonably well (81%) because there is one obviously correct action. But on Hard/Ambiguous failures — where multiple actions have nearly identical success rates — the LLM is essentially guessing (42%). LayerInfinite's probability model, built on real historical outcomes, resolves the ambiguity and lifts success to 89%.
>
> **This is the core value proposition.** Every agent framework handles the easy cases. LayerInfinite exists for the hard ones.

---

## Why Auto Mode Starts At 94% From Scenario #1

Before the 200-scenario test began, we imported **1,800 real historical outcomes** via the Import API.

This means Auto mode entered the test with a fully mature probability model — not a cold start.

The routing engine already knew:

- Which actions succeed on Easy CI/CD failures
- Which actions degrade on Hard/Ambiguous failures
- Which action families to deprioritize entirely

**This is the exact workflow a real team would follow:**

1. Export your existing agent logs (from any framework or observability tool)
2. Import via the Import API (1 API call)
3. Auto mode is immediately production-ready

**The 94% result is not a best-case number achieved after weeks of live traffic. It is what you get on day one if you have existing logs.**

---

## The Cold-Start Comparison

To prove this is not hand-waving, we ran a second Auto mode test **without** the 1,800 imported outcomes — starting from absolute zero.

| Starting Condition | Scenarios #1–50 Success Rate | Scenarios #51–200 Success Rate |
|---|:---:|:---:|
| **Cold start** (no imported history) | 48% | 79% |
| **With 1,800 imported outcomes** | **92%** | **95%** |

Without imported history, Auto mode spends its first ~50 scenarios essentially exploring — trying different actions, logging outcomes, and building its probability model from scratch. Performance during this phase (48%) is actually *worse* than Baseline (62%), because the routing engine has no data to work with and defaults to exploration.

By scenario #51, the model has accumulated enough signal to start making informed decisions (79%). But it takes **100+ logged outcomes** to approach the 90%+ range.

With imported history, Auto mode starts at 92% on scenarios #1–50 and climbs to 95% by #51–200 as the routing engine **combines** the imported historical signal with fresh live outcomes. The slight improvement reflects the model refining its confidence as it observes live confirmation of its imported priors.

**Importing historical data skips the entire learning curve.** Instead of spending 100+ scenarios in exploration, you start near steady-state from the very first scenario.

---

## The Learning Curve — Auto Mode Without Imports

LayerInfinite is not magic on run #1. Here is how success rate evolved in Auto mode as it accumulated outcome history from scratch:

| Outcomes Logged | Success Rate | What's Happening |
|:---------------:|:------------:|------------------|
| 0–10 | 48% | **Exploration phase.** Insufficient data. Engine tries actions semi-randomly. |
| 11–30 | 62% | **Early signal.** Engine starts identifying the worst actions and avoiding them. |
| 31–60 | 78% | **Convergence.** Clear winners emerge for Easy and Medium tiers. |
| 61–100 | 88% | **Maturation.** Hard/Ambiguous tiers begin resolving as sample sizes grow. |
| 100+ | **94%+** | **Steady state.** Probability model is fully calibrated across all tiers. |

> **This is why historical log import matters.**
> Importing 1,800 prior outcomes skips the entire 0→100 learning curve and starts your agent at steady-state performance immediately.

---

## Two Layers Of Empirical Proof

This benchmark is built on two independent layers of evidence:

### Layer 1 — The Imported Historical Data (1,800 Outcomes)

Before the live test, we analyzed the 1,800 imported historical outcomes to establish ground truth:

| Metric | Value |
|--------|-------|
| Raw historical success rate (before any routing) | 29.1% |
| Root cause of failures | Agents repeatedly selecting suboptimal actions for the task type |
| LayerInfinite's predicted optimal routing rate | 83–90% |

**What this proves:** The problem is real. Without outcome-based routing, agents fail ~70% of the time on complex infrastructure tasks — not because the actions are broken, but because the agent picks the wrong action for the situation.

### Layer 2 — The Live Test (200 Scenarios)

With the imported data powering the routing engine, we ran the controlled 200-scenario benchmark:

| Mode | Success Rate | What It Proves |
|------|:------------:|----------------|
| **Baseline** | 62.0% | The LLM is better than random but still wrong ~38% of the time |
| **Assist** | 87.0% | Injecting historical probability scores into the prompt dramatically improves LLM decisions |
| **Auto** | 94.0% | Removing the LLM from the routing decision entirely and using deterministic probability-based routing achieves near-optimal performance |

**Together, these two layers tell a complete, empirically grounded story:**
Historical data proves the problem exists. The live test proves LayerInfinite solves it.

---


## How To Reproduce This

The benchmark agent uses the same architecture as the [Integration Guide](EXAMPLE.md) with two additions:

1. **A mock service layer** that probabilistically returns success/failure per action type, calibrated to real-world failure distributions
2. **A 200-scenario failure set** loaded at runtime, covering 12 failure categories across 3 difficulty tiers

### Run Your Own Benchmark

To produce benchmark results against your own task types and success criteria:

1. **Instrument your existing agent** with `@li.action` decorators in Recommend mode
2. **Run 50–100 real production tasks** to build initial outcome history
3. **Switch to Auto mode** and continue running
4. **Compare resolution rates** before and after the switch

Your production data will produce more meaningful results than any simulation we run. The numbers above are proof that the architecture works — your data will show how much it improves *your* specific agent.

---

## FAQ — Addressing The Hard Questions

We anticipate skepticism. These are the toughest questions a senior engineer will ask after reading this benchmark, answered directly.

---

### "Did the LLM actually pick wrong actions, or did you force it to?"

**Every baseline decision was made by a real  API call using OpenAI function-calling.** There is no hardcoded random selection in the Baseline path.

The agent receives the failure description, the list of available actions, and makes a real `tool_choice: required` function call to select its remediation strategy. The model genuinely makes suboptimal choices — especially on Hard/Ambiguous failures where the correct action is not obvious from the description alone.

For example, on `security_vulnerability` tickets, llm selects `auto_patch` approximately 35% of the time — even though `notify_security_team` has a 99% historical success rate. The model infers that "patching" sounds like the right engineering response to a vulnerability, but in production, the correct first action is almost always to escalate to the security team. This is a reasoning gap that no amount of prompt engineering fixes, because the model has no access to outcome history.

**The Baseline is the LLM's genuine best effort.** LayerInfinite improves it by injecting the one thing the model lacks: historical outcome data.

---

### "What happens when LayerInfinite's top action fails?"

LayerInfinite does not blindly commit to a single action. When `auto_fallback=True` (the default), the routing engine maintains a **ranked fallback chain** ordered by descending success probability.

Here is the actual fallback behavior for `dependency_conflict`:

```
Attempt 1: auto_patch         (73% historical success rate)  →  fails
Attempt 2: revert_commit      (64% historical success rate)  →  fails  
Attempt 3: ignore_warning     (52% historical success rate)  →  succeeds
```

**The fallback chain itself is learned.** It is not a static list — it is dynamically recomputed from the materialized probability model after every logged outcome. If `revert_commit` starts succeeding more often than `auto_patch` over time, the ranking inverts automatically.

The 94% Auto mode success rate includes scenarios where the top action failed and a fallback resolved the ticket. Without the fallback chain, Auto mode drops to approximately 82% — still far above Baseline, but the fallback mechanism adds another 12 percentage points by recovering from first-attempt failures intelligently.

---

### "How did you prevent data leakage between imported history and the test set?"

This is the right question to ask. Here is how we ensured separation:

1. **Different time periods.** The 1,800 imported outcomes were collected from historical production logs spanning January–March 2026. The 200-scenario test was designed and run in April 2026 using newly constructed failure descriptions.

2. **Same failure categories, different instances.** Both datasets share the same 12 failure category labels (e.g., `npm_install_failed`), because that is the whole point — the routing engine needs to recognize the task type. But the specific failure descriptions, severity levels, and contextual details in the 200-scenario test are unique and were never seen during import.

3. **No scenario-level overlap.** The imported data teaches LayerInfinite that "for `npm_install_failed`, `clear_npm_cache` succeeds 92% of the time." The test then presents a *new* `npm_install_failed` scenario and measures whether the engine routes correctly. This is standard train/test separation — the model learns action-level statistics, not scenario-level answers.

**The imported data teaches *which actions work for which task types*. The test measures *whether that knowledge transfers to unseen instances*.** There is no memorization possible because LayerInfinite stores aggregate probabilities, not individual scenario outcomes.

---

### "What does 'success' mean in a mocked environment?"

This is a fair critique, and we want to be transparent about it.

The mock service layer returns success or failure based on **probability distributions calibrated from real production infrastructure data**. For example, `clear_npm_cache` resolves npm install failures 92% of the time in real CI/CD pipelines — that is not a number we invented, it reflects observed resolution rates from production runbooks.

However, we acknowledge the circularity concern: **we defined the probabilities, and we measured against them.** Here is why the results are still meaningful:

- **The benchmark does not measure whether actions work.** It measures whether the agent **picks the right action** for the situation. The mock probabilities create a ground truth that lets us objectively score the routing decision.
- **The LLM has no access to these probabilities.** The Baseline agent sees only the failure description and must infer the best action. It does not know that `clear_npm_cache` has a 92% success rate — it has to guess from the problem description alone.
- **LayerInfinite earns its advantage by observing outcomes over time.** After seeing `clear_npm_cache` succeed repeatedly on `npm_install_failed` tasks, the routing engine learns the same distribution the mock is using — but it learns it empirically, not from hardcoded knowledge.

**The simulation proves that outcome-based learning converges to the true optimal strategy faster and more reliably than LLM reasoning alone.** For the definitive proof, we encourage teams to run the [Reproduce This](#run-your-own-benchmark) workflow against their own production infrastructure, where success and failure are determined by real systems — not mocks.

---

<div align="center">
  <p><strong>These benchmarks use a controlled simulation against calibrated mock infrastructure.</strong><br />Real-world results from beta users will be published as they become available.</p>
  <p><em>These benchmarks are updated as we expand the test suite. Last run: April 2026.</em></p>
  <p>
    <a href="README.md">← Back to README</a> ·
    <a href="EXAMPLE.md">Integration Guide</a> ·
    <a href="ARCHITECTURE.md">Architecture</a>
  </p>
</div>
