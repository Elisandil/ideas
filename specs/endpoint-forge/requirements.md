# endpoint-forge — Requirements

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Context
Capability 12 of [ai-platform-bff.md](../../ai-platform-bff.md), designed in [endpoint-forge.md](../../endpoint-forge.md). It is a local Claude Code plugin that generates a new BFF endpoint for a C# BFF hosted in **Azure DevOps**: the endpoint, DTOs, mappers, validation and tests. The input is a description, one or more downstream OpenAPI operations, or both. The output follows the team's architecture profile. It is designed with SDD: requirements (EARS) → design → tasks, plus a risk document. It must be approved before any code is written.

Decisions taken:
- **Local Claude Code plugin**: the developer runs it in their own clone. The result is a local branch that they review and push like any other change. No pipeline in v0–v2.
- **Architecture as data**: a versioned profile (`.endpoint-forge/profile.yaml`) plus exemplar endpoints from the same repository. [v2] `init` proposes the profile from Roslyn facts.
- **Plan before code**: an endpoint plan (contract, field lineage, downstream calls, error mapping, validation, test scenarios) is checked deterministically and approved by the developer before any file is written.
- **Deterministic structure, model-written bodies**: the repository's own client generator produces the downstream DTOs. [v1] BFF DTOs, validators, direct mappings, registration lines, WireMock stubs and test skeletons are rendered from the approved plan. The model writes the composition logic, the expressions and the test bodies.
- **Contained and verified**: the tool writes only the files in the run manifest, on a local branch. It never pushes and never adds packages. The result must build, pass the formatter and pass the generated tests, and [v1] the BFF contract may only gain the new operation.
- **Phased delivery**: every requirement is tagged **[v0]** (the model writes all the code following the exemplars; build, format and test gate), **[v1]** (deterministic scaffold, client generation, full plan check, additive contract check, contract-first mode) or **[v2]** (Roslyn `init` and profile check, lineage verification, architecture tests, learning from edits). v3, the pipeline mode, is out of scope.

## Notation
EARS: **WHEN** (event), **WHILE** (state), **WHERE** (optional feature), **IF … THEN** (unwanted behaviour), no prefix for ubiquitous requirements. One obligation (**SHALL** / **SHALL NOT**) per requirement. Ids are stable and never renumbered.

## Acceptance criteria
- A1. No file in the repository is written before the developer approves the endpoint plan.
- A2. Every response field traces to a field of a provided downstream operation, a request parameter, a declared expression or constant, or an explicit `todo`. No downstream field is invented.
- A3. The generated code follows the architecture profile: file locations, names, libraries and registration points.
- A4. The generated code builds, passes the formatter and passes the generated tests.
- A5. Every downstream call has tests for success and for each failure the plan maps, and every validation rule has a test.
- A6. Only files in the run manifest change, on a local branch. Nothing is pushed and no package reference changes.
- A7. [v1] The BFF OpenAPI contract changes only by the new operation.
- A8. Neither the description nor the downstream spec content can make the tool write outside the manifest, run arbitrary code or reach the network.
- A9. Invalid inputs, failing checks, quota and credential errors end in a controlled way, with the branch in a known state and a report.
- A10. Every run leaves an auditable record with no source code or spec content.

## Configuration and profile
- **R1** [v0] The system SHALL read the architecture profile from the `profilePath` option (default `.endpoint-forge/profile.yaml`, relative to the repository root).
- **R2** [v0] IF the profile does not exist THEN the system SHALL stop without writing any file and show how to create it.
- **R3** [v0] IF the profile does not validate against `profile.schema.json` THEN the system SHALL stop and list the violations.
- **R4** [v0] IF an exemplar listed in the profile does not resolve to an existing endpoint THEN the system SHALL stop and name the exemplar.
- **R5** [v0] The system SHALL derive every path in the run manifest from the profile's `layout`, `registration`, `clients` and `openApi` entries.
- **R6** [v0] The system SHALL write the rendered plan and the run report in the `language` option (default `en`).

