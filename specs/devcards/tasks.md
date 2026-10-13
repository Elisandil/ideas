# devcards — Tasks

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## v0 — no Roslyn
- [ ] **T1 — Spike** on Windows, with the real BFF and the pinned Claude Code version. Confirm:
  - the exec-form `SessionStart` hook works (`command` + `args`) and how its stdout reaches the session context;
  - how to declare an agent with no tools (reader) and with `Read` only (editor);
  - that `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` are substituted in skill bodies;
  - that the `bin/` shims run from the Bash tool under Git Bash;
  - the case format for `claude plugin eval`;
  - which Azure Repos merge types leave `Merged PR <id>:` (merge, squash, semi-linear, rebase fast-forward);
  - [for v1] whether `MSBuildWorkspace` loads the real solution, private Azure Artifacts feeds included.

  Results go to `progress/spike.md`; if any contradicts the spec, the spec is updated first.
- [ ] **T2 — CLI skeleton + `vault`**. Tests: `Options_ReadsVaultPath` (R1), `Vault_NotSet_StopsWithoutWriting` (R2), `Vault_InsideRepo_Refused` (R3), `Vault_InPersonalOneDrive_WarnsOnce` (R4), `Vault_InGitRepoWithRemote_WarnsOnce` (R5), `VaultKey_FromAzureReposUrls_HttpsVisualStudioSsh` (R6), `VaultKey_NoOrigin_UsesFolderName` (R7), `Options_LanguageDefaultEn` (R8), `Arch_OnlyAdoModuleUsesHttp` (R10).
- [ ] **T3 — Schemas, budgets and lint**. Tests: `Schemas_AllCardTypesDefined` (R11), `Lint_SchemaViolation_Rejected` (R12), `Lint_OverBudget_Rejected` (R18), `Lint_BannedPhrase_Rejected` (R19), `Lint_LongCodeBlock_Rejected` (R20), `Lint_UnanchoredStep_Rejected` (R21), `Lint_SecretPattern_Rejected` (R22).
- [ ] **T4 — Anchors**. Tests: `Anchor_AllKindsParsed` (R26), `Hash_IgnoresCrlfAndWhitespace` (R27), `Anchor_MissingPathOrLines_Rejected` (R28), `Anchor_MovedSnippet_Relocated`, `Anchor_RenamedFile_Relocated` (R29), `Anchor_HumanMarkerRendered` (R31), `Anchor_RendersVsCodeLink` (R32).
- [ ] **T5 — Render and registry**. Tests: `Render_IgnoresMarkdownInDraft_UsesTemplate` (R13), `FrontMatter_HasRequiredFields` (R14), `Id_FromTypeAndSlug` (R15), `Id_Collision_AppendsSuffix` (R16), `Registry_RecordsPathAnchorsHashesStatus` (R17), `Rerender_PreservesHumanSections` (R97), `Notes_AlwaysHumanOwned` (R98), `Card_ResolvedByIdAfterMove` (R102).
- [ ] **T6 — Guard**. Tests: `Guard_SnapshotTaken` (R33), `Guard_RepoChanged_WarnsWithFiles` (R34), `AllCommands_LeaveRepoClean` (R9; runs every CLI command over a fixture repo).
- [ ] **T7 — Lint loop + `capture` skill**. Tests: `Render_WithoutPassingLint_Refused` (R24). Eval cases: `Eval_LintFailure_RetriesOnce` (R23), `Eval_SecondFailure_ShowsViolations` (R25), `Eval_Capture_WithQuestion` (R35), `Eval_Capture_ProposesQuestion` (R36), `Eval_Capture_NoReads_NoCard` (R37), `Eval_Capture_OffersSimilarCard` (R38).
- [ ] **T8 — Git context for PR and digest** (merge and squash fixtures). Tests: `Pr_FindsMergedPrCommit` (R40), `Pr_NoMatch_RequestsRange` (R41), `Pr_RangeDiff` (R42), `Pr_ListsCardsInChangedLines` (R44), `Pr_MarksAffectedCardsStale` (R45), `Digest_SinceLastDigest` (R46), `Digest_FirstRun_Last7Days` (R47), `Digest_OneLinePerPr` (R48), `Digest_GroupsByTopFolder` (R49), `Digest_NeedsAttention_AuthoredOrAnchored` (R50), `Digest_LinksExistingPrCards` (R51), `Digest_ShallowClone_Warns` (R52).
- [ ] **T9 — `pr` and `digest` skills + writer agent**. Eval: `Eval_PrCard_FillsBehaviourConfigRisk_WithinBudget` (R43).
- [ ] **T10 — Azure DevOps REST**, with WireMock.NET. Tests: `AdoLogin_StoresInCredentialManager` (R53), `PrCard_AddsDescriptionReviewersWorkItems` (R54), `Ado_IssuesOnlyGetRequests` (R55), `Ado_Failure_FallsBackToGitOnly` (R56), `Pat_NeverInVaultLogsOrStdout` (R57).
- [ ] **T11 — `map`**. Tests: `Map_OneLinePerProject` (R81), `Map_LinksAnchoredCards` (R82).
- [ ] **T12 — Run log**. Tests: `Runs_AppendsRecord` (R105), `Runs_NoDiffNoCredentials` (R106).

