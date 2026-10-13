# devcards — Requirements

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Context
Capability 29 of [ai-platform-bff.md](../../ai-platform-bff.md), designed in [devcards.md](../../devcards.md). The goal is a local Claude Code plugin that produces short, anchored, tested developer docs ("cards") for a C# BFF hosted in **Azure DevOps**. Cards are written into a personal vault outside the repository.

Decisions taken:
- **Local Claude Code plugin**: no pipeline and no team approval needed.
- **Personal vault outside the repo**: `<vaultPath>/<org>/<project>/<repo>/`, Obsidian-compatible markdown.
- **Card types**: Flow + field lineage, PR cards and digest, Q cards from sessions, Contract cards and a module map.
- **Phased delivery**: every requirement is tagged **[v0]** (no Roslyn), **[v1]** (Roslyn fact extraction) or **[v2]** (editor, verify, living docs). v3, running in the platform pipeline, is out of scope.

## Notation
EARS: **WHEN** (event), **WHILE** (state), **WHERE** (optional feature), **IF … THEN** (unwanted behaviour), no prefix for ubiquitous requirements. One obligation (**SHALL** / **SHALL NOT**) per requirement. Ids are stable and never renumbered.

## Acceptance criteria
- A1. Every card fits its budget and is rendered from a template; there are no free-form docs.
- A2. Every claim is anchored, and every anchor resolves.
- A3. Nothing is written in the company repository, and the vault location is safe.
- A4. After a `git pull`, a digest shows the merged Azure DevOps PRs, one line each, and highlights the ones that concern the user.
- A5. An investigation in a session becomes a Q card with one command.
- A6. Flow, lineage and contract cards are built from extracted facts, and unknowns are shown as `?`.
- A7. PR cards compute contract changes and mark affected cards stale.
- A8. Cards flag themselves stale when anchored code changes, and refreshing them preserves human edits.
- A9. Card quality is measured (lint, verify score, editor changes).
- A10. Credentials never reach the model, the vault or logs.
- A11. The agent uses the cards as context.

## Configuration and vault
- **R1** [v0] The system SHALL read the vault location from the `vaultPath` plugin option.
- **R2** [v0] IF `vaultPath` is not set THEN the system SHALL stop without writing any file and tell the user how to set it.
- **R3** [v0] IF `vaultPath` is inside the working tree of the analyzed repository THEN the system SHALL refuse to write cards.
- **R4** [v0] IF `vaultPath` is inside a personal cloud sync folder (personal OneDrive, Dropbox, iCloud Drive, Google Drive) THEN the system SHALL warn the user once before the first write.
- **R5** [v0] IF `vaultPath` is inside a git working tree that has a remote THEN the system SHALL warn the user once before the first write.
- **R6** [v0] The system SHALL store each repository's cards under `<vaultPath>/<org>/<project>/<repo>/`, derived from the Azure Repos `origin` URL.
- **R7** [v0] IF the repository has no `origin` remote THEN the system SHALL use the repository folder name as its vault key.
- **R8** [v0] The system SHALL write card content in the `language` option (default `en`).
- **R9** [v0] The system SHALL NOT write files inside the analyzed repository.
- **R10** [v0] The CLI SHALL NOT make network calls other than the optional Azure DevOps REST calls.

## Card model
- **R11** [v0] The system SHALL support the card types `flow`, `lineage`, `contract`, `pr`, `digest`, `q` and `map`.
- **R12** [v0] The system SHALL validate every draft against the JSON schema of its card type.
- **R13** [v0] The system SHALL produce card markdown only through the deterministic template of its type.
- **R14** [v0] The system SHALL write front matter with `id`, `type`, `title`, `repo`, `source_commit`, `status`, `anchors` and `generated_at` in every card.
- **R15** [v0] The system SHALL derive a card's `id` from its type and a slug of its subject.
- **R16** [v0] IF a new card's `id` collides with an existing card for a different subject THEN the system SHALL append a numeric suffix.
- **R17** [v0] The system SHALL record every card in `_meta/cards.json` with its path, anchors, section hashes and status.

## Budgets and lint
- **R18** [v0] The system SHALL reject a draft that exceeds any budget of its card type.
- **R19** [v0] The system SHALL reject a draft that contains a phrase from the banned-phrases list.
- **R20** [v0] The system SHALL reject a draft that contains a code block longer than `maxCodeLines` lines.
- **R21** [v0] The system SHALL reject a draft in which a step, answer bullet or lineage row has no anchor.
- **R22** [v0] The system SHALL reject a draft that contains text matching the secret patterns.
- **R23** [v0] IF a draft is rejected THEN the system SHALL send the violations back to the drafter and request one corrected draft.
- **R24** [v0] IF the corrected draft is also rejected THEN the system SHALL NOT write the card.
- **R25** [v0] IF the corrected draft is also rejected THEN the system SHALL show the remaining violations to the user.

