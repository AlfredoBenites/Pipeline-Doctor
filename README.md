# Pipeline Doctor 🩺

Automated failure reproduction, AI root-cause repair, and Pull Request generation built on BoxLang.

---

## Prerequisites

| Dependency | Requirement | Purpose |
| :--- | :--- | :--- |
| **Java** | OpenJDK 17+ | Runtime target execution |
| **Git** | CLI installed | Branch management & commits |
| **GitHub CLI** | `gh` authenticated | Remote PR creation |
| **BoxLang** | Installed CLI | Execution runtime (`.bx`, `.bxs`) |

```bash
# macOS
brew install openjdk@17 git gh

# Ubuntu / Debian
sudo apt update && sudo apt install -y openjdk-17-jdk git gh

# BoxLang CLI
curl -fsSL https://boxlang.io/install.sh | 
```

1. GitHub Setup
Authenticate gh CLI so the orchestrator can create PRs:

gh auth login

 - Select GitHub.com -> HTTPS -> Yes (authenticate Git) -> Browser Login.

2. Directory Layout
The demo target must sit side-by-side with Pipeline-Doctor to isolate git trees:

workspace/
├── Pipeline-Doctor/          # Tooling repo
└── pipeline-doctor-demo/     # Target repo to diagnose & fix

3. Clone Repositories

```bash
# Clone the orchestrator
git clone https://github.com/AlfredoBenites/Pipeline-Doctor.git
cd Pipeline-Doctor

# Clone the target demo repo alongside it
git clone https://github.com/AlfredoBenites/pipeline-doctor-demo.git ../pipeline-doctor-demo
```

4. Run the Pipeline
Ensure the demo repository is on a clean branch:

```bash
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo reset --hard origin/main
git -C ../pipeline-doctor-demo clean -fd
```

Run the end-to-end automation:

```bash
boxlang main.bxs
```

Resetting for Another Run

```bash
git -C ../pipeline-doctor-demo checkout main
git -C ../pipeline-doctor-demo reset --hard origin/main
git -C ../pipeline-doctor-demo clean -fd
boxlang main.bxs
```
