# Power Platform Release — GitHub Actions edition

> ⚠️ **REPO-TEMPLATE LOCATION NOTICE**: this file (and its neighbours,
> `power-platform-release.yml` + `deploy-to-env.yml`) lives here at
> `Pipelines/.github/workflows/` **only inside this repo-template**, to keep it
> organized next to the Azure DevOps port. GitHub Actions only discovers
> workflows under `.github/workflows/` **at the root of a repository** — when
> you copy these files into a real project repo, move all three together to
> `.github/workflows/` at that repo's root. Only then do the paths referenced
> in the table below (and the `uses: ./.github/workflows/deploy-to-env.yml`
> reference inside `power-platform-release.yml`) resolve correctly; in this
> repo-template they are shown in their **post-relocation** form for clarity.

GitHub Actions port of the Azure DevOps pipeline documented in the [root `README.md`](../../../README.md).

Both versions live side by side and are fully independent:

| Platform | Entry point | Reusable unit |
|---|---|---|
| Azure DevOps | `Pipelines/devops/power-platform-release.yaml` | `Pipelines/devops/templates/deploy-to-env.yaml` |
| GitHub Actions (post-relocation, real project repo) | `.github/workflows/power-platform-release.yml` | `.github/workflows/deploy-to-env.yml` |
| GitHub Actions (this repo-template, as-is) | `Pipelines/.github/workflows/power-platform-release.yml` | `Pipelines/.github/workflows/deploy-to-env.yml` |

The ADO files are **not** modified by this port — if you run on Azure DevOps, nothing changes for you.

---

## Table of Contents

