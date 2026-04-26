# Integrating LayerInfinite into a Real-World Agent

This guide provides a **fully complete, runnable agent** in both Python and TypeScript. By default, the agents are configured in **Recommend Mode** (silent telemetry). This is the fastest and safest way to verify LayerInfinite is working, as it poses zero risk to your existing architecture.

> **Note on OpenAI APIs:** This example uses standard Chat Completions for simplicity and broad compatibility. LayerInfinite is fully compatible with OpenAI's newer Responses API, and a dedicated Responses-based quickstart will be available soon.

---

## Prerequisites & Setup

Before running the examples, ensure your environment is set up.

**1. Install Dependencies**
*   **Python:** `pip install layerinfinite openai python-dotenv`
*   **TypeScript:** `npm install layerinfinite openai dotenv tsx typescript @types/node`

**2. Environment Variables**
Create a `.env` file in your project root containing your API keys:
```env
OPENAI_API_KEY="sk-proj-..."
LI_API_KEY="li_..."
```

---

## 1. Python Complete Agent (`agent.py`)

```python
import json
import os
import subprocess
from pathlib import Path
from typing import Any, Dict, List
from dotenv import load_dotenv
from openai import OpenAI
from layerinfinite import Layerinfinite

load_dotenv()

# --- 1. INITIALIZE LAYERINFINITE ---
li = Layerinfinite(
    api_key=os.getenv("LI_API_KEY"), 
    agent_id="coding-agent", 
    mode="recommend"  # Defaulting to silent telemetry mode
)

ROOT = Path(os.getenv("AGENT_WORKSPACE", ".")).resolve()
MODEL = os.getenv("MODEL", "gpt-4o-mini")
MAX_STEPS = int(os.getenv("MAX_STEPS", "20"))
MAX_FILE_BYTES = int(os.getenv("MAX_FILE_BYTES", "120000"))
MAX_OUTPUT_CHARS = int(os.getenv("MAX_OUTPUT_CHARS", "12000"))

SYSTEM_PROMPT = """
You are a senior software engineering agent operating inside a project workspace.
Return ONLY valid JSON with exactly one action per turn.
Allowed actions:
1. {"action":"list_files","path":"."}
2. {"action":"read_file","path":"src/main.py"}
3. {"action":"write_file","path":"src/main.py","content":"full file content here"}
4. {"action":"run_shell","command":"pytest -q"}
5. {"action":"final","message":"what changed and why"}
"""

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def truncate(text: str, limit: int = MAX_OUTPUT_CHARS) -> str:
    if len(text) <= limit: return text
    return f"{text[:limit//2]}\n...<truncated>...\n{text[-limit//2:]}"

def safe_path(path_str: str) -> Path:
    path = (ROOT / path_str).resolve()
    if not str(path).startswith(str(ROOT)): raise ValueError("Path escapes root")
    return path

# --- 2. DECORATE TOOLS ---
# By decorating these, LayerInfinite silently tracks their success rates.
# Note: **kwargs is added so LayerInfinite can safely swap tools even if arguments differ.

@li.action("coding_task")
def list_files(path_str: str = ".", **kwargs) -> Dict[str, Any]:
    path = safe_path(path_str)
    if path.is_file(): return {"ok": True, "files": [str(path.relative_to(ROOT))]}
    files = [str(p.relative_to(ROOT)) for p in path.rglob("*") if not any(part in {".git", "node_modules"} for part in p.parts)][:400]
    return {"ok": True, "files": files}

@li.action("coding_task")
def read_file(path_str: str, **kwargs) -> Dict[str, Any]:
    path = safe_path(path_str)
    if not path.is_file(): return {"ok": False, "error": "File not found"}
    return {"ok": True, "content": truncate(path.read_text(errors="ignore"))}

@li.action("coding_task")
def write_file(path_str: str, content: str, **kwargs) -> Dict[str, Any]:
    path = safe_path(path_str)
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content)
    return {"ok": True, "written": str(path.relative_to(ROOT))}

@li.action("coding_task")
def run_shell(command: str, **kwargs) -> Dict[str, Any]:
    try:
        result = subprocess.run(command, shell=True, cwd=ROOT, text=True, capture_output=True, timeout=90)
        return {"ok": result.returncode == 0, "stdout": truncate(result.stdout), "stderr": truncate(result.stderr)}
    except subprocess.TimeoutExpired:
        return {"ok": False, "error": "Timeout"}

def execute_action(action_obj: Dict[str, Any]) -> Dict[str, Any]:
    action = action_obj.get("action")
    try:
        if action == "list_files": return list_files(action_obj.get("path", "."))
        if action == "read_file": return read_file(action_obj["path"])
        if action == "write_file": return write_file(action_obj["path"], action_obj["content"])
        if action == "run_shell": return run_shell(action_obj["command"])
        if action == "final": return {"ok": True, "final": action_obj.get("message", "Completed")}
        return {"ok": False, "error": f"Unknown action: {action}"}
    except Exception as exc: return {"ok": False, "error": str(exc)}

def call_model(messages: List[Dict[str, str]]) -> str:
    response = client.chat.completions.create(model=MODEL, temperature=0, messages=messages)
    return (response.choices[0].message.content or "").strip()

def run_agent(user_task: str) -> str:
    messages = [{"role": "system", "content": SYSTEM_PROMPT}, {"role": "user", "content": user_task}]

    for step in range(1, MAX_STEPS + 1):
        raw = call_model(messages)
        try: action_obj = json.loads(raw)
        except json.JSONDecodeError:
            messages.extend([{"role": "assistant", "content": raw}, {"role": "user", "content": "Return valid JSON."}])
            continue

        # --- 3. THE EXECUTION LOOP ---
        result = execute_action(action_obj)

        if "final" in result: return result["final"]
        messages.extend([{"role": "assistant", "content": json.dumps(action_obj)}, {"role": "user", "content": json.dumps(result)}])
    return "Stopped: max steps reached."

if __name__ == "__main__":
    print(run_agent("Please read src/main.py and fix the syntax error."))
```

