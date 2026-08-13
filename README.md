# STO: Pipeline Integration — Add a Scan Step to an Existing Pipeline

> **Harness University Tidbit.** A 5–10 minute, self-paced micro-lesson backed by an executable repo.
> **Skill:** Add native STO scan steps (SAST + container SCA) to an existing CI Build stage — no shell scripts, just native Harness step types.

## What this skill accomplishes

- Adds a native Harness STO scan step to an existing CI Build or CD pipeline
- Runs Bandit (SAST) on source code and Aqua Trivy (SCA/container) on a built image — no shell scripts
- Introduces scan modes: **Orchestration** (scan + ingest in one step) vs **Ingestion** (run + ingest separately)
- Gates the pipeline on `fail_on_severity` so critical findings block the build automatically
- Surfaces scan results in the **Security Tests** tab of every pipeline execution

## How it works

```
┌─────────────────── CI Build Stage ──────────────────────────────┐
│                                                                   │
│  ┌──────────┐   ┌───────────────────┐   ┌──────────────────────┐│
│  │  Clone   │──▶│  Build & Push     │──▶│  Aqua Trivy Scan     ││
│  │ Codebase │   │  (Docker image)   │   │  type: AquaTrivy     ││
│  └──────────┘   └───────────────────┘   │  mode: orchestration ││
│                                          │  target.type: cont.. ││
│  ┌───────────────────┐                  └──────────────────────┘│
│  │  Bandit SAST Scan │                                           │
│  │  type: Bandit     │  ◀── runs on cloned source code          │
│  │  mode: orchestr.. │                                           │
│  │  target: repo     │                                           │
│  └───────────────────┘                                           │
│                                                                   │
│  Both steps feed results into the Security Tests tab             │
└───────────────────────────────────────────────────────────────────┘
```

## Repo layout

```
sto-tidbits-pipeline-integration/
├── README.md
└── .harness/
    └── pipeline.yaml        ← CI Build stage with Bandit + AquaTrivy native steps
```

## Prerequisites

| Requirement | Details |
|---|---|
| Harness STO license | Needed for native scan steps and the Security Tests tab |
| Docker connector | Required for building and scanning container images |
| Git connector | Required for cloning source code |
| Harness Cloud (or K8s infra) | Build infrastructure for the CI stage |
| No extra tools to install | STO scan steps pull their own scanner images automatically |

## Steps

### 1. Open an existing pipeline (or create one)

In Harness, navigate to your project → **Pipelines**. Open an existing CI Build pipeline or create a new one with a **Build** stage.

### 2. Add a Bandit step for SAST (code scanning)

In the **Execution** tab of your Build stage, click **Add Step** and search for **Bandit**.

Configure it:
- **Scan Mode**: Orchestration (the step runs Bandit and ingests results in one go)
- **Target Type**: Repository
- **Target Name / Variant**: set to `<+input>` (runtime input) or use stage variables like `<+stage.variables.REPO_NAME>`
- **Fail on Severity**: High (blocks the pipeline if any High or Critical findings are detected)

### 3. Add an Aqua Trivy step for container scanning

After your image build step, click **Add Step** and search for **Aqua Trivy**.

Configure it:
- **Scan Mode**: Orchestration
- **Target Type**: Container Image
- **Image Type**: Local Image (the image was just built in the previous step)
- **Image Name / Tag**: use expressions — `<+stage.variables.IMAGE_NAME>` / `<+stage.variables.IMAGE_TAG>`
- **Fail on Severity**: Critical

### 4. (Optional) Move the Bandit step before the build

STO scan steps can appear anywhere in the execution. A common pattern is:

```
Clone → Bandit SAST → Build → Push → Aqua Trivy
```

Drag the Bandit step to before the build step so code defects are caught before you spend time building.

### 5. Run the pipeline and review results

Click **Run**, provide runtime values if needed, and watch the execution. When a scan step completes:

1. Click the scan step in the execution view
2. Select the **Security Tests** tab at the top of the execution
3. Filter by severity (Critical, High, Medium, Low)
4. Click a finding to see the file, line number, and CWE details

### 6. Set baselines to track new issues only

In **STO → Test Targets**, find your target and set a **baseline** (usually the `main` branch or the latest `production` tag). STO then highlights only **new** issues introduced since the baseline — noise from pre-existing issues is suppressed.

## Key concepts

| Concept | Description |
|---|---|
| **Native scan step** | STO has dedicated step types (`Bandit`, `AquaTrivy`, `Snyk`, `Zap`, etc.) that pull scanner images automatically — no shell scripts required |
| **Orchestration mode** | The step runs the scanner AND ingests results in one action. Easiest way to start |
| **Ingestion mode** | A `Run` step runs the scanner and writes results to `/shared/scan_results/`. A scan step then ingests that file. Useful when you need custom scanner flags |
| **Target** | The object being scanned — a repository, a container image, or a running web instance |
| **Variant** | The specific version of the target — a branch name, image tag, or app version |
| **fail_on_severity** | Controls which finding severity causes the pipeline to fail. Options: `none`, `low`, `medium`, `high`, `critical` |
| **Security Tests tab** | A unified view in every pipeline execution showing all scan results across all STO steps |
| **Baseline** | The reference state for a target. STO highlights only NEW findings relative to the baseline |

## Scan step types quick reference

| Step Type | Target | Best for |
|---|---|---|
| `Bandit` | Repository (Python) | SAST: Python source code |
| `AquaTrivy` | Container / Repository | SCA: container image CVEs, filesystem |
| `Snyk` | Repository / Container | SCA: dependency vulnerabilities (requires Snyk token) |
| `Zap` | Instance (web app) | DAST: running web application |
| `Semgrep` | Repository | SAST: multi-language static analysis |
| `OWASPDependencyCheck` | Repository | SCA: open source dependency CVEs |
