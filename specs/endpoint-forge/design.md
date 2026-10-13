# endpoint-forge — Design

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Architecture
```
Claude Code session (developer's clone, Windows)
 ├─ /endpoint-forge:<command>  (skill = orchestration)
 │    ├─ forge CLI (.NET 9, deterministic)  profile · resolve · plan · approve · guard · scaffold · verify · runs · [v2] init · drift
 │    ├─ planner subagent      → plan.json (JSON against plan.schema.json)
 │    └─ implementer subagent  → code, only in manifest paths
 └─ PreToolUse hook → forge guard write   (writes only to the current phase's paths, R57)

Inputs   description · contracts/downstream/*.json · .endpoint-forge/profile.yaml · exemplar endpoints
Output   local branch forge/{feature}-{name} (never pushed) · run dir <runsPath>/<repo>/<runId>/
Shared   [v1] SpecNormalizer + OasdiffRunner (contract-guard) · [v2] Bff.Facts Roslyn extraction (devcards)
```
Core principle: **the plan is the contract. Deterministic code checks it, renders the mechanical parts from it and verifies the result against it. The model only proposes the plan and writes the bodies, inside a manifest it cannot leave.**

After generation, the developer pushes the branch and opens a normal PR, which goes through [ai-pr-reviewer](../ai-pr-reviewer/design.md) and [contract-guard](../contract-guard/design.md) like any other change.