## Anchors
- **R26** [v0] The system SHALL support the anchor kinds `code`, `pr`, `workitem` and `human`.
- **R27** [v0] The system SHALL compute the hash of a `code` anchor over its referenced lines, normalized for line endings and whitespace.
- **R28** [v0] IF a `code` anchor points to a path or line range that does not exist at `source_commit` THEN the system SHALL reject the draft.
- **R29** [v0] IF a `code` anchor's lines no longer match its hash THEN the system SHALL search the file, following renames, for the hashed snippet and relocate the anchor when found.
- **R30** [v1] WHERE a `code` anchor has a `symbol` and facts are available, the system SHALL resolve the anchor by symbol before using its line range.
- **R31** [v0] The system SHALL render `human` anchors with a visible "human note" marker.
- **R32** [v0] The system SHALL render each `code` anchor as a link that opens the file at the anchored line in VS Code.

## Repository safety
- **R33** [v0] WHEN a command starts, the system SHALL snapshot the `git status` of the analyzed repository.
- **R34** [v0] IF the repository's `git status` differs from the snapshot when the command ends THEN the system SHALL warn the user and list the changed files.

## Q cards
- **R35** [v0] WHEN the user runs `/devcards:capture <question>`, the system SHALL produce a `q` card that answers the question from the current session's investigation.
- **R36** [v0] WHEN the user runs `/devcards:capture` without a question, the system SHALL propose a question and ask the user to confirm it before drafting.
- **R37** [v0] IF the current session has not read any file of the repository THEN the system SHALL NOT create a card.
- **R38** [v0] WHEN capturing, the system SHALL list existing `q` cards with a similar title and ask whether to update one of them instead.
- **R39** [v1] WHERE the captured investigation follows a request from an endpoint to downstream services, the system SHALL offer to create a `flow` card instead.

## PR cards
- **R40** [v0] WHEN the user runs `/devcards:pr <id>`, the system SHALL locate the commit whose message starts with `Merged PR <id>:`.
- **R41** [v0] IF no commit matches the PR id THEN the system SHALL ask the user for a commit range.
- **R42** [v0] WHEN the user runs `/devcards:pr <range>`, the system SHALL build the card from the diff of that range.
- **R43** [v0] The system SHALL fill the PR card's behaviour, config/flags and risk sections from the diff.
- **R44** [v0] The system SHALL list in the PR card the existing cards whose `code` anchors fall in changed lines.
- **R45** [v0] WHEN a PR card lists an affected card, the system SHALL mark that card `stale`.

## Digest
- **R46** [v0] WHEN the user runs `/devcards:digest` without `since`, the system SHALL include the PRs merged after the commit recorded by the previous digest.
- **R47** [v0] IF no previous digest exists THEN the system SHALL include the PRs merged in the last 7 days.
- **R48** [v0] The system SHALL write exactly one line per merged PR in the digest.
- **R49** [v0] The system SHALL group digest lines by the top-level folder with the most changed lines in each PR.
- **R50** [v0] The system SHALL list under "Needs your attention" the PRs that change files the user has authored or files anchored by existing cards.
- **R51** [v0] WHERE a PR card exists for a digest line, the system SHALL link the line to it.
- **R52** [v0] IF the local git history is shallow THEN the system SHALL warn that the digest may be incomplete.

## Azure DevOps REST (optional)
- **R53** [v0] WHEN the user runs `devcards ado login`, the system SHALL store the PAT in the OS credential store.
- **R54** [v0] WHERE a PAT is stored, the system SHALL add the PR description, reviewers and linked Azure Boards work items to PR cards.
- **R55** [v0] The system SHALL NOT call Azure DevOps endpoints that modify data.
- **R56** [v0] IF an Azure DevOps REST call fails THEN the system SHALL continue with git-only data and note it in the card.
- **R57** [v0] The system SHALL NOT expose the PAT to the model, the vault or any log.

## Fact extraction
- **R58** [v1] WHEN extracting facts, the system SHALL load the solution with Roslyn semantic analysis.
- **R59** [v1] IF the semantic load fails THEN the system SHALL fall back to syntax-only analysis and mark the resulting facts `inferred`.
- **R60** [v1] The system SHALL extract endpoints from controllers and minimal API mappings.
- **R61** [v1] The system SHALL extract request and response DTO shapes, including nullability.
- **R62** [v1] The system SHALL extract outbound calls made through typed `HttpClient` and Refit clients.
- **R63** [v1] The system SHALL extract the resilience policies attached to each outbound client.
- **R64** [v1] The system SHALL extract property assignments in manual mappers and in AutoMapper and Mapster profiles.
- **R65** [v1] The system SHALL extract configuration keys bound through options classes and `IConfiguration`.
- **R66** [v1] The system SHALL cache facts per commit in `_facts/facts@<commit>.json`.
- **R67** [v1] IF cached facts exist for the current commit THEN the system SHALL reuse them.
- **R68** [v1] The system SHALL represent any fact it cannot resolve as `?`.