### Running and Verifying (Python)
1. **Run the agent:** `python agent.py`
2. **Verification:** Open your LayerInfinite Dashboard. Under the **Telemetry** or **Tasks** tab, you should instantly see `"coding_task"` populated with the actions (e.g., `list_files`, `read_file`) and their success rates!

---

## 2. TypeScript Complete Agent (`agent.ts`)

```typescript
import fs from "fs";
import path from "path";
import process from "process";
import { exec } from "child_process";
import { promisify } from "util";
import OpenAI from "openai";
import dotenv from "dotenv";
import { Layerinfinite, action } from "layerinfinite";

dotenv.config();

// --- 1. INITIALIZE LAYERINFINITE ---
const li = new Layerinfinite({
  apiKey: process.env.LI_API_KEY!,
  agentId: "coding-agent",
  mode: "recommend" // Defaulting to silent telemetry mode
});

const execAsync = promisify(exec);
const ROOT = path.resolve(process.env.AGENT_WORKSPACE || ".");
const MODEL = process.env.MODEL || "gpt-4o-mini";
const MAX_STEPS = Number(process.env.MAX_STEPS || 20);

const SYSTEM_PROMPT = `
You are a senior software engineering agent operating inside a project workspace.
Return ONLY valid JSON with exactly one action per turn.
Allowed actions:
1. {"action":"list_files","path":"."}
2. {"action":"read_file","path":"src/main.ts"}
3. {"action":"write_file","path":"src/main.ts","content":"full file content here"}
4. {"action":"run_shell","command":"npm test"}
5. {"action":"final","message":"what changed and why"}
`;

function safePath(pathStr: string): string {
  const resolved = path.resolve(ROOT, pathStr);
  if (!resolved.startsWith(ROOT)) throw new Error("Path escapes root");
  return resolved;
}

// --- 2. WRAP TOOLS ---
// By wrapping these, LayerInfinite silently tracks their success rates.
// Note: ...args is added so LayerInfinite can safely swap tools even if arguments differ.

const listFiles = action("coding_task", function listFiles(pathStr = ".", ...args: any[]) {
  const target = safePath(pathStr);
  if (fs.existsSync(target) && fs.statSync(target).isFile()) return { ok: true, files: [path.relative(ROOT, target)] };
  
  const files: string[] = [];
  function walk(dir: string) {
    if (files.length > 400) return;
    for (const file of fs.readdirSync(dir)) {
      if ([".git", "node_modules"].includes(file)) continue;
      const fullPath = path.join(dir, file);
      if (fs.statSync(fullPath).isDirectory()) walk(fullPath);
      else files.push(path.relative(ROOT, fullPath));
    }
  }
  walk(target);
  return { ok: true, files };
});

const readFile = action("coding_task", function readFile(pathStr: string, ...args: any[]) {
  const target = safePath(pathStr);
  if (!fs.existsSync(target) || !fs.statSync(target).isFile()) return { ok: false, error: "Not found" };
  return { ok: true, content: fs.readFileSync(target, "utf8").slice(0, 12000) };
});

const writeFile = action("coding_task", function writeFile(pathStr: string, content: string, ...args: any[]) {
  const target = safePath(pathStr);
  fs.mkdirSync(path.dirname(target), { recursive: true });
  fs.writeFileSync(target, content, "utf8");
  return { ok: true, written: path.relative(ROOT, target) };
});

const runShell = action("coding_task", async function runShell(command: string, ...args: any[]) {
  try {
    const { stdout, stderr } = await execAsync(command, { cwd: ROOT, timeout: 90_000 });
    return { ok: true, stdout, stderr };
  } catch (error: any) {
    return { ok: false, error: error.message };
  }
});

async function executeAction(actionObj: any) {
  try {
    switch (actionObj.action) {
      case "list_files": return listFiles(actionObj.path || ".");
      case "read_file": return readFile(actionObj.path);
      case "write_file": return writeFile(actionObj.path, actionObj.content);
      case "run_shell": return await runShell(actionObj.command);
      case "final": return { ok: true, final: actionObj.message || "Completed" };
      default: return { ok: false, error: `Unknown action` };
    }
  } catch (error: any) { return { ok: false, error: String(error) }; }
}

async function runAgent(userTask: string): Promise<string> {
  const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  const messages: any[] = [{ role: "system", content: SYSTEM_PROMPT }, { role: "user", content: userTask }];

  for (let step = 1; step <= MAX_STEPS; step += 1) {
    const response = await client.chat.completions.create({ model: MODEL, temperature: 0, messages });
    const raw = response.choices[0]?.message?.content || "";

    let actionObj;
    try { actionObj = JSON.parse(raw); } 
    catch {
      messages.push({ role: "assistant", content: raw }, { role: "user", content: "Return valid JSON." });
      continue;
    }

    // --- 3. THE EXECUTION LOOP ---
    const result = await executeAction(actionObj);

    if ((result as any).final) return (result as any).final;
    messages.push({ role: "assistant", content: JSON.stringify(actionObj) }, { role: "user", content: JSON.stringify(result) });
  }
  return "Stopped: max steps reached.";
}

async function main() {
  console.log(await runAgent("Please read src/main.ts and fix the bug."));
}
main();
```

