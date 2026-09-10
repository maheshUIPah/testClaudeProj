# testClaudeProj — UiPath CI/CD Reference Implementation

6 enterprise-grade sample automation processes demonstrating the Dev -> UAT
-> Prod promotion flow described in the "UiPath Dev-to-Production CI/CD
Delivery Framework" whitepaper, using two custom Automation Ops Pipeline
Processes instead of one native pipeline set per project.

Both pipeline processes are built directly on top of UiPath's own official
Pipeline templates rather than authored from scratch: the `upa:*` activities,
their argument names, and the stage structure are copied verbatim from the
currently-published OOB template packages, then extended with a manifest
loop so one pipeline run handles all 6 (or 1,000) independent projects.

| Custom pipeline process | Extends UiPath OOB template | Package version referenced |
|---|---|---|
| `pipelines/MultiProjectCI` | `Update with tests` | 2.0.1 (UiPath.Pipelines.Activities 2.0.0) |
| `pipelines/MultiProjectPromotion` | `Copy package between environments` | 2.0.1 (UiPath.Pipelines.Activities 2.0.0) |

The `Solution deployment pipeline` OOB package was intentionally NOT used —
it requires a formal Solution created via Solutions Management, which is out
of scope here (plain independent Projects only, per current scope).

## Orchestrator folder mapping (this tenant)

| Tier | Orchestrator folder |
|---|---|
| DEV  | `shared` |
| UAT  | `mk` |
| PROD | `mk-laptop` |

## Repository layout

```
processes/                                 6 independent Studio projects (enterprise-grade source)
  Invoice-Data-Extraction/
    project.json
    Main.xaml                              Init -> bounded-retry Process -> End; Business vs System exception handling
  Contract-Renewal-Reminder/
  Customer-Refund-Processing/
  Email-Attachment-Sorter/
  Employee-Onboarding-Bot/
  Expense-Report-Validator/

pipelines/manifest/
  projects-manifest.csv                    One row per process: PathToProjectJson, PackageName, OrchestratorFolder, ProcessName, RunTests

pipelines/MultiProjectCI/
  project.json                             Extends OOB "Update with tests" (UiPath.Pipelines.Activities 2.0.0)
  Main.xaml                                Clone once, then per manifest row: Analyze -> Run Tests -> Build -> Publish -> Update Process

pipelines/MultiProjectPromotion/
  project.json                             Extends OOB "Copy package between environments" (UiPath.Pipelines.Activities 2.0.0)
  Main.xaml                                Per manifest row: Download Package (source folder) -> Publish Package (destination folder) -> Update Process
```

## Sample processes (enterprise-grade design)

Each process follows the same structural pattern rather than a flat script:

- **Initialization** — generates a TransactionID (for correlated logging/traceability) and records the start time.
- **Process** — a bounded retry loop (`MaxRetryCount = 3`) wrapping the business steps. `System.Exception`s increment the retry counter and are retried; once `MaxRetryCount` is exhausted the exception is rethrown so the job/pipeline fails visibly. `BusinessRuleException`s are logged as a business failure and are NOT retried (retrying a policy violation would just fail the same way again).
- **End Process** — logs the final transaction status and elapsed time.

| Process | Description | Business rule enforced |
|---|---|---|
| Invoice-Data-Extraction | Extracts invoice header and line-item data from scanned PDFs into a structured queue item. | Header total must reconcile with the sum of extracted line items. |
| Contract-Renewal-Reminder | Scans the contracts repository for upcoming expirations and notifies account owners. | Flagged contract must have an assigned account owner on record. |
| Customer-Refund-Processing | Validates refund requests against policy rules and posts approved refunds to the finance system. | Refund must satisfy amount limit and eligibility window. |
| Email-Attachment-Sorter | Reads a shared mailbox, extracts attachments, and files them by sender/subject rules. | A target-folder rule must match the sender/subject. |
| Employee-Onboarding-Bot | Creates AD, email, and HR-system accounts for new hires from an onboarding request queue. | All mandatory onboarding fields must be present. |
| Expense-Report-Validator | Validates submitted expense reports against policy limits and routes exceptions for approval. | Line items exceeding policy limits require manager approval. |

Business steps inside "Process" are documented `Comment` + `LogMessage` placeholders describing exactly what each step does; replace each with the real activity(ies) for that step without changing the surrounding Init/retry/exception/logging scaffold — that scaffold is what makes the process enterprise-grade and is not meant to be stripped out.

## One-time manual setup still required (cannot be done outside Studio/Orchestrator UI)

1. Open `pipelines/MultiProjectCI` and `pipelines/MultiProjectPromotion` in UiPath Studio. Both already reference the real `UiPath.Pipelines.Activities` (2.0.0) activities used by UiPath's own OOB templates (Clone, Analyze, RunTests, Build, PublishPackage, DownloadPackage, UpdateProcess) — resolve/restore dependencies, fix any minor property-panel bindings Studio's designer prefers to re-wire itself, then Publish each to the `Pipelines` Orchestrator folder. This is a one-time bootstrap step: there is no pipeline yet to build these pipelines, so they must be published directly from Studio.
2. In Automation Ops - Pipelines, connect this GitHub repository as the source control provider (one-time).
3. Create the pipeline definitions:
   - **CI-Dev** -> `MultiProjectCI`, `ManifestRelativePath = pipelines/manifest/projects-manifest.csv`, `OrchestratorUrl` = this tenant, no `OrchestratorFolder` needed (comes from the manifest per row, all `shared` today).
   - **Promote-Dev-to-UAT** -> `MultiProjectPromotion`, `SourceOrchestratorFolder = shared`, `DestinationOrchestratorFolder = mk`.
   - **Promote-UAT-to-Prod** -> `MultiProjectPromotion`, `SourceOrchestratorFolder = mk`, `DestinationOrchestratorFolder = mk-laptop`.
   - **Hotfix-to-Prod** (optional) -> same as CI-Dev/Promotion pair but triggered off a `hotfix/*` branch per the whitepaper's branching model.
4. From then on, promoting all 6 (or 1,000) processes together at any tier is a single pipeline run — no per-project pipeline configuration needed going forward.
