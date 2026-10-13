# ai-pr-reviewer — Requirements

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Context
Capability 1 of [ai-platform-bff.md](../../ai-platform-bff.md), in its "no API" variant. The goal is to avoid per-token spend by using a Claude subscription. It is designed with SDD: requirements (EARS) → design → tasks, plus a risk document. It must be approved before any code is written.

Decisions taken:
- **Configurable plan**: `consumer` (Pro/Max) or `commercial` (Team/Enterprise). Each one has its own compliance requirements.
- **Self-hosted** Azure Pipelines agent.
- **Trigger**: full review when the PR is created, incremental review on each push. It can be re-run manually.
- **Advisory only**: it comments and summarizes; it never votes or blocks.
- **Teams notification**: when a review finishes, the configured users get a 1:1 Teams message through a Workflow (Power Automate) webhook. The message is sent as the Workflow owner if possible, otherwise as Flow bot. Email is not used because it gets read too late.

Facts verified against the official documentation (Oct 2026):
- `claude setup-token` generates a **1-year** OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`). It requires Pro, Max, Team or Enterprise.
- Credential precedence: if `ANTHROPIC_API_KEY` is present, it always wins in `-p` mode and **bills the API**.
- `--bare` **does not read** `CLAUDE_CODE_OAUTH_TOKEN`, so it is not used.
- Available flags: `--json-schema`, `--tools`, `--restricted` (v2.1.248+), `--permission-prompts none` (v2.1.259+), `--no-session-persistence`, `--max-turns`, `--strict-mcp-config` and `--disable-slash-commands`.
- `claude auth status` returns JSON with `authMethod`, which can be `oauth_token`, `api_key`, etc.
- Pro and Max are governed by the **Consumer Terms**. Their advertised limits assume "ordinary, individual usage". Anthropic does not allow routing other users' requests through Free/Pro/Max credentials. Team and Enterprise are governed by the **Commercial Terms**.

## Notation
EARS: **WHEN** (event), **WHILE** (state), **WHERE** (optional feature), **IF … THEN** (unwanted behaviour), no prefix for ubiquitous requirements. One obligation (**SHALL** / **SHALL NOT**) per requirement. Ids are stable and never renumbered.

## Acceptance criteria
- A1. When a PR is opened, inline findings and a summary are posted with no human intervention.
- A2. On push, only what changed since the last review is reviewed, without duplicate comments.
- A3. The reviewer never blocks, votes or changes the PR status.
- A4. The reviewer always uses the subscription, never the paid API, and knows which plan it is using.
- A5. With the `consumer` plan, only PRs authored by the token owner are reviewed.
- A6. Neither a PR author nor PR content can extract secrets or execute code on the agent.
- A7. Posted comments contain no secrets, images, external links or mentions.
- A8. Large diffs, quota or credential errors and transient failures end in a controlled way and are explained.
- A9. Low-confidence findings, or findings on made-up lines, are not posted inline.
- A10. Every run leaves an auditable report with no sensitive content.
- A11. When a review completes, the configured recipients get a Teams message with the outcome and the PR link, without email and without spam on incremental pushes.

## Activation and scope
- **R1** WHEN a PR is created against a branch with the validation policy configured, the system SHALL run a full review.
- **R2** WHEN a new iteration is pushed to an already reviewed PR, the system SHALL review only the files whose diff against the target branch has changed since the last review.
- **R3** IF the commit or merge-base of the last review is not available in the repository THEN the system SHALL run a full review.
- **R4** WHEN a run is triggered with no changes since the last review, the system SHALL update the summary saying so without invoking Claude.
- **R5** WHERE the PR has the `ai-review-full` label, the system SHALL run a full review.
- **R6** WHILE the PR is a draft and `skipDrafts` is enabled, the system SHALL skip the review without invoking Claude.
- **R7** WHEN the PR title contains `[skip ai-review]` or the PR has the `no-ai-review` label, the system SHALL skip the review.
- **R8** IF the PR author is in `skipAuthors` THEN the system SHALL skip the review.
- **R9** IF the PR comes from a fork THEN the system SHALL exit without accessing secrets.

## Plan and compliance
- **R10** The system SHALL read `plan`, `tokenOwnerUpn`, `consumerTermsAck` and `tokenExpiresOn` exclusively from the protected variable group.
- **R11** IF `plan` is neither `consumer` nor `commercial` THEN the system SHALL abort before invoking Claude.
- **R12** WHERE `plan` is `consumer`, the system SHALL review only PRs whose author matches `tokenOwnerUpn`.
- **R13** WHERE `plan` is `consumer`, the system SHALL require `consumerTermsAck=true` before invoking Claude.
- **R14** The system SHALL state in the summary the plan used, the model and the Claude Code version.

## Credentials and isolation
- **R15** WHEN it is about to invoke Claude, the system SHALL check that `claude auth status` reports `authMethod` = `oauth_token`.
- **R16** IF the authentication method is not `oauth_token` THEN the system SHALL abort without invoking the model.
- **R17** The system SHALL launch Claude with an allowlisted environment that excludes `SYSTEM_ACCESSTOKEN`, `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN`.
- **R18** The system SHALL NOT include Azure DevOps credentials in the prompt or in files accessible to the model.
- **R19** The system SHALL invoke Claude with only the `Read`, `Grep` and `Glob` tools.
- **R20** The system SHALL invoke Claude without MCP servers, skills or commands.
- **R21** The system SHALL invoke Claude in `--restricted` mode with the working directory limited to the checkout.
- **R22** WHEN preparing the workspace, the system SHALL delete symbolic links whose target is outside the checkout.
- **R23** WHEN preparing the workspace, the system SHALL delete `CLAUDE.md`, `CLAUDE.local.md`, `AGENTS.md`, `.claude/` and `.mcp.json` from the checkout.
- **R24** WHEN preparing the workspace, the system SHALL delete files matching `excludePaths`.
- **R25** The system SHALL execute only code from the reviewer repository at a pinned reference (tag).
- **R26** WHEN the job ends, the system SHALL delete Claude's temporary configuration directory (`CLAUDE_CONFIG_DIR`).
- **R27** IF the Claude Code version is lower than `minClaudeVersion` THEN the system SHALL abort before invoking the model.

## Context and diff
- **R28** The system SHALL compute the diff between the merge-base with the target branch and the PR source commit.
- **R29** The system SHALL exclude from the diff files matching `excludePaths`.
- **R30** The system SHALL exclude from the diff binary files and LFS pointers.
- **R31** IF the filtered diff exceeds `maxChangedLines` THEN the system SHALL split it into per-file chunks of at most `maxChangedLines` lines.
- **R32** IF the number of chunks exceeds `maxChunks` THEN the system SHALL review the first `maxChunks` chunks ordered by `priorityGlobs`.
- **R33** IF files remain unreviewed because of R32 THEN the system SHALL list them in the summary.
- **R34** IF the filtered diff is empty THEN the system SHALL post a summary saying so without invoking Claude.
- **R35** The system SHALL pass Claude the central rules from the reviewer repository.
- **R36** WHERE `.ai-review/rules.md` exists in the target branch, the system SHALL add it to the rules.
- **R37** The system SHALL NOT read rules, prompts or configuration from the PR source branch.
- **R38** The system SHALL delimit the PR title, description and diff as untrusted data within the prompt.

## Model execution
- **R39** The system SHALL request output conforming to `findings.schema.json` via `--json-schema`.
- **R40** The system SHALL limit each invocation to `maxTurns` turns.
- **R41** IF an invocation exceeds `timeoutMinutes` THEN the system SHALL terminate the Claude process.
- **R42** The system SHALL run the chunks of a given PR sequentially.
- **R43** IF Claude returns a usage-limit error THEN the system SHALL stop the remaining chunks without retrying.
- **R44** IF Claude returns a usage-limit error THEN the system SHALL state it in the summary.
- **R45** IF Claude returns an authentication error THEN the system SHALL state in the summary that the reviewer is unavailable due to credentials.
- **R46** IF a chunk's output does not validate against the schema THEN the system SHALL retry that chunk once.
- **R47** IF the R46 retry also fails THEN the system SHALL discard the chunk and note it in the summary.

## Finding validation
- **R48** The system SHALL discard findings with `confidence` lower than `minConfidence`.
- **R49** IF a finding points to a line that is not among the added or modified lines of a file in the diff THEN the system SHALL post it as a file-level comment.
- **R50** IF a finding points to a file that is not in the diff THEN the system SHALL discard it.
- **R51** The system SHALL post at most `maxFindings` inline findings per run, ordered by severity and confidence.
- **R52** The system SHALL compute a fingerprint per finding from the path, the `ruleId` and the normalized content of the affected lines.
- **R53** IF a reviewer thread with the same fingerprint exists, in any status, THEN the system SHALL NOT create another one.
- **R54** The system SHALL post `info` severity findings only in the summary.

## Output sanitization
- **R55** The system SHALL redact in comments any text matching `secretPatterns`.
- **R56** The system SHALL strip markdown images and HTML from generated comments.
- **R57** The system SHALL turn links whose domain is not in `allowedLinkDomains` into plain text.
- **R58** The system SHALL neutralize `@` mentions in generated comments.
- **R59** The system SHALL truncate each comment to `maxCommentChars` characters.

## Publishing to the PR
- **R60** The system SHALL post each inline finding as an `active` thread anchored to the right side of the file, on the reviewed iteration.
- **R61** The system SHALL mark each of its threads with the `AiReviewer.Fingerprint` property and a visible `🤖 AI Reviewer` prefix.
- **R62** The system SHALL keep a single summary thread per PR and update it on every run.
- **R63** The system SHALL store the reviewed iteration, commit and merge-base in the summary thread properties.
- **R64** IF, before publishing, there is a newer iteration than the reviewed one THEN the system SHALL discard the results without publishing them.
- **R65** IF the PR is completed or abandoned at publishing time THEN the system SHALL exit without publishing.
- **R66** IF the Azure DevOps API returns 429 or 5xx THEN the system SHALL retry with exponential backoff up to 3 times.
- **R67** The system SHALL NOT vote, approve, reject or post PR statuses.
- **R68** The system SHALL NOT modify threads it did not create.
- **R69** The system SHALL write comments in the `language` language (default `en`).

## Outcome and observability
- **R70** The system SHALL finish the job as `Succeeded` or `SucceededWithIssues`, never as `Failed`.
- **R71** IF the review does not complete because of an error THEN the system SHALL finish as `SucceededWithIssues` with a warning stating the cause.
- **R72** The system SHALL publish the `ai-review-report.json` artifact with duration, chunks, generated, filtered and published findings, reported usage and outcome.
- **R73** The system SHALL NOT write the full diff or secret values to logs or artifacts.
- **R74** WHILE fewer than 30 days remain until `tokenExpiresOn`, the system SHALL emit a warning in the pipeline log.

## Team notification
- **R75** WHERE `notifyRecipients` is non-empty, the system SHALL send a notification when a full review publishes its results.
- **R76** WHEN an incremental review publishes at least one new finding with a severity in `notifyOnIncrementalSeverities`, the system SHALL send a notification.
- **R77** IF the review was skipped (R6–R9, R12), discarded (R64, R65) or had no changes (R4, R34) THEN the system SHALL NOT send a notification.
- **R78** The system SHALL read `notifyWebhookUrl` and `notifyRecipients` exclusively from the protected variable group.
- **R79** The system SHALL send the notification to the `notifyWebhookUrl` Workflow webhook with the recipients, PR id, title, repo, author, finding counts by severity, outcome, plan and PR URL.
- **R80** The system SHALL NOT include finding bodies, the diff or secret values in the notification.
- **R81** The system SHALL apply the R55–R59 sanitization to the PR title and author before sending them.
- **R82** The system SHALL build the PR URL from the collection URI, project, repo and PR id of the pipeline context, never from PR content.
- **R83** IF sending the notification still fails after the R66 retries THEN the system SHALL finish as `SucceededWithIssues` without modifying the published comments.
- **R84** The system SHALL record the notification status (sent, skipped with its reason, or failed) in `ai-review-report.json`.
- **R85** The system SHALL exclude `notifyWebhookUrl` from Claude's environment.

## Acceptance criteria coverage
| Criterion | Covered by |
|---|---|
| A1 review on PR open | R1, R28, R35, R39, R60, R62 |
| A2 incremental, no duplicates | R2, R3, R4, R5, R52, R53, R63, R64 |
| A3 never blocks | R67, R68, R70 |
| A4 subscription, not API | R10, R11, R14, R15, R16, R17 |
| A5 consumer = owner only | R12, R13 |
| A6 no extraction or execution | R9, R17–R27, R37, R38, R85 |
| A7 safe comments | R55–R59 |
| A8 controlled failures | R31–R34, R40–R47, R65, R66, R71, R74 |
| A9 finding quality | R48–R51, R54 |
| A10 auditable | R72, R73, R84 |
| A11 Teams notification | R75–R85 |
| Exclusions and options | R6, R7, R8, R29, R30, R36, R69 |
