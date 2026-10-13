# contract-guard — breaking-change detector for BFF contracts

Capability 2 of [ai-platform-bff.md](ai-platform-bff.md). It runs as a Build Validation in Azure DevOps. A deterministic OpenAPI diff (oasdiff) finds the changes, and Claude in headless mode explains the impact for each consumer in a PR comment.

> SDD spec in [specs/contract-guard/](specs/contract-guard/requirements.md): [requirements](specs/contract-guard/requirements.md), [design](specs/contract-guard/design.md), [tasks](specs/contract-guard/tasks.md) and [risks](specs/contract-guard/risks.md). Where this document and the spec differ, the spec wins.

## Problem
The BFF sits between two contract surfaces:
- **Its own OpenAPI**, used by the web front end, the mobile apps and other teams.
- **Downstream OpenAPIs**, which the BFF itself consumes.

Breaking changes in either one show up late, in QA or production. A raw spec diff doesn't solve this:
- **Noise**: hundreds of lines, mostly irrelevant.
- **No "who"**: it says what changed, not which consumer breaks.
- **No "what now"**: it doesn't suggest an additive alternative, a version or a deprecation window.

## Principles
1. **The diff is deterministic.** oasdiff decides what is breaking and how severe it is (`ERR` / `WARN` / `INFO`). The model never decides severity.
2. **No consumer, no claim.** Every impact points to evidence: the consumer registry, a Pact contract, telemetry by client id, or a code reference. Without evidence it says "unknown consumers", never guesses.
3. **Explain and propose.** For each break: what fails for that consumer, and the cheapest fix (an optional field instead of a required one, `/v2`, a deprecation window).
4. **Comment, don't block (at first).** The job is not required in the branch policy. `--fail-on ERR` only becomes a gate once precision is proven.
5. **Intentional breaks are explicit.** They are acknowledged in a versioned file, with a reason and the consumers notified, so the audit trail stays in the repo.
6. **No spec change, no LLM call.** Most PRs cost zero quota.

## Two directions
| Direction | Trigger | Question | Evidence |
|---|---|---|---|
| **Outbound** | The BFF spec changes in the PR | Which consumers break? | Consumer registry, Pact, telemetry |
| **Inbound** | A downstream spec or client package is bumped | Which BFF code paths use what changed? | Code references (grep / Roslyn), [devcards](devcards.md) lineage cards |

## Pipeline
```
PR opened/updated
   └─► Build Validation (azure-pipelines.yml)
         1. build head; generate openapi.json at build time
            check it matches the committed snapshot (contracts/bff.openapi.json)
         2. no change in contracts/** → exit (no LLM call)
         3. oasdiff breaking base.json head.json --format json   (+ oasdiff changelog)
            apply ignore files and accepted-breaks.yaml
         4. map each change (method, path, location) to consumers
            via consumers.yaml + evidence sources
         5. claude -p "<impact prompt>" --output-format json
            (read-only tools; input: diff JSON + consumer matches)
         6. a script posts one summary thread + inline threads on the spec
            via the Azure DevOps REST API (pullRequests/{id}/threads)
            re-runs update the bot's own threads instead of adding new ones
```

- **Generating the spec**:
  - .NET 9: `Microsoft.AspNetCore.OpenApi` + `Microsoft.Extensions.ApiDescription.Server` write the document at build time.
  - Swashbuckle: `swagger tofile`.
- **Base spec**:
  - Preferred: commit the generated spec as a snapshot. The diff is then visible in the PR, and the base is just the target branch's file, with no second build.
  - Fallback: build the target branch in the same job.
- **Inbound**: downstream specs live in `contracts/downstream/*.json` (the ones used to generate the clients). When one changes, the same diff runs, and Claude receives the BFF references to the affected operations and fields instead of consumers.

## Consumer registry
`contracts/consumers.yaml`, maintained by the BFF team:

```yaml
- consumer: web
  owner: team-checkout
  contact: "#checkout-web"
  clientIds: [web-spa]
  uses:
    - operation: GET /orders/{id}
      fields: [id, status, price, lines[].sku]
    - operation: POST /orders
- consumer: mobile-ios
  owner: team-mobile
  clientIds: [ios-app]
  uses:
    - operation: GET /orders/{id}
      fields: [id, status, lines[].sku]
```

| Evidence source | What it gives | Cost |
|---|---|---|
| Registry (manual) | Consumers, owners, endpoints and fields used | Upkeep; it rots |
| Pact pacts (if they exist) | Exact fields each consumer reads | Already maintained by consumers |
| App Insights | Real calls per endpoint and client id / `User-Agent`, last 30 days | One KQL query |
| Front-end repo search (optional) | Field usage in the consumer's code | Needs read access to that repo |

