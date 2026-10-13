# ai-pr-reviewer — Design

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Architecture
```
Azure Repos (BFF repo)                         ai-pr-reviewer repo (tag vX.Y.Z)
 Branch policy: Build validation (Optional)     templates/review.yml  (extends)
   └─► pipelines/ai-review.yml ──extends──►     src/AiReviewer.Cli    (.NET 9)
                                                prompts/, rules/, schemas/
Self-hosted agent (dedicated pool, 1 agent)
 1. checkout PR (source commit) + fetch target        ← git
 2. AiReviewer.Cli review                              ← deterministic logic
      ├─ PolicyGuard / Preflight
      ├─ DiffProvider + WorkspaceSanitizer
      ├─ ClaudeRunner ──► claude -p (read-only) ──► Anthropic (subscription)
      ├─ FindingValidator + OutputSanitizer
      ├─ CommentPublisher ──► Azure DevOps REST (System.AccessToken)
      └─ TeamsNotifier    ──► Workflow webhook ──► Teams 1:1 chats
 3. publish ai-review-report.json
```
Core principle: **the model only reads and returns JSON. Everything written to Azure DevOps is written by deterministic code with a token the model never sees.**

## Files
**New `ai-pr-reviewer` repo:**
| Path | Contents |
|---|---|
| `templates/review.yml` | `extends` template: pool, checkout, variable group, CLI invocation, artifact (R25, R70) |
| `prompts/system.md` | Role, format, severities and the rule to treat PR content as untrusted data (R38, R69) |
| `rules/bff-csharp.md` | Central rules: HttpClient/Polly, `CancellationToken`, downstream N+1, PII in logs, async, DTOs and mappings, authorization (R35) |
| `schemas/findings.schema.json` | Output schema (R39) |
| `src/AiReviewer.Cli/Program.cs` | `review` command, composition and IoC |
| `src/AiReviewer.Cli/Config/ReviewerOptions.cs` | Options with defaults and validation (R10, R11) |
| `src/AiReviewer.Cli/Pipeline/PipelineContext.cs` | Reads Azure Pipelines predefined variables |
| `src/AiReviewer.Cli/Guards/PolicyGuard.cs` | Skips and plan rules (R6–R9, R12, R13) |
| `src/AiReviewer.Cli/Guards/Preflight.cs` | Version, `auth status` and token expiry (R15, R16, R27, R74) |
| `src/AiReviewer.Cli/Git/DiffProvider.cs` | Merge-base, diff, filters, incremental, chunks (R2–R4, R28–R34) |
| `src/AiReviewer.Cli/Git/WorkspaceSanitizer.cs` | Symlinks, agent configuration files, exclusions (R22–R24) |
| `src/AiReviewer.Cli/Claude/PromptBuilder.cs` | Assembles the prompt and the rules from the target branch (R35–R38) |
| `src/AiReviewer.Cli/Claude/ClaudeRunner.cs` | Process, arguments, allowlisted environment, timeout (R17, R19–R21, R40–R42) |
| `src/AiReviewer.Cli/Claude/ClaudeResultClassifier.cs` | Classification: success, quota, auth, max_turns, schema (R43–R47) |
| `src/AiReviewer.Cli/Findings/FindingValidator.cs` | Confidence, valid lines, cap, info→summary (R48–R51, R54) |
| `src/AiReviewer.Cli/Findings/Fingerprint.cs` | Stable fingerprint (R52) |
| `src/AiReviewer.Cli/Findings/OutputSanitizer.cs` | Secrets, images/HTML, links, mentions, truncation (R55–R59) |
| `src/AiReviewer.Cli/AzureDevOps/AdoClient.cs` | PR, iterations, labels, threads; retries (R65, R66) |
| `src/AiReviewer.Cli/AzureDevOps/ReviewStateStore.cs` | State in the summary thread properties (R62, R63) |
| `src/AiReviewer.Cli/AzureDevOps/CommentPublisher.cs` | Thread creation, deduplication, staleness (R53, R60, R61, R64, R67, R68) |
| `src/AiReviewer.Cli/Notifications/TeamsNotifier.cs` | Decides whether to notify, builds the payload and sends it to the Workflow webhook (R75–R84) |
| `src/AiReviewer.Cli/Reporting/RunReport.cs` | Artifact and `##vso` commands (R70–R73, R84) |
| `tests/AiReviewer.Tests/` | xUnit + WireMock.NET; fixtures of diffs, recorded Claude outputs and ADO responses |
| `eval/golden/` + `eval/run-eval.ps1` | Golden cases; manual run (consumes quota) |
| `docs/operations.md` | Onboarding a repo, token rotation, troubleshooting |
| `docs/notify-workflow.md` | Step-by-step setup of the Teams Workflow, the expected payload and how to rotate the webhook URL |

