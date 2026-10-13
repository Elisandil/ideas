# contract-guard — Requirements

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Context
Capability 2 of [ai-platform-bff.md](../../ai-platform-bff.md), designed in [contract-guard.md](../../contract-guard.md). It detects breaking changes in the BFF's OpenAPI contracts on every Azure DevOps PR, and explains the impact for each consumer. It is designed with SDD: requirements (EARS) → design → tasks, plus a risk document. It must be approved before any code is written.

Decisions taken:
- **Azure DevOps Build Validation**, Optional at first. It runs on the same self-hosted pool and protected variable group as [ai-pr-reviewer](../ai-pr-reviewer/design.md#azure-devops-infrastructure).
- **The diff is deterministic**: oasdiff (pinned version) decides what changed and how severe it is. Claude only explains the impact and proposes fixes; it never adds, removes or reclassifies changes.
- **The spec is a committed snapshot** (`contracts/bff.openapi.json`) generated at build time. The base is the snapshot at the merge-base, and the head is the spec generated from the PR's source commit.
- **Consumers come from evidence**: a registry (`contracts/consumers.yaml`), and later Pact contracts and App Insights telemetry. With no evidence it says "unknown consumers".
- **Two directions**: outbound (the BFF spec changes → consumers) and inbound (a downstream spec changes → BFF code).
- **Advisory by default**: comments only. Failing the job on unacknowledged `ERR` changes is an explicit opt-in (`failOnErr`).
- **Phased delivery**: every requirement is tagged **[v0]** (oasdiff only, no LLM), **[v1]** (consumer registry, Claude explanations, acknowledgments) or **[v2]** (telemetry and Pact evidence, inbound direction, gate). v3, moving into the platform, is out of scope.

## Notation
EARS: **WHEN** (event), **WHILE** (state), **WHERE** (optional feature), **IF … THEN** (unwanted behaviour), no prefix for ubiquitous requirements. One obligation (**SHALL** / **SHALL NOT**) per requirement. Ids are stable and never renumbered.

## Acceptance criteria
- A1. Every PR that changes a contract gets one summary with the breaking and potentially breaking changes, with no human intervention.
- A2. PRs that don't change a contract get no comment and use no LLM quota.
- A3. Severity always comes from oasdiff; the model cannot hide, add or reclassify a change.
- A4. Every consumer named as affected comes with evidence and its source.
- A5. Intentional breaks can be acknowledged in a versioned file, with a reason, notified consumers and an expiry.
- A6. Changes to downstream specs show which BFF code uses what changed.
- A7. The job never blocks a PR unless `failOnErr` is enabled.
- A8. Neither a PR author nor PR content (including spec descriptions) can extract secrets, hide consumers or execute code on the agent.
- A9. Spec generation failures, quota or credential errors and transient failures end in a controlled way and are explained.
- A10. Every run leaves an auditable report with no sensitive content.

## Activation and scope
- **R1** [v0] WHEN a PR is created or updated against a branch with the validation policy configured, the system SHALL run.
- **R2** [v0] WHERE the PR has the `no-contract-guard` label, the system SHALL skip the run.
- **R3** [v0] IF the PR comes from a fork THEN the system SHALL exit without accessing secrets.
- **R4** [v0] WHEN the run starts, the system SHALL generate the BFF OpenAPI document from the PR source commit at build time.
- **R5** [v0] IF the OpenAPI document cannot be generated THEN the system SHALL post a summary stating the generation failure without running the diff.
- **R6** [v0] The system SHALL take the base spec from `snapshotPath` at the merge-base between the target branch and the PR source commit.
- **R7** [v0] The system SHALL take the head spec from the document generated in R4.
- **R8** [v0] IF the generated document is not semantically equal to the `snapshotPath` committed in the source commit THEN the system SHALL post a "snapshot out of date" finding with the regeneration command.
- **R9** [v0] IF neither the BFF spec nor any file matching `downstreamSpecGlobs` changed against the base AND no previous summary thread exists THEN the system SHALL exit without posting.
- **R10** [v0] IF neither the BFF spec nor any downstream spec changed against the base AND a previous summary thread exists THEN the system SHALL update it to "no contract changes".
- **R11** [v0] IF the base snapshot does not exist at the merge-base THEN the system SHALL treat the run as "first snapshot" and post only the list of operations, without a diff.

## Diff
- **R12** [v0] The system SHALL run oasdiff at the `oasdiffVersion` version.
- **R13** [v0] WHEN it installs oasdiff, the system SHALL verify the binary checksum against `oasdiffSha256` before running it.
- **R14** [v0] The system SHALL take each change's level (`ERR`, `WARN`, `INFO`) exclusively from oasdiff's output.
- **R15** [v0] The system SHALL apply the oasdiff ignore files `contracts/oasdiff-ignore-err.txt` and `contracts/oasdiff-ignore-warn.txt` read from the target branch.
- **R16** [v0] The system SHALL NOT read ignore files, the consumer registry or configuration from the PR source branch.
- **R17** [v0] The system SHALL compute a stable change id from the oasdiff rule id, the HTTP method, the path and the affected element.
- **R18** [v0] The system SHALL post `INFO` changes only in the summary's changelog section.

## Consumer mapping
- **R19** [v1] The system SHALL read the consumer registry from `contracts/consumers.yaml` in the target branch.
- **R20** [v1] IF the registry does not validate against `consumers.schema.json` THEN the system SHALL ignore it and state it in the summary.
- **R21** [v1] The system SHALL match a change to a consumer WHEN the consumer lists the change's operation and either lists no fields for it or lists a field affected by the change.
- **R22** [v1] IF an `ERR` or `WARN` change matches no consumer THEN the system SHALL mark it "unknown consumers".
- **R23** [v1] The system SHALL attach to each match at least one evidence item with its source (`registry`, `pact`, `telemetry` or `code`).
- **R24** [v2] WHERE `telemetry.enabled` is true, the system SHALL query App Insights for calls per affected operation and client id over the last `telemetry.days` days.
- **R25** [v2] WHERE telemetry shows calls from a client id that is not in the registry, the system SHALL report it as an unregistered consumer.
- **R26** [v2] WHERE `pact.source` is configured, the system SHALL read the consumer pacts for the BFF and match the fields each consumer reads.
- **R27** [v2] The system SHALL NOT expose the telemetry or Pact credentials in Claude's environment.

## Inbound changes
- **R28** [v2] WHEN a file matching `downstreamSpecGlobs` changes against the base, the system SHALL diff it with the rules of R12–R18.
- **R29** [v2] For each inbound `ERR` or `WARN` change, the system SHALL search the BFF source for references to the affected operation and fields, using `inboundSearch` rules.
- **R30** [v2] The system SHALL attach the references found in R29 as `code` evidence with `file:line`.
- **R31** [v2] IF an inbound change has no references in the BFF source THEN the system SHALL mark it "not used by the BFF".

## Acknowledgments
- **R32** [v1] The system SHALL read acknowledgments from `contracts/accepted-breaks.yaml` in the PR source commit.
- **R33** [v1] WHEN an acknowledgment's change id matches a detected change, the system SHALL list the change as "acknowledged" instead of as a finding.
- **R34** [v1] IF an acknowledgment's `expires` date has passed THEN the system SHALL treat the change as unacknowledged.
- **R35** [v1] IF an acknowledgment lacks `reason`, `consumers_notified` or `expires` THEN the system SHALL ignore it and state it in the summary.
- **R36** [v1] WHEN an acknowledgment is added or changed in the PR itself, the system SHALL mark it "acknowledged in this PR" in the summary.

## Model execution
- **R37** [v1] The system SHALL invoke Claude only WHEN at least one `ERR` or `WARN` change remains after ignores and acknowledgments.
- **R38** [v1] The system SHALL apply the credential and isolation requirements R15–R27 of [ai-pr-reviewer](../ai-pr-reviewer/requirements.md#credentials-and-isolation) to every Claude invocation.
- **R39** [v1] WHERE `plan` is `consumer` and the PR author is not `tokenOwnerUpn`, the system SHALL post the deterministic summary without invoking Claude.
- **R40** [v1] The system SHALL pass Claude only the changes, the affected operations and schemas, the consumer matches and their evidence.
- **R41** [v1] IF the number of `ERR` and `WARN` changes exceeds `maxExplainedChanges` THEN the system SHALL explain the first `maxExplainedChanges` (ERR first) and list the rest without explanation.
- **R42** [v1] The system SHALL delimit spec descriptions, examples and PR content as untrusted data within the prompt.
- **R43** [v1] The system SHALL request output conforming to `impact.schema.json` via `--json-schema`.
- **R44** [v1] IF an explanation references a change id that was not in the input THEN the system SHALL discard it.
- **R45** [v1] IF an explanation names a consumer that is not among that change's matches THEN the system SHALL discard that consumer entry.
- **R46** [v1] IF the output does not validate against the schema THEN the system SHALL retry once.
- **R47** [v1] IF Claude fails (invalid schema twice, usage limit, authentication or timeout) THEN the system SHALL post the deterministic summary and state the cause.

## Publishing to the PR
- **R48** [v0] The system SHALL keep a single summary thread per PR and update it on every run.
- **R49** [v0] WHERE the snapshot is updated in the PR and an `ERR` or `WARN` change can be located in it, the system SHALL post an inline thread on that line of the snapshot.
- **R50** [v0] The system SHALL compute a fingerprint per thread from the change id.
- **R51** [v0] IF a contract-guard thread with the same fingerprint exists, in any status, THEN the system SHALL NOT create another one.
- **R52** [v0] The system SHALL mark each of its threads with the `ContractGuard.Fingerprint` property and a visible `🤖 contract-guard` prefix.
- **R53** [v0] IF, before publishing, there is a newer iteration than the analysed one THEN the system SHALL discard the results without publishing them.
- **R54** [v0] IF the PR is completed or abandoned at publishing time THEN the system SHALL exit without publishing.
- **R55** [v0] IF the Azure DevOps API returns 429 or 5xx THEN the system SHALL retry with exponential backoff up to 3 times.
- **R56** [v0] The system SHALL NOT vote, approve, reject or post PR statuses.
- **R57** [v0] The system SHALL NOT modify threads it did not create.
- **R58** [v1] The system SHALL apply the output sanitization R55–R59 of [ai-pr-reviewer](../ai-pr-reviewer/requirements.md#output-sanitization) to every generated text.
- **R59** [v0] The system SHALL write comments in the `language` language (default `en`).

## Outcome and observability
- **R60** [v0] The system SHALL finish the job as `Succeeded` or `SucceededWithIssues` except as allowed by R61.
- **R61** [v2] WHERE `failOnErr` is true, IF at least one unacknowledged outbound `ERR` change exists THEN the system SHALL finish the job as `Failed`.
- **R62** [v0] IF the run does not complete because of an error THEN the system SHALL finish as `SucceededWithIssues` with a warning stating the cause.
- **R63** [v0] The system SHALL publish the `contract-guard-report.json` artifact with duration, change ids and levels, matches, acknowledgments, whether Claude was invoked, reported usage and outcome.
- **R64** [v0] The system SHALL NOT write spec bodies, telemetry rows or secret values to logs or artifacts.

## Acceptance criteria coverage
| Criterion | Covered by |
|---|---|
| A1 summary on contract change | R1, R4, R6, R7, R12, R14, R48, R49 |
| A2 silence and zero quota otherwise | R9, R10, R37 |
| A3 severity from oasdiff | R14, R15, R44, R45 |
| A4 evidence per consumer | R19–R27 |
| A5 acknowledgments | R32–R36 |
| A6 inbound | R28–R31 |
| A7 advisory by default | R56, R57, R60, R61 |
| A8 no extraction or tampering | R3, R13, R16, R27, R38, R42, R58 |
| A9 controlled failures | R5, R8, R11, R39, R41, R46, R47, R53–R55, R62 |
| A10 auditable | R63, R64 |
| Options | R2, R17, R18, R50–R52, R59 |
