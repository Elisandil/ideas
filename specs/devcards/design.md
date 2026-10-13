# devcards — Design

SDD spec: [requirements](requirements.md) · [design](design.md) · [tasks](tasks.md) · [risks](risks.md)

## Architecture
```
Claude Code session (your subscription, Windows)
 ├─ /devcards:<command>  (skill = orchestration)
 │    ├─ devcards CLI (.NET 9, deterministic)  vault · guard · context · extract · lint · render · check · verify · ado
 │    ├─ writer subagent   → draft.json (JSON against the card schema)
 │    ├─ editor subagent   → JSON patch (read-only)                [v2]
 │    └─ reader subagent   → answers (card content only)           [v2]
 └─ SessionStart hook → devcards check --hook → stale count + card index   [v2]

Azure Repos clone (read-only)  ──►  vault: <vaultPath>/<org>/<project>/<repo>/
Azure DevOps REST (optional, PAT in Windows Credential Manager)
```
Core principle: **facts and markdown come from deterministic code; the model only fills small, budgeted, anchored slots in a JSON schema.**

## Plugin files
| Path | Contents |
|---|---|
| `.claude-plugin/plugin.json` | Name `devcards`, `userConfig` (below), version |
| `skills/{capture,pr,digest,map}/SKILL.md` | [v0] command orchestration |
| `skills/{flow,contract}/SKILL.md` | [v1] |
| `skills/{refresh,ask}/SKILL.md` | [v2] |
| `agents/writer.md` | Tools `Read`, `Grep`, `Glob`, `Write`; writes only `draft.json` in the run directory |
| `agents/editor.md` | Tool `Read` only; returns a JSON patch as its final message (R84) |
| `agents/reader.md` | No tools; the card content is passed inline (R87) |
| `hooks/hooks.json` | `SessionStart` exec-form hook: `dotnet <root>/tool/Devcards.Cli.dll check --hook --timeout-ms 3000` (R91–R95) |
| `bin/devcards`, `bin/devcards.cmd` | Shims on the Bash `PATH` that run `dotnet <plugin>/tool/Devcards.Cli.dll` |
| `tool/` | Framework-dependent .NET 9 build of `Devcards.Cli` |
| `schemas/<type>.schema.json` | One per card type, with `maxItems` / `maxLength` (R11, R12) |
| `templates/<type>.md` | Deterministic rendering templates (R13) |
| `rules/budgets.json`, `rules/banned-phrases.txt`, `rules/secret-patterns.txt` | Lint rules (R18–R22) |
| `evals/` | `claude plugin eval` cases over a sample BFF fixture |

**Source repo layout (`devcards/`):** `src/Devcards.Cli/` (Vault, Guard, Git, Ado, Facts, Lint, Anchors, Render, Check, Verify, Style, Runs), `tests/Devcards.Tests/` (xUnit, git fixture repos, WireMock.NET), `fixtures/SampleBff/` (a small BFF with controllers, minimal APIs, Refit, Polly, AutoMapper and manual mappers), `plugin/` (the files above).

## Configuration (`userConfig`)
| Key | Type | Default |
|---|---|---|
| `vaultPath` | directory | — (required, R1, R2) |
| `language` | string | `en` (R8) |
| `maxCodeLines` | number | `6` (R20) |
| `maxStyleExamples` | number | `3` (R101) |
| `indexTokenBudget` | number | `1500` (R95) |
| `verifyByDefault` | boolean | `false` (R86) |
| `solution` | file | auto: the only `.sln`/`.slnx` at the repo root, otherwise ask |

The Azure DevOps PAT is **not** a `userConfig` option. Plugin options do not reach commands run through Bash, and the PAT must never pass through the model. Instead, `devcards ado login` reads it from the terminal and stores it in Windows Credential Manager (R53, R57).

## Vault layout
```
<vaultPath>/<org>/<project>/<repo>/
├── index.md                      # map card (R81)
├── flows/ lineage/ contracts/ prs/ digests/ questions/
├── _facts/facts@<commit>.json    # R66
├── _style/examples/<type>/       # R100
└── _meta/
    ├── cards.json                # registry: id → path, anchors, section hashes, status (R17, R102)
    ├── state.json                # last digest commit, warnings already shown (R4, R5, R46)
    └── runs.jsonl                # R105
```
Vault key from the `origin` URL (R6). Supported formats: `https://dev.azure.com/<org>/<project>/_git/<repo>`, `https://<org>.visualstudio.com/<project>/_git/<repo>` and `git@ssh.dev.azure.com:v3/<org>/<project>/<repo>`.

## Card schemas (drafts)
The model fills only the fields marked ✍. Fields marked ⚙ are computed by the CLI and injected at render time.