**In each BFF repo:** only `pipelines/ai-review.yml`, about 10 lines with `extends`, and optionally `.ai-review/rules.md`.

**Not touched:** the BFF code, its existing build and test pipelines, or the required branch policies.

## Azure DevOps infrastructure
- **Dedicated pool** `ai-reviewer-pool` with 1 agent. This satisfies R42 naturally and consumes quota serially. The pool is authorized only for this pipeline.
- **Variable group** `ai-reviewer-secrets`, linked to Key Vault. It holds `CLAUDE_CODE_OAUTH_TOKEN` (secret), `plan`, `tokenOwnerUpn`, `consumerTermsAck`, `tokenExpiresOn`, `notifyWebhookUrl` (secret) and `notifyRecipients` (R78). It has two *Checks*: **Required template** (`templates/review.yml@ai-pr-reviewer` on `refs/tags/*`) and restricted pipeline permissions. So even if a PR modifies `ai-review.yml`, it cannot get the secret (R10, R25).
- **Branch policy**: Build validation **Optional**, automatic trigger and expiry "immediately when the source branch is updated" (R1, R2, R70).
- **Build Service**: on the repo it has only "Contribute to pull requests", without "Contribute" or "Create branch". *Job authorization scope* limited to the current project. *Make secrets available to builds of forks* disabled (R9).
- **Teams Workflow** (Power Automate), created by the reviewer's owner. The flow is: trigger "When a Teams webhook request is received" → `Apply to each` over `recipients` → "Post card in a chat or channel" in a 1:1 chat. The card is sent **as the owner's user** ("Post as User") if T1 confirms that it supports 1:1 chats, otherwise as **Flow bot** ("Chat with Flow bot"). The pipeline never has the owner's credentials; they stay in the Power Automate connection. The self-hosted agent needs outbound access to the Power Automate endpoint (checked in T1).

## Claude invocation (R15–R21, R39–R41)
```
claude auth status                       # must return authMethod=oauth_token
claude -p "<instruction>" < chunk.patch \
  --restricted --tools "Read,Grep,Glob" \
  --disallowedTools "mcp__*" --strict-mcp-config --disable-slash-commands \
  --permission-prompts none --no-session-persistence \
  --append-system-prompt-file <assembled prompt+rules> \
  --json-schema "<findings.schema.json>" --output-format json \
  --max-turns <maxTurns> --model <model>
```
Process environment (allowlist): `PATH`, `HOME`/`USERPROFILE`, `TMP`, `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CONFIG_DIR=<job temp>`, `DISABLE_AUTOUPDATER=1` and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`. The names of these variables are confirmed in spike T1. `--bare` is **never** used, because it ignores the OAuth token. `notifyWebhookUrl` is not on the allowlist (R85).

## Output schema (`findings.schema.json`)
```json
{ "summary": "string",
  "findings": [{
    "file": "string", "line": "int>=1", "endLine": "int|null",
    "severity": "critical|major|minor|info",
    "category": "security|correctness|resilience|performance|conventions|tests",
    "ruleId": "string", "title": "string<=120", "body": "string",
    "suggestion": "string|null", "confidence": "number 0..1" }] }
```

## Notification payload (R79, R80)
```json
{ "recipients": ["lead@company.com"],
  "prId": 1234, "title": "string (sanitized, R81)", "repo": "string",
  "author": "string (sanitized, R81)", "mode": "full|incremental",
  "outcome": "Succeeded|SucceededWithIssues", "plan": "consumer|commercial",
  "counts": { "critical": 0, "major": 2, "minor": 3, "info": 1 },
  "prUrl": "<collectionUri><project>/_git/<repo>/pullrequest/<prId>" }
