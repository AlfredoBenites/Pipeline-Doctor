# RepairAgent (`src/RepairAgent.bx`)

## Overview
`RepairAgent` is the core AI-driven diagnostic and remediation engine of Pipeline Doctor. It consumes a structured failure context (`FailureContext`) containing runtime errors, stack traces, and candidate files, orchestrates code remediation with strict safety constraints, applies an isolated source patch, and returns a machine-readable `RepairResult` contract.

---

## Module Contract

### Consumes: `FailureContext`
* `repoPath`: Absolute root directory of the target repository.
* `failureOutput`: Combined `stdout` and `stderr` execution logs (e.g., stack trace).
* `candidateFiles`: Identified `.java` source files from stack traces.
* `sourceSnippets`: Contextual source code snippets.

### Produces: `RepairResult`
* `diagnosis`: Short root-cause explanation.
* `changedFiles`: List of edited relative file paths (strictly $\le 1$ file in MVP).
* `summary`: PR-ready description of the applied fix.
* `confidence`: Confidence rating (`high`, `medium`, `low`).
* `safeToVerify`: Boolean flag indicating strict compliance with boundary safety checks.

---

## Safety & MVP Boundaries
* **Path Traversal Guard:** Canonical path resolution enforces that all file accesses and modifications remain strictly inside `repoPath` (`../` attacks blocked).
* **Extension Enforcement:** Rejects file reads and writes to non-`.java` targets.
* **Single-File Scope:** Restricts edits to a maximum of one file per remediation run.
* **No Blind Merges:** Emits the patch and metadata contract for independent verification by Person C before opening pull requests.

---

## Challenges Encountered & Engineering Solutions

* **`bx-ai` Tool Serialization Mismatch (`INVALID_ARGUMENT: 400`):**
  * *Problem:* The `bx-ai` module formatted `aiTool()` definitions using the OpenAI schema (`type: function`), which Google Gemini endpoints reject.
  * *Solution:* Decoupled file I/O from the LLM tool-calling layer. The prompt supplies bounded file context directly, and BoxLang deterministically handles regex extraction and file writes.

* **Hardcoded Model Retirement (`404 NOT_FOUND`):**
  * *Problem:* The installed `bx-ai` module hardcoded deprecated model endpoints (`gemini-2.5-flash`), ignoring runtime model configuration overrides.
  * *Solution:* Bypassed the rigid module wrapper by invoking Google's REST endpoint directly via `systemExecute` using `curl`, targeting active endpoints (`gemini-3.6-flash`).

* **BoxLang String Parsing Errors:**
  * *Problem:* Standard C-style backslash quote escaping (`\"`) causes compile-time syntax errors in BoxLang.
  * *Solution:* Refactored all prompt and JSON string literals to use CFML/BoxLang doubled quotes (`""`) and single-quote delimiters (`'...'`).

* **Process Buffer Hanging:**
  * *Problem:* Passing large prompt payloads directly into `systemExecute(arguments=[...])` caused standard I/O buffer deadlocks.
  * *Solution:* Streamed requests via temporary JSON payload files (`--data-binary "@file"` with a `-m 10` timeout) to ensure fast, predictable process completion.

* **Free-Tier Rate Limits & Capacity Spikes (`503` / `Limit: 0`):**
  * *Problem:* Google's free API tier denied access to Pro models (`limit: 0`) and experienced periodic high-demand spikes on Flash models.
  * *Solution:* Built a resilient multi-tier remediation strategy: the agent attempts live generation against `gemini-3.6-flash`, and automatically falls back to an internal deterministic remediation engine if Google's API throttles or times out. This guarantees zero pipeline downtime.

---

## How to Run & Verify

### 1. Configure Environment
Set your API key in your terminal session:
```bash
export GEMINI_API_KEY="your-gemini-api-key"

### 2. Execute in Terminal

boxlang test_agents.bxs

### 3. Verify Output

javac test-repo/src/App.java && java -cp test-repo/src App
# Expected Output: Welcome, DEVELOPER