**flow**: ⚙ `entry` (route + anchor) and diagram (from facts, R71).
- ✍ `steps[1..8]` {`text` ≤ 20 words, `anchor`}
- ✍ `failureModes[0..6]` {`downstream`, `policy`, `bffReturns`, `anchor`}, pre-filled ⚙ from policies (R72)
- ✍ `gotchas[0..5]` {`text` ≤ 30 words, `anchor`}

**lineage**: ⚙ `rows[]` {`field`, `type`, `source`, `transform` (`direct` / `convention` / `expression` / `?`), `anchor`} (R73–R75); ✍ `notes[0..3]` ≤ 25 words.

**contract**: ⚙ `route`, `auth`, `params[]`, `statusCodes[]` (from facts and OpenAPI, R76–R78); ✍ `statusCodes[].when` ≤ 15 words; ✍ `nullability[0..3]` ≤ 20 words.

**pr**: ⚙ `pr` {`id`, `title`, `author`, `mergedAt`}, ⚙ `contractChanges[]` [v1] (R79, R80), ⚙ `affectedCards[]` (R44).
- ✍ `behaviour[1..3]` {`text` ≤ 25 words, `anchor`}
- ✍ `config[0..5]` {`key`, `change`, `default`}
- ✍ `risk` {`level`, `reason` ≤ 25 words}

**digest**: ⚙ `entries[]` {`prId`, `title`, `area`, `author`, `attention`} (R46–R50); ✍ `entries[].summary` ≤ 15 words.

**q**: ✍ `question` ≤ 20 words, ✍ `answer[1..5]` {`text` ≤ 30 words, `anchor`}, ✍ `related[]` card ids.

**map**: ⚙ `projects[]` {`name`, `path`, `cards[]`} (R81, R82); ✍ `projects[].purpose` ≤ 15 words.

Every schema has a `notes` field that the model must leave `null`. It renders as the human-owned `## Notes` section (R98).

**Anchor:** `{ kind: code|pr|workitem|human, path?, lines?, symbol?, ref?, text? }`. The drafter supplies `path` + `lines` (+ `symbol` in v1). The CLI computes `hash` (R27) and rejects missing paths or lines (R28). Code anchors render as `vscode://file/<repoRoot>/<path>:<line>` (R32). The repo root comes from the registry, so moving the clone only requires re-rendering.

## Section ownership
```markdown
<!-- dc:begin steps owner=machine hash=3fa2c1 -->
| # | Step | Where |
…
<!-- dc:end steps -->
```
- `render` replaces only `owner=machine` sections and never touches `owner=human` ones (R97). `## Notes` is always human (R98).
- [v2] When `check` or `render` finds that a machine section's content hash differs from `hash`, the user edited it. The section flips to `owner=human` (R99) and the pair (generated, edited) is saved to `_style/examples/<type>/` (R100).
- Obsidian hides the HTML comments in reading view.

## CLI (`devcards`)
| Command | Purpose | Requirements |
|---|---|---|
| `vault` | Resolve and validate the vault for the current repo; print its path | R1–R7 |
| `guard begin\|end --run <dir>` | Snapshot and compare `git status --porcelain` | R33, R34 |
| `context capture\|pr\|digest\|map --run <dir> [args]` | Deterministic context JSON: commits, diff stats, changed lines, affected cards, authored files, projects | R40–R42, R44, R46–R52, R81, R82 |
| `extract [--commit <sha>]` | Roslyn facts with cache | R58–R68 |
| `facts diff <base> <head>` | Contract changes, additive or breaking | R79, R80 |
| `lint <draft> --type <t> --attempt <n>` | Schema, budgets, banned phrases, code blocks, anchors, secrets; writes `lint.json` | R12, R18–R22, R28 |
| `render --run <dir>` | Picks `edited.json` if its lint passed, otherwise `draft.json`; refuses if neither passed; writes the card and the registry | R13–R17, R24, R85, R97, R98, R102 |
| `check [--hook] [--timeout-ms]` | Anchors vs `HEAD`, relocation, stale marking, edit detection, card index | R29, R91–R95, R99 |
| `style --type <t>` | Top `maxStyleExamples` examples | R100, R101 |
| `verify questions\|score --run <dir>` | Fact-template questions; deterministic scoring; uncited sections | R86, R88–R90 |
| `ado login\|logout\|pr <id>` | PAT in Windows Credential Manager; read-only REST | R53–R57 |
| `runs append --run <dir>` | Observability record | R105, R106 |

The only module allowed to use `HttpClient` is `Ado` (R10, enforced by an architecture test).

