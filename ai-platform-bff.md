# AI agent platform for quality and automation of a BFF (C#)

## Context
- Development team of a C# BFF on **Azure DevOps**.
- Goal: a **shared platform** (not standalone tools) that hosts AI "capabilities" to automate tasks and improve software quality.
- Scope: team/company (security and data handling matter).
- Models: **Claude API** behind a gateway, so the provider can be changed.
- Status: ideation and design only; no implementation for now.

## Architecture

```
 Triggers                Core                            Capabilities (plugins)
 ─────────────          ──────────────────────────      ─────────────────────────
 ADO webhooks      ──►  Agent orchestrator       ──►    PR review, tests, docs,
 Alerts/Observab.       (queues + workflows)            incidents, migrations...
 CLI / Chat / VSCode    LLM gateway
 Cron                   System context:                 Outputs
                         · repos + code index           ─────────────
                         · service catalog              PRs/comments, work items,
                         · OpenAPI/contracts            Teams, dashboards
                         · telemetry (traces, logs)
                        MCP servers (tools)
                        Evaluation + audit + HITL
```

### Key pieces
- **LLM gateway**: model per task, prompt caching, cost/token tracking per capability, limits, and redaction of sensitive data before sending.
- **Context catalog**: graph of services, endpoints, contracts, owners and downstream dependencies. This is the differentiator: agents reason about the real system.
- **Capabilities as plugins**: trigger + tools (MCP) + prompt + success criterion. Adding one does not touch the core.
- **Human in the loop**: every change comes out as a PR or comment, never straight to main. Every run is audited.
- **Continuous evaluation**: golden cases (PRs with known bugs, past incidents) to measure whether each capability gets better or worse when prompts/models change.