## Rules and severity
| oasdiff level | Comment | Examples |
|---|---|---|
| `ERR` | **Breaking** | Endpoint removed; new required param; response field removed; type changed; field becomes nullable in the response |
| `WARN` | **Potentially breaking** | Enum value added in a response; pattern or max length changed; default changed |
| `INFO` | Changelog only | New optional field; new endpoint |

Team rules on top, in the oasdiff ignore files: for example, treat `x-extensible-enum` as non-breaking.

## Acknowledgments
`contracts/accepted-breaks.yaml`:

```yaml
- change: "response-property-removed|GET /orders/{id}|legacyTotal"
  reason: "Unused for 90 days (App Insights)"
  consumers_notified: [web, mobile-ios]
  expires: 2026-12-31
```

An acknowledged change is shown as **acknowledged**, not as a finding. An expired acknowledgment flags again.

### What good looks like
~~~markdown
## contract-guard — 2 breaking, 1 potentially breaking

| Change | Level | Consumers affected |
|---|---|---|
| `GET /orders/{id}`: new required query param `currency` | ERR | web, mobile-ios |
| `OrderView.price`: `number?` → `number` | ERR | none found |
| `OrderView.status`: enum value `ON_HOLD` added | WARN | mobile-ios |

**`currency` required**
- **web** (registry + 12k calls/day): every call without `currency` gets a 400.
- **mobile-ios** (registry; versions < 5.4 can't send it): 400 on the order screen.
- **Suggestion**: make it optional with default `EUR`, or publish `/v2/orders/{id}`.

**`ON_HOLD` added**
- **mobile-ios**: the `status` switch in versions ≤ 5.3 has no default branch (Pact contract). Map it to `PENDING` until 5.4 is the minimum.
~~~

## Pipeline setup
```yaml
- script: dotnet build src/Bff -c Release           # writes openapi.json
- script: ./tools/contract-guard/diff.sh            # oasdiff (pinned) → changes.json
- script: npm i -g @anthropic-ai/claude-code@<pinned>
  condition: eq(variables.HAS_CONTRACT_CHANGES, 'true')
- script: ./tools/contract-guard/explain-and-post.sh
  condition: eq(variables.HAS_CONTRACT_CHANGES, 'true')
  env:
    CLAUDE_CODE_OAUTH_TOKEN: $(CLAUDE_CODE_OAUTH_TOKEN)
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

Authentication, least privilege, quota and terms of use are the same as in the [PR reviewer's no-API variant](ai-platform-bff.md#no-api-variant-pr-reviewer-on-a-claude-subscription-capability-1): an OAuth token as a secret variable, the build service with only "Contribute to pull requests", and `claude` with read-only tools.

## Quality metrics
| Metric | Target |
|---|---|
| Precision of "breaking" flags (team agrees) | ≥ 0.9 |
| Breaking changes that reached QA/prod without a flag | 0 |
| Consumers correctly identified (vs. later incidents) | trending up |
| PRs with spec changes covered by the job | 100 % |
| LLM calls on PRs without spec changes | 0 |
| Time from push to comment | < 5 min |

## Roadmap
| Phase | Scope | Value |
|---|---|---|
| **v0** (days) | Spec generated at build + committed snapshot, oasdiff, changelog comment on the PR. No LLM | Breaking changes visible in every PR |
| **v1** | `consumers.yaml`, Claude impact per consumer, `accepted-breaks.yaml` | "Who breaks and what to do" |
| **v2** | App Insights and Pact evidence, inbound direction (downstream bumps), `--fail-on ERR` gate | Evidence-based impact; a safe gate |
| **v3** | Move into the platform: the context catalog as the consumer source, LLM gateway, golden cases from past breaking changes; feeds release notes (25) and contract testing (10) | From pipeline step to platform capability |

## Risks
- **An unreliable spec gives a useless diff**: minimal APIs without metadata, `object` or dynamic responses, missing `[ProducesResponseType]`. v0 also means cleaning up the spec.
- **Registry rot**: without telemetry or Pact, `consumers.yaml` drifts. That's why it shows "unknown consumers" instead of guessing, and why v2 adds real evidence.
- **False positives** on enums, nullability and defaults: start by commenting only on `ERR`, and tune with ignore files.
- **Subscription and quota**: same terms of use and shared limits as capability 1; a team-wide job needs a Team/Enterprise seat.
- **Data**: the diff and code snippets go to Anthropic. Send only the changed operations, never the whole spec or repo.
- **Huge specs**: cap the payload to changed operations and their schemas.

## Prior art
- oasdiff (breaking-change rules, changelog, ignore files).
- OpenAPITools openapi-diff.
- Optic (API diff and governance in CI).
- Pact `can-i-deploy` (consumer-driven compatibility).
- Bump.sh (changelogs and API diffs).
- Azure API Center and API Management (API inventory and governance).