```
The Workflow renders it as an Adaptive Card with an **Open PR** button. Finding bodies and the diff are never sent. `prUrl` comes from `System.CollectionUri`, `System.TeamProject`, `Build.Repository.Name` and `System.PullRequest.PullRequestId` (R82).

## Configuration (template parameters with defaults)
| Key | Default | Source |
|---|---|---|
| `plan`, `tokenOwnerUpn`, `consumerTermsAck`, `tokenExpiresOn` | — | Protected variable group (R10) |
| `model` | `sonnet` | Template |
| `language` | `en` | Template |
| `minConfidence` | `0.7` | Template |
| `maxFindings` | `15` | Template |
| `maxChangedLines` / `maxChunks` | `1500` / `4` | Template |
| `maxTurns` / `timeoutMinutes` | `25` / `15` | Template |
| `maxCommentChars` | `4000` | Template |
| `skipDrafts` | `true` | Template |
| `skipAuthors` | bots (Renovate/Dependabot) | Template |
| `excludePaths` | `**/Migrations/**`, `*.Designer.cs`, `*.g.cs`, `**/wwwroot/lib/**`, `appsettings.*.json`, `*.pfx`, `*.snk`, `.env*`, `**/packages.lock.json` | Template and target repo rules |
| `priorityGlobs` | `**/Controllers/**`, `**/Endpoints/**`, `**/Clients/**`, `**/Auth*/**`, `**/*Service*.cs`, rest, `**/*Tests*/**` | Template |
| `secretPatterns` | `sk-ant-*`, ADO PATs, `AccountKey=`, `Password=`, JWTs, PEM keys | Template |
| `allowedLinkDomains` | `learn.microsoft.com`, `dev.azure.com/<org>` | Template |
| `minClaudeVersion` | `2.1.259` | Template |
| `notifyWebhookUrl` (secret), `notifyRecipients` (comma-separated UPNs) | — (empty = notification disabled) | Protected variable group (R78) |
| `notifyOnIncrementalSeverities` | `critical,major` | Template |

## Algorithm
1. Read `PipelineContext` and `ReviewerOptions`, and validate the plan → R10, R11.
2. If the PR comes from a fork, exit → R9. Query the PR in ADO: status, draft, labels, author and title.
3. `PolicyGuard`: skip on draft, label, title or author → R6, R7, R8. If the plan is `consumer`, require the author to be `tokenOwnerUpn` and `consumerTermsAck` to be true → R12, R13.
4. `Preflight`: minimum version → R27; `auth status` = `oauth_token` → R15, R16; expiry warning → R74.
5. `ReviewStateStore`: read the summary thread (previous iteration, commit and merge-base).
6. `DiffProvider`:
   - a. `mb = merge-base(origin/<target>, sourceCommit)` and `D_now = diff(mb...sourceCommit)`, filtered by `excludePaths` and binaries/LFS → R28, R29, R30.
   - b. Pick the mode: `full` on the first run, with the `ai-review-full` label, or when the previous commits are missing → R1, R3, R5. Otherwise `incremental`: compute `D_prev = diff(mb_prev...commit_prev)` and include the files whose patch in `D_now` differs from `D_prev`, compared by hash of the normalized patch → R2. This ignores merges of the target branch into the source branch.
   - c. If nothing is left to review: "no changes" or "no reviewable changes" summary → R4, R34.
   - d. Split into per-file chunks (`maxChangedLines`), order by `priorityGlobs` and trim to `maxChunks`, listing what was skipped → R31, R32, R33.
7. `WorkspaceSanitizer`: delete external symlinks, agent configuration files and `excludePaths` from the checkout → R22, R23, R24.
8. `PromptBuilder`: `prompts/system.md` + `rules/bff-csharp.md` + `git show origin/<target>:.ai-review/rules.md`. The title, description and diff go inside `<untrusted_pr_content>` delimiters → R35–R38, R69.
9. `ClaudeRunner`, sequential per chunk: launch with the allowlisted environment and flags → R17–R21, R39–R42. `ClaudeResultClassifier`:
   - quota → stop and note it → R43, R44;
   - authentication → note it → R45;
   - invalid schema → 1 retry, then discard → R46, R47;
   - timeout → kill → R41.
10. `FindingValidator`:
    - confidence → R48;
    - map of added/modified lines per file: invalid line → file-level comment; file not in the diff → discard → R49, R50;
    - `info` → summary → R54;
    - sort and trim → R51;
    - fingerprint → R52.
11. `OutputSanitizer`: apply to every comment body and to the summary → R55–R59.
12. `CommentPublisher`:
    - query the PR again: if completed or abandoned → exit (R65); if there is a newer iteration → discard (R64);
    - list threads, skip existing fingerprints (R53) and create the new threads with `threadContext.rightFileStart/End` and `pullRequestThreadContext` (iteration and `changeTrackingId` obtained from `iterations/{id}/changes`) → R60, R61;
    - upsert the summary and its state → R14, R62, R63;
    - retries on 429/5xx → R66. Never votes or statuses → R67. Only threads with its own marker are touched → R68.
    - b. `TeamsNotifier`, only if step 12 published:
      - full review → notify (R75); incremental → notify only if new findings with a severity in `notifyOnIncrementalSeverities` were published (R76); skip, discard or no changes → no notification (R77);
      - sanitize the title and author with `OutputSanitizer` (R81), build `prUrl` from the pipeline context (R82) and POST the payload (R79, R80);
      - retries per R66; if it still fails, warning and `SucceededWithIssues`, without touching the published comments (R83).
13. `RunReport`: JSON artifact without the diff or secrets, with the notification status, outcome `Succeeded` or `SucceededWithIssues` → R70–R73, R84. Delete `CLAUDE_CONFIG_DIR` in the `finally` block and in an `always()` step of the template → R26.

## Error handling
| Situation | Behaviour | Job outcome |
|---|---|---|
| Invalid configuration, consumer without ack, old version | Does not invoke Claude; warning in the log | SucceededWithIssues |
| `authMethod` ≠ `oauth_token` | Aborts; summary "credential not valid for subscription" | SucceededWithIssues |
| Usage limit | Stops the chunks; posts the valid partial findings and a "quota exhausted" summary | SucceededWithIssues |
| Timeout / `error_max_turns` | Discards the chunk; continues with the next one | SucceededWithIssues |
| Invalid JSON twice | Discards the chunk | SucceededWithIssues |
| Persistent ADO 429/5xx | Reported in the artifact; nothing posted | SucceededWithIssues |
| Teams webhook fails or times out | Comments already published; warning in the log and status in the artifact | SucceededWithIssues |
| PR closed or new iteration | Exits silently (the new run will review) | Succeeded |
| Skip (draft, label, author, fork) | Informational log | Succeeded |

## Rejected alternatives
1. **Letting Claude post the comments itself** (Bash with `az`/curl or an ADO MCP server). Rejected because it would give the model a write-capable token and execution tools. A prompt injection in the PR code could then approve, delete threads or exfiltrate the token. Separating "the model reads and returns JSON" from "deterministic code writes" is the main defence.
2. **PowerShell scripts only.** Rejected because JSON parsing, line mapping and sanitization need proper unit tests. A .NET CLI with xUnit fits the team and enables R↔test traceability.
3. **Reviewing the merge commit `refs/pull/N/merge`.** Rejected because line numbers may not match the iteration the human reviewer sees. The source commit is reviewed against the merge-base.
4. **Incremental diff `commit_prev..commit_now`.** Rejected because, if the source branch merges in the target, it would include code that is not part of the PR. The comparison is done at per-file patch level.
5. **Sending the Teams message "as my user" through Microsoft Graph** (`POST /chats/{id}/messages`). Rejected because Graph only allows it with delegated permissions. The user's refresh token would have to be stored in the pipeline, where it expires and clashes with MFA and Conditional Access. The Workflow keeps the credentials inside Power Automate.
6. **Azure Repos app for Teams, or @mentions in ADO comments.** Rejected because the app's subscriptions cannot filter on "AI review finished" and would add noise. Mentions trigger email, and R58 neutralizes them anyway.