## Flow and lineage
- **R69** [v1] WHEN the user runs `/devcards:flow <route|symbol>`, the system SHALL produce a `flow` card and a `lineage` card for that entry point.
- **R70** [v1] IF the argument matches no extracted endpoint THEN the system SHALL list the closest endpoints and write nothing.
- **R71** [v1] The system SHALL generate the flow's sequence diagram from the extracted outbound calls.
- **R72** [v1] The system SHALL fill the failure-modes table from the extracted resilience policies and the downstream error-handling code.
- **R73** [v1] The system SHALL produce one lineage row per response field.
- **R74** [v1] IF a response field's source cannot be traced THEN the system SHALL show its source as `?`.
- **R75** [v1] WHERE a field is mapped by naming convention, the system SHALL mark its transform as `convention`.

## Contract
- **R76** [v1] WHEN the user runs `/devcards:contract <route>`, the system SHALL produce a `contract` card from the extracted facts.
- **R77** [v1] WHERE an OpenAPI document exists in the repository, the system SHALL use it to complement the extracted facts.
- **R78** [v1] The system SHALL list each response status code with the condition that produces it.
- **R79** [v1] WHEN building a PR card, the system SHALL compute contract changes by diffing the facts of the base and head commits.
- **R80** [v1] The system SHALL classify each contract change as `additive` or `breaking`.

## Module map
- **R81** [v0] WHEN the user runs `/devcards:map`, the system SHALL produce a `map` card with one line per project in the solution.
- **R82** [v0] The system SHALL link each map line to the cards anchored in that project.

## Editor
- **R83** [v2] WHEN a draft passes lint, the system SHALL run the editor pass before rendering.
- **R84** [v2] The system SHALL run the editor without file-writing tools.
- **R85** [v2] IF the edited draft fails lint THEN the system SHALL render the pre-edit draft.

## Verify
- **R86** [v2] WHERE verification is requested, the system SHALL generate questions for the card from fact templates.
- **R87** [v2] WHERE verification is requested, the system SHALL have the reader answer using only the card content.
- **R88** [v2] The system SHALL score each answer by checking the expected facts of its question.
- **R89** [v2] The system SHALL store the score in the card's `verify_score`.
- **R90** [v2] The system SHALL report the card sections that no answer cited.

## Staleness and context
- **R91** [v2] WHEN a Claude Code session starts in a repository with cards, the system SHALL check every `code` anchor against `HEAD`.
- **R92** [v2] WHEN the check finds an anchor that cannot be resolved or relocated, the system SHALL mark its card `stale`.
- **R93** [v2] WHEN the check finishes, the system SHALL print the number of stale cards.
- **R94** [v2] IF the session-start check exceeds 3 seconds THEN the system SHALL stop it without blocking the session.
- **R95** [v2] WHEN a session starts in a repository with cards, the system SHALL inject the repository's card index into the session context within `indexTokenBudget` tokens.
- **R96** [v2] WHEN the user runs `/devcards:refresh`, the system SHALL regenerate only the machine-owned sections of stale cards.

## Ownership and learning
- **R97** [v0] The system SHALL NOT overwrite human-owned sections.
- **R98** [v0] The system SHALL treat the `## Notes` section of every card as human-owned.
- **R99** [v2] WHEN a machine-owned section differs from its last rendered content, the system SHALL make that section human-owned.
- **R100** [v2] WHEN a section becomes human-owned through an edit, the system SHALL save its generated and edited versions as a style example for the card type.
- **R101** [v2] The system SHALL give the writer at most `maxStyleExamples` style examples of the card type being drafted.
- **R102** [v0] The system SHALL resolve cards by the `id` in their front matter, even if the file was moved or renamed in the vault.

## Ask
- **R103** [v2] WHEN the user runs `/devcards:ask <question>`, the system SHALL answer from the repository's cards before reading code.
- **R104** [v2] IF answering required reading code THEN the system SHALL offer to capture the answer as a `q` card.

## Observability
- **R105** [v0] The system SHALL append every run to `_meta/runs.jsonl` with the command, card ids, lint violations, whether the editor changed the draft, and the verify score.
- **R106** [v0] The system SHALL NOT write diff contents or credentials to `_meta/runs.jsonl`.

## Acceptance criteria coverage
| Criterion | Covered by |
|---|---|
| A1 budget + template | R12, R13, R18–R20, R22–R25 |
| A2 anchored claims | R21, R26–R32 |
| A3 nothing in repo, safe vault | R1–R7, R9, R33, R34 |
| A4 Azure DevOps digest | R46–R52 |
| A5 capture | R35–R38 |
| A6 facts-based flow, lineage, contract | R58–R78 |
| A7 PR cards | R40–R45, R79, R80 |
| A8 living docs | R91, R92, R96–R100, R102 |
| A9 measured quality | R83–R90, R105 |
| A10 credentials safe | R10, R53–R57, R106 |
| A11 agent uses cards | R95, R103, R104 |
| Model and options | R8, R11, R14–R17, R39, R81, R82, R93, R94, R101 |