## Generic pipeline (every command except `capture`)
1. `devcards vault` → stop on failure (R1–R7).
2. `devcards guard begin` (R33).
3. `devcards context <type>` and, in v1, `devcards extract` → `context.json`, `facts.json`.
4. [v2] `devcards style --type` → examples for the writer (R101).
5. **writer** subagent receives the run directory, writes `draft.json` and returns one line: `done -> draft.json`.
6. `devcards lint --attempt 1`. If it fails, the writer gets `lint.json` and retries; then `--attempt 2`. If that fails too, stop and show the violations (R23–R25).
7. [v2] **editor** subagent returns a JSON patch. The skill writes `edited.json` and lints it (R83–R85).
8. `devcards render` (R13–R17, R97, R98).
9. [v2] With `--verify` or `verifyByDefault`: `verify questions`, then the **reader** answers with the card passed inline, then the skill writes `answers.json`, then `verify score` (R86–R90).
10. `devcards guard end` (R34) and `devcards runs append` (R105).

Subagents return file references or JSON, never summaries, to avoid the "telephone game".

## Command specifics
- **capture** [v0]: the main session agent drafts the `q` card itself, because only it has seen the investigation; a fresh subagent would not. Steps:
  1. Check that the session read repo files (R37).
  2. Propose or accept the question (R35, R36).
  3. Run `context capture` → list similar `q` titles (R38).
  4. Draft, then lint, then render.
  5. [v1] If the investigation is a request flow, offer `/devcards:flow` (R39).
- **pr** [v0]: `context pr <id>` finds `Merged PR <id>:` (merge and squash merges) or uses the given range (R40–R42). The writer fills behaviour, config and risk (R43). `render` marks the affected cards `stale` (R44, R45). [v1] `facts diff` adds the contract changes (R79, R80). With a PAT, `ado pr` adds the description, reviewers and work items (R54, R56).
- **digest** [v0]: entries are computed from `git log --first-parent` since `state.json.lastDigestCommit`, or the last 7 days (R46, R47). The writer adds one ≤ 15-word summary per PR (R48). Areas, attention and links are computed (R49–R51). A shallow clone triggers a warning (R52).
- **flow** [v1]: resolve the route or symbol in the facts, or list the closest ones (R69, R70). The diagram, lineage rows and pre-filled failure modes come from the facts (R71–R75). The writer fills steps, gotchas and notes.
- **contract** [v1]: the skeleton comes from facts plus OpenAPI (R76, R77). The writer fills the `when` conditions and the nullability notes (R78).
- **map** [v0]: projects come from the `.sln`/`.slnx` (or a `*.csproj` scan) and their cards from the registry (R81, R82). The writer fills one purpose line per project.
- **refresh** [v2]: for each stale card, the writer receives the old card, the anchored diff and the facts, and regenerates only the machine sections (R96, R97).
- **ask** [v2]: search the vault index and the cards first and answer with anchors. If it had to read code, offer `/devcards:capture` (R103, R104).

## Default rules
- **Budgets**: as in the card schemas, plus a total of ≤ 250 words of prose per flow card, ≤ 150 per contract and q card, and ≤ 120 per PR card.
- **Banned phrases**: "it is important to note", "this section", "this document", "in summary", "in conclusion", "robust", "seamless", "leverage", "comprehensive", "plays a crucial role", "it's worth mentioning".
- **Secret patterns**: `Password=`, `AccountKey=`, `SharedAccessSignature=`, ADO PATs, JWTs, PEM blocks, `sk-ant-`.

## Error handling
| Situation | Behaviour |
|---|---|
| Vault not configured or inside the repo | Stop with instructions; nothing written (R2, R3) |
| Vault in a personal sync folder or a git repo with a remote | Warn once, remembered in `state.json` (R4, R5) |
| Lint fails twice | No card; violations shown (R24, R25) |
| Edited draft fails lint | Render the pre-edit draft (R85) |
| Roslyn semantic load fails (restore, private feeds) | Syntax-only analysis; facts `inferred`; warning in the card (R59) |
| No `Merged PR <id>:` commit | Ask for a commit range (R41) |
| ADO REST error or 401 | Git-only data; note in the card (R56) |
| `check` exceeds 3 s | Stop; one-line notice; the session continues (R94) |
| Repo changed during a command | Warning with the changed files (R34) |

## Rejected alternatives
1. **Model writes markdown directly with a "be concise" prompt.** This is the baseline that produces books. Schema + budgets + template is what makes length a constraint instead of a request.
2. **RAG or embeddings over the code.** Retrieval does not fix verbosity or invented facts; deterministic extraction does. The vault stays plain markdown plus a JSON registry: greppable, Obsidian-friendly, no vector store.
3. **Writer subagent for `capture`.** A fresh subagent cannot see the investigation, so it would re-investigate or invent. The main session agent drafts, and lint keeps it honest.
4. **PAT as a sensitive `userConfig` option** (as first sketched in `devcards.md`). Plugin options are not passed to commands run through Bash, and anything the skill could read the model could see. The credential store plus `devcards ado login` keeps the PAT out of the model entirely.
5. **Cards in the company repository.** This conflicts with the team culture and needs approval. The vault lives outside the repo; sharing selected cards is v3.