## Inputs
- **R7** [v0] The system SHALL accept a free-text description of the endpoint.
- **R8** [v0] The system SHALL accept zero or more downstream operations, each as `<specPath>#<operationId>` or `<specPath>#<METHOD> <path>`.
- **R9** [v0] IF neither a description nor a downstream operation is given THEN the system SHALL stop and ask for one.
- **R10** [v0] IF a downstream spec file is outside the repository THEN the system SHALL reject it.
- **R11** [v0] IF a referenced operation does not exist in its spec THEN the system SHALL list the closest operations and write nothing.
- **R12** [v0] The system SHALL pass the planner only the selected operations and the closure of the schemas they reference.
- **R13** [v0] IF the selected operations still exceed `maxSpecTokens` after their examples are removed THEN the system SHALL stop and ask the user to select fewer operations.
- **R14** [v0] The system SHALL delimit the description and the spec descriptions and examples as untrusted data within every prompt.

## Repository safety
- **R15** [v0] IF the working tree has uncommitted changes when `/endpoint-forge:new` starts THEN the system SHALL stop without writing.
- **R16** [v0] WHEN generation starts, the system SHALL create and check out a local branch named after `branchPattern` (default `forge/{feature}-{name}`) from the current `HEAD`.
- **R17** [v0] IF the branch name already exists THEN the system SHALL append a numeric suffix.
- **R18** [v0] The system SHALL record in the run manifest every file it will create or edit before writing any of them.
- **R19** [v0] The system SHALL edit existing files only at the profile's registration anchors, inside the generated-client folder (R51) and in the OpenAPI snapshot (R69).
- **R20** [v0] IF `git status` at the end of the run lists a changed file outside the manifest THEN the system SHALL fail verification and list the file.
- **R21** [v0] The system SHALL NOT add, remove or change package references or central package versions.
- **R22** [v0] The system SHALL NOT push, create pull requests or call Azure DevOps.
- **R23** [v0] WHERE `commit` is true, the system SHALL commit the generated files to the run branch after verification passes.

## Plan
- **R24** [v0] WHEN the inputs are resolved, the system SHALL have the planner produce a `plan.json` that conforms to `plan.schema.json`.
- **R25** [v0] The plan check SHALL reject a response field without a `source` of kind `downstream`, `expr`, `request`, `constant` or `todo`.
- **R26** [v0] IF a source references a call, operation or JSON path that is not among the resolved operations THEN the plan check SHALL reject the plan.
- **R27** [v1] IF a downstream source's type is not compatible with the type of its BFF field THEN the plan check SHALL reject the plan.
- **R28** [v1] IF a non-nullable BFF field has a downstream source that is optional or nullable and declares no default THEN the plan check SHALL reject the plan.
- **R29** [v0] IF the plan's method and route collide with an existing endpoint THEN the plan check SHALL reject the plan.
- **R30** [v0] IF the plan's calls contain a dependency cycle or reference an unknown call THEN the plan check SHALL reject the plan.
- **R31** [v0] WHERE a call lacks an error mapping for `notFound`, `clientError`, `serverError` or `timeout`, the plan check SHALL fill it from the profile's `errorMapping`.
- **R32** [v0] IF an error mapping degrades to a response with null fields and one of those fields is non-nullable THEN the plan check SHALL reject the plan.
- **R33** [v0] IF the scenarios do not cover the success and every mapped error of each call, and every validation rule, THEN the plan check SHALL reject the plan.
- **R34** [v0] IF a name in the plan does not match the profile's `naming` patterns THEN the plan check SHALL reject the plan.
- **R35** [v0] The plan check SHALL reject a plan in which a field sourced as `todo` is not nullable.
- **R36** [v0] IF the plan check rejects the plan THEN the system SHALL send the violations to the planner and request one corrected plan.
- **R37** [v0] IF the corrected plan is also rejected THEN the system SHALL stop without writing and show the violations.

