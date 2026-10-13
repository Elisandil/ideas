# contract-guard — Design

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Architecture
```
Azure Repos (BFF repo)                          contract-guard repo (tag vX.Y.Z)
 Branch policy: Build validation (Optional)      templates/contract-guard.yml (extends)
   └─► pipelines/contract-guard.yml ──extends──► src/ContractGuard.Cli (.NET 9)
                                                 prompts/, schemas/, tools/oasdiff.lock
Self-hosted agent (ai-reviewer-pool)
 1. checkout PR (source commit) + fetch target           ← git
 2. dotnet build → generated openapi.json                ← BFF build, no tests
 3. ContractGuard.Cli run                                ← deterministic logic
      ├─ PolicyGuard / Preflight
      ├─ SpecProvider (base @ merge-base, head = generated, snapshot check)
      ├─ OasdiffRunner ──► oasdiff (pinned, checksum) ──► changes
      ├─ AckStore + IgnoreRules
      ├─ ConsumerMatcher ◄── consumers.yaml · [v2] Pact · App Insights
      ├─ [v2] InboundScanner ──► references in BFF code
      ├─ [v1] ClaudeRunner ──► claude -p (read-only) ──► Anthropic (subscription)
      ├─ ImpactValidator + OutputSanitizer
      └─ CommentPublisher ──► Azure DevOps REST (System.AccessToken)
 4. publish contract-guard-report.json
```
Core principle, as in ai-pr-reviewer: **the model only reads and returns JSON. oasdiff decides what is breaking, deterministic code decides who is affected, and deterministic code writes to Azure DevOps with a token the model never sees.**

## Shared components with ai-pr-reviewer
Both tools need the same guards and plumbing. They are extracted into an `AiTools.Common` project in the ai-pr-reviewer repo, published to the Azure Artifacts feed and consumed by both CLIs. Whichever tool is built first creates it.

| Component | Used here for |
|---|---|
| `Preflight` | Claude version and `authMethod = oauth_token` (R38) |
| `ClaudeRunner` + `ClaudeResultClassifier` | Allowlisted environment, read-only flags, timeout, quota/auth classification (R38, R43, R46, R47) |
| `WorkspaceSanitizer` | Agent configuration files and external symlinks (R38) |
| `OutputSanitizer` | Secrets, images/HTML, links, mentions, truncation (R58) |
| `AdoClient` | PR, iterations, labels, threads, retries (R53–R55) |
| `PipelineContext` | Azure Pipelines predefined variables |

## Files
**New `contract-guard` repo:**
| Path | Contents |
|---|---|
| `templates/contract-guard.yml` | `extends` template: pool, checkout, variable group, BFF build, oasdiff install, CLI invocation, artifact (R3, R4, R12, R13, R60–R62) |
| `tools/oasdiff.lock` | `oasdiffVersion`, download URL and `oasdiffSha256` per OS (R12, R13) |
| `prompts/system.md` | Role, output format, "never change levels or invent consumers", untrusted data rule (R42, R59) |
| `schemas/impact.schema.json` | Claude output (R43) |
| `schemas/consumers.schema.json` | Registry (R20) |
| `schemas/accepted-breaks.schema.json` | Acknowledgments (R35) |
| `src/ContractGuard.Cli/Program.cs` | `run` and `snapshot` commands |
| `src/ContractGuard.Cli/Config/GuardOptions.cs` | Options with defaults and validation |
| `src/ContractGuard.Cli/Guards/PolicyGuard.cs` | Label, fork, consumer plan (R2, R3, R39) |
| `src/ContractGuard.Cli/Specs/SpecProvider.cs` | Base, head, snapshot equality, first snapshot (R4–R11) |
| `src/ContractGuard.Cli/Specs/SpecNormalizer.cs` | Canonical JSON for semantic equality and stable line mapping (R8, R49) |
| `src/ContractGuard.Cli/Diff/OasdiffRunner.cs` | Process, arguments, JSON parsing (R12, R14, R15, R18) |
| `src/ContractGuard.Cli/Diff/ChangeId.cs` | Stable change id (R17) |
| `src/ContractGuard.Cli/Acks/AckStore.cs` | Reading, validation, expiry, "in this PR" (R32–R36) |
| `src/ContractGuard.Cli/Consumers/ConsumerMatcher.cs` | Registry matching and evidence (R19–R23) |
| `src/ContractGuard.Cli/Consumers/TelemetrySource.cs` | [v2] App Insights KQL (R24, R25, R27) |
| `src/ContractGuard.Cli/Consumers/PactSource.cs` | [v2] Pact files or broker (R26, R27) |
| `src/ContractGuard.Cli/Inbound/InboundScanner.cs` | [v2] References to operations and fields in BFF code (R28–R31) |
| `src/ContractGuard.Cli/Claude/ImpactPromptBuilder.cs` | Payload, cap, untrusted delimiters (R40–R42) |
| `src/ContractGuard.Cli/Claude/ImpactValidator.cs` | Unknown change ids and consumers discarded (R44, R45) |
| `src/ContractGuard.Cli/Publishing/SpecLineLocator.cs` | Change → line in the snapshot (R49) |
| `src/ContractGuard.Cli/Publishing/CommentPublisher.cs` | Summary upsert, inline threads, fingerprints (R48–R57) |
| `src/ContractGuard.Cli/Publishing/SummaryRenderer.cs` | Deterministic markdown for the summary (R10, R11, R18, R47, R59) |
| `src/ContractGuard.Cli/Reporting/RunReport.cs` | Artifact, `##vso` commands, outcome (R60–R64) |
| `tests/ContractGuard.Tests/` | xUnit + WireMock.NET; spec pairs, recorded oasdiff and Claude outputs, ADO responses |
| `eval/golden/` + `eval/run-eval.ps1` | Past breaking changes with known affected consumers; manual run |
| `docs/onboarding.md` | Enabling a repo: spec generation, snapshot, registry, ignore files |