## Shared components
| Component | Comes from | Used here for |
|---|---|---|
| `SpecNormalizer`, `OasdiffRunner`, `tools/oasdiff.lock` | [contract-guard](../contract-guard/design.md#files) | [v1] Additive contract check and snapshot update (R67–R69). It writes the same normalized snapshot, so contract-guard sees only the new operation in the PR |
| `Bff.Facts` (Roslyn: endpoints, DTOs, clients, mappers, DI, policies) | devcards [`extract`](../devcards/design.md#cli-devcards) | [v2] `init`, `check-profile` and lineage verification (R71, R83–R89) |
| Repo key from the Azure Repos `origin` URL | devcards [`vault`](../devcards/design.md#vault-layout) | Run directory key (R92) |
| `AiTools.Common` | [ai-pr-reviewer](../ai-pr-reviewer/design.md) | Not used in v0–v2; the v3 pipeline mode reuses its hardened path to Claude |

As with `AiTools.Common`, whichever tool is built first extracts the component into a package on the Azure Artifacts feed. v0 needs neither: it uses Roslyn syntax trees only (no solution load), and route collisions come from the OpenAPI snapshot or a syntax scan.

## Plugin files
| Path | Contents |
|---|---|
| `.claude-plugin/plugin.json` | Name `endpoint-forge`, `userConfig` (below), version |
| `skills/{plan,new,repair}/SKILL.md` | [v0] command orchestration |
| `skills/{init,check-profile}/SKILL.md` | [v2] |
| `agents/planner.md` | Tools `Read`, `Grep`, `Glob`, `Write` (R55); the hook only lets it write `plan.json` in the run directory |
| `agents/implementer.md` | Tools `Read`, `Grep`, `Glob`, `Edit`, `Write` (R56); no `Bash`, no web tools, no MCP; the hook only lets it write manifest paths |
| `hooks/hooks.json` | `PreToolUse` on `Write\|Edit\|MultiEdit\|NotebookEdit` → `forge guard write` (R57) |
| `bin/forge`, `bin/forge.cmd` | Shims on the Bash `PATH` that run `dotnet <plugin>/tool/EndpointForge.Cli.dll` |
| `tool/` | Framework-dependent .NET 9 build of `EndpointForge.Cli` |
| `schemas/profile.schema.json`, `plan.schema.json`, `verify.schema.json` | R3, R24, R73 |
| `templates/<style>/<library>/*.scriban` | [v1] Built-in templates for the mechanical parts only: DTO, validator, direct mapping assignments, registration line, WireMock stub, test skeleton (R45–R50) |
| `evals/` | `claude plugin eval` cases over `fixtures/SampleBff` |

**Source repo layout (`endpoint-forge/`):** `src/EndpointForge.Cli/` (Profile, Specs, Plan, Approval, Guard, Scaffold, Clients, Verify, Runs), `tests/EndpointForge.Tests/` (xUnit, git fixture repos), `fixtures/SampleBff/` (minimal APIs and controllers, Refit and NSwag clients, FluentValidation, manual and Mapster mappers, WebApplicationFactory + WireMock.NET tests, three downstream specs), `eval/golden/`, `plugin/` (the files above).

## Configuration (`userConfig`)
| Key | Type | Default |
|---|---|---|
| `runsPath` | directory | `~/.endpoint-forge/runs` (must be outside the repository, R92) |
| `profilePath` | string | `.endpoint-forge/profile.yaml` (R1) |
| `language` | string | `en` (R6) |
| `branchPattern` | string | `forge/{feature}-{name}` (R16) |
| `commit` | boolean | `false` (R23) |
| `maxRepairAttempts` | number | `2` (R74) |
| `maxSpecTokens` | number | `30000` (R13) |
| `maxExemplars` / `dominanceThreshold` / `maxEditExamples` | number | `2` / `0.7` / `3` [v2] (R85, R86, R91) |

## Architecture profile (`.endpoint-forge/profile.yaml`)
Versioned in the BFF repo and reviewed like code. Placeholders: `{Feature}`, `{Name}`, `{Response}`, `{Request}`, `{Downstream}`, `{OperationId}`, `{Scenario}`.
```yaml
version: 1
solution: Bff.sln
style: minimal-api                      # minimal-api | controllers
naming:                                 # R34
  name: "{Verb}{Resource}"              # GetCustomerSummary
  request: "{Name}Request"
  response: "{Resource}View"            # CustomerSummaryView
dto: { kind: record, nullable: enabled, jsonNames: policy }   # record | class; policy | attribute
layout:                                 # R5: every manifest path comes from here
  endpoint:  src/Bff.Api/Features/{Feature}/{Name}Endpoint.cs
  request:   src/Bff.Api/Features/{Feature}/{Request}.cs
  response:  src/Bff.Api/Features/{Feature}/{Response}.cs
  validator: src/Bff.Api/Features/{Feature}/{Name}Validator.cs
  handler:   src/Bff.Application/{Feature}/{Name}Handler.cs
  mapper:    src/Bff.Application/{Feature}/{Name}Mapper.cs
  port:      src/Bff.Application/{Feature}/Ports/I{Downstream}Source.cs     # contract-first (R80)
  unitTests: tests/Bff.UnitTests/{Feature}/{Name}MapperTests.cs
  apiTests:  tests/Bff.IntegrationTests/{Feature}/{Name}EndpointTests.cs
  stubs:     tests/Bff.IntegrationTests/Stubs/{Downstream}/{OperationId}.{Scenario}.json
registration:                           # R19, R48: the only edits to existing files
  - file: src/Bff.Api/Features/{Feature}/{Feature}Module.cs
    anchor: "// endpoint-forge:endpoints"
  - file: src/Bff.Application/DependencyInjection.cs
    anchor: "// endpoint-forge:handlers"
libraries:
  validation: fluentvalidation          # fluentvalidation | dataannotations
  mapping: manual                       # manual | mapster | automapper
  mediator: none                        # none | mediatr
  tests: { framework: xunit, http: wiremock-net, host: webapplicationfactory, assertionHelpers: [] }
clients:                                # [v1] R51–R53
  tool: refitter                        # refitter | nswag | kiota | typed-httpclient
  command: "dotnet refitter --settings-file {settings}"
  settings: src/Bff.Infrastructure/Clients/{Downstream}/.refitter
  generatedPath: src/Bff.Infrastructure/Clients/{Downstream}/Generated
conventions:
  routePrefix: /api
  auth: bff                             # default authorization policy
  errors: problem-details
  cancellationToken: required
errorMapping:                           # R31: defaults per downstream outcome
  notFound: 404
  clientError: 502
  serverError: 502
  timeout: 504
downstreamSpecGlobs: ["contracts/downstream/*.json"]   # same folder as contract-guard's inbound mode
openApi:
  snapshotPath: contracts/bff.openapi.json             # [v1] R67–R69
bannedApis:                             # R59; defaults shown
  - System.Diagnostics.Process
  - System.Reflection.Assembly.Load*
  - System.IO.File.Delete
  - System.IO.Directory.Delete
  - System.Net.Sockets
  - System.Net.WebClient
  - System.Environment.GetEnvironmentVariables
  - Microsoft.Win32.Registry
exemplars:                              # R4
  - GET /api/orders/{id}
  - POST /api/orders
architectureTests: tests/Bff.ArchitectureTests          # [v2] R70
```

## Plan schema (`plan.json`)
The planner fills the fields marked ✍. Fields marked ⚙ are computed or filled by the CLI.
```jsonc
{
  "mode": "compose",                                        // ⚙ compose | expose (no description) | contract-first (no operation, R79)
  "feature": "Customers", "name": "GetCustomerSummary",    // ✍ checked against naming (R34)
  "method": "GET", "route": "/api/customers/{id}/summary", // ✍ collision check (R29)
  "auth": "bff",                                            // ✍ default from the profile
  "request": {                                              // ✍
    "params": [{ "name": "id", "in": "path", "type": "guid", "required": true }],
    "body": null,
    "validation": [{ "id": "v-id", "target": "id", "rule": "notEmpty" }]
  },
  "response": {                                             // ✍ every field needs a source (R25)
    "type": "CustomerSummaryView",
    "fields": [
      { "name": "id", "type": "guid", "nullable": false, "source": { "kind": "request", "param": "id" } },
      { "name": "displayName", "type": "string", "nullable": false,
        "source": { "kind": "expr", "inputs": ["customer:$.firstName", "customer:$.lastName"], "expr": "first + \" \" + last" } },
      { "name": "openOrders", "type": "int", "nullable": false, "source": { "kind": "downstream", "ref": "orders:$.totalCount" } },
      { "name": "tier", "type": "string", "nullable": true, "source": { "kind": "downstream", "ref": "loyalty:$.tier" } },
      { "name": "marketingConsent", "type": "bool", "nullable": true, "source": { "kind": "todo", "note": "Not in any provided spec" } }
    ]
  },
  "calls": [                                                // ✍ DAG checked (R30)
    { "id": "customer", "spec": "contracts/downstream/customers.json", "operationId": "getCustomer",
      "args": { "customerId": "request.id" }, "dependsOn": [] },
    { "id": "orders", "spec": "contracts/downstream/orders.json", "operationId": "listOrders",
      "args": { "customerId": "request.id", "status": "OPEN" }, "dependsOn": [] },
    { "id": "loyalty", "spec": "contracts/downstream/loyalty.json", "operationId": "getMember",
      "args": { "memberId": "customer:$.loyaltyId" }, "dependsOn": ["customer"] }
  ],
  "errors": [                                               // ✍ overrides; ⚙ gaps filled from errorMapping (R31)
    { "call": "customer", "outcome": "notFound", "bff": { "status": 404 } },
    { "call": "loyalty", "outcome": "timeout", "bff": { "status": 200, "nullFields": ["tier"] } }
  ],
  "scenarios": [                                            // ✍ coverage checked (R33); excerpt, the full plan has 15
    { "id": "ok", "given": "all calls succeed", "expect": "200 with every field" },
    { "id": "customer-not-found", "given": "customer returns 404", "expect": "404 problem details" },
    { "id": "loyalty-timeout", "given": "loyalty times out", "expect": "200 with tier = null" },
    { "id": "v-id-empty", "given": "id is Guid.Empty", "expect": "400 validation problem" }
  ]
}
```
- **Source kinds**: `downstream` (`ref` = `<callId>:<JSONPath>`, optional `default`), `expr` (several `inputs` and an expression the implementer writes in C#), `request` (`param`), `constant` (`value`), `todo` (`note`; the field must be nullable, R35).
- **Refs** resolve against the response schema of the call's operation in `context.json` (R26). [v1] The JSON schema type and the `required`/`nullable` flags at that path drive the type and nullability checks (R27, R28).
- **Contract-first** (R79–R81): calls have `port` and `method` (`"port": "ILoyaltySource", "method": "GetTierAsync"`) instead of `spec` and `operationId`, and no field may be `downstream`.

## CLI (`forge`)
| Command | Purpose | Requirements |
|---|---|---|
| `profile check` | Load and validate the profile; resolve the exemplars | R1–R5 |
| `resolve --run <dir> [--spec <ref>]… [--description-file <f>]` | Inputs, operations, `$ref` closure without examples, token budget, existing routes → `context.json` | R7–R13, R29 |
| `plan check --run <dir> --attempt <n>` | Schema, sources, [v1] types and nullability, routes, DAG, error fill, scenario coverage, names, `todo` → `plan-check.json` | R24–R37, R81 |
| `plan render --run <dir>` | `plan.md` with tables and Mermaid | R6, R38, R39, R82 |
| `approve --run <dir>` / `approve --verify <plan.json>` | Record or check the SHA-256 of `plan.json` in `approval.json` | R40–R42, R44 |
| `guard begin --run <dir>` | Requires an approval; clean tree, branch, manifest, active-run marker | R15–R18, R40 |
| `guard phase --run <dir> <plan\|fill\|repair\|idle>` | Sets the phase in the marker | R57 |
| `guard write` | Hook entry point: reads the tool call from stdin, allows or denies | R57 |
| `guard end --run <dir>` | `git status` vs manifest; optional commit; clears the marker | R20, R23 |
| `scaffold --run <dir>` | [v1] Templates, registration, stubs, test skeletons, client generation | R45–R53 |
| `verify --run <dir> --attempt <n>` | The verification gate → `verify.json` | R54, R59–R73 |
| `runs append --run <dir>` / `report --run <dir>` | Observability record and final report | R92–R95 |
| `init` | [v2] Facts → proposed profile and exemplars | R83–R87 |
| `drift` | [v2] Endpoints and exemplars off profile; survival of generated lines | R88–R90 |
| `examples --kind <layout-kind>` | [v2] Earlier (generated, current) pairs for the implementer | R91 |

`forge` never calls the network itself. Child processes (`dotnet build`, `dotnet test`, the client generator) restore packages as they would in normal development.

## Run directory
```
<runsPath>/<org>/<project>/<repo>/<runId>/
├── input.json        # description and selected operations
├── context.json      # resolved operations + schema closure, existing routes, exemplar paths
├── plan.json         # planner output (✍) + filled error mappings (⚙)
├── plan-check.json
├── plan.md           # rendered plan; can be pasted into the PR description
├── approval.json     # plan SHA-256, approved at
├── manifest.json     # per path: kind, owner (rendered | model | registration | client | snapshot), rendered hash
├── verify.json       # one entry per attempt (R73)
└── report.md
<runsPath>/runs.jsonl                    # R93
<runsPath>/active/<repo-hash>.json       # active-run marker: runDir, phase, repoRoot
```

## Write guard (R57)
`forge guard write` runs on every `Write`, `Edit`, `MultiEdit` and `NotebookEdit` call while the plugin is enabled:
1. Read the hook input (tool name, target path, `cwd`) and find the repository root of `cwd`.
2. No active-run marker for that repository → allow. The guard only constrains runs.
3. Phase `plan` → allow only `<runDir>/plan.json`. Phase `fill` or `repair` → allow only manifest paths. Paths are normalized (case-insensitive on Windows, `..` and symlinks resolved) before comparing.
4. Deny with a reason that names the allowed paths. The denial is recorded and listed in the report.

Plugin-shipped agents cannot declare their own hooks, so the guard is plugin-wide and keyed on the marker. While a phase is active it also constrains the main session, which is intended. T1 confirms the hook input fields on Windows.

## Generic pipeline (`/endpoint-forge:new`)
1. `forge profile check` → stop on failure (R1–R5).
2. `forge resolve` → `context.json` (R7–R13).
3. Phase `plan`: the **planner** receives the run directory (description and specs inside untrusted delimiters, R14), the profile and the exemplar paths. It writes `plan.json` and returns one line: `done -> plan.json`.
4. `forge plan check --attempt 1`. On failure the planner gets `plan-check.json` and retries once; a second failure stops the run (R24–R37).
5. `forge plan render` → the skill shows `plan.md`. The developer approves, edits `plan.json` by hand or asks for changes (back to 3). `forge approve` (R38–R42).
6. `forge guard begin` → branch, manifest, marker (R15–R18, R40).
7. [v1] `forge scaffold` (R45–R53).
8. Phase `fill`: the **implementer** receives the plan, profile, exemplar files, [v1] scaffold and [v2] edit examples, and returns `done` (R56, R58, R91).
9. `forge verify --attempt 1` (R59–R73).
10. On failure: phase `repair`; the implementer gets the latest attempt in `verify.json`; full re-verification after every attempt, up to `maxRepairAttempts` (R74–R76).
11. `forge guard end` (R20, R23), `forge report` (R95), `forge runs append` (R92–R94).

Subagents return file references, never summaries. The main session only orchestrates: it runs `forge` and launches the subagents, and it does not write code itself.

## Command specifics
- **plan** [v0]: steps 1–5 without `guard begin`. Nothing is written to the repository, so it also runs on a dirty working tree (R43).
- **new** [v0]: the full pipeline. With a plan path instead of a description, `forge approve --verify` checks the recorded hash and the run starts at step 6 (R44).
- **repair** [v0]: finds the run from the current branch in `runs.jsonl` and resumes at step 9 (R78). The developer may have edited files by hand, but only manifest paths (R20).
- **contract-first** [v1]: with no operations, the mode is `contract-first`. Calls become ports (`layout.port`), fields that would come from downstream are `todo`, and tests use a fake of the port instead of WireMock (R79–R82).
- **init** [v2]: `Bff.Facts` → endpoint clusters by style, layout and libraries. Fields below `dominanceThreshold` stay `?` and the user is asked. Exemplars are the endpoints closest to the cluster centre. The profile is written only after confirmation (R83–R87).
- **check-profile** [v2]: reports endpoints off profile, failing or drifted exemplars and the survival of generated lines per earlier run (R88–R90).

## Verification gate (`forge verify`)
| # | Check | How | Fails when | Requirements |
|---|---|---|---|---|
| 1 | `manifest` | `git status --porcelain` vs `manifest.json` | A changed file is not in the manifest | R20 |
| 2 | `packages` | Diff of `*.csproj`, `Directory.Packages.props`, `packages.lock.json` | Any package reference or version changes | R66 |
| 3 | `banned-apis` | Roslyn syntax walk of the manifest's `.cs` files | An API in `bannedApis` is used; stops before step 6 | R59 |
| 4 | `rendered` [v1] | Hash of rendered files vs `manifest.json` | A rendered DTO or validator was changed | R54 |
| 5 | `todo-markers` | Syntax walk of the mapper | A `todo` field has no `TODO(endpoint-forge)` comment | R65 |
| 6 | `build` | `dotnet build <solution>` | Compiler errors | R60 |
| 7 | `format` | `dotnet format --verify-no-changes --include <files>` | Formatting differences | R61 |
| 8 | `scenarios` | Syntax walk: scenario marker per test, `Skip`, assertions | A scenario has no test; a test is skipped or asserts nothing | R63, R64 |
| 9 | `tests` | `dotnet test --filter` on the generated test classes | A test fails | R62 |
| 10 | `openapi` [v1] | Build-time generation, `SpecNormalizer`, `oasdiff breaking` + `changelog` | Breaking output not empty, or a changelog entry outside the new operation | R67–R69 |
| 11 | `architecture` [v2] | `dotnet test <architectureTests>` | A test fails | R70 |
| 12 | `lineage` [v2] | `Bff.Facts` mapper assignments vs plan sources | An assignment contradicts the plan | R71, R72 |

- **Scenario marker**: xUnit `[Trait("scenario", "<id>")]`, NUnit `[Property("scenario", "<id>")]`, MSTest `[TestProperty("scenario", "<id>")]`. An assertion is a call to `Assert.*`, `Should*`, `Verify*` or `Received*`, or to a helper listed in `libraries.tests.assertionHelpers`.
- A build failure marks steps 7–12 `skipped`. **Environment** failures (package restore, missing SDK, locked files) are classified apart and never sent to repair (R77).
- `verify.json`: `{ attempts: [{ attempt, checks: [{ id, status: pass|fail|skipped|environment, items: [{ path, line, code, message, scenario? }] }] }] }`. Compiler and test messages are kept; file contents are not.

## Error handling
| Situation | Behaviour |
|---|---|
| Profile missing, invalid, or exemplar not found | Stop; nothing written (R2–R4) |
| No input, unknown operation, spec outside the repo | Stop; closest operations listed where relevant (R9–R11) |
| Selected operations over `maxSpecTokens` | Ask for fewer operations (R13) |
| Dirty working tree on `new` | Stop before creating the branch (R15) |
| Plan rejected twice | Violations shown; nothing written (R36, R37) |
| `plan.json` edited after approval | Re-check and ask again (R42) |
| Client generation command not allowed | Refused; stop (R52) |
| Client generation touched other files | Reverted; stop (R53) |
| Write denied by the guard | The agent receives the reason; the denial is listed in the report (R57) |
| Verification still failing after repairs | Branch kept with the changes; failing checks in the report (R76) |
| Environment failure (restore, SDK, locked files) | Stop without repair; cause in the report (R77) |
| Claude quota or authentication error | Stop; branch as it is; cause in the report (R77) |
| [v2] Semantic load fails | `init` and `lineage` fall back to syntax facts marked `inferred`; lineage check reported as `skipped` |

## Rejected alternatives
1. **One prompt: "write an endpoint like this one".** This is the baseline: it compiles, drifts from the architecture, invents downstream fields and tests only the happy path. A checked plan and a verification gate turn those expectations into checks.
2. **Team-maintained templates for every file** (`dotnet new`, Scriban). Templates rot as the architecture evolves, and composition logic (parallel calls, partial failures, degraded responses) does not fit a template. Built-in templates cover only the parts the plan fully determines; the exemplars are the living reference for the rest.
3. **The model writes downstream DTOs and clients.** The repository already has a generator; hand-written DTOs drift from the spec and from the other clients.
4. **Inferring the architecture on every run.** The output depends on whichever endpoint looks closest, legacy ones included. The profile makes the target explicit and reviewable; `init` gives the convenience without the drift.
5. **Server stubs from NSwag or OpenAPI Generator.** They produce a 1:1 proxy of the downstream, which is what a BFF should not be, and they ignore the team's layers.
6. **Pipeline first** (Azure Boards → PR). Plan approval would have to be asynchronous, and it needs a Team/Enterprise seat and the shared pipeline infrastructure. The local loop is where the developer iterates on the plan; the pipeline is v3.
7. **Letting the implementer run `dotnet build` through Bash.** Bash would let injected spec content run arbitrary commands. The CLI runs a fixed set of commands and returns structured results.
8. **Profile outside the repository by default.** The profile describes the team's architecture, so it is versioned and reviewed like code. `profilePath` still allows a local profile while the team has not adopted it.
