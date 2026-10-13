# devcards — Risks, gaps and edge cases

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Compliance and data
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| D1 | Company code with a personal subscription | Sessions send BFF code to Anthropic under the Consumer Terms | Confirm the company's data policy; use a Team/Enterprise seat if available | High if the policy forbids it: then don't use it on company code |
| D2 | Vault leak through sync | Windows often redirects Desktop and Documents to a personal OneDrive; the vault syncs without the user noticing | R4 (personal sync folders), R5 (git repo with a remote), and documenting a local path such as `C:\work-notes` | Low-medium |
| D3 | Secrets copied into cards | A short code block or gotcha quotes a connection string | R20 (short code blocks) and R22 (secret patterns) | Low |
| D4 | PAT exposure | The PAT ends up in plugin config, the transcript or a log | Credential store via `devcards ado login` (R53), never in `userConfig` or Bash, R57, R106 | Low |

## Security
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| S1 | Writer writes into the repo | A prompt injection in a code comment ("update this file…") makes the writer use `Write` outside the run directory | Instructions, guard snapshot and warning (R33, R34); `render` only reads from the run directory | Low: detected, not prevented |
| S2 | Injected content in cards | A malicious comment gets copied into a gotcha or note | Anchors must resolve to real lines (R28), budgets, and personal-only use | Low |
| S3 | Hook runs untrusted code | The `SessionStart` hook runs in every repo the user opens | The hook runs only the plugin's own CLI, reads the repo, never executes repo code, and has a 3 s cap (R94) | Low |

## Extraction quality (v1)
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| E1 | Semantic load fails | Restore needs private Azure Artifacts feeds, or the solution doesn't load in `MSBuildWorkspace` | Syntax-only fallback with `inferred` (R59); checked in spike T1 | Medium |
| E2 | Dynamic wiring | Reflection, assembly scanning (Scrutor), source generators, endpoints registered in loops | `?` instead of guesses (R68, R74) | Medium |
| E3 | Mapping conventions | AutoMapper maps by name with no explicit `ForMember` | `convention` transform (R75) | Low-medium |
| E4 | Generated clients | NSwag or Kiota clients with huge generated files | Treat generated clients as outbound calls by type; exclude generated files from steps | Medium |
| E5 | Large solutions | Extraction takes minutes | Per-commit cache (R66, R67); extraction runs on demand, not in the hook | Low |

## Living docs
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| L1 | Stale false positives | A mass reformat or a namespace change breaks every hash | Whitespace-normalized hashes (R27), relocation (R29), symbol resolution in v1 (R30) | Medium on big refactors |
| L2 | Stale false negatives | Behaviour changes in code that no anchor covers | PR cards list affected cards (R44); digest attention (R50) | Medium: anchors only cover what was documented |
| L3 | Human edits lost | Refresh overwrites a fix the user made | Section ownership (R97–R99) | Low |
| L4 | Edit detection noise | Obsidian or a formatter rewrites whitespace and every section flips to human | Hash sections over normalized content | Low |

## Operations and usage
| # | Risk | Scenario | Mitigation | Residual |
|---|---|---|---|---|
| O1 | Quota use | Editor and verify passes on every card | Deterministic extraction; verify off by default; editor only after lint passes | Low |
| O2 | Context bloat | The injected index grows with the vault | `indexTokenBudget` (R95); index lists titles only | Low |
| O3 | Slow session start | `check` on a large vault | 3 s cap (R94); incremental checks against the last checked commit | Low |
| O4 | Plugin API changes | Exec-form hooks, agent tool declarations or `claude plugin eval` change | Pinned Claude Code version; spike T1 repeated on upgrades | Low |
| O5 | Card sprawl | Dozens of near-duplicate Q cards | Similar-title check on capture (R38); map and index | Medium |
| O6 | Abandonment | Cards stop being generated after the novelty wears off | v0 targets the daily habit (digest after pull, capture after an investigation); metrics in T21 | Medium: it is behavioural |

## Edge cases to cover in tests
- Azure Repos merge types: merge, squash and semi-linear leave `Merged PR <id>:`; rebase fast-forward does not, so the user is asked for a range (R41).
- Several `Merged PR` commits for the same id (a reverted and re-merged PR): use the latest and mention the others.
- Revert PRs (`Revert "Merged PR <id>…"`): link to the original PR card.
- Renamed or deleted anchored files: relocate by following renames; deleted → `stale` with the reason "deleted".
- Detached HEAD, several remotes (only `origin` is used), no remote (R7).
- Repos with several solutions: the `solution` option, otherwise ask.
- CRLF vs LF and UTF-8 BOM in anchored files (R27).
- Clone moved to another folder: the VS Code links are rebuilt from the registry on the next render (R32).
- A card file moved or renamed in Obsidian (R102); front matter edited by hand (registry wins, with a warning).
- Paths with spaces or Unicode in the vault or the repo.
- Shallow clones and partial history (R52).
- Very large PRs (hundreds of files): the writer receives diff stats and the top changed files, not the full diff.

## Open questions
1. Does company policy allow sending BFF code to Anthropic under a personal plan, and is there a Team/Enterprise seat?
2. Is a personal PAT with *Code (read)* and *Work Items (read)* acceptable to the company, or should v0 stay git-only?
3. Obsidian as the viewer, or plain markdown in VS Code? It only affects link style (wikilinks vs relative links).
4. Which BFF solution and which endpoints should be the first dogfooding targets?
