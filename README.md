# STO Tidbit — Pipeline Integration: Add Gitleaks to an Existing CI Stage

> **Harness University Tidbit.** A 5–10 minute, self-paced micro-lesson backed by an executable repo.
> **Skill:** Keep the CI steps you already have, then add a native Gitleaks step so secret detection runs in the same Build stage and lands in the **Vulnerabilities** tab.

This page is the full walkthrough. You can follow it without watching the video.

The pipeline to import or copy is [`.harness/pipeline.yaml`](.harness/pipeline.yaml). Comments in that file mark every value to edit before import and every value Harness prompts for at **Run**.

---

## What this skill accomplishes

- Leaves an existing CI step in place and adds a scan after it
- Runs [Gitleaks](https://github.com/gitleaks/gitleaks) as a native STO step (`type: Gitleaks`) so Harness runs the scanner, then ingests, normalizes, and deduplicates the findings
- Lets the step detect the scan target and variant from the cloned repo, so you do not type a target name or branch into the step
- Shows the findings on the execution **Vulnerabilities** tab (this tab was previously named **Security Tests**)

Gitleaks looks for hardcoded secrets, API keys, tokens, and passwords in the repository, including commit history. It is open source, so the scanner itself does not need a paid scanner license. You still need Harness STO so the step can publish results, and pipeline executions are billed as usual.

---

## How it works

```
Pipeline: Scan Pipeline Multiple Scanners For Existing CI Process
└── Stage: Build and Scan (CI)
    │
    │  Clone codebase  (cloneCodebase: true)
    │  Harness Cloud · Linux · AMD64
    │
    ├── 1. Run: Install Dependency
    │      shell: Sh
    │      command: echo "Installing Dependency"
    │      ↑ stand-in for the CI work this pipeline already does
    │
    └── 2. Gitleaks
           mode: orchestration
           config: default
           target.type: repository
           target.detection: auto
           advanced.log.level: info
```

`detection: auto` is only available in orchestration mode. On a repository target, the step runs:

- `git config --get remote.origin.url` to set the **target** (the repo)
- `git rev-parse --abbrev-ref HEAD` to set the **variant** (the branch that was cloned)

The default workspace is `/harness`, which is where the stage clone lands. This pipeline does not override the workspace, so Gitleaks scans that checkout.

---

## Repo layout

```
sto-tidbits-pipeline-integration/
├── README.md
├── video_script.md
└── .harness/
    └── pipeline.yaml
```

---

## Prerequisites

| Requirement | Why you need it |
|---|---|
| Harness project with STO enabled | The Gitleaks step publishes normalized findings into STO |
| A code repo connector | The Build stage clones through the pipeline codebase. Use **Connect through Harness Platform** when the build runs on Harness Cloud |
| Harness Cloud (Linux, AMD64) | Matches `runtime.type: Cloud` and `platform` in the YAML. A Kubernetes build infrastructure also works, with extra setup |
| A repository to scan | At run time you supply the repo name and the branch or tag |

You do not install Gitleaks on the runner. The step pulls the scanner.

---

## Steps

### 1. Open the pipeline this repo describes

**Import the YAML**

1. In your Harness project, go to **Pipelines** → **Create a Pipeline** → **Import From Git** or paste YAML, depending on how your project is set up.
2. Use [`.harness/pipeline.yaml`](.harness/pipeline.yaml).
3. Apply every `# EDIT` comment in that file before you import: org, project, Git connector, and the command in the existing CI step.
4. Leave the `# INPUT` lines as `<+input>`. Harness asks for the repository name and the branch or tag when you click **Run**.

**Or build the same pipeline in Pipeline Studio**

1. **Pipelines** → **Create a Pipeline**.
2. Name: `Scan Pipeline Multiple Scanners For Existing CI Process`.
3. **Add Stage** → **Build**. Stage name: `Build and Scan`.
4. Leave **Clone Codebase** enabled.
5. Select your Git provider connector.
6. Set **Repository Name** to runtime input (the tack icon next to the field → **Runtime Input**). That is `repoName: <+input>`.
7. Set the build (branch or tag) to runtime input. That is `build: <+input>`.
8. Leave sparse checkout empty so the stage clones the full repo.
9. On **Infrastructure**, choose **Cloud**, **Linux**, and **AMD64**.

### 2. Keep the existing CI step

The first execution step is ordinary CI work. In this repo it is a placeholder so the scan has something to follow:

1. On the stage **Execution** tab, **Add Step** → **Run**.
2. Name: `Install Dependency`. Identifier: `Install_Dependency`.
3. Shell: **Sh**.
4. Command:

```sh
echo "Installing Dependency"
```

In a real pipeline this is your install, test, or build step. Add the scan after those steps. Do not replace them.

### 3. Add the Gitleaks step

Recommended placement is a **Build** or **Security** stage. This pipeline uses the Build stage you already have.

1. After **Install Dependency**, select **Add Step**.
2. Search for **Gitleaks**.
   You can also open **Security Tests** → **Built-in Scanners** → **Secret Detection**. That built-in scanner is Gitleaks (v8.22.1 in current docs) with orchestration and auto-detection already filled in. Either path should match the YAML below.
3. Name the step `Gitleaks`.

Match the Gitleaks step in [`.harness/pipeline.yaml`](.harness/pipeline.yaml). The comments on that step say which settings to keep, which ones to leave empty, and which optional lines to uncomment (Fail on Severity, CLI flags, workspace).

Leave target **Name** and **Variant** empty. Auto fills them from git after the clone. Auto is the default when you add the step, and it is not available if Scan Mode is **Ingestion**.

Save the pipeline.

### 4. Run it

1. Select **Run**.
2. Fill the two codebase prompts marked `# INPUT` in [`.harness/pipeline.yaml`](.harness/pipeline.yaml): the repository name, and the branch or tag to clone.
3. You are not prompted for a Gitleaks target name or variant. Those come from the clone.
4. Select **Run Pipeline**.

For a first scan of a repo, use the root branch (`main` or `master`). That gives you a sensible baseline later.

The stage clones the repo, runs **Install Dependency**, then Gitleaks scans `/harness` and ingests the results in the same step.

### 5. Read the results

When the execution finishes:

1. Open the execution from the pipeline **Execution History**, or from **STO** → **Executions**.
2. Open the **Vulnerabilities** tab.
3. Filter by severity (**Critical**, **High**, **Medium**, **Low**, **Info**), and by **Scanner**, **Step**, or **Issue Type**. Secret findings from Gitleaks use issue type **Secret**.
4. Open a finding for the file, the rule, and the severity.
5. Optional: **Download CSV** exports the issues on this execution.

**Active Issues** counts findings that are still open. Exempted and remediated issues are left out of that count.

### 6. Set a baseline

After the first successful scan:

1. Go to **Security Tests** → **Test Targets**.
2. Select the target Gitleaks created (the remote URL detected from `git config --get remote.origin.url`).
3. Set the baseline variant to your root branch.

Later scans then split issues into:

- only in the scanned variant
- common to the target baseline

If no baseline is set, STO compares the current scan with the previous scan of that target.

### 7. Add another scanner the same way

This baseline shows one scanner after the existing CI step. To cover more of the same repo, **Add Step** again in this stage and choose another native scanner (for example a SAST or SCA step), still in **Orchestration**, still with **Repository** and **Auto** when you are scanning the clone. Each step publishes into the same **Vulnerabilities** tab. Filter by **Scanner** or **Step** to separate them.

---

## Orchestration and ingestion

| Mode | What the step does | When to use it |
|---|---|---|
| **Orchestration** | Runs Gitleaks and ingests the results in one step | This pipeline. No results file to manage |
| **Ingestion** | Reads a results file you already produced, then normalizes and deduplicates it | You run `gitleaks detect` yourself (often in a Run step) and write SARIF somewhere the scan step can read, such as `/shared/scan_results/...` |

Ingestion cannot use **Auto** detection. You set the target name and variant yourself, and you point **Ingestion File** at the results file. The stage needs a shared path for that file (**Overview** → **Shared Paths**).

---

## Allowlist inactive secrets

Gitleaks scans commit history. A secret that you rotated in a later commit still shows up in older commits until you allowlist it.

1. Review the findings.
2. Rotate or deactivate anything that is still live.
3. Add the inactive, rotated, or false-positive values to a `.gitleaks.toml` at the repo root.

Harness recommends listing them in the `regexes` array (plain text), with `useDefault = true` so you keep the stock rules:

```toml
title = "example gitleaks config"

[extend]
useDefault = true

[allowlist]
regexTarget = "match"
description = "whitelist public and test secrets"
regexes = [
  '''1234567890abcdef1234567890abcdef''',
  '''abcdef1234567890abcdef1234567890''',
]
```

---

## Key concepts

| Concept | In this pipeline |
|---|---|
| **Existing CI step** | The Run step `Install Dependency`. The scan is added after it |
| **Native scan step** | `type: Gitleaks`. Harness runs the scanner and ingests results. A plain Run step only prints logs |
| **Orchestration** | Scan and ingest in one step |
| **Scan configuration `default`** | The predefined Gitleaks configuration shipped with the step |
| **Target** | The repository. With auto detection, the remote URL |
| **Variant** | The branch checked out for this run |
| **Baseline** | The variant you treat as the root (usually `main`). STO uses it to tell new issues from known ones |
| **Log level `info`** | Minimum log severity. Also available: `debug`, `warning`, `error` |
| **Fail on Severity** | Not set in this YAML. Add `fail_on_severity` when a chosen severity should fail the build |
| **Vulnerabilities tab** | Per-execution list of normalized issues. Formerly named Security Tests |

---

## Docs used for this tidbit

- [Gitleaks step configuration](https://developer.harness.io/security-testing-orchestration/use-sto/sto-scanner-configuration/gitleaks-scanner-reference)
- [Built-in scanners](https://developer.harness.io/security-testing-orchestration/use-sto/set-up-sto-scans/built-in-scanners) (Secret Detection uses Gitleaks)
- [Secret Detection](https://developer.harness.io/security-testing-orchestration/use-sto/set-up-sto-scans/secret-detection)
- [Run an orchestrated scan](https://developer.harness.io/security-testing-orchestration/new-to-sto/key-concepts/run-an-orchestrated-scan-in-sto)
- [Targets, variants, and baselines](https://developer.harness.io/security-testing-orchestration/new-to-sto/key-concepts/targets-and-baselines)
- [View scan results — Vulnerabilities tab](https://developer.harness.io/security-testing-orchestration/use-sto/sto-security-issues/view-scan-results)
- [Configure the CI codebase](https://developer.harness.io/continuous-integration/use-harness-ci/use-harness-ci/codebase-configuration/create-and-configure-a-codebase)