### Running and Verifying (TypeScript)
1. **Run the agent:** `npx tsx agent.ts`
2. **Verification:** Check your LayerInfinite Dashboard. You will see your TypeScript actions logged under `"coding_task"`.

---

## Next Steps: Advanced Routing Modes

Once you have verified that Recommend Mode is successfully logging your telemetry, you can unlock LayerInfinite's active routing capabilities.

### Option A: Assist Mode (Probabilistic Prompt Injection)
**Why use this:** Use this when your LLM struggles with decision paralysis or hallucinates incorrect tools. By fetching the recommendation and injecting it into the system prompt *before* generation, you use statistical history to bias the LLM's attention, significantly reducing hallucination rates while keeping the LLM ultimately in control.

**How to activate:**
1. Change initialization to `mode="assist"` (or `mode: "assist"` in TS).
2. Inject the hint right before calling your LLM:
   * **Python:**
     ```python
     suggestion = li.suggest(task="coding_task")
     if suggestion and suggestion.action_name:
         messages.append({"role": "system", "content": f"Hint: Recommend '{suggestion.action_name}'"})
     ```
   * **TypeScript:**
     ```typescript
     const suggestion = await li.suggest("coding_task");
     if (suggestion?.actionName) {
         messages.push({ role: "system", content: `Hint: Recommend '${suggestion.actionName}'` });
     }
     ```

### Option B: Auto Mode (Deterministic Circuit Breaker)
**Why use this:** Use this for highly brittle tasks where LLM unpredictability is unacceptable. By stripping the LLM's chosen action and delegating to `li.run()`, you bypass the LLM's tool-selection reasoning entirely. LayerInfinite acts as a hardcoded statistical router, guaranteeing execution of the historically most successful tool.

**How to activate:**
1. Change initialization to `mode="auto"` (or `mode: "auto"` in TS).
2. Replace your manual tool execution with `li.run()`:
   * **Python:**
     ```python
     # Remove the 'action' key so we are left with only the tool arguments (e.g., {"path": "main.py"})
     action_name = action_obj.pop("action", None) 
     
     # Pass those exact tool arguments into li.run using standard Python unpacking (**)
     result = li.run(task="coding_task", **action_obj)
     ```
   * **TypeScript:**
     ```typescript
     // Extract the tool arguments (e.g., { path: "main.ts" })
     const { action: llmActionName, ...args } = actionObj;
     
     // Pass those arguments into li.run
     const result = await li.run("coding_task", args);
     ```
