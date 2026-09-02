# Integrate the Pipeline Doctor MVP

You are working inside the `Pipeline-Doctor` BoxLang repository. The sibling repository `../pipeline-doctor-demo` is the target demo project.

Do not assume the application currently works end-to-end. The individual components were developed separately and merged, but `main.bxs` is still a fake placeholder pipeline.

## Goal

Build and test one real BoxLang pipeline that:

1. Reads the target repository’s `pipeline-doctor.json`.
2. Confirms the target repository is clean and on its configured base branch.
3. Runs the configured Java command.
4. Detects the non-zero exit code and captures the error.
5. Builds failure context containing the correct source path and source code.
6. Creates a new `pipeline-doctor/` repair branch before changing code.
7. Calls `RepairAgent.bx` to diagnose and repair the source file.
8. Runs the same Java command again to verify the repair.
9. Refuses to publish if verification fails or unsafe/unexpected files changed.
10. Commits only the verified repaired file.
11. Pushes the repair branch.
12. Creates a GitHub pull request into `main`.
13. Prints understandable progress followed by a final JSON result containing the diagnosis, changed files, verification result, commit SHA, and PR URL.
14. Never merges the pull request automatically.

Keep this limited to the current Java MVP. Do not redesign the entire project.

## Existing components

Inspect and reuse these files:

* `src/CommandRunner.bx`
* `src/FailureContext.bx`
* `src/RepairAgent.bx`
* `src/publisher/GitHubPublisher.bx`
* `src/publisher/SafetyGuard.bx`
* `main.bxs`
* All existing test scripts

Target demo configuration:

* Repository: `../pipeline-doctor-demo`
* Configuration: `../pipeline-doctor-demo/pipeline-doctor.json`
* Source file: `scr1/App.java`
* Base branch: `main`
* Initial command: `java scr1/App.java`

The demo repository’s `main` branch must remain intentionally broken.

## Required compatibility fixes

Inspect the code before editing, but the currently merged versions have these known mismatches:

### Command result

`CommandRunner.bx` currently returns `success`, while the publisher expects `succeeded`.

Make `succeeded` the shared field. Keeping `success` temporarily for compatibility is acceptable.

Ensure both the normal result and caught-error result include it.

### Source path

`FailureContext.bx` currently places only `App.java` in `candidateFiles`.

The repair agent needs the repository-relative path from the configuration:

`scr1/App.java`

Use `relativeSourcePath` in `candidateFiles` and use the same relative path as the `sourceSnippets` key.

Do not hardcode `scr1/App.java` inside the reusable classes.

### Repair result

`RepairAgent.bx` currently returns `safeToVerify`, while `SafetyGuard.bx` expects `repaired`.

Every RepairAgent result must include:

* `diagnosis`
* `summary`
* `changedFiles`
* `repaired`

A successful single-file modification should return `repaired: true`. Aborted or unsuccessful repairs should return `repaired: false`.

Keeping `safeToVerify` as an additional field is acceptable.

## AI behavior

The current repair agent reads `GEMINI_API_KEY` and falls back to a hardcoded null-pointer repair when the AI request fails.

Do not falsely report that AI produced the repair when the fallback was used.

Add an explicit field such as:

* `repairMode: "ai"`
* `repairMode: "fallback"`

Print which mode was used.

Check whether the configured Gemini model and response parsing actually work. Never print or commit the API key. If no API key is available, test the fallback but clearly report that it was the fallback.

Do not replace the current repair approach with a large new framework unless absolutely necessary for this MVP.

## Pipeline behavior

Replace the fake functions in `main.bxs` with the real pipeline.

The target path should be configurable. Prefer supporting a command such as:

`boxlang main.bxs ../pipeline-doctor-demo`

If that is not compatible with the installed BoxLang version, use a clearly documented environment variable or a simple safe default.

Stop immediately with a useful message when:

* The configuration file is missing or invalid.
* The repository is dirty.
* The starting branch is not `main`.
* The initial command already succeeds.
* Failure context cannot identify a permitted source file.
* The repair does not change a file.
* Verification fails.
* Git detects files not reported by the repair agent.
* Commit, push, or pull-request creation fails.

Never commit `.env`, API keys, absolute paths, files outside the target repository, or files that the repair agent did not report.

## Existing demo state

There is already an open demonstration PR using:

`pipeline-doctor/fix-null-pointer`

Do not merge, close, delete, or overwrite that PR or branch.

Generate a unique safe branch name for integration testing, such as a `pipeline-doctor/` name with a timestamp or short unique suffix.

Before testing, ensure the local demo repository is clean and switched back to its broken `main` branch. Do not modify remote `main`.

You may create one new repair branch and pull request to prove the complete pipeline, but never merge it, force-push, or delete existing branches.

## Working method

1. Inspect the complete repository and Git status.
2. Explain briefly what is currently disconnected.
3. Make the smallest necessary changes.
4. Run existing tests before and after changes where practical.
5. Add a focused integration test or documented demo command.
6. Run the actual end-to-end flow against `../pipeline-doctor-demo`.
7. Confirm the resulting pull request changes only `scr1/App.java`.
8. Do not merge anything into `main`.
9. Do not push changes to the `Pipeline-Doctor` repository unless I approve it.
10. At the end, summarize:

* Files changed
* Contract fixes made
* Tests run and their results
* Whether real AI or fallback made the repair
* The created PR URL
* Any remaining manual setup needed for the presentation

Do not merely write a plan. Inspect, implement, test, and debug the integration.
