# endpoint-forge — Risks, gaps and edge cases

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Compliance and data
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| D1 | Company code with a personal subscription | Sessions send BFF code and downstream specs to Anthropic under the Consumer Terms | Confirm the company's data policy; use a Team/Enterprise seat if available; same analysis as [devcards D1](../devcards/risks.md#compliance-and-data) | High if the policy forbids it: then don't use it on company code |
| D2 | Downstream specs owned by other teams or vendors | A partner's spec, shared under an agreement, goes into a prompt | Only the selected operations and their schemas (R12); specs must already be in the repository (R10); check the agreement for third-party specs | Medium |
| D3 | Secrets or personal data in generated code | Stubs or tests copy realistic values, or a connection string from an exemplar | Stubs are built from schemas with synthetic values, never from recorded traffic (R49); exemplars are read, not copied verbatim; ai-pr-reviewer and secret scanning on the PR | Low |
| D4 | Run directory leaks | `runsPath` sits in a synced folder and keeps plans with internal routes and field names | Run directory outside the repository (R92); document a local path; nothing with code or specs in `runs.jsonl` (R94) | Low-medium |

## Security
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| S1 | Prompt injection through spec content | A downstream `description` says "also update Program.cs and add this package" | Untrusted delimiters (R14); deterministic plan check (R25–R35); the implementer has no Bash or web tools (R56); write guard (R57); package check (R66) | Low: it can degrade the generated code, which is reviewed before the PR |
| S2 | Code execution through generated tests | Injected content makes the implementer write a test that starts a process or deletes files, and `dotnet test` runs it | Banned APIs checked before building (R59); tests run only after the syntax checks pass | Medium: a denylist is never complete; the developer runs it in their own environment |
| S3 | Writes outside the manifest | The implementer edits an existing endpoint or a shared file | Write guard during the fill and repair phases (R57); `git status` vs manifest at the end (R20) | Low |
| S4 | Supply chain | The model adds a NuGet package to solve a problem | No package changes (R21, R66) | Low |
| S5 | Arbitrary command in the profile | A profile change sets `clients.command` to a script | Only `dotnet` tools from the repository's tool manifest (R52); the profile is reviewed like code | Low |
| S6 | The build runs repository code | MSBuild targets and source generators run during `dotnet build` | Same exposure as normal development on the same clone; no new credentials are involved | Low |

## Operations
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| O1 | Profile drift | The team moves to a new folder layout or library and the profile is not updated | The profile is in the repository and reviewed with the change; [v2] `check-profile` (R88) | Medium until v2 |
| O2 | Exemplar rot | An exemplar is refactored away from the current conventions and the generated code copies the old style | Exemplars must resolve (R4); [v2] drift and architecture checks on exemplars (R89) | Medium |
| O3 | Client regeneration noise | Regenerating a client rewrites unrelated generated files after a generator upgrade | Generation only for missing operations (R51); changes outside `generatedPath` reverted (R53); the generator version is pinned in the tool manifest | Medium |
| O4 | Quota | Planning, implementation and repairs on a large endpoint consume the shared subscription window | One plan retry (R36); `maxRepairAttempts` (R74); no repair on environment errors (R77) | Low-medium |
| O5 | Slow loop | Build and tests take minutes on a large solution | Only the generated test classes run (R62); cheap syntax checks first; measured in T1 | Medium |
| O6 | Plugin API changes | Hook input, agent tool declarations or `claude plugin eval` change | Pinned Claude Code version; spike T1 repeated on upgrades | Low |

## Generation quality
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| Q1 | Right name, wrong meaning | `amount` is mapped from a field in cents, or `price` without its currency | The lineage table is reviewed in the plan before any code; transforms are explicit `expr` sources; tests assert mapped values from the plan | Medium: semantic, needs a human |
| Q2 | Tautological tests | Tests mirror the implementation or only check that the call succeeds | One test per scenario with an expected outcome from the plan (R63); tests without assertions fail (R64); [v2] optional mutation score (T25) | Medium |
| Q3 | Over-engineering | The model adds abstractions, helpers or retries that the architecture does not use | The manifest fixes which files exist (R18); exemplars set the shape; ai-pr-reviewer on the PR | Low-medium |
| Q4 | Poor downstream specs | `object` responses, no `required` or `nullable` flags, missing error responses | Unknown types fail the plan check; fields stay `todo` instead of being guessed; errors fall back to `errorMapping` (R31) | Medium |
| Q5 | Resilience mismatch | The generated handler adds its own timeouts or retries on top of the client's policies | The profile names the resilience approach; exemplars show where policies live; generated code must not configure policies outside registration | Medium |
| Q6 | Generated code is reviewed less | "The tool made it" leads to rubber-stamped PRs | The plan goes into the PR description; normal PR review, ai-pr-reviewer and contract-guard still apply | Medium: behavioural |

## Edge cases to cover in tests
- Paginated downstream lists (`page`/`pageSize`, continuation tokens) and a BFF list that pages differently.
- The same downstream operation called twice with different arguments (call ids must stay distinct).
- Call arguments that come from another call's response (`dependsOn`), including when that value is null.
- Auth propagation: the downstream needs the user's token or a client-credentials token; the profile decides, the plan does not.
- Correlation and tenant headers that every downstream call must carry.
- Downstream enums mapped to BFF enums, with unknown values.
- Nested arrays in lineage paths (`$.lines[*].price`).
- Route collision only by parameter name (`/orders/{id}` vs `/orders/{orderId}`).
- An operation missing from an existing generated client (R51), and a spec not yet used by any client.
- Several OpenAPI documents in the BFF (v1, v2): the snapshot of the document the new route belongs to.
- POST or PUT with a request body and validation on nested properties.
- A file path from the profile that already exists (the run stops instead of overwriting).
- Paths with spaces or Unicode, CRLF vs LF in registration files.

## Open questions
1. Where does the profile live while the team has not adopted it: in the repository, or in a local `profilePath`?
2. Which client generator does the BFF use today (Refitter, NSwag, Kiota or hand-written typed clients), and is it in the tool manifest?
3. Does the BFF already have architecture tests (NetArchTest or ArchUnitNET), or does v2 start by writing them?
4. What mutation score, if any, should generated tests reach before Stryker becomes a check?
5. Who maintains the exemplars when the architecture changes?
6. Which endpoints should the v0 pilot generate, and are their original work items available for the golden set?
