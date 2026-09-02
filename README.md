Pipeline Doctor
Pipeline Doctor is an automated failure reproduction, AI-assisted diagnosis, and patch-verification system built with BoxLang. It reproduces runtime and test failures inside an isolated target repository, synthesizes and applies targeted fixes via an AI agent, verifies execution success, and automatically opens a GitHub Pull Request.

Architecture Overview
src.CommandRunner: Runs reproduction and verification commands in isolated processes, returning structured results (exitCode, stdout, stderr, execution timing).

src.FailureContext: Parses stack traces and compiler logs to extract candidate failure files and build the diagnostic context.

src.RepairAgent: Analyzes the root cause and generates safe, targeted code patches within bounded project constraints.

src.publisher.GitHubPublisher: Interfaces with Git and the GitHub CLI (gh) to branch, commit, push, and submit Pull Requests.

main.bxs: Orchestrates the 7-stage reproduction-to-PR pipeline.

Prerequisites & Installation
1. Required Tooling
Java SDK (17+): Required for BoxLang and Java runtime targets.

Git: System version control.

GitHub CLI (gh): Required for automated push and PR creation.

BoxLang CLI: The runtime used to execute .bx and .bxs files.

macOS (Homebrew)
Bash
brew install openjdk@17 git gh
Ubuntu / Debian
Bash
sudo apt update
sudo apt install -y openjdk-17-jdk git gh
2. BoxLang Installation
If BoxLang is not already installed on your machine, install it via the official installer:

Bash
curl -fsSL https://boxlang.io/install.sh | bash
Verify your environment:

Bash
boxlang --version
java -version
git --version
gh --version
GitHub Authentication Setup
Pipeline Doctor uses the gh CLI to publish branches and open pull requests directly from your local environment or VS Code.

Authenticate your GitHub account:

Bash
gh auth login
Choose the following options:

What account do you want to log into? GitHub.com

What is your preferred protocol for Git operations on this host? HTTPS

Authenticate Git with your GitHub credentials? Yes

How would you like to authenticate GitHub CLI? Login with a web browser

Confirm authentication:

Bash
gh auth status
Important: Your authenticated GitHub account must have collaborator/write permissions on the target repository to push branches and open Pull Requests.

Workspace Directory Layout
To prevent Git state conflicts, the target demo repository must live as an isolated sibling directory directly alongside the tooling repository:

Plaintext
workspace/
├── Pipeline-Doctor/          # Tooling repo (Orchestrator, Runner, RepairAgent, Publisher)
│   ├── src/
│   │   ├── CommandRunner.bx
│   │   ├── FailureContext.bx
│   │   ├── RepairAgent.bx
│   │   └── publisher/
│   │       └── GitHubPublisher.bx
│   └── main.bxs
│
└── pipeline-doctor-demo/     # Target repo to diagnose and fix
    ├── pipeline-doctor.json  # Project config & test command
    └── scr1/
        └── App.java          # Target application with bug
Quickstart Guide
1. Clone Repositories Side-by-Side
Bash
# Clone the main Pipeline Doctor orchestrator
git clone https://github.com/AlfredoBenites/Pipeline-Doctor.git
cd Pipeline-Doctor

# Clone the demo target repository in the parent directory
git clone https://github.com/AlfredoBenites/pipeline-doctor-demo.git ../pipeline-doctor-demo
2. Prepare the Target Demo Repository
Ensure the target repository starts on a clean main branch with the failing bug present:

Bash
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo reset --hard origin/main
git -C ../pipeline-doctor-demo clean -fd
3. Run Pipeline Doctor
From the root of the Pipeline-Doctor directory, run:

Bash
boxlang main.bxs
Target Project Configuration (pipeline-doctor.json)
Pipeline Doctor dynamically loads configuration rules from pipeline-doctor.json located in the root of the target repository:

JSON
{
  "executable": "java",
  "arguments": ["scr1/App.java"],
  "timeoutSeconds": 60,
  "allowedExtensions": ["java"],
  "maxFilesChanged": 1,
  "baseBranch": "main"
}
Expected Output
Plaintext
==================================================
             PIPELINE DOCTOR v1.0.0               
  Automated Failure Reproduction & Pull Request   
==================================================

[Stage 1 & 2: Setup & Branching]
Target Repository: /path/to/pipeline-doctor-demo
Creating isolation branch: pipeline-doctor/fix-npe-2026-09-02-15-59-23
Switched to branch: pipeline-doctor/fix-npe-2026-09-02-15-59-23

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
Committed repair to branch: pipeline-doctor/fix-npe-2026-09-02-15-59-23
Pushing branch to origin...
Creating GitHub Pull Request on target repository...

==================================================
PIPELINE DOCTOR RUN COMPLETE!
Pull Request URL: https://github.com/AlfredoBenites/pipeline-doctor-demo/pull/2
==================================================
Resetting Between Test Runs
To reset the target demo repo and trigger a clean end-to-end run:

Bash
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo reset --hard origin/main
git -C ../pipeline-doctor-demo clean -fd
boxlang main.bxs
