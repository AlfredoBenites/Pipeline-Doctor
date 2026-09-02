# Pipeline Doctor

Pipeline Doctor is an automated failure reproduction, AI-assisted diagnosis, and patch-verification system built with BoxLang. It reproduces build/runtime failures inside an isolated target repository, synthesizes and applies targeted fixes via an AI agent, verifies execution success, and opens a GitHub Pull Request.

Architecture Overview
src.CommandRunner: Executes configured target commands in isolated processes and captures runtime results (exitCode, stdout, stderr, execution timing).

src.FailureContext: Parses stack traces, extracts candidate files, and compiles failure context bounded by configuration rules.

src.RepairAgent: Analyzes the root cause and safely generates code patches without violating structural constraints.

src.publisher.GitHubPublisher: Interacts with local Git and the GitHub CLI (gh) to branch, commit, push, and submit Pull Requests.

main.bxs: Orchestrates the 7-stage pipeline loop.

Prerequisites
Ensure the following tools are installed and available on your system PATH:

BoxLang: Runtime environment for executing .bxs and .bx files.

Java SDK: Version 17+ (required to run Java reproduction targets).

Git: Version control CLI.

GitHub CLI (gh): Required for remote branch publishing and pull request creation.

GitHub Authentication Setup
To allow GitHubPublisher to push branches and open Pull Requests directly from your terminal or VS Code, authenticate the gh CLI:

Bash
gh auth login
Select the following configuration options during login:

Account: GitHub.com

Preferred protocol: HTTPS

Authenticate Git with your GitHub credentials? Yes

Authentication method: Browser or Personal Access Token

Verify active authentication:

Bash
gh auth status
Permission Requirement: The authenticated GitHub account must have collaborator or write permissions on the target repository to push branches and publish PRs.

Local Directory Layout
To keep the orchestrator tool clean and prevent Git working tree collisions, the target repository must be cloned as a sibling directory:

Plaintext
workspace/
├── Pipeline-Doctor/          # Core tooling repository (Orchestrator)
│   ├── src/
│   │   ├── CommandRunner.bx
│   │   ├── FailureContext.bx
│   │   ├── RepairAgent.bx
│   │   └── publisher/
│   │       └── GitHubPublisher.bx
│   └── main.bxs
│
└── pipeline-doctor-demo/     # Target repository to diagnose & repair
    ├── pipeline-doctor.json  # Target run configuration
    └── scr1/
        └── App.java          # Source code containing the bug
Setup & Execution Instructions
1. Clone the Repositories Side-by-Side
Bash
# Clone the core tool repository
git clone https://github.com/AlfredoBenites/Pipeline-Doctor.git
cd Pipeline-Doctor

# Clone the demo target repository in the parent directory
git clone https://github.com/AlfredoBenites/pipeline-doctor-demo.git ../pipeline-doctor-demo
2. Verify Target Repository Cleanliness
The orchestrator requires the target demo repo to start on a clean main branch before branching:

Bash
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo pull origin main
git -C ../pipeline-doctor-demo status
Ensure pipeline-doctor-demo contains its reproduction failure (e.g., String developerName = null; in scr1/App.java).

3. Run Pipeline Doctor
From within the Pipeline-Doctor root folder, execute the orchestrator:

Bash
boxlang main.bxs
Expected Terminal Output
Plaintext
==================================================
             PIPELINE DOCTOR v1.0.0               
  Automated Failure Reproduction & Pull Request   
==================================================

[Stage 1 & 2: Setup & Branching]
Target Repository: /Users/.../pipeline-doctor-demo/
Creating isolation branch: pipeline-doctor/fix-npe-2026-XX-XX...
Switched to branch: pipeline-doctor/fix-npe-2026-XX-XX...

[Stage 3: Reproducing Failure]
Failure confirmed (Exit code: 1).
Error output preview: Exception in thread "main" java.lang.NullPointerException...

[Stage 4: Context Building]
Candidate file identified: scr1/App.java

[Stage 5: AI Diagnosis & Repair]
Diagnosis : Uninitialized variable 'developerName' caused NullPointerException on toUpperCase() invocation
Summary   : Initialized 'developerName' to a default string literal to prevent runtime null dereference
Confidence: high

[Stage 6: Verifying Fix]
VERIFICATION PASSED: Program exited cleanly (Exit code: 0)!
Output: Welcome, DEVELOPER

[Stage 7: Publishing Pull Request]
Committed repair to branch: pipeline-doctor/fix-npe-2026-XX-XX...
Pushing branch to origin...
Creating GitHub Pull Request on target repository...

==================================================
PIPELINE DOCTOR RUN COMPLETE!
Pull Request URL: https://github.com/AlfredoBenites/pipeline-doctor-demo/pull/<PR_NUMBER>
==================================================
Resetting Between Test Runs
To run the pipeline again against pipeline-doctor-demo:

Bash
# Reset demo repo back to the default failing state on main
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo reset --hard origin/main
git -C ../pipeline-doctor-demo clean -fd

# Rerun orchestrator
boxlang main.bxs
