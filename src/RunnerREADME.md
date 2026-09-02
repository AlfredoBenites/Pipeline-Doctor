# CommandRunner & FailureContext (`src/CommandRunner.bx`, `src/FailureContext.bx`)

## Overview
Responsible for Stage 3 (Reproduce) and supporting the verification flow of the Pipeline Doctor lifecycle. It executes the target repository's configured command using BoxLang `systemExecute()`, captures stdout, stderr, exit codes, execution time, and timeout information, and transforms Java failure traces into bounded context packages for AI repair.

---

## Module Contracts

### Produces: `CommandResult`
Returned by `CommandRunner.run(config, repoPath)`:

* `command`: Full CLI invocation string (e.g., `java scr1/App.java`).
* `success`: Boolean indicating whether the process exited successfully with code `0`.
* `exitCode`: Exit code returned by the executed process.
* `stdout`: Standard output produced by the target program.
* `stderr`: Standard error produced by the target program, including compiler errors or uncaught exceptions.
* `combinedOutput`: Combined stdout and stderr used for failure analysis.
* `durationMs`: Execution time in milliseconds.
* `timedOut`: Boolean indicating whether the configured timeout was exceeded.
* `terminated`: Boolean indicating whether the process was terminated by the runner.
* `started`: Boolean indicating whether the configured executable successfully started.

If the executable cannot be started, `CommandRunner` returns a failed `CommandResult` instead of crashing Pipeline Doctor.

### Produces: `FailureContext`
Built by `FailureContext.build(commandResult, repoPath, config)`:

* `repoPath`: Directory path of the target repository.
* `failureOutput`: Captured failure output from the executed command.
* `candidateFiles`: Array containing the Java source file identified from the failure trace.
* `sourceSnippets`: Struct mapping the identified source filename to its source-code contents.
* `lineNumber`: Line number extracted from the Java source reference in the failure output.
* `contextFound`: Boolean indicating whether a usable Java source-file reference was found.

If no usable Java source reference is found, `FailureContext` returns safely with `contextFound: false` instead of throwing an exception.

---

## Safety & Boundaries

* **Timeout Enforcement:** Stops stuck or long-running processes using the timeout value provided by `config.timeoutSeconds`.
* **Execution Boundary:** Commands execute in the designated target repository directory supplied through `repoPath`.
* **Missing Executable Handling:** If the configured executable cannot start, `CommandRunner` returns `success: false`, `started: false`, and `exitCode: -1` instead of crashing.
* **Source Context Boundary:** The current MVP only accepts Java source references and respects the configured `allowedExtensions`.
* **Clean Exit Handling:** If `CommandRunner` reports `success == true`, the controller should skip `FailureContext` creation because no repair is needed.

---

## Verification

Run Person A's test harness by supplying the path to the target repository:

```bash
boxlang src/TestFailureContext.bxs C:/path/to/demo-java-bug
