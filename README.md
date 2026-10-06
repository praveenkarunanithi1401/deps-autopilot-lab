# deps-autopilot-lab

A tiny rehearsal repo for automatic library upgrades with **Renovate**, using the same
dependency file formats as Blackboard Learn, but with public libraries only and no Learn code.

## What is in here

| File | Mimics in Learn | Who edits it |
| --- | --- | --- |
| `project-dependencies.xml` | `project-dependencies.xml` (Ant `<property>` + `<declare-lib>`) | Humans / Renovate |
| `distribution/includes/library-versions.gradle` | same path (key-only Groovy map) | Humans / Renovate |
| `tools/report/build.gradle` | hand-written `build.gradle` with hardcoded versions | Humans / Renovate |
| `generated/**` | the ~120 generated Gradle files | **Only** `bin/generate_gradle.py` |
| `bin/generate_gradle.py` | `cd workspace/build/developer && ./build gradle` | - |
| `renovate.json5` | the proposed Learn config | - |
| `.github/workflows/renovate.yml` | Jenkins job `platform/renovate-learn` (runs the bot) | - |
| `.github/workflows/ci.yml` | Jenkins job `platform/learn` (tests `main` and `feature/**`) | - |

Libraries are pinned to older versions on purpose, so Renovate has something to propose:
`commons-io 2.15.0` and Jackson `2.17.0` are allowed; Tomcat and the `-bb-` fork must never be touched.

## Set up (personal GitHub account)

1. Create an empty **private** repo named `deps-autopilot-lab` on github.com (no README).
2. Push this folder:
   ```bash
   cd deps-autopilot-lab
   git init -b main
   git add -A
   git commit -m "Initial lab"
   git remote add origin https://github.com/<your-user>/deps-autopilot-lab.git
   git push -u origin main
   ```
3. Create a **fine-grained personal access token**: GitHub > Settings > Developer settings >
   Fine-grained tokens > Generate. Repository access: only `deps-autopilot-lab`. Permissions:
   Contents, Pull requests, Issues, Commit statuses: **Read and write** (Metadata: read is added automatically).
4. In the repo: Settings > Secrets and variables > Actions > New repository secret:
   name `RENOVATE_TOKEN`, value = the token.
5. In the repo: Settings > Actions > General > Workflow permissions: leave as default.

## Run it

1. **Dry run:** Actions > `renovate` > Run workflow > `dry_run = full`.
   The log shows `Dependency extraction complete` and what it *would* create. Nothing is pushed.
2. **Real run:** Run workflow with `dry_run = none`.
   An issue **"Renovate Dependency Dashboard (lab)"** appears.
3. In that issue, tick the box next to `commons-io` (under *Pending Approval* or *Pending Status Checks*).
4. Run the `renovate` workflow again (`none`).
5. A branch `feature/renovate/commons-io` and a PR appear. The PR changes 5 files:
   `library-versions.gradle`, `project-dependencies.xml`, `tools/report/build.gradle`,
   and the two files under `generated/` (rewritten by `bin/generate_gradle.py`).
6. The `ci` workflow runs on the PR: generated files in sync, dependencies resolve, config valid.
7. Merge the PR. On the next `renovate` run the branch is deleted.

Try also: close a PR without merging (Renovate then ignores that version), or add a library to the
allow-list in `renovate.json5` and re-run.

## Checks you can run locally (Node 24+)

```bash
npx --yes --package renovate@44.127.0 -- renovate-config-validator --strict
LOG_LEVEL=debug npx --yes renovate@44.127.0 --platform=local --dry-run=lookup > lookup.log 2>&1
grep -i commons-io lookup.log | head
python3 bin/generate_gradle.py && git diff --exit-code -- generated/
```

## What differs from Learn

- The bot runs in GitHub Actions with a personal token; in Learn it runs in Jenkins with a GitHub App.
- `prHourlyLimit` is `0` here; Learn uses `1`.
- Versions are looked up on Maven Central; Learn also uses its internal Nexus.