## Approval
- **R38** [v0] WHEN the plan passes the check, the system SHALL render `plan.md` with the contract, the lineage table, the call diagram, the error table, the validation rules and the scenarios.
- **R39** [v0] WHERE the plan has fields sourced as `todo`, the system SHALL list them at the top of `plan.md`.
- **R40** [v0] The system SHALL NOT write any file in the repository before the developer approves the plan.
- **R41** [v0] The system SHALL record each approval with the SHA-256 of `plan.json`.
- **R42** [v0] IF `plan.json` changes after its approval THEN the system SHALL re-run the plan check and require a new approval.
- **R43** [v0] WHEN the user runs `/endpoint-forge:plan`, the system SHALL stop after rendering the plan.
- **R44** [v0] WHEN `/endpoint-forge:new` receives a plan file whose hash has a recorded approval, the system SHALL generate from it without re-planning.

## Scaffold
- **R45** [v1] WHEN generation starts from an approved plan, the system SHALL render the request and response DTOs from the plan's fields, types and nullability.
- **R46** [v1] The system SHALL render validators from the plan's validation rules, using the profile's validation library.
- **R47** [v1] The system SHALL render the mapping assignments of fields whose source is `request`, or `downstream` without an expression.
- **R48** [v1] The system SHALL insert the registration lines at the profile's registration anchors.
- **R49** [v1] The system SHALL render WireMock.NET stubs for each call and scenario from the downstream response schemas.
- **R50** [v1] The system SHALL render one test method skeleton per plan scenario, marked with the scenario id.
- **R51** [v1] IF a selected operation is missing from the repository's generated client THEN the system SHALL run the profile's client generation command for that spec.
- **R52** [v1] IF the client generation command is not `dotnet` followed by a tool listed in `.config/dotnet-tools.json` THEN the system SHALL refuse to run it.
- **R53** [v1] IF client generation changes files outside the profile's `clients.generatedPath` THEN the system SHALL revert the generation and stop.
- **R54** [v1] IF a rendered DTO or validator differs from its rendered content after the implementer runs THEN the system SHALL fail verification.

## Implementation
- **R55** [v0] The system SHALL run the planner with only the `Read`, `Grep`, `Glob` and `Write` tools.
- **R56** [v0] The system SHALL run the implementer with only the `Read`, `Grep`, `Glob`, `Edit` and `Write` tools.
- **R57** [v0] WHILE a run is in the plan, fill or repair phase, the system SHALL block every file write outside the phase's allowed paths.
- **R58** [v0] The system SHALL give the implementer the approved plan, the profile, the exemplar files and, [v1] once it exists, the scaffold.

## Verification
- **R59** [v0] IF a generated file calls an API listed in `bannedApis` THEN the system SHALL fail verification without building or running tests.
- **R60** [v0] WHEN the implementer finishes, the system SHALL build the profile's solution with `dotnet build`.
- **R61** [v0] The system SHALL run `dotnet format --verify-no-changes` on the manifest's C# files.
- **R62** [v0] The system SHALL run the generated test classes with `dotnet test --filter`.
- **R63** [v0] IF a plan scenario has no generated test marked with its id THEN the system SHALL fail verification.
- **R64** [v0] IF a generated test is skipped or contains no assertion THEN the system SHALL fail verification.
- **R65** [v0] IF the mapping of a field sourced as `todo` lacks a `TODO(endpoint-forge)` comment THEN the system SHALL fail verification.
- **R66** [v0] IF the change touches a package reference or a central package version THEN the system SHALL fail verification.
- **R67** [v1] WHERE `openApi.snapshotPath` is set, the system SHALL generate the BFF OpenAPI document and diff it against the snapshot with oasdiff.
- **R68** [v1] IF the OpenAPI diff contains a breaking change or a change to any operation other than the new one THEN the system SHALL fail verification.
- **R69** [v1] WHEN the OpenAPI diff passes, the system SHALL update the snapshot with the normalized document.
- **R70** [v2] WHERE `architectureTests` is set, the system SHALL run that test project.
- **R71** [v2] The system SHALL extract the mapper's property assignments with Roslyn and compare them with the plan lineage.
- **R72** [v2] IF an extracted assignment contradicts the plan lineage THEN the system SHALL fail verification.
- **R73** [v0] The system SHALL write the result of every check in every attempt to `verify.json`.

