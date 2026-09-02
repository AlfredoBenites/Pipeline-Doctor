# GitHub Publisher (`src/publisher/GitHubPublisher.bx`, `src/publisher/SafetyGuard.bx`)

## Overview

The GitHub Publisher handles the final stage of Pipeline Doctor. After another component diagnoses and repairs a failure, this feature safely records the verified change and opens a GitHub pull request for human review.

It does not diagnose errors, edit source code, approve pull requests, or merge changes automatically.

## Main Responsibilities

`GitHubPublisher` can:

* Confirm that Git and GitHub CLI are available.
* Confirm that GitHub CLI is authenticated.
* Inspect the current Git repository, branch, and changed files.
* Create a separate `pipeline-doctor/` repair branch.
* Commit a repair only after successful verification.
* Push the repair branch to GitHub.
* Open a pull request into the configured base branch.
* Return the commit SHA, branch name, and pull-request URL.

`SafetyGuard` decides whether a repair is safe to publish.

## Safety Checks

Before committing or publishing, the feature checks that:

* The target folder is a valid Git repository.
* The repair is on a branch beginning with `pipeline-doctor/`.
* The verification command completed successfully.
* The repair stage reports that it produced a repair.
* At least one changed file was reported.
* Git’s actual changed files exactly match the repair agent’s reported files.
* Files such as `.env` are not changed.
* Absolute paths and paths containing `..` are rejected.
* The repository has no leftover uncommitted changes before pushing or opening a pull request.

Only reported and verified files are staged. The publisher never runs a general `git add .`.

## Important Methods

### `checkPrerequisites(workingDirectory)`

Checks that Git is installed and that GitHub CLI is authenticated.

### `inspectRepository(workingDirectory)`

Returns the current branch, changed files, repository status, and whether the working tree is clean.

### `createRepairBranch(workingDirectory, baseBranch, branchName)`

Creates an isolated repair branch. It refuses to continue if the repository is dirty or is not currently on the expected base branch.

### `commitVerifiedRepair(workingDirectory, repairResult, verificationResult, commitMessage)`

Validates the repair, compares the reported files with Git’s actual changes, stages only those files, creates a commit, and returns the real commit SHA.

### `pushRepairBranch(workingDirectory)`

Pushes the current repair branch to the repository’s `origin` remote and configures its upstream branch.

### `createPullRequest(workingDirectory, baseBranch, pullRequestTitle, pullRequestBody)`

Uses GitHub CLI to create a pull request and returns its URL. It does not merge the pull request.

## Expected Input

The commit stage expects a repair result similar to:

```json
{
  "diagnosis": "The program called toUpperCase() on null.",
  "summary": "Replaced the null value with a valid string.",
  "changedFiles": ["scr1/App.java"],
  "repaired": true
}
```

It also expects a verification result containing:

```json
{
  "exitCode": 0,
  "succeeded": true
}
```

These field names must match the results returned by the runner and repair components.

## Challenges Encountered

### Reading Git status output

Git returns changed files as multiple newline-separated status entries. BoxLang did not recognize `chr(10)` in the installed runtime, so the implementation uses `char(10)` to separate the lines correctly.

### Running commands safely

Git and GitHub commands are executed through BoxLang’s `systemExecute()`. Arguments are passed as arrays so filenames and commit messages are treated as arguments instead of being combined into an unsafe shell command.

### Preventing unrelated files from being committed

The repair agent’s reported file list cannot be trusted by itself. The safety guard compares it with the files Git actually sees. Publication is rejected if either list contains a file missing from the other.

### GitHub authentication

GitHub CLI and Git authentication are separate concerns. GitHub CLI must be authenticated, and pushing through SSH may require unlocking the user’s local SSH key.

### Repeatable demonstrations

A branch or pull request with the same name may already exist after a rehearsal. Integration and presentation runs should use a unique `pipeline-doctor/` branch name or reset the demo repository carefully. Existing pull requests must never be merged or deleted automatically.

## How to Test

Run tests from the root of the `Pipeline-Doctor` repository.

### Read-only tests

```bash
boxlang test-publisher.bxs
boxlang test-safety-guard.bxs
```

`test-publisher.bxs` checks Git, GitHub authentication, and repository inspection.

`test-safety-guard.bxs` confirms that failed verification, unsafe files, and unexpected Git changes are rejected.

### Git and GitHub tests

The following tests perform real state-changing operations against the sibling `../pipeline-doctor-demo` repository:

```bash
boxlang test-create-branch.bxs
boxlang test-commit-repair.bxs
boxlang test-push-branch.bxs
boxlang test-create-pr.bxs
```

They cover:

1. Creating a repair branch.
2. Committing a verified changed file.
3. Pushing the repair branch.
4. Opening a real GitHub pull request.

Before running them, the demo repository must be clean and on its broken `main` branch. The branch name used by the tests must not already exist locally or remotely. A repair must also be applied and successfully verified before running the commit test.

These scripts should not be rerun blindly because they create real Git branches, commits, and pull requests.

## Current MVP Boundary

The publisher supports the current single-repository Java demonstration. It assumes that the runner and repair agent provide compatible result structures.

The full Pipeline Doctor integration must connect these methods in the correct order:

1. Run the broken project.
2. Capture the failure.
3. Create the repair branch.
4. Apply the repair.
5. Verify the repaired project.
6. Commit the verified file.
7. Push the branch.
8. Open a pull request for human review.