- [Pipeline shape](#pipeline-shape)
- [1. Required secrets](#1-required-secrets)
- [2. Required GitHub Environments](#2-required-github-environments)
- [3. Required permissions](#3-required-permissions)
- [4. Running the workflow](#4-running-the-workflow)
- [5. Deployment Settings File (DSF) overrides](#5-deployment-settings-file-dsf-overrides)
  - [5.1 When secrets and variables are read](#51-when-secrets-and-variables-are-read)
- [6. Known differences vs the Azure DevOps pipeline](#6-known-differences-vs-the-azure-devops-pipeline)
- [7. ADO task → GitHub Action mapping](#7-ado-task--github-action-mapping)
- [8. Troubleshooting](#8-troubleshooting)

---

## Pipeline shape

```
build ──► deploy_uat ──┬──► deploy_test
                       │
                       └──► deploy_pre_prod ──► deploy_prod
```

`deploy_test` and `deploy_pre_prod` run **in parallel**, both gated on `deploy_uat`.
`deploy_prod` depends on `deploy_pre_prod` only. This matches the ADO `dependsOn` graph exactly.

- **`build`** — validates inputs, exports the solution from DEV, bumps the version in Dataverse, unpacks the unmanaged solution into `Solutions/<SolutionName>/Components`, commits + tags + pushes, generates the DSF template and uploads everything as the `release-bundle` artifact.
- **`deploy_*`** — each calls the reusable workflow `./.github/workflows/deploy-to-env.yml`, which downloads the artifact, merges the environment overrides into the DSF, decides fresh-install vs upgrade, and imports the solution.

---

## 1. Required secrets

All secrets below are **repository-level** secrets
(*Settings → Secrets and variables → Actions → Repository secrets*).

> **Why repository scope and not environment scope?**
> The `deploy_*` jobs call a reusable workflow, and the `secrets:` block of a
> `uses:` job is evaluated in the **caller's** context. The caller job has no
> `environment:` of its own (the `environment:` lives inside the called
> workflow), so an environment secret **cannot be referenced from that
> `secrets:` block**. Scoping the credentials at repository level with an
> environment prefix in the name keeps one secret per environment and keeps the
> wiring explicit and auditable.
>
> ⚠️ This does **not** mean environment secrets are unreachable: the *called*
> job does declare `environment:`, so an environment secret with the matching
> name is injected there and **takes precedence** over whatever the caller
> passed. That route also has different read timing, which matters a lot for the
> DSF overrides — see [§5.1](#51-when-secrets-and-variables-are-read) and the
> [environment-scoped wiring](#environment-scoped-secrets-recommended-for-the-dsf-overrides).

### DEV (used by the `build` job)

| Secret | Value |
|---|---|
| `PP_DEV_URL` | `https://<dev-org>.crm4.dynamics.com` |
| `PP_DEV_APP_ID` | Service principal application (client) id |
| `PP_DEV_CLIENT_SECRET` | Service principal client secret |
| `PP_DEV_TENANT_ID` | Entra ID tenant id |

### UAT / TEST / PRE_PROD / PROD (used by the `deploy_*` jobs)

For each of `UAT`, `TEST`, `PRE_PROD`, `PROD`:

| Secret | Required | Value |
|---|---|---|
| `PP_<ENV>_URL` | ✅ | `https://<org>.crm4.dynamics.com` |
| `PP_<ENV>_APP_ID` | ✅ | Service principal application (client) id |
| `PP_<ENV>_CLIENT_SECRET` | ✅ | Service principal client secret |
| `PP_<ENV>_TENANT_ID` | ✅ | Entra ID tenant id |
| `PP_<ENV>_DSF_OVERRIDES` | ⬜ legacy | Raw JSON with the environment-specific DSF overrides. **Prefer an environment secret named `PP_DSF_OVERRIDES`** — a repository secret is frozen at queue time, see [§5.1](#51-when-secrets-and-variables-are-read) |

Concretely, that is 20 secrets (16 if you move the four `*_DSF_OVERRIDES` to
environment scope, which is the recommended wiring):

```
PP_DEV_URL            PP_DEV_APP_ID            PP_DEV_CLIENT_SECRET            PP_DEV_TENANT_ID
PP_UAT_URL            PP_UAT_APP_ID            PP_UAT_CLIENT_SECRET            PP_UAT_TENANT_ID            PP_UAT_DSF_OVERRIDES
PP_TEST_URL           PP_TEST_APP_ID           PP_TEST_CLIENT_SECRET           PP_TEST_TENANT_ID           PP_TEST_DSF_OVERRIDES
PP_PRE_PROD_URL       PP_PRE_PROD_APP_ID       PP_PRE_PROD_CLIENT_SECRET       PP_PRE_PROD_TENANT_ID       PP_PRE_PROD_DSF_OVERRIDES
PP_PROD_URL           PP_PROD_APP_ID           PP_PROD_CLIENT_SECRET           PP_PROD_TENANT_ID           PP_PROD_DSF_OVERRIDES
```

The service principals are the same ones behind the ADO service connections
`pp-dev-spn`, `pp-uat-spn`, `pp-test-spn`, `pp-pre-prod-spn`, `pp-prod-spn` — see
*Configure service connections using a service principal* in the root README for how to create them.

---

## 2. Required GitHub Environments

Create four environments under *Settings → Environments*. The names must match
**exactly** (they are passed as `environmentName` to the reusable workflow and used as the job's `environment:`):

| Environment | Replaces ADO | Suggested protection |
|---|---|---|
| `UAT` | ADO environment `UAT` | Required reviewers (1+) |
| `TEST` | ADO environment `TEST` | Required reviewers (1+) |
| `PRE_PROD` | ADO environment `PRE_PROD` | Required reviewers (1+) |
| `PROD` | ADO environment `PROD` | Required reviewers (2+), optional wait timer, branch restriction to `master` |

**To replicate ADO manual approvals:**

1. *Settings → Environments → `<name>` → Configure environment*
2. Tick **Required reviewers** and add the approver users/teams (max 6).
3. Optionally set **Wait timer** and **Deployment branches and tags** (e.g. restrict `PROD` to `master` only).

The run will pause on the corresponding `deploy_*` job with a *Review deployments* prompt, exactly like an ADO environment check.

> 💡 The environment is also where the per-environment `PP_DSF_OVERRIDES` secret
> belongs: values stored there are read **when the deploy job starts**, so they
> can still be corrected while the job waits for approval. See
> [§5.1](#51-when-secrets-and-variables-are-read).

> ⚠️ Required reviewers are only enforced on **public** repos and on **private/internal** repos in GitHub Team / Enterprise plans. On a private repo under the Free plan the environment is created but the approval gate is silently skipped.

---

## 3. Required permissions

The workflow declares a read-only default token and elevates only the `build` job:

```yaml
permissions:
  contents: read      # workflow default

jobs:
  build:
    permissions:
      contents: write # needed to push the exported solution + version tag
```

`contents: write` is required because `build` commits `Solutions/<SolutionName>/**`
and pushes the `<SolutionName>-v<version>` tag using the default `GITHUB_TOKEN`
persisted by `actions/checkout`.

Also make sure that, under *Settings → Actions → General → Workflow permissions*,
**"Read and write permissions"** is available to workflows (or at least that
"Read repository contents permission" is not hard-locked), otherwise the `contents: write`
escalation is denied and the commit step fails with `403`.

If the target branch is protected, either exclude the workflow's identity from the
restriction or allow the bot to bypass — a protected branch requiring pull requests
will reject the direct push.

---

## 4. Running the workflow

*Actions → **Power Platform Release** → **Run workflow***

| Input | Type | Default | Meaning |
|---|---|---|---|
| `SolutionName` | string (**required**) | — | Unique solution name (not display name). Must match `^[a-zA-Z0-9_]+$` |
| `AutoIncrementSolutionVersion` | boolean | `true` | Read the current version from Dataverse and bump the **4th** part only |
| `SolutionVersion` | string | `1.0.0.0` | Used only when auto-increment is disabled. Must be `major.minor.build.revision` |
| `deployManaged` | boolean | `true` | Deploy the managed package |
| `forceOverwrite` | boolean | `false` | Overwrite unmanaged customizations on import |
| `ExportBothManagedAndUnmanaged` | boolean | `true` | Export both flavours from DEV |

The branch selected in the *Run workflow* dropdown is the branch the build job commits and pushes to.

---

## 5. Deployment Settings File (DSF) overrides

GitHub has no equivalent of Azure DevOps **Secure Files**, so the four
`deploy.<env>.settings.json` Secure Files become four secrets.

Put the raw JSON directly in the secret value. **Store it as an environment
secret named `PP_DSF_OVERRIDES`** on the matching GitHub Environment — that is
the only variant whose read timing matches the ADO Secure File it replaces
([§5.1](#51-when-secrets-and-variables-are-read)). The repository-scoped
`PP_<ENV>_DSF_OVERRIDES` still works as a fallback but is frozen at queue time.

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "cr123_ApiBaseUrl", "Value": "https://uat.api.contoso.com" }
  ],
  "ConnectionReferences": [
    {
      "LogicalName": "cr123_sharedoffice365_abc12",
      "ConnectorId": "/providers/Microsoft.PowerApps/apis/shared_office365",
      "ConnectionId": "00000000-0000-0000-0000-000000000000"
    }
  ],
  "CopilotAgents": []
}
```

Merge semantics are identical to the ADO template:

- **`EnvironmentVariables`** — matched on `SchemaName`; only `Value` is replaced. Unmatched overrides are ignored.
- **`ConnectionReferences`** — matched on `LogicalName`; `ConnectorId` and `ConnectionId` are replaced when non-empty. Unmatched overrides are ignored.
- **`CopilotAgents`** — matched on `Id` first, then `Name`; every non-null property of the override is copied over. **If no match is found the agent is appended** (add-if-missing).

If the secret is empty or not set, the DSF template is used as-is and the deploy
does **not** fail — same behaviour as a missing Secure File in ADO.

If the secret is set but is not valid JSON, the job **fails fast** with an explicit
error rather than silently deploying the untouched template (a small hardening over the ADO template).

> 🔒 The values injected from the override secret are registered with `::add-mask::`
> before the merged DSF is dumped to the log. GitHub only masks a secret as a whole
> string, so without that the "Final DSF Content" dump would print each connection id
> and environment-variable value in clear text.

### 5.1 When secrets and variables are read

This is the single most load-bearing fact in this pipeline, and it is **not** the
same for every kind of secret. From the GitHub
[Secrets reference](https://docs.github.com/en/actions/reference/security/secrets#when-github-actions-reads-secrets):

> **Organization and repository secrets are read when a workflow run is queued, and environment secrets are read when a job referencing the environment starts.**

Consequences for this workflow:

| What | Read when | Can you change it mid-run? |
|---|---|---|
| `PP_<ENV>_*` repository secrets (incl. `PP_<ENV>_DSF_OVERRIDES`) | the **whole run** is queued | ❌ **No.** A `deploy_prod` job that waits two days for approval still uses the value that existed when `build` was queued. |
| `PP_DSF_OVERRIDES` / `PP_URL` / … as **environment** secrets | the **deploy job** starts, i.e. **after** the approval gate | ✅ **Yes.** Edit it while the job is pending review; the approved job reads the new value. |
| `vars.*` at repository/organization scope | not documented; treat as queue-time | ❌ assume no |
| `vars.*` at environment scope | ["only available on the runner after the job starts executing"](https://docs.github.com/en/actions/reference/workflows-and-actions/variables#configuration-variable-precedence) | ✅ yes (but not masked — never put credentials there) |
| the workflow YAML itself | resolved once for the run (`GITHUB_WORKFLOW_SHA` is fixed per run; a `./…` reusable workflow is taken "from the same commit as the caller workflow") | ❌ no — editing `deploy-to-env.yml` mid-run does not affect the running run |

> This is exactly the Azure DevOps *variable-group-snapshot-at-queue-time*
> limitation analysed in [`poc/README.md`](../../../poc/README.md) — and it is the
> reason that POC was closed. **On GitHub the problem is solvable without
> splitting the pipeline**, because environment secrets are read per job, after
> the gate. Keep the single `build → UAT → TEST‖PRE_PROD → PROD` run.

#### How to update an override without re-running the whole pipeline

1. *Settings → Environments → `UAT` (or `TEST` / `PRE_PROD` / `PROD`) →
   **Environment secrets** → `PP_DSF_OVERRIDES`* → paste the new JSON → **Save**.
   Leave the repository-level `PP_<ENV>_DSF_OVERRIDES` unset (or stale — it is
   shadowed, see below).
2. Approve the pending `Deploy to <ENV>` job.
3. Check the **DSF Override Fingerprint** row in the job summary.

The precedence that makes step 1 work is documented in
[Reuse workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#using-inputs-and-secrets-in-a-reusable-workflow):

> Environment secrets cannot be passed from the caller workflow as `on.workflow_call` does not support the `environment` keyword. **If you include `environment` in the reusable workflow at the job level, the environment secret will be used, and not the secret passed from the caller workflow.**

`deploy-to-env.yml` does declare `environment: ${{ inputs.environmentName }}` at
job level, so no YAML change is needed — only *where you store the JSON*.

#### Verifying which revision was actually used

Secrets are opaque, so the deploy job prints a **fingerprint** instead: the first
12 hex characters of the SHA-256 of the override bytes, in the step log and in the
job summary. Reproduce it locally over the exact text you pasted:

```powershell
$json  = Get-Content .\deploy.uat.settings.json -Raw
$bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
$sha   = [System.Security.Cryptography.SHA256]::Create()
([System.BitConverter]::ToString($sha.ComputeHash($bytes)) -replace '-', '').Substring(0, 12).ToLowerInvariant()
```

> ⚠️ Hash over the **exact** bytes. `Get-Content -Raw` keeps the file's trailing
> newline; if you pasted the JSON into the GitHub UI without it, the two
> fingerprints will differ even though the JSON is semantically identical.

To check *when* the environment secret was last written (the value is never
returned — only metadata), from your own machine:

```powershell
gh api repos/:owner/:repo/environments/UAT/secrets/PP_DSF_OVERRIDES --jq '.updated_at'
```

A `404` here means no environment secret exists for that environment, so the
run is falling back to the queue-time repository secret.

#### Re-runs

Whether a re-run re-reads repository secrets is **not documented**. What *is*
documented is only the adjacent behaviour for the reusable-workflow reference:
"Re-running all jobs in a workflow will use the reusable workflow from the
specified reference" while "Re-running failed jobs or a specific job in a workflow
will use the reusable workflow from the same commit SHA of the first attempt"
([docs](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#behavior-of-reusable-workflows-when-re-running-jobs)).
Do not infer secret behaviour from that. Do not rely on a re-run to refresh a
repository secret; use the environment secret instead, whose read timing *is*
documented.

---

## 6. Known differences vs the Azure DevOps pipeline

| # | Azure DevOps | GitHub Actions | Why / impact |
|---|---|---|---|
| 1 | `DownloadSecureFile@1` + Secure Files | `PP_DSF_OVERRIDES` **environment** secret (repo-level `PP_<ENV>_DSF_OVERRIDES` as fallback) | GitHub has no Secure Files. Same "missing override ⇒ deploy anyway" semantics. ⚠️ Read timing differs: the ADO Secure File is fetched **at task execution time**, a GitHub *repository* secret is frozen **at queue time**, a GitHub *environment* secret is read **when the deploy job starts**. Only the environment secret matches ADO's behaviour — see [§5.1](#51-when-secrets-and-variables-are-read). |
| 2 | Service connections (`pp-*-spn`) + `PowerPlatformSetConnectionVariables@2` | Four secrets per environment passed directly | There is no service-connection concept; credentials are handed to the actions/`pac` directly. |
| 3 | `##vso[build.addbuildtag]` / `##vso[build.updatebuildnumber]` | **Dropped.** Replaced by `$GITHUB_STEP_SUMMARY` + a `run-name:` | GHA has no API to add tags to a run or rename it after it started. `run-name` is evaluated *before* the run, so it cannot contain the resolved solution version. |
| 4 | `retryCountOnTaskFailure: 2` on export | `continue-on-error` + one guarded retry step | GHA has no built-in step retry. The pattern gives 1 initial attempt + 1 retry (ADO allowed 2 retries); the retry step has no `continue-on-error`, so a second failure fails the job. |
| 5 | `NuGetToolInstaller@1` + `Cache@2` + `NuGetCommand@2` + "Find pac CLI folder" | A single `actions-install@v1` with `add-tools-to-path: 'true'` | The GHA action installs pac **and** puts it on `PATH`, which the ADO `PowerPlatformToolInstaller@2` does not do. The NuGet cache was dropped: `actions-install` pins its own pac version and the restore is a few seconds. |
| 6 | ~90-line rebase / `-X ours` / `rebase --abort` / `merge --allow-unrelated-histories` recovery block | Plain `git pull --rebase` + workflow-level `concurrency:` group | The recovery block only existed to survive two concurrent runs writing to the same `Solutions/<SolutionName>` folder. `concurrency: pp-release-<SolutionName>` removes the race structurally, so a conflict now means a real problem and fails loudly instead of silently forcing "ours". |
| 7 | `PowerPlatformSetSolutionVersion@2` | `set-online-solution-version@v1` | Exists in `microsoft/powerplatform-actions` but is **not** listed on the Microsoft Learn "Available GitHub Actions" page. No PowerShell fallback needed. |
| 8 | `PowerPlatformWhoAmi@2` (commented out) | `who-am-i@v1` (**enabled**) | Cheap fail-fast connectivity check against DEV before anything is exported. |
| 9 | Booleans render as `True` / `False` | Booleans render as `true` / `false` | Every PowerShell comparison was switched to lowercase. Copy/pasting a comparison from the ADO files will silently always be false. |
| 10 | `${{ parameters.X }}` inlined into PowerShell | Inputs passed via `env:` and read as `$env:X` | Avoids the GHA script-injection class of bug and all the quoting pitfalls. |
| 11 | Full DSF dumped to the log | Same dump, but override values are `::add-mask::`-ed first | Prevents leaking connection ids / environment-variable values into public logs. |
| 12 | Environment approvals via ADO Environments | GitHub Environments + Required reviewers | Equivalent, but see the plan caveat in [§2](#2-required-github-environments). |
| 13 | Artifacts kept per ADO retention policy | `retention-days: 30` on `upload-artifact@v4` | Explicit, tune to taste. |

### Environment-scoped secrets (recommended for the DSF overrides)

If you prefer real environment-scoped secrets (so that PROD credentials are only
readable by a job targeting the `PROD` environment), you can create secrets named
`PP_URL`, `PP_APP_ID`, `PP_CLIENT_SECRET`, `PP_TENANT_ID`, `PP_DSF_OVERRIDES`
**inside each GitHub Environment**, and leave the repository-level ones unset.

For `PP_DSF_OVERRIDES` this is not a matter of taste: it is the **only** wiring
in which an override edited after the run started is actually picked up
([§5.1](#51-when-secrets-and-variables-are-read)).

Per [GitHub's documentation on reusing workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#using-inputs-and-secrets-in-a-reusable-workflow):

> Environment secrets cannot be passed from the caller workflow as `on.workflow_call` does not support the `environment` keyword. If you include `environment` in the reusable workflow at the job level, the environment secret will be used, and not the secret passed from the caller workflow.

Because `deploy-to-env.yml` declares `environment: ${{ inputs.environmentName }}` at
job level and names its secrets `PP_URL` / `PP_APP_ID` / …, an environment secret with
the same name **takes precedence** over whatever the caller passed. No change to the
YAML is needed — only to where you store the secrets.

> ⚠️ Environment secrets require a **public** repo, or GitHub Pro / Team /
> Enterprise for a private one. On a private repo under the Free plan the
> environment exists but its secrets are ignored, and you are stuck with the
> queue-time repository secrets.

---

## 7. ADO task → GitHub Action mapping

Actions are pinned to the `@v1` major tag of
[`microsoft/powerplatform-actions`](https://github.com/microsoft/powerplatform-actions)
(latest release at time of writing: `v1.10.0`).

| Azure DevOps task | GitHub Action | Notes |
|---|---|---|
| `PowerPlatformToolInstaller@2` | `microsoft/powerplatform-actions/actions-install@v1` | `add-tools-to-path: 'true'` is **mandatory** for the `pac` steps |
| `PowerPlatformWhoAmi@2` | `microsoft/powerplatform-actions/who-am-i@v1` | |
| `PowerPlatformSetSolutionVersion@2` | `microsoft/powerplatform-actions/set-online-solution-version@v1` | inputs `name`, `version` |
| `PowerPlatformExportSolution@2` | `microsoft/powerplatform-actions/export-solution@v1` | `solution-name`, `solution-output-file`, `managed`, `overwrite`, `run-asynchronously`, `max-async-wait-time` |
| `PowerPlatformUnpackSolution@2` | `microsoft/powerplatform-actions/unpack-solution@v1` | `solution-file`, `solution-folder`, `solution-type`, `overwrite-files` |
| `PowerPlatformImportSolution@2` | `microsoft/powerplatform-actions/import-solution@v1` | see input mapping below |
| `PowerPlatformSetConnectionVariables@2` | *(none)* | Not needed: secrets are already available |
| `publish:` / `- download: current` | `actions/upload-artifact@v4` / `actions/download-artifact@v4` | artifact name `release-bundle` |
| `DownloadSecureFile@1` | *(none)* | Replaced by the `PP_DSF_OVERRIDES` environment secret — **not** by a repository secret, whose read timing differs ([§5.1](#51-when-secrets-and-variables-are-read)) |

### `PowerPlatformImportSolution@2` input mapping

| ADO input | GHA input | Value used |
|---|---|---|
| `AsyncOperation` | `run-asynchronously` | `true` |
| `StageAndUpgrade` | `stage-and-upgrade` | computed (solution already present ⇒ `true`) |
| `MaxAsyncWaitTime` | `max-async-wait-time` | `120` |
| `PublishWorkflows` | `activate-plugins` | `true` |
| `PublishCustomizationChanges` | `publish-changes` | `not(deployManaged)` |
| `OverwriteUnmanagedCustomizations` | `force-overwrite` | `forceOverwrite` input |
| `UseDeploymentSettingsFile` | `use-deployment-settings-file` | computed |
| `DeploymentSettingsFile` | `deployment-settings-file` | computed |

> The `import-solution` inputs above are **not** documented on Microsoft Learn's
> *Available GitHub Actions* page (which only lists the auth + `solution-file` inputs).
> They were verified against
> [`import-solution/action.yml`](https://github.com/microsoft/powerplatform-actions/blob/main/import-solution/action.yml)
> in the official repository.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `pac : The term 'pac' is not recognized` | `add-tools-to-path` missing or not the literal string `'true'` | The action does `core.getInput('add-tools-to-path') === 'true'`; pass it quoted as a string |
| Commit step fails with `403` | `contents: write` not granted, or branch protection | See [§3](#3-required-permissions) |
| `Solution '<name>' not found in Dataverse` | Auto-increment enabled but the solution has never been created in DEV | Create it in DEV first, or disable auto-increment and pass an explicit `SolutionVersion` |
| Deploy job never asks for approval | Private repo on the Free plan | Required reviewers need Team/Enterprise (or a public repo) |
| `No solution file found matching pattern` | `deployManaged` differs between build and deploy, or the export step was skipped | Keep `ExportBothManagedAndUnmanaged: true` |
| Deploy job can't see the secrets | Secrets created as **environment** secrets but the caller passes them explicitly | Move them to repository scope, or adopt the environment-scoped wiring described in [§6](#environment-scoped-secrets-recommended-for-the-dsf-overrides) |
| I edited the DSF override but the deploy used the old JSON | The override lives in a **repository** secret, and those are read when the run is *queued* — an approval pause does not re-read them | Move the JSON to an environment secret named `PP_DSF_OVERRIDES`; verify via the *DSF Override Fingerprint* row in the job summary ([§5.1](#51-when-secrets-and-variables-are-read)) |
| A step fails with `NativeCommandError` right after a `pac`/`git` call | With `shell: pwsh` GitHub prepends `$ErrorActionPreference='Stop'`, turning native stderr output into a terminating error | The steps that tolerate this already set `$ErrorActionPreference = 'Continue'` and end with `exit 0`; do the same in any new step |

### Local validation

Both workflows are validated with [actionlint](https://github.com/rhysd/actionlint):

```powershell
actionlint.exe .github\workflows\power-platform-release.yml .github\workflows\deploy-to-env.yml
```

`actionlint` catches expression/context errors that plain YAML parsing misses — for
instance that the `runner` context is **not** available in a job-level `env:` block.