**In each BFF repo:**
- `pipelines/contract-guard.yml`, about 10 lines with `extends`;
- `contracts/bff.openapi.json` (snapshot), `contracts/consumers.yaml`, `contracts/accepted-breaks.yaml`, optional ignore files;
- the build-time OpenAPI generation in the API project (see below).

**Not touched:** the BFF code, its existing pipelines and the required branch policies.

## Azure DevOps infrastructure
Reuses [ai-pr-reviewer's](../ai-pr-reviewer/design.md#azure-devops-infrastructure) pool, variable group (with its *Required template* check extended to `templates/contract-guard.yml@contract-guard`) and Build Service permissions. Additions:
- **Branch policy**: Build validation **Optional**, automatic trigger, path filter `/contracts/*; /src/*` (R1). With `failOnErr` (R61) the team decides whether to make it Required.
- **[v2] Telemetry identity**: an Azure service connection (workload identity federation) with *Monitoring Reader* on the App Insights resource. Used via `AzureCLI@2` in a step before the CLI, which writes the query result to a file; the token never reaches the CLI process that launches Claude (R27).
- **[v2] Pact**: pacts read from a folder in the repo or from a broker with a read-only token from the variable group, excluded from Claude's environment (R27).

## Spec generation (R4, R8)
In the BFF API project (.NET 9):
```xml
<PackageReference Include="Microsoft.AspNetCore.OpenApi" />
<PackageReference Include="Microsoft.Extensions.ApiDescription.Server" PrivateAssets="all" />
<PropertyGroup>
  <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/../../artifacts/openapi</OpenApiDocumentsDirectory>
</PropertyGroup>
```
`dotnet build` writes `artifacts/openapi/<Project>.json`. Swashbuckle projects use `swagger tofile` instead. Exact properties are confirmed in T1.

`contract-guard snapshot` (developer command) copies the generated file to `contracts/bff.openapi.json` after normalizing it with `SpecNormalizer`: sorted object keys, 2-space indentation, LF. The summary for R8 shows that command. Semantic equality compares the normalized forms.

## oasdiff invocation (R12–R18)
```
oasdiff breaking base.json head.json --format json \
  --err-ignore <target>:contracts/oasdiff-ignore-err.txt \
  --warn-ignore <target>:contracts/oasdiff-ignore-warn.txt
oasdiff changelog base.json head.json --format json --level INFO
```
- Ignore files are extracted with `git show origin/<target>:<path>` into a temp folder (R15, R16).
- The level, rule id, operation, path and text of each change are read from the JSON; field names are confirmed in T1.
- `--fail-on` is not used: the CLI decides the outcome (R60, R61).

**Change id** (R17): `<ruleId>|<METHOD> <path>|<subject>`, where `<subject>` is the affected parameter, property path or enum value, normalized. Example: `response-property-removed|GET /orders/{id}|legacyTotal`. The same string is the acknowledgment key (R33) and the thread fingerprint (R50).

## Data formats
**`contracts/consumers.yaml`** (R19–R21):
```yaml
- consumer: web
  owner: team-checkout
  contact: "#checkout-web"
  clientIds: [web-spa]                # [v2] match with telemetry
  uses:
    - operation: GET /orders/{id}
      fields: [id, status, price, lines[].sku]   # empty = whole operation
    - operation: POST /orders
```

**`contracts/accepted-breaks.yaml`** (R32–R35):
```yaml
- change: "response-property-removed|GET /orders/{id}|legacyTotal"
  reason: "Unused for 90 days (App Insights)"
  consumers_notified: [web, mobile-ios]
  expires: 2026-12-31
```

**`schemas/impact.schema.json`** (R43):
```json
{ "explanations": [{
    "changeId": "string",
    "summary": "string<=200",
    "consumers": [{ "consumer": "string", "impact": "string<=300",
                    "evidence": ["registry|pact|telemetry|code"] }],
    "suggestion": "string<=300|null" }] }
```
Claude never returns a level: the renderer takes it from oasdiff (R14).

## Claude invocation (R37–R47)
Same command line, environment allowlist and preflight as [ai-pr-reviewer](../ai-pr-reviewer/design.md#claude-invocation-r15r21-r39r41), with `--json-schema impact.schema.json`. The input is a single JSON document on stdin:
```
<untrusted_contract_content>
  changes (id, level, text), affected operations and schemas from head.json
  (descriptions and examples included, hence untrusted), consumer matches with evidence,
  [v2] inbound code references (file:line + 3 lines of context)
</untrusted_contract_content>
```
`Read`, `Grep` and `Glob` stay available so the model can look at the mappers or endpoints behind an operation. One invocation per run; `maxExplainedChanges` caps the payload (R41).

## Configuration (template parameters with defaults)
| Key | Default | Source |
|---|---|---|
| `plan`, `tokenOwnerUpn`, `consumerTermsAck`, `tokenExpiresOn` | — | Protected variable group (shared) |
| `apiProject` | — (required) | Template |
| `snapshotPath` | `contracts/bff.openapi.json` | Template |
| `downstreamSpecGlobs` | `contracts/downstream/*.json` | Template |
| `oasdiffVersion` / `oasdiffSha256` | from `tools/oasdiff.lock` | contract-guard repo |
| `model` | `sonnet` | Template |
| `language` | `en` | Template |
| `maxExplainedChanges` | `10` | Template |
| `maxTurns` / `timeoutMinutes` | `15` / `10` | Template |
| `failOnErr` | `false` | Template |
| `telemetry.enabled` / `telemetry.days` / `telemetry.serviceConnection` | `false` / `30` / — | Template [v2] |
| `pact.source` | — (disabled) | Template [v2] |
| `inboundSearch` | operation → generated client method name; field → DTO property name (`PascalCase`) | Template [v2] |

## Algorithm
1. Read `PipelineContext` and `GuardOptions`. Fork → exit (R3). Query the PR: status, labels, author. `no-contract-guard` label → skip (R2).
2. The template has already built the BFF. `SpecProvider`:
   - a. Generated file missing → summary "spec generation failed" and `SucceededWithIssues` (R5).
   - b. `mb = merge-base(origin/<target>, sourceCommit)`; `base = git show mb:<snapshotPath>`. Missing → "first snapshot" summary (R11).
   - c. `head` = normalized generated spec (R7). Compare with the normalized snapshot in the source commit; different → "snapshot out of date" finding (R8).
   - d. List changed downstream specs (`git diff --name-only mb sourceCommit -- <downstreamSpecGlobs>`).
   - e. No change in the BFF spec or downstream specs → R9 or R10 and exit.
3. `OasdiffRunner`: outbound diff `base` vs `head`; [v2] inbound diff per changed downstream spec (R28). Ignore files from the target branch (R15). Build change ids (R17).
4. `AckStore`: read `accepted-breaks.yaml` from the source commit (R32), validate (R35), apply expiry (R34), compare with the target-branch version to mark "in this PR" (R36). Matched changes move to "acknowledged" (R33).
5. `ConsumerMatcher` [v1]: registry from the target branch (R19, R20); match by operation and fields (R21); no match → "unknown consumers" (R22). [v2] `PactSource` (R26) and the telemetry file (R24, R25) add evidence and unregistered consumers.
6. `InboundScanner` [v2]: for each inbound change, `git grep` over the source commit for the client method and DTO property names → `code` evidence with `file:line` (R29–R31).
7. If no `ERR`/`WARN` remains, or the plan is `consumer` and the author is not the token owner, skip to 9 without Claude (R37, R39).
8. `ImpactPromptBuilder` → `ClaudeRunner` (shared preflight and isolation, R38) → `ImpactValidator`: drop unknown change ids (R44) and consumers outside the matches (R45); 1 retry on invalid schema (R46); any failure → deterministic summary with the cause (R47).
9. `SummaryRenderer`: header with counts, table of `ERR`/`WARN` changes (level from oasdiff), per-change consumers with evidence and explanation, acknowledged section, changelog (`INFO`, R18), unexplained changes beyond the cap (R41). `OutputSanitizer` on every generated text (R58).
10. `CommentPublisher`:
    - query the PR again: completed or abandoned → exit (R54); newer iteration → discard (R53);
    - upsert the summary (R48);
    - for each `ERR`/`WARN` change, if the snapshot is in the PR diff and `SpecLineLocator` finds the line of the operation, parameter or schema property in the normalized snapshot, create an inline thread unless its fingerprint exists (R49–R52);
    - retries on 429/5xx (R55); never votes or statuses (R56); only its own threads (R57).
11. `RunReport`: artifact without spec bodies or telemetry rows (R63, R64). Outcome: `Failed` only under R61; otherwise `Succeeded`, or `SucceededWithIssues` with the cause (R60, R62).

## Error handling
| Situation | Behaviour | Job outcome |
|---|---|---|
| BFF build or spec generation fails | Summary "spec generation failed"; no diff | SucceededWithIssues |
| Snapshot out of date | Finding with the regeneration command; the diff still runs against the generated spec | Succeeded |
| No base snapshot | "First snapshot" summary with the operation list | Succeeded |
| oasdiff checksum mismatch or crash | Not run; warning in the log | SucceededWithIssues |
| Invalid `consumers.yaml` | Ignored and stated in the summary; all changes "unknown consumers" | Succeeded |
| Invalid acknowledgment | Ignored and stated in the summary | Succeeded |
| Claude quota, auth, timeout or invalid JSON twice | Deterministic summary with the cause | SucceededWithIssues |
| Telemetry or Pact unavailable | Registry evidence only; stated in the summary | SucceededWithIssues |
| Persistent ADO 429/5xx | Reported in the artifact; nothing posted | SucceededWithIssues |
| Unacknowledged outbound `ERR` with `failOnErr` | Normal summary plus a "gate failed" line | Failed |
| PR closed or new iteration | Exits silently | Succeeded |

## Rejected alternatives
1. **Letting the model decide what is breaking.** Rejected because severity must be reproducible and auditable, and a prompt injection in a spec description could otherwise hide a break. oasdiff's rules are deterministic and versioned.
2. **Diffing against the spec served by a deployed environment** (`/openapi.json` in DEV). Rejected because the environment may not match the target branch, and the job would need network access and credentials to that environment.
3. **Building the target branch on every run** to get the base spec. Rejected as the default because it doubles build time and fails if the target branch is broken. The committed snapshot is cheaper and makes the contract diff visible to human reviewers. It remains the fallback while a repo has no snapshot (R11).
4. **Reading the registry and ignore files from the PR source branch.** Rejected because a PR could remove a consumer or add an ignore rule to hide its own break. Acknowledgments are the exception: they belong to the PR that introduces the break, are validated, and are flagged "acknowledged in this PR" for the human reviewer.
5. **Blocking from day 1** (`--fail-on ERR` in a Required policy). Rejected because false positives on enums and nullability would teach the team to bypass it. The gate is opt-in after the pilot measures precision.
6. **A separate pipeline and infrastructure from ai-pr-reviewer.** Rejected because the pool, variable group, token, checks and isolation would be duplicated. Sharing `AiTools.Common` keeps one hardened path to Claude.
