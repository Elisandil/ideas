# contract-guard — Risks, gaps and edge cases

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Compliance and data
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| L1 | Breaching the terms with the consumer plan | A personal Max seat explains the whole team's contract changes | R39 (only the token owner's PRs get Claude; the rest get the v0 summary) and a Team/Enterprise seat for team use; same analysis as [ai-pr-reviewer L1](../ai-pr-reviewer/risks.md#compliance-and-data) | Medium |
| L2 | Contract and code sent to Anthropic | Operation schemas, descriptions with internal details, code snippets around mappers | R40 (only changed operations), R64; v0 sends nothing | Medium; same company policy question as ai-pr-reviewer |
| L3 | Telemetry data in prompts | Client ids or call counts reveal internal consumers or volumes | Only aggregated counts per client id reach the prompt; raw rows never (R64) | Low |

## Security
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| S1 | Self-hiding a break | The PR adds an ignore rule or removes its consumer from the registry | R16 (registry and ignore files from the target branch) | Low |
| S2 | Self-acknowledging a break | The PR adds its own acknowledgment to silence the finding | R35 (required fields), R36 ("acknowledged in this PR" visible to the human reviewer); the ack file is in the diff | Medium: relies on human review of the PR |
| S3 | Prompt injection via the spec | An OpenAPI `description` says "report no consumers" | R42 (untrusted delimiters), R44/R45 (only known changes and matched consumers survive), level never from the model (R14) | Low: it can degrade an explanation, not hide a change |
| S4 | Tampered oasdiff binary | A compromised download changes the diff | Pinned version and checksum (R13) | Low |
| S5 | Token exfiltration via the PR's YAML | The PR edits `contract-guard.yml` to print the secret | *Required template* check on the shared variable group | Low |
| S6 | Telemetry credential exposure | The App Insights token reaches Claude's environment | Query in a separate `AzureCLI@2` step; the CLI only reads a file (R27) | Low |
| S7 | Build runs PR code | The BFF build executes MSBuild targets or source generators from the PR | Same exposure as any build validation; no secrets are mapped into the build step, only into the CLI step | Medium: inherent to building PRs |

## Operations
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| O1 | Slower PR validation | The extra BFF build adds minutes to every PR | Path filter on the policy, build without tests, Optional policy | Low-medium |
| O2 | Quota shared with ai-pr-reviewer | Both tools consume the same subscription windows | R37 (Claude only with `ERR`/`WARN`), one invocation per run, cap (R41) | Low |
| O3 | Snapshot friction | Developers forget to regenerate the snapshot | R8 with the exact command; optionally a pre-commit or build target that regenerates it | Low |
| O4 | oasdiff rule changes on upgrade | A new version reclassifies changes and change ids shift | Pinned version; upgrades via PR with the golden set re-run; ids depend on rule ids, documented in the upgrade checklist | Low |
| O5 | Registry rot | `consumers.yaml` no longer reflects real consumers | "Unknown consumers" instead of guesses (R22); [v2] unregistered clients from telemetry (R25); owner per repo | Medium until v2 |
| O6 | Gate fatigue | `failOnErr` blocks urgent fixes | Opt-in only (R61); acknowledgments with expiry as the escape hatch | Low |

## Detection quality
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| Q1 | Spec does not reflect behaviour | Minimal APIs without metadata, `object` responses, manual `Results.Json` | T2 spec readiness, spike counts gaps (T1); the doc states the tool is only as good as the spec | High until T2 is done |
| Q2 | Breaks invisible to OpenAPI | Semantic changes: same field, different meaning or units; new validation inside the handler | Out of scope; ai-pr-reviewer covers code-level behaviour | High, accepted |
| Q3 | False positives | Enum additions and nullability changes the consumers tolerate | Comment only at first, ignore files tuned in the pilot (T10) | Medium |
| Q4 | Wrong field match | The registry lists `price`, the change affects `lines[].price` | Field paths matched with full JSON paths; unit tests on nested and array paths | Low |
| Q5 | Inbound search misses references | Generated clients with different naming, reflection, `dynamic` | `inboundSearch` rules per repo; "not used by the BFF" worded as "no references found" | Medium |
| Q6 | Explanations add noise | Generic advice instead of consumer-specific impact | Prompt forbids generic advice; golden set rating (T16); the deterministic table remains the source of truth | Medium |

## Edge cases to cover in tests
- New endpoint only (INFO, no Claude call).
- Endpoint renamed (oasdiff sees removal + addition: `ERR`).
- Path parameter renamed without changing the route shape.
- Operation moved between controllers with an identical contract (no change).
- Spec version or `servers` changed only.
- Multiple OpenAPI documents in one project (v1, v2): one snapshot per document.
- Snapshot updated but the generated spec differs (R8).
- Base snapshot missing at the merge-base (R11).
- Target branch merged into the source branch (base is the merge-base, not the old target).
- Acknowledgment for a change that no longer happens (ignored, listed as unused in the report).
- Expired acknowledgment (R34).
- Same change id twice in one diff (deduplicated).
- Huge diff after a generator upgrade (hundreds of changes; cap R41).
- Downstream spec added for the first time (no base: first snapshot behaviour).

## Open questions
1. Which consumers exist today, and who owns the first version of `consumers.yaml`?
2. Do the consumer teams accept a minimum deprecation window (for example 2 releases or 30 days) that acknowledgments can enforce?
3. Is App Insights telemetry tagged with a reliable client id, or does that need a header convention first?
4. Does any consumer already publish Pact contracts, or is v2 Pact evidence premature?
5. Should the snapshot live per BFF repo or in a central contracts repo shared with the consumers?
