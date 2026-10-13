# ai-pr-reviewer — Tasks

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

- [ ] **T1 — CLI validation spike**, on the pinned version and on the self-hosted agent. Confirm:
  - that `--restricted` works with `CLAUDE_CODE_OAUTH_TOKEN`;
  - the name of the structured output field with `--json-schema`;
  - the shape of the quota and authentication errors in `--output-format json`;
  - whether `--restricted` follows symlinks and whether it loads `CLAUDE.md`;
  - the names of `DISABLE_AUTOUPDATER` and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`;
  - the format of `properties` on ADO threads;
  - whether build validation triggers on draft PRs;
  - whether the Teams Workflow action "Post as User" can post to a 1:1 chat or only to group chats and channels (if not, Flow bot);
  - whether the "When a Teams webhook request is received" trigger is available in the tenant without a premium license;
  - outbound connectivity from the self-hosted agent to the Power Automate endpoint.

  Results go to `progress/spike_cli.md` and, if any of them contradicts this spec, the spec is adjusted before continuing. Covers: groundwork for R15–R21, R39, R43–R45, R61, R79.
- [ ] **T2 — Infrastructure**: dedicated pool, Key Vault and variable group with *Required template* checks, least-privilege Build Service permissions, Optional branch policy and forks without secrets. Create the Teams Workflow, store its URL in Key Vault as `notifyWebhookUrl` and fill in `notifyRecipients`. Documented in `docs/operations.md` and `docs/notify-workflow.md`. Covers: R9, R10, R25, R42, R67, R78.
- [ ] **T3 — Skeleton**: .NET 9 solution `AiReviewer.Cli`, `AiReviewer.Tests` and `ReviewerOptions` with validation. Tests: `Options_InvalidPlan_Aborts`, `Options_SecretsOnlyFromVariableGroup`. Covers: R10, R11.
- [ ] **T4 — `PolicyGuard`**. Tests: `Skip_Draft_WhenSkipDraftsEnabled`, `Skip_TitleTagOrLabel`, `Skip_AuthorInSkipList`, `Fork_EndsWithoutSecrets`, `Consumer_OtherAuthor_Skipped`, `Consumer_WithoutAck_Aborts`. Covers: R6–R9, R12, R13.
- [ ] **T5 — `Preflight`**. Tests: `Preflight_OldVersion_Aborts`, `Preflight_ApiKeyAuth_Aborts`, `Preflight_OAuthToken_Passes`, `Preflight_TokenExpiringIn29Days_Warns`. Covers: R15, R16, R27, R74.
- [ ] **T6 — `DiffProvider`**, on a fixture git repo. Tests: `Diff_UsesMergeBaseAndSourceCommit`, `Diff_ExcludesPathsBinariesLfs`, `Incremental_OnlyFilesWithChangedPatch`, `Incremental_IgnoresTargetMergedIntoSource`, `MissingPrevCommit_FallsBackToFull`, `LabelFull_ForcesFull`, `NoChanges_SkipsClaude`, `EmptyFilteredDiff_SkipsClaude`, `Chunking_RespectsMaxLines`, `Chunking_PriorityAndMaxChunks_ListsSkipped`. Covers: R1–R5, R28–R34.
- [ ] **T7 — `WorkspaceSanitizer`**. Tests: `RemovesExternalSymlinks`, `KeepsInternalSymlinks`, `RemovesAgentConfigFiles`, `RemovesExcludedFiles`. Covers: R22–R24.
- [ ] **T8 — `PromptBuilder`**. Tests: `Rules_CentralAlwaysIncluded`, `Rules_RepoRulesReadFromTargetBranch`, `Rules_NeverFromSourceBranch`, `PrContent_WrappedAsUntrusted`, `Language_Applied`. Covers: R35–R38, R69.
- [ ] **T9 — `ClaudeRunner` + `ClaudeResultClassifier`**, with a fake executable that replays the outputs recorded in T1. Tests: `Env_IsWhitelisted_NoAdoTokenNoApiKey`, `Args_ReadOnlyToolsRestrictedNoMcp`, `Args_JsonSchemaAndMaxTurns`, `Timeout_KillsProcess`, `Chunks_RunSequentially`, `UsageLimit_StopsRemainingChunks`, `UsageLimit_ReportedInSummary`, `AuthError_ReportedInSummary`, `InvalidSchema_RetriesOnce`, `InvalidSchemaTwice_DropsChunk`. Covers: R17–R21, R39–R47.
- [ ] **T10 — `FindingValidator` + `Fingerprint`**. Tests: `LowConfidence_Dropped`, `LineOutsideHunk_BecomesFileComment`, `FileNotInDiff_Dropped`, `MaxFindings_SortedBySeverityConfidence`, `Info_GoesToSummaryOnly`, `Fingerprint_StableAcrossLineShifts`. Covers: R48–R52, R54.
- [ ] **T11 — `OutputSanitizer`**. Tests: `Redacts_SecretPatterns`, `Strips_ImagesAndHtml`, `Links_NonAllowlistedToPlainText`, `Mentions_Neutralized`, `Truncates_MaxChars`. Covers: R55–R59.
- [ ] **T12 — `AdoClient` + `ReviewStateStore` + `CommentPublisher`**, with WireMock.NET. Tests: `Publish_InlineThreadRightSideWithIteration`, `Publish_MarkerAndFingerprintProperty`, `Publish_SkipsExistingFingerprintAnyStatus`, `Summary_SingleThreadUpserted`, `Summary_StoresIterationCommitMergeBase`, `Summary_IncludesPlanModelVersion`, `NewerIteration_DiscardsResults`, `ClosedPr_ExitsWithoutPublishing`, `Transient_RetriesThreeTimes`, `NeverCallsVoteOrStatusEndpoints`, `NeverTouchesForeignThreads`. Covers: R14, R53, R60–R68.
- [ ] **T13 — `RunReport` and outcome**. Tests: `Job_NeverFailed`, `Error_SucceededWithIssuesWithReason`, `Report_ContainsCounts`, `Report_NoDiffNoSecrets`, `ConfigDir_DeletedInFinally`. Covers: R26, R70–R73.
- [ ] **T14 — `templates/review.yml`, `prompts/system.md`, `rules/bff-csharp.md` and `schemas/findings.schema.json`.** The template checks out the source commit with `fetchDepth: 0` and `persistCredentials: false`, adds an `always()` cleanup step and publishes the artifact. Covers: R21, R25, R26, R35, R39.
- [ ] **T15 — Golden set**: 10 or more BFF diffs with known bugs (missing `CancellationToken`, `HttpClient` without timeout, PII in logs, N+1, hardcoded secret, prompt injection) and 3 clean diffs, in `eval/golden/`. `run-eval.ps1` measures precision and false positives. Run manually.
- [ ] **T16 — Pilot**: enable on 1 repo with an Optional policy for 2 weeks, review the rate of threads resolved as "Won't fix" or "By design", and tune `minConfidence` and the rules.
- [ ] **T18 — `TeamsNotifier`**, with WireMock.NET. Tests: `Notify_FullReview_Sends`, `Notify_IncrementalOnlyWithNewNotifySeverities`, `Notify_SkippedDiscardedOrNoChanges_DoesNotSend`, `Notify_NoRecipients_Disabled`, `Config_NotifyOnlyFromVariableGroup`, `Payload_NoBodiesNoDiffNoSecrets`, `Payload_TitleAndAuthorSanitized`, `Payload_UrlFromPipelineContext`, `Notify_Failure_SucceededWithIssuesCommentsIntact`, `Report_ContainsNotificationStatus`, `Env_ExcludesWebhookUrl`. Covers: R75–R85.
- [ ] **T17 — Write the R↔test map** to `progress/impl_ai-pr-reviewer.md` and run `dotnet test` green.

## Verification
- Deterministic components are tested with `dotnet test`, covering every R through the tests named above. They use recorded Claude outputs and consume no quota.
- Model quality is measured with `eval/run-eval.ps1` on the golden set: precision and false positives, run manually.
- End-to-end test on a sandbox Azure DevOps repo:
  - open a PR with seeded bugs → threads and summary;
  - push → only the new files and no duplicates;
  - modify `ai-review.yml` to print the secret → the *Required template* check blocks it;
  - add `.claude/settings.json` with a hook and an external symlink → no effect;
  - set `ANTHROPIC_API_KEY` on the agent → aborts per R16;
  - open a PR → the recipients get a Teams message with the outcome and an Open PR button;
  - push without new critical or major findings → no Teams message;
  - set a wrong `notifyWebhookUrl` → comments published and job `SucceededWithIssues`.
