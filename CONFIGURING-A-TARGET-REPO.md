# Pointing Pipeline Doctor at your own repository

Pipeline Doctor repairs a repository other than itself. To let it work on yours, add a
`pipeline-doctor.json` file to that repository's root.

```bash
boxlang main.bxs /path/to/your-repo
boxlang main.bxs /path/to/your-repo --dry-run   # rehearse: no commit, no push, no PR
```

## Minimal configuration

```json
{
  "executable": "java",
  "arguments": ["src/App.java"],
  "timeoutSeconds": 60,
  "allowedExtensions": ["java"],
  "maxFilesChanged": 1,
  "baseBranch": "main"
}
```

| Field | Meaning |
|---|---|
| `executable` | The program to run, e.g. `java`, `python3`, `node`, `sh` |
| `arguments` | Its arguments. Together these must **fail** on a broken commit |
| `sourceFile` | **Optional but usually needed.** The one file Pipeline Doctor may repair |
| `allowedExtensions` | Extensions it is permitted to modify. Nothing else can be touched |
| `maxFilesChanged` | Keep at `1` for now |
| `baseBranch` | The branch to start from and open the PR against |
| `timeoutSeconds` | Budget for the command and the AI call |

## When you need `sourceFile`

Without it, Pipeline Doctor assumes `arguments[1]` is the source file. That holds for
`java src/App.java`, but not for anything with flags or a build tool:

```json
{
  "executable": "sh",
  "arguments": ["-c", "javac -d out src/*.java && java -cp out Main"],
  "sourceFile": "src/Greeter.java",
  "allowedExtensions": ["java"],
  "baseBranch": "main"
}
```

Without `sourceFile` here, it would try to treat `-c` as a filename and stop.

## Other languages

Nothing is Java-specific any more. Set `allowedExtensions` and point at the file:

```json
{
  "executable": "python3",
  "arguments": ["src/app.py"],
  "allowedExtensions": ["py"],
  "baseBranch": "main"
}
```

Verified working on Java (single-file and multi-file builds) and Python.

## Requirements before a run

Pipeline Doctor refuses to start unless all of these hold, and tells you which one failed:

1. `pipeline-doctor.json` exists and is valid JSON.
2. The working tree is **clean** — no uncommitted changes.
3. You are on the configured `baseBranch`.
4. The configured command **fails** (there is nothing to repair otherwise).
5. The failure output mentions the configured source file.

## Ignore your build artifacts

If your command compiles anything, add the output directory to `.gitignore`. Otherwise the
safety guard sees compiled files change that the repair agent never reported, and refuses to
publish:

```
Refusing to publish: Git found an unexpected changed file: out/Greeter.class
```

That refusal is correct — it is what stops build output being committed into a PR.

## What it will not do

- Change more than one file.
- Touch a file whose extension is not in `allowedExtensions`.
- Commit anything the repair agent did not report.
- Publish when verification fails.
- **Merge the pull request.** A human always does that.

## Known limits

- One file per run, so failures needing coordinated multi-file edits are out of scope.
- The failure must name the source file in its output; a test that fails an assertion without
  a stack trace may not be located.
- Needs `gh` authenticated and push access to open the pull request.