### Proposed stack
.NET 9 (Worker Services + ASP.NET Core), Microsoft.Extensions.AI or the Anthropic SDK, PostgreSQL + pgvector, Roslyn (C# analysis), OpenTelemetry, Aspire (local orchestration), Azure DevOps REST API and Service Hooks.

## Capability catalog

### Quality and security
1. **PR reviewer with domain context**: BFF conventions, HttpClient/Polly usage, `CancellationToken`, N+1 across downstream calls, logs with sensitive data.
2. **Contract breaking-change detector**: OpenAPI diff between branches (oasdiff) + impact explanation per consumer. Design in [contract-guard.md](contract-guard.md).
3. **Test generator**: xUnit + WireMock.NET (happy paths, 4xx/5xx, timeouts, partial responses), reinforced with Stryker.NET (mutation testing).
4. **CI and flaky test triage**: classifies failures (flaky, infrastructure, real bug) and proposes a fix or opens a work item.
5. **Architecture guardian**: layer violations and drift from ADRs (NetArchTest + explanation).
6. **Per-PR security auditor**: injections, JWT/tokens, SSRF in downstream calls, secrets, PII exposure.
7. **PII detector in logs and responses**.
8. **Dependency watcher**: NuGet vulnerabilities and licenses, with a summary of the risk of upgrading.
9. **Downstream resilience review**: consistent timeouts, retries, circuit breakers and fallbacks.
10. **Contract testing (Pact) with self-repair**.
11. **Guided chaos and load testing**: k6/NBomber scenarios from real traffic and contracts.

### Automation and operations
12. **BFF endpoint generator (endpoint-forge)**: endpoint, DTOs, mappers, validation and tests from a description and/or downstream OpenAPI operations, following an architecture profile, with an approved plan before any code. Starts as a personal Claude Code plugin. Design in [endpoint-forge.md](endpoint-forge.md).
13. **Incident diagnosis**: given a `correlationId`, rebuilds the call chain from telemetry and summarizes the likely cause.
14. **Custom BFF MCP server**: endpoints, dependencies, feature flags and status as tools for any agent.
15. **Automatic documentation**: README, ADRs and Mermaid diagrams updated after each merge.
16. **Migrations and upgrades**: .NET/NuGet upgrades, breaking-change fixes, tests and PR.
17. **Performance analysis**: traces and BenchmarkDotNet; detects sequential calls that could run in parallel, missing caching and large payloads.
18. **Production replay**: anonymized traces turned into regression tests.
19. **Automatic postmortems**: timeline, root cause and action items.
20. **SLO guardian and per-release performance regressions**.
21. **Cloud and token FinOps**.
22. **Alert triage**: groups duplicates and links them to the latest deploy.

### Team productivity
23. **Conversational onboarding** over real code and diagrams.
24. **Ticket → PR**: from an Azure Boards story to a PR with tests, going through the automatic reviewer first.
25. **Release notes and changelog** focused on consumer impact.
26. **Prioritized technical debt detector** (frequent changes + bugs + complexity) with small, safe refactors.
27. **Dead feature flag cleanup**.
28. **Mocks and simulated environments** for downstream services, built from contracts.
29. **Lean living docs engine (devcards)**: short, anchored, tested cards (flows, field lineage, contracts, PRs, Q&A) that flag themselves stale when the code changes. Starts as a personal Claude Code plugin. Design in [devcards.md](devcards.md).

## No-API variant: PR reviewer on a Claude subscription (capability 1)

> Full SDD spec in [specs/ai-pr-reviewer/](specs/ai-pr-reviewer/requirements.md): [requirements](specs/ai-pr-reviewer/requirements.md), [design](specs/ai-pr-reviewer/design.md), [tasks](specs/ai-pr-reviewer/tasks.md) and [risks](specs/ai-pr-reviewer/risks.md).

To avoid per-token API spend, capability 1 can be built with **Claude Code in headless mode** (`claude -p`) inside an Azure DevOps pipeline, authenticated with the subscription. It does not require the .NET platform or the gateway.

```
PR opened/updated
   └─► Build Validation (azure-pipelines.yml)
         1. checkout the PR + git diff against the target branch
         2. install Claude Code (pinned version)
         3. claude -p "<review prompt>" --output-format json
         4. a script posts the findings as comment threads
            via the Azure DevOps REST API (pullRequests/{id}/threads)
```

- **Authentication**: `claude setup-token` generates a long-lived OAuth token tied to the subscription. It is stored as a secret pipeline variable and passed as `CLAUDE_CODE_OAUTH_TOKEN`.
- **Team rules**: review rules with the BFF conventions (Polly, `CancellationToken`, downstream N+1, PII in logs...).
- **Structured output**: JSON (file, line, severity, comment) that the script turns into inline comments.
- **Least privilege**: the build service token with only "Contribute to pull requests"; `claude` with read-only tools (`Read`, `Grep`, `Glob`).

### Watch points
1. **Terms of use**: Pro/Max are governed by the Consumer Terms, intended for individual use; reviewing a whole team's PRs requires a Team/Enterprise seat. Details in the spec's [risks](specs/ai-pr-reviewer/risks.md).
2. **Shared limits**: reviews consume the same quota (hourly and weekly windows) as interactive use.
3. **Token**: it can expire or be revoked; the job must fail without blocking the PR (do not make it required in the branch policy).
4. **Diff size**: cap it and review per folder or file; huge diffs degrade quality.
5. **Data**: the PR code goes to Anthropic; check the plan's retention policy.

## Roadmap

| Phase | Contents |
|------|-----------|
| 0. Foundation | LLM gateway, repo ingestion, contract catalog, Azure DevOps PR webhook |
| 1. Quick wins | PR reviewer (1), security auditor (6), breaking changes (2) |
| 2. Operational context | BFF MCP server (14), incident diagnosis (13) |
| 3. Generation | Tests (3), endpoints (12), ticket→PR (24), continuous evaluation |
| 4. Maturity | Architecture (5), FinOps (21), replay (18), chaos/load (11), postmortems (19), metrics dashboard |

## Risks and open decisions
- **Data and privacy**: which code/telemetry may be sent to the Claude API; redaction and retention policy.
- **Cost**: budget per capability and per repo; caching and smaller models for simple tasks.
- **False positives**: if the reviewer is noisy, the team ignores it; start with a few high-confidence rules.
- **Permissions**: the platform with least privilege in Azure DevOps (PRs and comments only, never push to main).
- **Hosting**: where it runs (Azure Container Apps, AKS, on-prem) according to company policy.
- **Ownership**: who maintains prompts, golden cases and the catalog.

## Verification (once implemented)
Golden cases per capability run in the platform's own CI; precision, false-positive, cost-per-run and latency metrics; mandatory human review during the first weeks.