## Repair
- **R74** [v0] IF verification fails for a reason other than an environment error (package restore, missing SDK, locked files) THEN the system SHALL send `verify.json` to the implementer and request a fix, up to `maxRepairAttempts` times.
- **R75** [v0] The system SHALL run every verification check after each repair attempt, not only the checks that failed.
- **R76** [v0] IF verification still fails after `maxRepairAttempts` attempts THEN the system SHALL leave the changes on the run branch and report the failing checks.
- **R77** [v0] IF Claude returns a usage-limit or authentication error, or verification fails with an environment error, THEN the system SHALL stop and report the cause, leaving the run branch as it is.
- **R78** [v0] WHEN the user runs `/endpoint-forge:repair` on a run branch, the system SHALL re-run verification and the repair loop for that run.

## Contract-first mode
- **R79** [v1] WHERE no downstream operation is given, the system SHALL plan the BFF contract from the description alone.
- **R80** [v1] WHERE no downstream operation is given, the system SHALL represent each downstream dependency as a port interface named in the plan.
- **R81** [v1] WHERE no downstream operation is given, the plan check SHALL reject any field sourced as `downstream`.
- **R82** [v1] The system SHALL mark contract-first runs as such in `plan.md` and in the run report.

## Init and profile check
- **R83** [v2] WHEN the user runs `/endpoint-forge:init`, the system SHALL extract facts from the solution with Roslyn.
- **R84** [v2] The system SHALL propose a profile with the dominant style, layout, libraries and conventions found in the facts.
- **R85** [v2] WHERE no pattern reaches `dominanceThreshold` for a profile field, the system SHALL leave that field as `?` and ask the user.
- **R86** [v2] The system SHALL propose as exemplars the `maxExemplars` endpoints that best match the proposed profile.
- **R87** [v2] The system SHALL write the proposed profile only after the user confirms it.
- **R88** [v2] WHEN the user runs `/endpoint-forge:check-profile`, the system SHALL report the endpoints that do not match the profile's layout or naming.
- **R89** [v2] WHEN `/endpoint-forge:check-profile` runs, the system SHALL report the exemplars that fail the architecture tests or no longer match the profile.

## Learning from edits
- **R90** [v2] WHEN `/endpoint-forge:check-profile` runs, the system SHALL compute, for each recorded run whose files exist at `HEAD`, the share of generated lines that survived.
- **R91** [v2] The system SHALL give the implementer at most `maxEditExamples` pairs of generated and current files of the same layout kind from earlier runs.

## Observability
- **R92** [v0] The system SHALL keep each run's plan, manifest, verification results and report in `<runsPath>/<repo>/<runId>/`, outside the repository.
- **R93** [v0] The system SHALL append every run to `<runsPath>/runs.jsonl` with the command, mode, plan hash, manifest paths, check outcomes, repair attempts, duration and outcome.
- **R94** [v0] The system SHALL NOT write source code, spec content or the description to `runs.jsonl`.
- **R95** [v0] WHEN a run ends, the system SHALL show a report with the branch, the manifest, the check results and the fields sourced as `todo`.

## Acceptance criteria coverage
| Criterion | Covered by |
|---|---|
| A1 no code before approval | R15–R18, R38, R40–R44 |
| A2 no invented fields | R12, R24–R28, R30, R35, R39, R65, R71, R72, R79–R82 |
| A3 follows the profile | R1–R5, R19, R29, R34, R45–R48, R51, R54, R58, R70, R83–R89 |
| A4 build, format, tests | R60–R62, R73–R76, R78 |
| A5 failure coverage | R31–R33, R49, R50, R63, R64 |
| A6 contained | R18–R22, R53, R57, R66 |
| A7 additive contract | R67–R69 |
| A8 no escape | R10, R14, R52, R55–R57, R59 |
| A9 controlled failures | R2–R4, R9, R11, R13, R36, R37, R76, R77 |
| A10 auditable | R92–R95 |
| Inputs and options | R6–R8, R23, R90, R91 |