## v1 — fact extraction
- [ ] **T13 — Roslyn `extract`** on `fixtures/SampleBff`. Tests: `Extract_SemanticLoad` (R58), `Extract_SyntaxFallback_MarksInferred` (R59), `Extract_ControllersAndMinimalApis` (R60), `Extract_DtoShapesWithNullability` (R61), `Extract_TypedAndRefitClients` (R62), `Extract_ResiliencePolicies` (R63), `Extract_ManualAutoMapperMapsterAssignments` (R64), `Extract_ConfigKeys` (R65), `Facts_CachedPerCommit` (R66), `Facts_CacheReused` (R67), `Facts_UnresolvedAsQuestionMark` (R68), `Anchor_ResolvedBySymbol` (R30).
- [ ] **T14 — `flow` + `lineage`**. Tests: `Flow_UnknownRoute_ListsClosest` (R70), `Flow_DiagramFromFacts` (R71), `Flow_FailureModesPrefilledFromPolicies` (R72), `Lineage_OneRowPerField` (R73), `Lineage_Untraced_IsQuestionMark` (R74), `Lineage_ConventionTransform` (R75). Eval: `Eval_Flow_ProducesFlowAndLineage` (R69), `Eval_Capture_OffersFlow` (R39).
- [ ] **T15 — `contract` + contract deltas**. Tests: `Contract_SkeletonFromFacts` (R76), `Contract_OpenApiComplements` (R77), `Contract_ListsAllStatusCodes` (R78), `FactsDiff_DetectsContractChanges` (R79), `FactsDiff_AdditiveVsBreaking` (R80).

## v2 — quality loop and living docs
- [ ] **T16 — Editor**. Tests: `Agent_EditorHasNoWriteTools` (R84), `EditedFailsLint_RendersPreEdit` (R85). Eval: `Eval_Editor_RunsAfterLint` (R83).
- [ ] **T17 — Verify**. Tests: `Verify_QuestionsFromFactTemplates` (R86), `Agent_ReaderHasNoTools` (R87), `Verify_ScoresExpectedFacts` (R88), `Verify_ScoreInFrontMatter` (R89), `Verify_ReportsUncitedSections` (R90).
- [ ] **T18 — `check` hook + index**. Tests: `Check_AllCodeAnchorsAgainstHead` (R91), `Check_Unresolvable_MarksStale` (R92), `Check_PrintsStaleCount` (R93), `Check_Timeout_DoesNotBlock` (R94), `Index_WithinTokenBudget` (R95).
- [ ] **T19 — `refresh`, ownership and style learning**. Tests: `Refresh_OnlyMachineSectionsOfStaleCards` (R96), `EditedSection_BecomesHuman` (R99), `EditedSection_SavesStyleExample` (R100), `Style_MaxExamplesPerType` (R101).
- [ ] **T20 — `ask`**. Eval: `Eval_Ask_UsesCardsFirst` (R103), `Eval_Ask_OffersCapture` (R104).

## Closing
- [ ] **T21 — Eval suite and metrics**: `fixtures/SampleBff` with question sets per card type, the `claude plugin eval` suite, and `metrics.ps1` reading `runs.jsonl` (first-draft lint pass rate, verify score, words per answered question, editor change rate).
- [ ] **T22 — Traceability**: write the R↔test map to `progress/impl_devcards.md`; `dotnet test` and `claude plugin eval` green.
- [ ] **T23 — Dogfooding**: two weeks on the real BFF; review the metrics; tune budgets, banned phrases and templates.

## Verification
- Deterministic parts (vault, guard, git context, lint, anchors, render, check, facts, verify scoring, ADO client): `dotnet test`, with no LLM calls.
- Model-driven behaviour (skills, writer, editor, reader): `claude plugin eval` cases over `fixtures/SampleBff`. These consume subscription quota, so run them on demand.
- End-to-end on the real BFF:
  - run `/devcards:digest` after a pull and check it against the Azure DevOps PR list;
  - capture one investigation;
  - [v1] produce the flow card of one known endpoint and check every lineage row by hand;
  - [v2] change an anchored method and confirm the session start reports it stale;
  - edit a section and confirm `/devcards:refresh` keeps it;
  - confirm `git status` of the BFF is clean after every command.
