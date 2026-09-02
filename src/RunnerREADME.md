# CommandRunner & FailureContext (`src/CommandRunner.bx`, `src/FailureContext.bx`)

## Overview
Responsible for Stage 3 (Reproduce) and Stage 6 (Verify) of the Pipeline Doctor lifecycle[cite: 1]. It executes the target repository's configured command via OS process isolation, captures stdout/stderr streams and exit codes, and transforms crash traces into bounded context packages for AI repair[cite: 1].

---

## Module Contracts

### Produces: `CommandResult`
Returned by `CommandRunner.run(config, repoPath)`[cite: 1]:
* `command`: Full CLI invocation string (e.g., `java src/App.java`)[cite: 1].
* `success`: Boolean indicating whether the process exited cleanly with code `0`[cite: 1].
* `exitCode`: Native exit code returned by `systemExecute()`[cite: 1].
* `stdout`: Standard output stream[cite: 1].
* `stderr`: Standard error stream containing compiler errors or uncaught exceptions[cite: 1].
* `combinedOutput`: Interleaved output stream preserving failure order[cite: 1].
* `durationMs`: Execution time in milliseconds[cite: 1].
* `timedOut`: Boolean indicating whether the timeout threshold was exceeded[cite: 1].

### Produces: `FailureContext`
Built by `FailureContext.build(commandResult, repoPath, config)`[cite: 1]:
* `repoPath`: Absolute directory path of target repository[cite: 1].
* `failureOutput`: Captured execution error log[cite: 1].
* `candidateFiles`: Array of identified `.java` source files extracted from stack traces[cite: 1].
* `sourceSnippets`: Struct mapping source filenames to their in-memory file contents[cite: 1].
* `lineNumber`: Line number extracted from the top of the stack trace[cite: 1].

---

## Safety & Boundaries
* **Timeout Enforcement:** Kills stuck or long-running processes via a configurable timeout ceiling (default: 60s)[cite: 1].
* **Execution Boundary:** Commands execute strictly within the designated target repository directory[cite: 1].
* **Clean Exit Short-Circuit:** If initial execution succeeds (`exitCode == 0`), the pipeline halts immediately without invoking the repair agent[cite: 1].

---

## Verification

Run Person A's test harness:
```bash
boxlang src/TestFailureContext.bxs