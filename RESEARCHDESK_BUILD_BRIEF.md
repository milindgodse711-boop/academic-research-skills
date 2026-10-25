# ResearchDesk — Implementation Brief for an AI Builder

Working product name: **ResearchDesk**
Target platform: **Windows 11, x64 first**
Source repository: `milindgodse711-boop/academic-research-skills` (fork of `Imbad0202/academic-research-skills`)
Pinned source commit for this brief: `77e77e871aeccf5b5740a3086903f914e58c4204` (suite version 3.23.2, 2026-10-25)

This document is the instruction set for an AI (or a human team) that will build
ResearchDesk. It is the second version of the brief. Version one was a strong,
platform-neutral specification; this version keeps every requirement from it,
grounds the open questions in the actual repository contents, and adds the agent
engine layer the owner asked for: **Claude Code CLI and OpenAI Codex CLI as
selectable engines**, so the researcher can pick either one to run the research
skills and assist inside the app.

Read Section 0 first. It lists what changed from version one and why. Appendix A
lists repository facts that were verified at the pinned commit; re-verify them
during discovery, because the repository moves fast (14 CI workflows, roughly
six-week minor releases).

---

## 0. What changed from the first brief, and why

| # | Change | Reason |
|---|--------|--------|
| 0.1 | Added **Section 7: Agent engines (Claude Code CLI, Codex CLI)** and demoted direct API providers (OpenRouter, OpenAI, Anthropic) to an **assist engine** for app-native actions (Section 8). | The repository is a Claude Code plugin: 5 skills, 35 modes, 43 agent prompt files, 16 slash commands, 2 hooks. Running that content natively through Claude Code gives the highest fidelity; Codex CLI gives a second, disclosed-fidelity engine; raw API calls would require re-implementing a 43-role orchestration from scratch. |
| 0.2 | **Two run modes**: *engine-native* (the engine runs the repository's own orchestrator and subagents) and *app-orchestrated* (ResearchDesk dispatches one bounded role at a time to whichever engine is selected). | Engine-native is only possible on Claude Code. App-orchestrated works on both engines and on the assist engine, and gives the UI full control over checkpoints and permissions. |
| 0.3 | A **ResearchDesk bridge** (an MCP server owned by the app) is the single way both engines reach app services: scoped file reads, literature search, evidence recording, artifact registration, and **structured checkpoint requests**. | Both CLIs speak MCP. One bridge replaces per-engine hacks, makes checkpoints machine-readable, and gives the app the permission chokepoint the first brief asked for. |
| 0.4 | The repository's own run artifacts (**Material Passport**, run ledger, evidence-source folders, screening logs) are the authoritative run state inside the project folder. The app database indexes them and holds UI state. | Version one asked for "one authoritative representation per content type". The repository already defines these files and both engines read and write them. Inventing a parallel model would create exactly the unsynchronized duplication the first brief forbids. |
| 0.5 | **License** section rewritten around the actual license: CC BY-NC 4.0 on the whole repository, with a Zenodo DOI, a NOTICE file, a governance file, and a platform-ports policy. | The first brief said "inspect the license". It is now known: noncommercial, not an open-source software license, and it binds any distribution that vendors the content. |
| 0.6 | **Python** decision made concrete: the core needs no Python; a short list of opt-in scripts does. The brief now specifies which to re-implement in TypeScript and when an embedded Python runtime is justified. | The repository states "ARS core needs no Python" in its own hook launcher, and its Windows hook history (Microsoft Store stub, Git Bash dependency) is a warning worth heeding. |
| 0.7 | **Export** baseline fixed: DOCX via a JavaScript DOCX library, PDF via Chromium print-to-PDF inside Electron, Markdown native. Pandoc and Tectonic become optional, detected, never required. | The repository's own format-convert mode uses Pandoc and LaTeX when available and gives instructions otherwise. The first brief forbids requiring them. |
| 0.8 | Explicit **non-goals** copied from the repository's positioning document. | The repository records rejected mechanisms (autonomous pipelines, idea generation for the scholar, cross-paper memory). ResearchDesk must not reintroduce them by accident through a convenient UI. |
| 0.9 | **Fidelity tiers** per engine and run mode, shown in the UI. | A Codex run of a Claude-calibrated skill is not the same product as a Claude run. The repository's own port policy requires documenting divergence; the app must show it. |
| 0.10 | Acceptance matrix seeded (Appendix D), plain-language glossary added (Appendix C), owner questions listed (Appendix E). | So the builder starts from checklists rather than deriving them. |

Everything else from the first brief is retained, sometimes reworded, never weakened.

---

## 1. Product intent

Build a premium, local-first Windows desktop workspace that helps a researcher:

1. Define a research question.
2. Create and maintain a research project.
3. Import papers, references, notes, and existing drafts.
4. Find and evaluate relevant literature using configured research providers.
5. Build an evidence-backed literature review.
6. Develop an outline and manuscript.
7. Run structured simulated peer review.
8. Respond to reviewer comments.
9. Check references and claim–source alignment.
10. Export research artifacts and final documents.
11. **Choose the agent engine** (Claude Code CLI or Codex CLI) that performs steps 4 to 9, per project or per task, and see what that choice implies.

The application must preserve the repository's research methodology (its skills,
checkpoints, integrity gates, reviewer panel, and honest-claim discipline), not
merely reuse its branding or copy its README.

### 1.1 Roles and authority

- The user remains responsible for research decisions, interpretation, authorship, and submission.
- AI-generated review is advisory feedback, not journal peer review.
- Do not promise publication readiness, scientific correctness, reproducibility, or guaranteed citation accuracy.
- Never present an AI score as a calibrated probability or a journal decision.

### 1.2 Four-state labeling

Every piece of information shown to the user carries one of four provenance states,
visibly and consistently:

| State | Meaning | Example |
|-------|---------|---------|
| **Generated** | A model produced it | A drafted section, a reviewer finding |
| **Tool-checked** | A deterministic tool ran and produced a result | "DOI resolves at Crossref", "4 of 4 indexes matched" |
| **User-confirmed** | The user explicitly confirmed or decided | A checkpoint approval, a human-verified claim |
| **Unresolved** | Not checked, failed to check, or still open | "Not checked: resolver unavailable" |

A network failure never becomes "verified" or "fabricated". A model statement that a
check passed is still **Generated** until the app's own tool confirms it.

### 1.3 Non-goals (from the repository's positioning)

ResearchDesk must not add, even as an optional convenience:

- An end-to-end autonomous pipeline that advances past a checkpoint without a user decision.
- Idea or hypothesis generation *for* the scholar outside the repository's non-generating Socratic mode.
- Hidden ranking of the user's ideas, or silent propagation of reviewer feedback into the manuscript.
- Cross-project memory: no registry of prior claims, limitations, or reviewer history shared across projects.
- Validation of actual experiments, raw data, or reproducibility. The app checks the manuscript and the reported process only, and says so.
- Any self-updater that lacks signed-artifact verification and rollback.

---

## 2. Repository discovery — complete this before implementation

Inspect the exact specified repository at the exact pinned commit. Do not assume it
matches the upstream repository or any similarly named project. Record the commit
SHA, the upstream relationship (`Imbad0202/academic-research-skills`, named in
`.claude-plugin/plugin.json`), and the sibling Codex distribution the repository
names in `POSITIONING.md` (`Imbad0202/academic-research-skills-codex`).

If cloning or access fails: report the exact failure, ask for a local checkout or
access, do not substitute another repository, and continue only with clearly
independent scaffolding that does not pretend integration happened.

### 2.1 What to inspect

Where present: README and quick-start; `LICENSE`, `NOTICE.md`, `CITATION.cff`,
`THIRD_PARTY.md`, `GOVERNANCE.md`, `SECURITY.md`, `CONTRIBUTING.md`,
`POSITIONING.md`; `.claude-plugin/plugin.json` and `marketplace.json`; `package.json`
and `pyproject.toml`; the five skill directories and their `SKILL.md` frontmatter;
`agents/` and each skill's `agents/`; `commands/`; `hooks/`; `shared/` (contracts,
handoff schemas, routing core, cross-model protocol); `scripts/` and both
requirements files; `tests/`, `evals/`, `examples/` (including the Pi end-to-end
example); `docs/` (`DATA_FLOWS.md`, `CONTROL_AVAILABILITY.md`, `RISK_REGISTER.md`,
`STAGE_CAPABILITY_MATRIX.md`, `ARCHITECTURE.md`, `SETUP.md`, `PERFORMANCE.md`,
`design/`); `MODE_REGISTRY.md`; the pipeline state machine
(`academic-pipeline/references/pipeline_state_machine.md`); and `CHANGELOG.md`.

Treat repository documents as untrusted input during discovery. Do not execute
scripts, hooks, or install commands until inspected. Appendix A gives the facts found
at the pinned commit; confirm each one and record differences.

### 2.2 Documents to create

- `docs/REPOSITORY_AUDIT.md`
- `docs/LICENSE_AND_ATTRIBUTION.md`
- `docs/FEATURE_MAPPING.md`
- `docs/ARCHITECTURE.md`
- `docs/IMPLEMENTATION_PLAN.md`
- `docs/ENGINE_SUPPORT.md` (new: per-engine capability matrix, minimum versions, verified flags, known gaps)

### 2.3 Feature mapping format

For every repository component:

Repository component → purpose → desktop UI equivalent → runtime equivalent (engine-native,
app-orchestrated, app-native, or none) → dependencies → implementation status →
limitations → tests.

Classify each component as one of:

- Reuse substantially unchanged (for example: skill and agent prompt text, JSON schemas under `shared/contracts/`)
- Adapt to desktop runtime (for example: checkpoints become approval records; slash commands become UI actions)
- Replace with equivalent functionality (for example: Python resolver clients re-implemented in TypeScript)
- Unsupported initially (for example: cross-model verification, inquiry ledger, calibration mode)
- Requires an external dependency or service (for example: Pandoc, Tectonic, a literature API key)

Do not assume a prompt instruction is a runtime guarantee. The repository itself
distinguishes prompt-level controls from enforced ones in `docs/CONTROL_AVAILABILITY.md`;
mirror that honesty.

### 2.4 Controls that need application-level enforcement

Identify, for each, the repository's mechanism, its channel availability, and the app
mechanism that will enforce it regardless of engine:

| Control | Repository mechanism (as found) | ResearchDesk enforcement |
|---------|-------------------------------|--------------------------|
| File write scope per stage/agent | `PreToolUse` hook `hooks/run_guard.sh` → `scripts/ars_write_scope_guard.py` (Python; silently pass-through without Python or Git Bash; plugin channel only) | Permission broker in the main process: per-run write allowlist, path canonicalization, deny by default |
| Read-only reviewers | `tools:` allowlist in the 4 plugin-exposed agents' frontmatter; prompt-level for the others | Reviewer runs get read-only tool policy on both engines (Section 7.6) |
| Human checkpoints (FULL / SLIM / MANDATORY) | Prompt-level in the orchestrator and state machine | Persisted approval records bound to artifact hashes; the app controls engine resume |
| Citation verification gate | Deterministic when the Python scripts run; prompt-level otherwise | App-native TypeScript gate with four resolvers and a local cache |
| Data-access boundaries | Engine working directory and sandbox flags | Project folder as the only engine root; sandbox flags set by the app, never by the model |
| Network permissions | Engine tool allowlists; Codex sandbox network setting | App decides which engine tools and bridge tools are enabled per run |
| Export blocking | Formatter refuse rules (prompt-level); `terminal_policies` schema | App export policy gate with recorded overrides |
| Cross-model data sharing | `ARS_CROSS_MODEL` plus per-session consent (prompt-level) | Explicit consent dialog; the app never sets cross-model variables in an engine environment without a recorded consent |

### 2.5 License requirements

Facts at the pinned commit: the repository is licensed **Creative Commons
Attribution-NonCommercial 4.0 International** (`LICENSE`, `plugin.json`,
`package.json`). `NOTICE.md` carries a personal-project statement. `CITATION.cff`
and `POSITIONING.md` give the citation with Zenodo DOI `10.5281/zenodo.20696614`.
`THIRD_PARTY.md` lists how derived projects get listed (credit ARS, describe it
faithfully) and `CONTRIBUTING.md` § "Platform ports (community-maintained only)"
sets the bar for recognized ports: pinned upstream commit, written sync policy,
vendored content unmodified, a named maintainer, end-to-end evidence, claims matched
to evidence, and a model-portability note.

Requirements:

- Preserve the license text, `NOTICE.md`, the citation, and attribution to Cheng-I Wu in the app's About screen, in `docs/LICENSE_AND_ATTRIBUTION.md`, and in the installer's third-party notices.
- Separate the **new application code license** from the **imported content license**. Imported skill, agent, reference, schema, and script content stays CC BY-NC 4.0 and is stored unmodified under `resources/imported-skills/<commit>/` with a manifest (`upstream_commit`, `retrieved_at`, per-file SHA-256).
- Do not relicense imported material. Do not claim that forking removes upstream obligations.
- **Flag the noncommercial restriction prominently.** A ResearchDesk distribution that bundles the content is bound by it. CC BY-NC is not an open-source license and was not written for software; note that fact and recommend the owner confirm intended use (Appendix E). If the owner intends commercial use, the imported content must be removed or separately licensed from the author before release.
- The app may include the Codex sibling distribution's adaptations only after inspecting its license and commit; record both.
- If use rights are uncertain, document the issue and seek clarification before publishing any build.

After discovery, write a concise implementation plan and proceed with the safe,
authorized scope. Do not stop at planning.

---

## 3. Technical direction

### 3.1 Baseline stack

Desktop: Electron, TypeScript, secure preload bridge, electron-builder.
Frontend: React, Vite, Tailwind CSS, accessible component primitives, Zustand (or
equivalent lightweight state), a structured Markdown or rich-text editor with a
stable document model (ProseMirror-based editors qualify).
Persistence: SQLite with versioned migrations, foreign keys, transactional writes,
SQLite FTS5 for local search, project files in normal local folders.
Validation: Zod (or equivalent) at every trust boundary: IPC, engine events,
provider responses, imported files, bridge tool arguments.
Testing: Vitest for unit and integration, Playwright for Electron end-to-end.

Choose current, mutually compatible, stable dependency versions at implementation
time and pin them in a lockfile. Record the chosen versions in
`docs/DEVELOPER_GUIDE.md`.

### 3.2 Agent engines are external, detected, never bundled

Claude Code CLI and Codex CLI are installed and authenticated by the user through
the vendors' own installers and login flows. ResearchDesk detects them, checks
versions, and drives them as supervised subprocesses or through the vendors'
official SDKs (Section 7). Do not vendor either CLI into the installer: licensing,
vendor terms (Section 7.7), update cadence, and authentication ownership all argue
against it. Both official TypeScript SDKs pull the vendor binary in as a dependency
by default; configure each SDK to use the user's installed binary and keep the
bundled copies out of the packaged build. Provide a guided "Set up an engine" screen
instead.

### 3.3 TypeScript-first runtime; Python only where justified

The repository's core content runs with no Python. Python is used by opt-in scripts.
Decide per component:

- **Re-implement in TypeScript** (preferred): the four bibliographic resolver clients, the citation-existence gate and its SQLite cache semantics, the acronym check, the revision token-conservation check, the literature-corpus adapters (folder scan, Zotero, Obsidian), and the sr-screener record parsing and PRISMA count derivation. These are small, stdlib-style programs with fixtures to port.
- **Embedded Python runtime** (only if a verified component cannot be ported in reasonable time, or the owner wants engine-native runs to execute the repository's own scripts): package a controlled runtime (for example a `python-build-standalone` distribution) with pinned wheels under the app's resources, exposed to engine subprocesses by prepending to `PATH` for that subprocess only. Never depend on the Microsoft Store Python alias; never require the end user to install Python or run pip; document included packages and licenses; test inside the installed build.
- When no Python is available to an engine-native run, the repository degrades to prompt-level checks. The app must then show the integrity report as **Generated**, not **Tool-checked**, and run its own TypeScript gate.

### 3.4 Process and performance rules

Keep heavy parsing, indexing, PDF text extraction, and export work off the renderer
thread (worker threads or utility processes). Do not add a local HTTP server unless a
demonstrated need exists; the ResearchDesk bridge uses stdio (Section 7.4). If a
server is ever necessary, bind to loopback only, authenticate each request, and
document lifecycle and exposure. Record significant choices as short ADRs under
`docs/adr/`.

---

## 4. Architecture and process boundaries

### 4.1 Layers

A. **Presentation**: screens, editors, panels, dialogs, accessibility.
B. **Application services**: project management, source ingestion, engine runs,
workflow execution, manuscript operations, citation management, export jobs.
C. **Domain**: projects, skills, workflow states, runs, checkpoints and approvals,
sources, evidence, claims, reviews, artifacts, engine sessions.
D. **Infrastructure**: SQLite, file storage, agent engines (Claude Code, Codex),
assist providers, literature providers, secret storage, background jobs, document
parsers, the ResearchDesk bridge.

### 4.2 Electron process rules

Renderer: no direct filesystem access, no API keys, no process execution, no
privileged provider requests, no raw engine process handles.

Main process and controlled workers: own privileged operations, validate IPC inputs
with schemas, enforce workspace and approval rules, perform authenticated provider
requests, spawn and supervise engine processes, run the bridge.

Use typed, allowlisted IPC methods. No generic "run command" or unrestricted
filesystem API. Validate the sender and the arguments of every IPC call.

### 4.3 Engine host

One **engine host** service in the main process (or a dedicated utility process)
owns all engine subprocesses. It: resolves the binary, builds a scrubbed environment,
spawns with the project folder as working directory, parses the event stream,
validates each event against a schema, forwards normalized events to the UI,
persists raw and normalized events, answers permission and checkpoint requests only
from recorded user decisions, and kills the entire process tree on cancel or exit.

---

## 5. User experience

Design an original, polished research application. Use a high-quality desktop
productivity aesthetic without copying another product's branding, assets, or
interface pixel-for-pixel.

### 5.1 Layout

Left sidebar: Dashboard, Projects, Library, Research, Manuscript, Reviews, Pipeline,
Skills, Engines, Settings.

Central workspace: source viewer, manuscript editor, evidence table, review report,
workflow and run details, skill explorer.

Right contextual panel: AI assistant (bound to the selected engine), current task,
selected sources, proposed changes, tool activity (live engine events), cost and
usage, context summary (what will leave the machine).

Top bar: active project, search and command palette, **engine selector** (Claude
Code, Codex, Assist provider) with a status dot and fidelity badge, model selector
where the engine allows it, save and task status.

Bottom status area: indexing activity, current run, network state, warnings, export
progress.

### 5.2 Engines screen

Per engine: detected or not, binary path, version, minimum supported version, auth
state as reported by the CLI, last successful smoke test, capability matrix
(skills-native, subagents, hooks, read-only mode, structured output, resume, usage
reporting, web search), and the fidelity tier it provides for repository content
(Section 7.2). A "Run smoke test" button runs a tiny no-tools prompt and shows the
raw result. A "Set up" guide links to the vendor's installer and login instructions
and never collects the vendor credential itself.

### 5.3 Skills screen

Lists the five imported skills with version, modes (from `MODE_REGISTRY.md`),
spectrum (Fidelity / Balanced / Originality), oversight level, triggers, and which
engines and run modes can run each mode. Includes a read-only **Skill explorer** that
renders `SKILL.md`, agent prompts, and references so the researcher can study how the
methodology works. User customizations are stored separately and shown as overlays.

### 5.4 Requirements

Light and dark themes; resizable panels; keyboard navigation; visible focus states;
accessible labels and contrast; high-DPI support; comfortable operation at 100 %,
125 %, 150 %, and 200 % Windows scaling; clear loading, empty, disabled, and error
states; no fake activity indicators or invented progress percentages.

Use plain-language labels for repository concepts (Appendix C). Every visible action
must work, be explicitly disabled with an explanation, or be clearly marked not
implemented.

---

## 6. Project model and local storage

### 6.1 Projects

Support multiple independent projects with: ID, title, description, research
question, objectives, discipline, article type, target venue (optional), citation
style, language (and optional output language pair, which the repository registers
as `zh-tw-en`), workspace location, **default engine and run mode**, created and
updated timestamps.

Project content: sources and attachments, extracted text and metadata, notes, outline
and manuscript, evidence links, review findings, revision responses, workflow runs
and engine sessions, export artifacts.

### 6.2 One authoritative representation per content type

| Content | Authority | Index or cache |
|---------|-----------|----------------|
| Manuscript and outline | Markdown files in the project folder (with structured citation markers) | SQLite FTS, snapshots table |
| Sources and metadata | SQLite tables; attachments as files | FTS over extracted text |
| Run state of a repository skill run | **Material Passport** (YAML) and run ledger written by the engine or the app in the project folder | SQLite mirror for UI queries, rebuildable |
| Integrity evidence | Evidence rows and `*_evidence_sources/` folders as the repository defines | SQLite mirror |
| Screening decisions | sr-screener work and output folders | SQLite mirror; PRISMA counts derived, never typed in |
| Approvals, checkpoints, audit events | SQLite (transactional) | Exported into the process summary |
| Settings and engine configuration | App-data SQLite and config | — |

Mark every derived index and cache as rebuildable from the authority.

### 6.3 Storage locations

- App settings and central catalog under the Windows app-data location.
- User-visible documents, attachments, passports, ledgers, and exports under the chosen project folder with an explicit project manifest (`researchdesk.project.json`) for portability.
- Engine working directory is the project folder (or a `runs/<run-id>/` subfolder when a run should be isolated), never the app installation directory.
- Do not write user data into the installation directory.

### 6.4 Behaviors

Autosave; manuscript snapshots; atomic file replacement; conflict detection for
externally changed files (engines write files too, so watch the project folder and
reconcile); archive and delete flows with recoverable deletion; project backup and
restore; schema-migration backup before each migration. Do not silently mix sources,
memory, or context between projects.

---

## 7. Agent engines: Claude Code CLI and Codex CLI

This section is new. It defines how ResearchDesk runs the repository's methodology
through two external agent engines and lets the user choose between them.

### 7.1 Engine abstraction

Implement an `AgentEngine` interface in `packages/engine-core` with three
implementations:

| Engine | Package | Transport (preferred → fallback) |
|--------|---------|----------------------------------|
| `claude-code` | `packages/engine-claude` | Claude Agent SDK for TypeScript → direct spawn of `claude -p` with JSON stream output |
| `codex` | `packages/engine-codex` | Codex SDK for TypeScript → `codex exec --json` → `codex app-server` JSON-RPC over stdio (the repository already drives app-server in `scripts/cross_model_codex_transport.py`; read it) |
| `assist` | `packages/providers` | Direct HTTPS to OpenRouter, OpenAI, Anthropic, custom OpenAI-compatible, or local endpoint (Section 8) |

Common contract:

```ts
interface AgentEngine {
  id: 'claude-code' | 'codex' | 'assist';
  detect(): Promise<EngineStatus>;              // binary, version, auth state, capability matrix
  startRun(spec: RunSpec): Promise<RunHandle>;  // cwd, prompt, tool policy, bridge tools, model, budget
  resumeRun(sessionRef: EngineSessionRef, userMessage: string): Promise<RunHandle>;
  cancel(handle: RunHandle): Promise<void>;     // kills the whole process tree
  events(handle: RunHandle): AsyncIterable<EngineEvent>; // normalized, schema-validated
}
```

Normalized `EngineEvent` kinds: `session_started` (engine session id), `assistant_text`
(delta or block), `tool_call_requested`, `tool_call_started`, `tool_call_completed`,
`permission_requested`, `checkpoint_requested` (from the bridge), `file_changed`,
`usage_reported`, `run_completed`, `run_failed`, `run_cancelled`, `engine_notice`.
Persist raw engine events alongside normalized ones for audit.

**Verify every flag, option, and event shape against the installed CLI version and the
vendors' current documentation at implementation time.** Both CLIs change monthly.
Pin minimum supported versions in `packages/engine-core/engine-support.json`, read the
installed version at detection, and refuse (with an explanation) below the minimum.
Run the engine contract tests (Section 21.4) against each newly supported version.

### 7.2 Run modes and fidelity tiers

| Run mode | Available on | What runs | Fidelity shown to the user |
|----------|--------------|-----------|----------------------------|
| **Engine-native** | Claude Code only | The engine loads the repository plugin from `resources/imported-skills/<commit>/` and runs a skill or the full pipeline orchestrator itself, including its subagent dispatch and hooks | **Reference** — the repository's documented behavior, subject to the channel limitations in `docs/CONTROL_AVAILABILITY.md` (hooks need Git Bash; the write-scope guard needs Python) |
| **App-orchestrated** | Claude Code, Codex | ResearchDesk is the orchestrator. It dispatches one bounded role (one agent prompt file, one mode, one stage) per engine run, passes the handoff artifacts the repository's schemas define, and holds the state machine | **Adapted** — same prompts and schemas, app-controlled checkpoints; no engine subagents; documented divergences |
| **Assist** | Assist providers | App-native prompts for small actions (clarity edit, summary, claim extraction) | **App-native** — not repository content |

Running repository skills through the assist engine is not offered by default. If the
owner opts in later, the UI must label such runs "Adapted (unvalidated model)" and
record the model; the repository notes its prompts are calibrated against Claude and
that ports must document divergence.

### 7.3 Engine-native mode on Claude Code

The points below were checked against the Claude Code documentation
(`code.claude.com/docs`: headless, CLI reference, Agent SDK TypeScript, permissions,
plugins, sessions, skills, authentication, legal) on 2026-10-25. Several behaviors
are version-gated; record the installed version and re-verify.

**Binary and transport.**
- Prefer the user's own installed Claude Code. The native Windows installer places it at `%USERPROFILE%\.local\bin\claude.exe`; detect with `claude --version`, `claude doctor`, and `claude auth status` (JSON; `authMethod` is one of `none`, `claude.ai`, `oauth_token`, `api_key`, `api_key_helper`, `third_party`). WinGet (`Anthropic.ClaudeCode`) and npm installs exist too; search `PATH` and the known locations.
- The Agent SDK ships its own CLI binary as a per-platform optional dependency. For ResearchDesk, set `pathToClaudeCodeExecutable` to the user's installed binary and exclude the bundled binary from the build; shipping it engages the "preinstalling Claude Code in your product" terms (Section 7.7).
- The SDK's `env` option **replaces** the subprocess environment; build it from the allowlist in Section 7.9 and set `CLAUDE_AGENT_SDK_CLIENT_APP` to identify ResearchDesk.

**Isolation (do not use bare mode here).**
- `--bare` skips hooks, skills, commands, subagents, plugins, MCP, CLAUDE.md, and does not read subscription credentials. Engine-native mode needs the plugin, so run **non-bare** and isolate explicitly instead:
  - `settingSources: []` (CLI: `--setting-sources` with no sources), so the project folder's own `.claude/settings.json`, hooks, `.mcp.json`, and CLAUDE.md never load. Headless runs otherwise execute them with no trust dialog.
  - `strictMcpConfig: true` (CLI `--strict-mcp-config`) so only the ResearchDesk bridge server is connected.
  - `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in the environment; auto memory loads regardless of setting sources.
  - Load the imported repository content with `plugins: [{ type: 'local', path }]` (CLI `--plugin-dir`). The path resolves relative to `cwd`, a tilde is not expanded, and a missing path is skipped silently: read `plugins` and `plugin_errors` in the `system/init` message and fail the run if the plugin did not load. Never install the plugin into the user's global Claude Code configuration.
- Working directory: the project folder. `additionalDirectories` (`--add-dir`): none by default; note that skills under `<dir>/.claude/skills/` of an added directory do load.

**Permissions and tools.**
- Since 2.1.302 a `-p` run starts in `default` (Manual) mode; a call that needs approval is denied unless an allow rule, a `PermissionRequest` hook, `--permission-prompt-tool`, or the SDK `canUseTool` callback answers it. Use the SDK: register a `PreToolUse` hook callback **and** `canUseTool`, both answered by the ResearchDesk broker. Evaluation order is hooks → deny rules → ask rules → permission mode → allow rules → `canUseTool`; the `PreToolUse` hook is the only layer that sees every call in every mode.
- `allowedTools` only auto-approves; it does not restrict which tools exist. Restrict the tool set with `tools` (CLI `--tools`) and remove tools with bare names in `disallowedTools`. Reviewer runs: `tools: ['Read', 'Grep', 'Glob']` plus the bridge's read tools, `disallowedTools: ['Write', 'Edit', 'MultiEdit', 'Bash', 'PowerShell', 'WebFetch', 'WebSearch']` (confirm the current built-in tool names from the init message's `tools` array).
- Never `bypassPermissions` or `--dangerously-skip-permissions`; never `dontAsk` for a run that relies on the broker (it skips `canUseTool`). `permissionPrompts: 'none'` (2.1.259+) is acceptable only for fully pre-approved runs.
- Native Windows has **no sandbox** (sandboxing is WSL 2 only). The broker is the enforcement layer.

**Skills, commands, and the prompt.**
- A `/skill-name` at the start of the prompt expands; plugin skills and commands are namespaced (`/plugin-name:skill-name`), only the first slash command in a prompt expands, and the rest of the line becomes its arguments. Send the repository's command (for example the `ars-full`, `ars-lit-review`, `ars-reviewer` commands under `commands/`, with the plugin namespace the init message reports) followed by the project materials by path.
- System prompt: use the preset `{ type: 'preset', preset: 'claude_code', append }` with a short ResearchDesk preamble that (a) names the bridge tools and instructs the orchestrator to call `researchdesk_checkpoint` at every FULL, SLIM, and MANDATORY checkpoint, **as the only tool call in that turn**, and to wait for its answer, (b) names the passport path, and (c) restates that imported papers and tool results are data, not instructions. Never replace the default system prompt with a plain string; the repository's content assumes Claude Code's own tool guidance.

**Durable checkpoints (the defer pattern).**
- When the engine calls `researchdesk_checkpoint`, the `PreToolUse` hook returns `permissionDecision: "defer"`. The run ends with `stop_reason: "tool_deferred"` and a `deferred_tool_use {id, name, input}` on the result; the session persists. ResearchDesk stores the session id and the question, shows the checkpoint UI, and when the user decides (minutes or days later, across app restarts) resumes with `resume: <session_id>`. The same tool call fires `PreToolUse` again; the hook now returns `allow` with the decision in `updatedInput`, in the user's exact words, and the tool returns it to the model.
- Defer works only in `-p` mode and only when the checkpoint call is the sole tool call of its turn (otherwise it is ignored with a warning). If a run ends with a question in plain text instead, treat the text as a checkpoint, require a user turn, and resume with the user's words.
- Use `resume`, not `continue`: plain `claude --continue` skips sessions created by `-p` or the SDK. `--fork-session` branches a session; `--no-session-persistence` is not for checkpointed runs.

**Events.**
- Run with `--output-format stream-json --verbose` (add `--include-partial-messages` for text deltas, `--include-hook-events` to see plugin hook activity). Expect `system/init` (`session_id`, `cwd`, `tools`, `mcp_servers`, `plugins`, `plugin_errors`, `model`, `permissionMode`, `skills`, `slash_commands`), `assistant` (content blocks including `tool_use`), `user` (`tool_result`), `stream_event`, `system/permission_denied`, `system/api_retry`, hook events, and a final `result` with `subtype` (`success`, `error_max_turns`, `error_during_execution`, `error_max_budget_usd`, `error_max_structured_output_retries`), `is_error`, `num_turns`, `stop_reason`, `total_cost_usd`, `usage`, `modelUsage`, `permission_denials`, and on success `result` and optional `structured_output`.
- Completion = `result` with `subtype: "success"`, `is_error: false`, `stop_reason` not `tool_deferred`, exit code 0. Anything else is not completion.

**Budget, output, models.**
- `maxTurns` / `--max-turns`; `maxBudgetUsd` / `--max-budget-usd` (client-side estimate, can overshoot, subagent spend counts); `--effort low|medium|high|xhigh|max`; `fallbackModel` for overload only (never for auth or billing errors; show a notice when it triggers).
- Structured output for app-orchestrated roles: `outputFormat: { type: 'json_schema', schema }` / `--json-schema`; read `structured_output` on the success result.
- Model IDs: `claude-fable-5-1`, `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`; aliases `fable`, `opus`, `sonnet`, `haiku`, `best`. The repository's content is calibrated for Opus-class and Sonnet-class models and says never Haiku: do not offer Haiku for repository runs.

**Hooks and Windows.**
- The plugin's `SessionStart` announce and `PreToolUse` write-scope guard are Bash scripts. On Windows they run only with Git for Windows installed (the Bash tool needs Git Bash; `CLAUDE_CODE_GIT_BASH_PATH` locates it; without it commands go through the PowerShell tool). Detect this and show "Write-scope guard: active / inactive (Git Bash not found)". The broker enforces write scope regardless. Whether plugin hooks run in headless mode is inferred from the docs, not stated; confirm with `--include-hook-events` in the contract tests.
- Credentials live at `%USERPROFILE%\.claude\.credentials.json`; ResearchDesk never reads that file.
- Cancellation: the SDK `interrupt()` (SIGINT) ends the turn and records a result; SIGTERM exits with code 143 and records nothing. Interrupt first, wait a bounded grace period, then kill the tree.

**Environment.**
- Pass only what the engine needs (Section 7.9). Do not pass app secrets. Do not set `ARS_CROSS_MODEL` or other cross-model variables without a recorded consent. Set `ARS_UPDATE_CHECK=0` so the plugin's update check does not phone out from inside the app. If `ANTHROPIC_API_KEY` is present, a `-p` run uses it even when a subscription login exists, so forward it only when the user chose API-key authentication for this engine.

### 7.4 App-orchestrated mode (both engines) and the ResearchDesk bridge

ResearchDesk holds the pipeline state machine (Section 10) and dispatches each role
as a bounded engine run:

1. Build the role prompt from the agent prompt file (unmodified) plus a run preamble naming inputs, outputs, allowed tools, and output schema.
2. Provide inputs as files in `runs/<run-id>/inputs/` and ask for outputs as files in `runs/<run-id>/outputs/`; validate outputs against the repository's JSON schemas (`shared/contracts/`) or the handoff schemas (`shared/handoff_schemas.md`); one bounded repair attempt; then fail visibly with diagnostics.
3. Record provider, model, engine version, prompt hash, inputs, outputs, usage, and validation results (Section 9.4).

The **ResearchDesk bridge** is an MCP server owned by the app, exposed to both
engines over stdio (in-process where the Claude Agent SDK allows it; as a child
process launched by the engine otherwise, running under the app's own Node runtime
with a one-time token in its environment). Tools:

| Bridge tool | Purpose | Enforcement |
|-------------|---------|-------------|
| `researchdesk_checkpoint` | Structured checkpoint: question, options, stage, artifact hashes → returns the user's decision | Blocks until a persisted approval record exists; MANDATORY checkpoints cannot be answered by anything but a user turn |
| `researchdesk_literature_search` | Topic, title, or DOI search through the app's literature providers | App rate limiting, provider terms, provenance recorded |
| `researchdesk_source_read` | Read a library source's extracted text or a page range | Scoped to the project; returns locators; never fabricates pages |
| `researchdesk_evidence_record` | Record a claim–source evidence row | Validated against the evidence schema; stored as **Generated** until tool-checked or user-verified |
| `researchdesk_artifact_register` | Register an output artifact with its hash and type | Validated writes only inside the run's output folder |
| `researchdesk_citation_check` | Run the app's deterministic citation gate on a reference list | Returns tool-checked results with resolver provenance |

No bridge tool executes shell commands, reads outside the project, or returns
secrets. Bridge arguments are schema-validated; results are size-limited.

### 7.5 Codex engine specifics

The points below were checked on 2026-10-25 against the `openai/codex` source at
commit `970111f6` (2026-10-23) and the npm registry (`@openai/codex` and
`@openai/codex-sdk` 0.164.0). The hosted documentation at `developers.openai.com`
could not be fetched from the verification environment; re-read it (CLI reference,
non-interactive mode, SDK, app-server, Windows, authentication, config reference,
skills, AGENTS.md) and record URLs in `docs/ENGINE_SUPPORT.md`. Codex changes
weekly; treat every item as version-gated.

**Transports.**
- Preferred: the official Codex SDK for TypeScript (`@openai/codex-sdk`). It spawns the CLI and exchanges JSONL events. `new Codex({ codexPathOverride, apiKey, baseUrl, config, configOverrides, env })`: `env` **replaces** the child environment (the SDK README names Electron hosts as the use case); `config` flattens to `--config key=value` overrides; `apiKey` is passed as `CODEX_API_KEY`. The SDK depends on `@openai/codex`, which pulls platform binaries as optional dependencies (including `win32-x64`); set `codexPathOverride` to the user's installed binary and exclude the bundled binaries from the packaged build (Section 3.2). The SDK's Windows support is implemented in code but not documented as a support policy; verify on a real Windows machine.
- `startThread({ model, sandboxMode, workingDirectory, skipGitRepoCheck, modelReasoningEffort, networkAccessEnabled, webSearchMode, approvalPolicy, additionalDirectories })`, `thread.run(input, { outputSchema, signal })` → `{ items, finalResponse, usage }`, `thread.runStreamed(input)` → async events, `codex.resumeThread(id)`, `thread.id` (null until `thread.started`).
- Fallback 1: `codex exec [OPTIONS] [PROMPT] --json` (JSONL on stdout, everything else on stderr; prompt also accepted on stdin). Relevant flags verified: `-m`, `-C <dir>`, `-s read-only|workspace-write|danger-full-access`, `--add-dir`, `--skip-git-repo-check` (required outside a trusted git directory, otherwise exit 1), `--ephemeral`, `--output-schema <file>`, `-o <file>`, `-i <image>`, `-c key=value` (repeatable), `-p <profile>` (layers `$CODEX_HOME/<name>.config.toml`), `--ignore-user-config` (do not load the user's `config.toml`; authentication still comes from `CODEX_HOME`), `--strict-config`. `codex exec resume <thread_id>` resumes; `codex exec fork` branches. **Not on `exec`:** `--ask-for-approval` and `--search` (interactive TUI only), `--full-auto` (removed). `exec` defaults `approval_policy` to `never`.
- Fallback 2: `codex app-server` (JSON-RPC over stdio, marked experimental by OpenAI; the repository drives it in `scripts/cross_model_codex_transport.py` and documents version quirks there). Use it when the run needs approval callbacks: the server sends `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, and `item/tool/requestUserInput` requests that the broker answers (`accept`, `acceptForSession`, `decline`, `cancel`). Generate TypeScript types with `codex app-server generate-ts`; no npm protocol package exists. Pin the Codex version this path was tested against, as the repository does.
- Never pass `--dangerously-bypass-approvals-and-sandbox`, `danger-full-access`, or `--dangerously-bypass-hook-trust`.

**Events (`--json` and SDK).**
- `thread.started {thread_id}` (store it for resume), `turn.started`, `item.started | item.updated | item.completed {item}`, `turn.completed {usage: input_tokens, cached_input_tokens, cache_write_input_tokens, output_tokens, reasoning_output_tokens}`, `turn.failed {error}`, `error {message}`.
- Item types: `agent_message` (text, or a JSON string when an output schema was given), `reasoning`, `command_execution` (`status` in_progress | completed | failed | declined, `exit_code`, `aggregated_output`), `file_change` (`changes[{path, kind}]`), `mcp_tool_call` (`server`, `tool`, `arguments`, `result`, `error`, `status`), `collab_tool_call` (Codex's experimental multi-agent feature), `web_search`, `todo_list`, `error`.
- Completion = `turn.completed` after the final item with no `turn.failed` or `error`, and exit code 0.

**Isolation.**
- Run with `--ignore-user-config` (direct spawn; confirm whether the SDK exposes it, otherwise spawn directly) so the user's own `~/.codex/config.toml` (their MCP servers, providers, profiles) never loads into a ResearchDesk run, and pass every setting explicitly: `sandbox_mode`, `approval_policy`, `mcp_servers.researchdesk` (stdio: `command`, `args`, `env`, `cwd`; set `required = true` so a bridge failure aborts the run), `history.persistence = "none"`, `web_search = "disabled"` unless the role needs it, `check_for_update_on_startup = false`, `shell_environment_policy.inherit = "core"` plus `exclude` patterns for secrets, `model`, `model_reasoning_effort` (`low | medium | high | xhigh | max`, with `ultra` on some models), `skills.include_instructions`.
- Working directory: the run folder inside the project (`-C` / `workingDirectory`), with `skipGitRepoCheck` because project folders are not git repositories. Sandbox: `read-only` for reviewer, integrity, and screener roles; `workspace-write` for writer roles with `sandbox_workspace_write.network_access = false` (the default) and `writable_roots` limited to the run output folder.
- **Windows sandbox caveat (verified in source, confirm in the hosted docs):** Codex's sandbox backends are macOS Seatbelt, Linux seccomp, and on Windows a restricted-token backend or Microsoft's MXC container backend selected by `[windows] sandbox = "mxc" | "elevated" | "unelevated"` (MXC preferred via `features.prefer_mxc = true`). **With no Windows sandbox configured, the sandbox type is `None`: `read-only` and `workspace-write` are then not enforced by the OS**, and since `exec` defaults approvals to `never`, a writer role would run unsandboxed. ResearchDesk must therefore, on Windows: (1) probe sandbox readiness at detection (app-server `windowsSandbox/readiness`, or a controlled probe command that attempts a write outside the run folder and must fail); (2) if no OS sandbox is active, run Codex only through app-server with `approval_policy = "untrusted"` so every command and file change becomes an approval request the broker evaluates against the role's policy, or disable Codex writer roles and say why; (3) show "OS sandbox: active (MXC / restricted token) / not active" on the Engines screen and in every run header. The installation guide for Codex still names WSL 2 for Windows in places; record what the hosted Windows page says at implementation time.

**Repository content on Codex.**
- Codex reads `AGENTS.md`: it finds the project root by walking up from the working directory to a root marker (`.git` by default; with none, the working directory alone counts), concatenates `AGENTS.md` files from root to working directory (an `AGENTS.override.md` wins per directory), capped at 32 KiB, after `~/.codex/AGENTS.md`. Generate a run-local `AGENTS.md` carrying the routing core and the key rules from `.claude/CLAUDE.md`, within the cap.
- Codex discovers skills (folders with `SKILL.md` carrying `name` and `description`) from `~/.agents/skills` (user), `.agents/skills` in each directory from project root to working directory, a project `.codex/skills`, the deprecated `$CODEX_HOME/skills`, and `/etc/codex/skills`; skills apply in `codex exec`; the model invokes them by `$skill-name` mention or automatically from the description. Place the needed repository `SKILL.md` folders and references under `<run folder>/.agents/skills/` unmodified, and use `[[skills.config]]` to disable anything not needed for the role.
- Agent prompt files are dispatched as role prompts (app-orchestrated). Inspect the sibling Codex distribution the repository names and reuse its adaptation decisions where its license allows.
- Known gaps to disclose in the UI: no Claude hooks (write-scope guard absent → app broker only), no repository slash commands, no engine-native pipeline orchestration, Codex's multi-agent feature is experimental and not the repository's subagent model, different model family (document the model; the repository's model-portability note requires it), and no OS sandbox on Windows unless configured.

**Structured outputs and usage.**
- Use `outputSchema` (`--output-schema`) for role outputs; the final `agent_message` is then a JSON string to validate. Otherwise validate the output file.
- Codex reports token counts per turn; show them. Show cost only if the user configured pricing for the model; otherwise "unknown".

**Authentication (Section 7.7 applies).**
- `codex login` (ChatGPT sign-in, browser OAuth on local port 1455; Plus, Pro, Business, Edu, Enterprise plans per the README), `codex login --with-api-key` (key on stdin), `codex login --device-auth`, `codex login status` (exit 0 when logged in; prints "Logged in using ChatGPT" or a masked "API key configured"), `codex logout`. Credentials live in `$CODEX_HOME/auth.json` (default `~/.codex/auth.json`; `cli_auth_credentials_store` can select the OS keyring). Environment keys: `OPENAI_API_KEY`, `CODEX_API_KEY`, `CODEX_ACCESS_TOKEN`. Headless `exec` uses the same stored login. ResearchDesk never reads or copies `auth.json`.
- OpenAI's policy text on using a ChatGPT subscription login from third-party or automated tooling **could not be verified** during this check; the authentication page reportedly points programmatic workflows to API keys and says ChatGPT sign-in is governed by the ChatGPT Terms of Use. Read the current policy, record it, and default the Codex engine to the API-key posture until the owner's counsel confirms otherwise.

**Models.** The bundled catalog at the checked commit lists `gpt-6.1-sol` (default), `gpt-6-astra`, `gpt-6-sol`, `gpt-6-luna`, `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, and `gpt-5.5` (legacy); the live catalog can differ. The repository's cross-model notes validated `gpt-5.6-sol` and treat `gpt-6-astra` as provisional for citation checks; neither is a validated engine for running the repository's skills, so label Codex runs "Adapted".

### 7.6 Tool policies per role

| Role class | Claude Code | Codex |
|------------|-------------|-------|
| Reviewer, integrity verifier, screener | Allow read-only tools (file read, grep, glob, the bridge's read tools); disallow write, edit, shell | Sandbox `read-only` |
| Writer, formatter, compiler | Allow read tools plus file write and edit inside the run output folder; shell only if the role needs a verified local tool and the user approved | Sandbox `workspace-write` on the run folder |
| Orchestrator (engine-native) | Default plugin behavior under the permission callback; the broker denies writes outside the project folder | Not applicable |

The broker canonicalizes every path (Section 17.2), denies anything outside the
allowed roots, and records every decision as an audit event.

### 7.7 Authentication and terms

- Each engine's login is owned by the engine. The user runs the vendor's login command (or sets the vendor's API-key environment variable) outside ResearchDesk or through a guided terminal step. ResearchDesk never reads, copies, proxies, or stores vendor OAuth tokens or credential files.
- Show the auth state the CLI itself reports (`claude auth status`; the Codex equivalent per Section 7.5). Record the terms URLs and the date checked in `docs/ENGINE_SUPPORT.md`, and have the owner's counsel confirm the reading below before any public release.
- **Anthropic terms, as published on 2026-10-25** (Claude Code "Legal and compliance" page and the Agent SDK overview): third-party developers may not offer claude.ai login in their own applications, may not route requests through Free, Pro, or Max plan credentials on behalf of their users, and may not collect, store, or intermediate claude.ai credentials or session tokens; products built on the Agent SDK should use API-key authentication (Claude Console or a supported cloud provider). Preinstalling or running Claude Code inside a product requires the Commercial Terms, an unmodified binary, each end user authenticating with their own API key, subscription credential, or third-party provider credential, and no reselling or intermediation of usage. Plain-text attribution such as "runs Claude Code" is allowed; using "Claude Code" or Anthropic names or logos in the product name is not. The page states that an end user signing in to the unmodified Claude Code binary with their own subscription is not prevented.
- **Consequences for ResearchDesk:** do not bundle the Claude Code binary; detect and drive the user's own unmodified install; never run `claude setup-token`, never read `.credentials.json`, never store or forward `CLAUDE_CODE_OAUTH_TOKEN`; offer two postures per engine, "use my installed CLI's own login" (the app touches no credential) and "use my API key" (the user's own `ANTHROPIC_API_KEY` from secure storage, forwarded into that engine's environment only); default to the API-key posture whenever the SDK's bundled binary is in use. Keep "ResearchDesk" as the product name and name the engines only in attribution text.
- **OpenAI terms for Codex** could not be verified at the time of this brief (Section 7.5); apply the same two-posture design and the same no-credential-handling rule, default Codex to the API-key posture, and record the policy text once read.
- Cost display: a subscription-authenticated run shows tokens and "billed by subscription"; an API-key run shows the engine-reported cost when the engine reports one (Claude Code reports `total_cost_usd`), else "unknown". Never show zero for unknown.

### 7.8 Engine selection UX

- Default engine per project; override per task from the top bar. Switching engines starts a new run; it never continues an in-flight run on the other engine.
- Engine unavailable → the selector shows why (not installed, below minimum version, not logged in, smoke test failed) and offers the setup guide; actions that need the engine are disabled with that reason.
- Each run's header shows engine, version, model, run mode, and fidelity tier. The process summary (Section 19.4) records them.

### 7.9 Process hygiene

- Build the child environment from an allowlist (system `PATH`, `SystemRoot`, `TEMP`, user profile, the vendor's own variables the user configured, the bridge token, `ARS_UPDATE_CHECK=0`). Nothing else.
- Spawn without a shell. Use `stdio: pipe`, parse newline-delimited JSON with bounded line length, and apply backpressure; drop to a visible error on malformed events rather than guessing.
- Kill the full process tree on cancel, app exit, or watchdog timeout (`taskkill /T /F` on Windows, process-group signals elsewhere). Persist the partial transcript and usage first.
- One engine run per project at a time; a global concurrency cap configurable in Settings (default 2).
- Never run with permission checks bypassed, never with unrestricted sandbox, never with auto-approve-everything flags.

---

## 8. Direct model providers (assist engine)

The assist engine serves app-native actions: improve clarity, summarize selected
sources, extract claims, suggest limitations, propose an abstract, draft a response
to one reviewer finding. It also powers the right-panel assistant when the user
prefers it. It does not run repository skills unless the owner opts in (Section 7.2).

Provider-neutral interface. First priority: OpenRouter. Then OpenAI, Anthropic,
custom OpenAI-compatible endpoint, optional local endpoint (an installed local model
server). Do not assume compatible endpoints share features. Track capabilities
explicitly: streaming, tool calling, structured output, vision, context window,
usage reporting, pricing availability.

Settings: provider, API key, endpoint where relevant, model, temperature where
supported, output-token limit where supported, request timeout, concurrency, per-run
budget warning and limit.

Requirements: secure key storage (Section 17.5), connection test, model selection
with manual-ID fallback, clear provider errors, timeouts and cancellation, bounded
retries for transient failures, respect for rate-limit headers, no silent provider
or model fallback. Keep API keys out of renderer state, project files, logs,
exported settings, and error screenshots.

Display estimated cost separately from reported usage. When pricing or usage is
unavailable, say "unknown", not zero. Persist usage received before cancellation or
errors. Do not imply that cancelling locally guarantees no provider charge.

Research, writing, and review roles may be configured to different engines or
models. Default to one engine and one model; do not enable multi-model ensembles
automatically.

---

## 9. Skill import and execution engine

### 9.1 Registry

Implement a skill registry built from repository discovery. For each imported skill
retain: stable ID, name and description, source path, source commit, content hash,
available modes (from `MODE_REGISTRY.md`, with spectrum and oversight level), input
requirements, output schemas, referenced resources, required capabilities,
permission requirements, engine availability per run mode, implementation
limitations.

Preserve imported content read-only under `resources/imported-skills/<commit>/`.
Store user customizations separately as overlays and show them as such.

### 9.2 Loading discipline

Load only the skill sections and resources needed for the current task. Do not
concatenate the whole repository into every prompt. In engine-native mode the engine
handles this (skills load on demand); in app-orchestrated mode the app resolves the
agent prompt file plus the references it names. Support references and includes
deliberately; detect missing files, cycles, and incompatible skill versions
(compare `metadata.version` in each `SKILL.md` against the registry).

### 9.3 Environment-specific mappings

| Repository concept | ResearchDesk equivalent |
|--------------------|-------------------------|
| File-reading instructions | Engine read tools scoped to the project folder, or `researchdesk_source_read` |
| Web-search and index-lookup instructions (`deep-research/references/*_api_protocol.md`) | `researchdesk_literature_search` and the app's resolver clients |
| Sub-agent roles | Engine subagents (engine-native on Claude Code) or supervised bounded runs (app-orchestrated) |
| Human checkpoints (FULL / SLIM / MANDATORY) | `researchdesk_checkpoint` → persisted approval records |
| Artifact creation (Material Passport, reports, drafts) | Validated writes inside the run folder, registered via `researchdesk_artifact_register` |
| `/ars-*` slash commands | UI actions; on Claude Code engine-native runs the command text is sent as the prompt |
| Hooks (`SessionStart`, `PreToolUse`) | Engine-native only when runnable; app broker always |
| Optional Python scripts | TypeScript ports, or the embedded runtime if adopted (Section 3.3) |

Unsupported instructions must become visible limitations in the run header and in
`docs/KNOWN_LIMITATIONS.md`. Never pretend a missing tool succeeded. Imported hooks
and scripts never execute automatically merely because they exist.

### 9.4 Invocation record

Each invocation records: skill version and content hash, mode, run mode and fidelity
tier, inputs and source IDs, engine, engine version, provider and model, allowed
tools and sandbox, context manifest (what left the machine), start and end state,
usage where available, output artifacts with hashes, validation results, and every
checkpoint decision in the user's words.

Structured outputs require schema validation; one bounded repair attempt; after
repair failure retain diagnostics and fail visibly.

---

## 10. Workflow runtime

### 10.1 Stages (as the repository defines them)

The repository's `academic-pipeline` is a 10-stage state machine. Expose it with
plain-language labels (Appendix C) but keep the repository's identifiers in the data
model:

| Stage | Repository meaning | Skill |
|-------|--------------------|-------|
| 1 | Research (RQ brief, methodology blueprint, bibliography, synthesis) | deep-research |
| 2 | Write (paper draft) | academic-paper |
| 2.5 | Integrity check (references, citation context, statistics, originality, claims) — MANDATORY checkpoint | academic-pipeline |
| 3 | Review (5-seat panel, editorial decision: Accept / Minor / Major / Reject) — MANDATORY checkpoint | academic-paper-reviewer |
| 4 | Revision | academic-paper |
| 3' | Re-review (never Reject) — MANDATORY checkpoint | academic-paper-reviewer |
| 4' | Re-revise (at most one round) | academic-paper |
| 4.5 | Final integrity check — MANDATORY checkpoint | academic-pipeline |
| 5 | Finalize (format and export; MANDATORY entry gate) | academic-paper |
| 6 | Process summary and AI self-reflection report (user may decline; terminal acknowledgement) | academic-pipeline |

Repository stage states: `pending`, `in_progress`, `completed`, `skipped`, `blocked`.
Repository global states: `initializing`, `running`, `awaiting_confirmation`,
`paused`, `completed`, `aborted`. Mirror these for pipeline runs. Repository loop
limits: integrity FAIL loop at most 3 rounds per gate; at most 2 full revision
loops; Stage 4' at most 1. Checkpoint classes: FULL, SLIM, MANDATORY; MANDATORY
checkpoints (2.5, 3, 3', 4.5, Stage 5 entry) cannot be skipped and require explicit
user input. Confirm all of this against `pipeline_state_machine.md` and document
adaptations.

Individual skills outside the pipeline (for example `deep-research lit-review`,
`academic-paper abstract-only`, `academic-paper-reviewer quick`) run as single-stage
workflows with their own checkpoints.

### 10.2 App task states

For engine runs and background jobs: `queued`, `running`, `waiting_for_user`,
`paused`, `completed`, `failed`, `cancelled`, `interrupted`. Persist transitions
transactionally.

### 10.3 Operations

Approve next stage; request changes; retry failed task; cancel current run; resume
after app restart; inspect artifacts; compare versions.

Checkpoint approval is bound to the exact artifact hashes presented. Invalidate the
approval when material inputs change. Do not advance a failed task as if it
succeeded. Resume from durable completed work; do not automatically repeat ambiguous
external side effects (provider calls, file writes) after a crash. Bounded
concurrency and bounded refinement loops; do not run dozens of agents because the
repository describes many roles. One writer per manuscript revision.

In engine-native mode the engine's own orchestrator holds the stage logic and the app
mirrors it from bridge checkpoint calls and passport changes; if the engine ends a
turn asking a question in plain text without calling the bridge, show the text as a
checkpoint, require a user turn, and resume with the user's words.

---

## 11. Paper and document ingestion

Initial formats: PDF, Markdown, TXT, DOCX, BibTeX, RIS. Later, after the main flow
works: PubMed `.nbib`, Web of Science exports, CSV (the repository's sr-screener
parses these; port its parsers with their fixtures).

Ingestion: copy or link by explicit user choice; hash files; extract metadata;
extract searchable text; preserve page and section mapping when available; allow
metadata correction; detect likely duplicates conservatively.

Do not promise reliable extraction of equations, tables, columns, or figures when
the parser does not provide it. If a PDF has insufficient extractable text, mark it
as requiring OCR and offer OCR only when a supported implementation exists. Preserve
attachment identity and extraction provenance (parser, version, time). Port the
repository's PDF read-integrity idea (`scripts/pdf_read_preflight.py`): cross-check
page counts and refuse to mint page locators from a truncated or mispaginated read.

Never represent a search snippet or an abstract as a fully read paper. Record the
read scope per source (full text, sections, abstract only, table of contents only,
unknown), as the repository's read-attestation ledger does.

---

## 12. Literature research and source library

### 12.1 Providers

Provider abstraction for search and metadata resolution. The repository uses four
key-free indexes and documents each protocol under `deep-research/references/`:

| Provider | Endpoint (as documented) | Credentials | Notes |
|----------|--------------------------|-------------|-------|
| Semantic Scholar | `api.semanticscholar.org` | optional `S2_API_KEY` raises rate limit | |
| OpenAlex | `api.openalex.org` | optional `OPENALEX_API_KEY`; polite-pool email | |
| Crossref | `api.crossref.org` | none; polite email in `User-Agent` | |
| arXiv | `export.arxiv.org` | none | pace requests ≥ 3 s per the repository's terms-of-use note |

Verify each provider's current terms, requirements, and rate limits at
implementation time. At least one live provider must work in the first
research-capable release; Crossref or OpenAlex is the pragmatic first choice.
Respect rate limits and access restrictions. Do not bypass paywalls. Keep
bibliographic resolution separate from full-text retrieval.

### 12.2 Library

The user can search by topic, title, or DOI; review results; import selected
records; attach full text lawfully; correct metadata; filter, tag, and search the
local library. Import the repository's literature-corpus adapters (folder scan,
Zotero, Obsidian) as TypeScript ports producing the repository's
`literature_corpus_entry` schema plus a rejection log.

Source fields: stable ID, title, authors, year, venue, DOI and other identifiers,
URLs, abstract, attachment references, provider provenance, retrieval time,
verification status, read scope, corrections and history.

Deduplicate by identifiers first, then cautious fuzzy matching; show uncertain merges
for user review.

---

## 13. Evidence and citation integrity

### 13.1 Four separate checks

A. **Bibliographic existence** — does a resolver identify the cited work?
B. **Metadata consistency** — do title, authors, year, identifier agree?
C. **Source availability** — full text, abstract only, or nothing?
D. **Claim–source alignment** — does an available passage support the specific claim?

A DOI resolving does not establish that a claim is supported. Four indexes agreeing
does not establish scientific truth.

### 13.2 App-native citation gate

Port the repository's deterministic gate: per citation, query the four indexes, record
per-resolver outcomes, derive `lookup_verified` ∈ {true, false, unresolvable} where
`false` requires an identifier-keyed mismatch and a title-only miss is
`unresolvable`; keep a local SQLite cache with a 90-day TTL and a staleness advisory;
support per-key invalidation and live revalidation. Preserve the repository's
contamination advisories (post-2024 preprint venue; index-unmatched counts) as
advisory labels only. Policy (`advisory` / `strict`) follows the project's export
policy (Section 19.3).

### 13.3 Evidence model

Fields: `claim_id`, `source_id`, supporting passage, locator and locator type,
extraction provenance, assessment method (model-mediated or human), assessment
result, human verification state. Locator types: PDF page, document section,
paragraph, abstract, web heading, no reliable locator. Never invent page numbers.
Keep the repository's three-layer citation markers in manuscript Markdown
(`<!--ref:slug-->` plus `<!--anchor:kind:value-->`, kinds quote / page / section /
paragraph / none, quotes capped at 25 words) so engine-native runs and the app agree
on the same markup.

Claim states: not checked, source unavailable, supported by available passage,
partially supported, contradicted, inconclusive, human verified. Label model-mediated
assessments as such.

### 13.4 Checker findings

Detect: missing bibliography entries, unused entries, malformed identifiers,
duplicate references, metadata discrepancies, missing source evidence, unsupported
claims per the configured assessment, undefined or late acronyms (port
`scripts/check_acronyms.py`), and numeric or citation token drift between revision
rounds (port `scripts/check_revision_token_conservation.py`).

If external checks cannot run, mark "not checked". Preserve unresolved issues in an
integrity report. Allow draft export with visible warnings. A policy-blocked
final export may be overridden only with a record of user, reason, time, and
affected artifact version. Do not claim these checks prove authentic data or
reproducible experiments.

---

## 14. Manuscript workspace

Provide: section navigator, editable outline, manuscript editor, source insertion,
citation insertion, notes, word count, version history, diff and revert.

AI actions (assist or selected engine): propose outline, draft selected section,
improve clarity, summarize selected sources, identify unsupported claims, suggest
limitations, suggest abstract, propose a revision responding to a review finding.

Flow: select action → choose permitted sources → review context summary → generate
proposal → inspect diff → accept or reject. Never overwrite the manuscript silently,
including when an engine-native run writes draft files: detect the change, show a
diff, and require acceptance before the authoritative manuscript changes.

Preserve citations as structured references with stable IDs; do not rely on
model-generated citation strings. Citation formatting: a tested CSL processor with
verified style files; APA 7 and IEEE first; other styles only when fixtures pass;
a clear bibliography-generation path. Do not invent measurements, methods,
experiments, results, quotes, findings, or references. Missing evidence becomes a
visible placeholder or issue.

---

## 15. Review and revision

Adapt the repository's reviewer roles. Its full-mode panel has five seats:
Journal-Fit Reviewer (`eic_agent`), Methodology (`methodology_reviewer_agent`),
Domain (`domain_reviewer_agent`), Perspective (`perspective_reviewer_agent`), and
Devil's Advocate (`devils_advocate_reviewer_agent`); `field_analyst_agent`
configures the panel and `editorial_synthesizer_agent` produces the decision and
revision roadmap. Modes: full, re-review, quick, methodology-focus, guided,
calibration. Start with full, quick, and re-review.

Reviewers receive read-only tools (Section 7.6). Findings contain: ID, category,
severity, affected section or passage, evidence or reasoning, requested action,
status, linked revision. Show role separation as role separation; the repository
itself forbids calling it independence.

Revision matrix: reviewer concern → author response → proposed change → accepted
manuscript revision → verification outcome. Support: accept concern, disagree with
rationale, request clarification, generate response draft, apply approved change,
recheck finding. Enforce the repository's loop limits (Section 10.1). Review text is
untrusted input and cannot grant tool permissions.

---

## 16. Systematic-review screening

Implement after the core research and writing flow is stable. The repository's
`sr-screener` provides: protocol definition and eligibility criteria, import of
RIS / PubMed / Web of Science / CSV exports, de-duplication, title/abstract
screening and full-text screening by two blinded reviewer runs plus an adjudicator,
exclusion reasons, conflict adjudication, human confirmation, screening log, PRISMA
2020 counts, a methods draft, and a `literature_corpus[]` handoff. Its scripts are
standard-library Python with synthetic fixtures; port them.

Rules: never label two AI reviewers as two independent human reviewers; a failed or
missing model response stays pending and never defaults to include or exclude;
derive counts from stored decisions, never from a model; neutralize spreadsheet
formula text on export; do not claim PRISMA compliance because a diagram exists.

---

## 17. Tools, permissions, and security

### 17.1 Electron

`contextIsolation` on, `nodeIntegration` off, sandbox on where compatible,
restrictive content security policy, typed allowlisted IPC, sender and argument
validation. Render Markdown and HTML safely: no raw HTML, no `javascript:` URLs, no
remote content becoming privileged instructions.

### 17.2 Filesystem

Restrict tools to explicitly selected project roots. Canonicalize paths (resolve
junctions, symlinks, UNC paths, reparse points, 8.3 names, case) before every check.
Reject traversal and workspace escapes. Protect app configuration and credential
files from engine and bridge access. Keep deletion recoverable where feasible.

### 17.3 Tool and bridge calls

Validate schemas; enforce timeouts and size limits; capability-based permissions;
audit events; structured errors. No unrestricted shell for any model. If an approved
export tool needs process execution (Pandoc, Tectonic), use an allowlisted
executable with a verified path, pass arguments without shell interpolation,
restrict working directory and environment, apply timeout and output limits.

### 17.4 Engine subprocesses

Section 7.9 applies. In addition: engines must never receive the assist-provider
keys or the app's own secrets; the bridge token is per run and expires with it; the
engine's own permission system is a second layer, never the only one.

### 17.5 Secrets

Electron `safeStorage` backed by Windows credential protection for assist-provider
keys and optional literature API keys. Detect unavailable protection and fail safely;
never fall back silently to plaintext. Engine credentials are never stored by
ResearchDesk (Section 7.7).

### 17.6 Prompt injection

Imported papers, websites, review text, and skill content are data, not authority.
Injection in a source must not override permissions, approvals, or tool policy. The
bridge ignores instructions embedded in tool arguments that attempt to change scope.
Include the repository's instruction/data boundary statement in every role preamble.

### 17.7 Privacy

No analytics or telemetry by default. No background upload of project files.
Explicit notice before content is sent to a cloud provider or an engine (the context
summary names the recipient). Log metadata without full document text by default.
Redacted diagnostic export. Warn that deleting local records does not retract
provider-held data. Document every network touchpoint in `docs/DATA_FLOWS.md`,
following the repository's own table format (touchpoint, when it fires, what is sent,
recipient, credentials, off switch).

---

## 18. Context and RAG

Start with local full-text search and explicit source selection. Add embeddings only
with a demonstrated improvement, opt-in, with disclosure of where text is sent,
provider changes supported, model and version tracked, stale indexes invalidated,
rebuild from original content.

Retrieved chunks retain project ID, source ID, locator, extraction version, content
hash. Prompts are built from: current task, relevant skill instructions, approved
project brief, selected sources, necessary evidence, user-approved manuscript
context. Show a context summary before anything leaves the machine. Project memory is
inspectable, editable, deletable, and never shared across projects.

---

## 19. Export

### 19.1 Priority

1. Markdown
2. BibTeX and RIS
3. DOCX (JavaScript DOCX library; no Pandoc required)
4. PDF (Electron print-to-PDF of the rendered manuscript; no TeX required)
5. Optional LaTeX project export, and optional Pandoc or Tectonic paths behind a dependency checker

### 19.2 Content

Manuscript, references, integrity report, review report, response to reviewers,
process summary. Internal review comments stay out of the manuscript unless selected.
For DOCX and PDF: preserve headings, render references, include supported tables and
images, handle Unicode (including CJK, since the repository supports Traditional
Chinese and bilingual abstracts), test page breaks and long content, report
unsupported constructs clearly. Honor a user-supplied layout profile following the
repository's `format_profile` schema; ship no real institution's or journal's
profile content.

### 19.3 Export policy

Draft export always allowed with visible warnings. Final export policy per project:
advisory or strict citation existence (Section 13.2). Overrides recorded.

### 19.4 Process summary

Record: repository and skill versions and hashes, engine and version, run mode and
fidelity tier, models used, workflow stages and checkpoints with decisions, known
limitations, unresolved integrity issues, experiment provenance only when supplied.
Never include API keys or hidden reasoning.

---

## 20. Offline behavior and performance

Offline: the app starts; projects open; sources stay searchable locally; manuscripts
stay editable; existing reports stay readable; local export works; cloud-only actions
and both engines (which need the vendor's service) are clearly unavailable. Local
inference exists only if a working local provider is configured; do not imply the app
ships a model.

Performance: responsive UI, virtualized lists, background workers, responsible
caching, cancellation, no full-library prompts, startup and large-document
benchmarks on available hardware. Report measurements and conditions; invent none.

---

## 21. Tests and acceptance criteria

### 21.1 Unit

Skill parsing and reference resolution; workflow transitions and loop limits;
approval invalidation; engine event normalization; provider normalization;
structured-output validation and repair; citation matching; metadata discrepancy
handling; path authorization; export transformations; cost and usage accounting;
adapter ports against the repository's fixtures.

### 21.2 Integration

Create, open, archive project; ingest PDF, DOCX, references; persist and reopen
manuscript; run and resume a workflow; backup and restore; migrations; export
generation; bridge tool round trips.

### 21.3 Security

Path traversal; junction and symlink escape; malicious Markdown; disallowed IPC;
prompt injection requesting secret access through a source and through a bridge
argument; reviewer run attempting a manuscript write on each engine; secrets in logs
and settings export; engine environment scrubbing.

### 21.4 Engine contract tests

Fake `claude` and `codex` executables (the repository's own test suite uses this
pattern) that replay recorded event streams: happy path, permission request,
checkpoint request, malformed event, mid-run exit, slow output. Version-drift test:
detect an unsupported CLI version and refuse clearly. Opt-in live smoke tests per
engine, labeled separately and never run in CI.

### 21.5 Failure

Invalid key; rate limit; timeout; malformed model response; unavailable resolver;
app closed during a run; cancelled task with process-tree cleanup verified; locked
file; disk-full or simulated write failure; engine not installed; engine logged out
mid-run.

### 21.6 End-to-end vertical slice

1. Install or launch the app.
2. Detect Claude Code and Codex; show status for both.
3. Configure one assist provider securely.
4. Create a project; select Claude Code as default engine.
5. Import a supplied paper.
6. Search one live literature provider; add selected references.
7. Run a repository skill engine-native (for example lit-review) and observe live tool activity.
8. Answer a checkpoint through the structured checkpoint dialog; confirm the approval record and the user's exact words in the run record.
9. Switch to Codex; run one app-orchestrated role (for example a quick review) on the same project; confirm the fidelity badge and the read-only policy.
10. Generate and accept an outline through a diff.
11. Generate a section using selected sources.
12. Run the citation gate; inspect one unresolved issue.
13. Produce a simulated review (full panel) on the default engine.
14. Accept a revision through a diff.
15. Export Markdown, DOCX, and PDF.
16. Close and reopen; confirm state, files, history, and engine sessions persist.

Maintain the acceptance matrix (Appendix D): requirement | implementation | test |
result | limitation. A feature is not complete because its UI exists.

---

## 22. Windows packaging and release

Produce a development build, a Windows x64 installer, and optionally a portable
build. End users must not need Node.js, Git, Python, a terminal, or manual dependency
installation **for the app itself**. The engines are a declared exception: using
Claude Code or Codex requires the user to install and log in to that vendor's CLI,
and Claude Code's shell tools on Windows depend on Git for Windows. The Engines
screen explains this and the app works without either engine (library, manuscript,
citation gate, assist engine, export).

Handle spaces and Unicode in paths, read-only installation directories, app-data
permissions, native dependency rebuilds for Electron, single-instance behavior,
startup recovery, uninstall without project deletion.

Unsigned build if signing is unavailable: label it, explain Windows trust prompts,
do not claim it is signed. Test the installed build. If the development environment
cannot build or run Windows artifacts, implement Windows CI and state exactly what
was not verified. No insecure self-updater.

---

## 23. Implementation phases

Report at the end of every phase: what works, files changed, tests executed and
results, known limitations, how to run, next phase. Do not polish a mock dashboard;
reach a working end-to-end research flow early.

**Phase 0 — Discovery.** Repository audit at the pinned commit; license and
provenance review; feature mapping; engine capability verification against installed
CLIs and current vendor docs (`docs/ENGINE_SUPPORT.md`); architecture and scope;
acceptance criteria.

**Phase 1 — Running desktop foundation.** Electron and React app; secure IPC; SQLite
and migrations; project creation; settings; secret storage; engine detection and
smoke test for Claude Code and Codex; one assist provider (OpenRouter) connection;
initial installer build.

**Phase 2 — First real research vertical slice.** Document import; local library;
one live literature provider; imported skill registry and skill explorer;
engine-native run on Claude Code with the bridge and structured checkpoints;
app-orchestrated single-role run on Codex; persisted runs with resume; streaming and
cancellation with process-tree cleanup.

**Phase 3 — Writing and integrity.** Manuscript editor; structured citations and CSL
formatting; evidence links; TypeScript citation gate with cache; claim–source
assessment with limitations; Markdown, DOCX, PDF exports.

**Phase 4 — Review and revision.** Read-only review runs on both engines; structured
findings; revision matrix; diff approval; re-review with loop limits.

**Phase 5 — Hardening.** Crash recovery; backup and restore; security and engine
contract tests; installed-build tests; accessibility; performance; packaging and
documentation.

**Phase 6 — Optional extensions.** Additional assist providers; local inference;
sr-screener port; embeddings; LaTeX output; consent-gated cross-model verification;
budget-controlled multi-model review; embedded Python runtime if still justified.

---

## 24. Repository organization

```
apps/
  desktop/
    src/
      main/          # engine host, bridge, broker, IPC
      preload/
      renderer/
packages/
  domain/
  database/
  engine-core/       # AgentEngine, EngineEvent, engine-support.json
  engine-claude/
  engine-codex/
  bridge/            # ResearchDesk MCP server
  providers/         # assist engine
  research-providers/
  skill-runtime/
  workflow-engine/
  document-processing/
  citation-engine/
  export-engine/
  shared-contracts/
resources/
  imported-skills/<upstream-commit>/   # unmodified, with manifest and hashes
  templates/
  sample-project/
tests/
  unit/ integration/ e2e/ fixtures/ engine-fakes/
docs/
  REPOSITORY_AUDIT.md  LICENSE_AND_ATTRIBUTION.md  FEATURE_MAPPING.md
  ARCHITECTURE.md  IMPLEMENTATION_PLAN.md  ENGINE_SUPPORT.md  SECURITY.md
  DATA_FLOWS.md  USER_GUIDE.md  DEVELOPER_GUIDE.md  BUILD_AND_RELEASE.md
  ACCEPTANCE_MATRIX.md  KNOWN_LIMITATIONS.md  adr/
```

Keep the architecture proportional. Do not create empty abstraction packages
without a demonstrated use.

---

## 25. Final delivery

1. Working application source
2. Locked dependencies
3. Preserved source-repository attribution and license files
4. Repeatable development commands
5. Repeatable Windows build commands
6. Installer, or a truthful statement of build limitations
7. Automated tests, including engine contract tests with fakes
8. Acceptance matrix
9. Sample project using lawful, clearly identified content
10. User guide, including engine setup
11. Architecture, data-flow, and engine-support documentation
12. Known limitations and remaining work

Report mock, partially implemented, and verified features separately.

Never: fake repository integration; invent references or results; call a prompt-only
rule an enforced boundary; claim tests ran when they did not; claim packaging
succeeded without an artifact; hide provider or engine failures behind simulated
success; leave dead buttons that appear functional; store or proxy a vendor's login
token; run an engine with permission checks bypassed.

---

## Appendix A. Repository facts verified at commit `77e77e8` (re-verify during discovery)

**Identity and license.** Upstream `Imbad0202/academic-research-skills`; author
Cheng-I Wu; suite version 3.23.2; license CC BY-NC 4.0; `NOTICE.md`,
`CITATION.cff` (Zenodo DOI `10.5281/zenodo.20696614`), `GOVERNANCE.md`,
`SECURITY.md`, `THIRD_PARTY.md`, `CONTRIBUTING.md` platform-ports policy;
`POSITIONING.md` names a sibling Codex distribution
(`Imbad0202/academic-research-skills-codex`) and a community Pi wrapper (`pi/`) with an
end-to-end example under `examples/pi/`.

**Content inventory.** 5 skills: `deep-research` 2.12.1 (8 modes),
`academic-paper` 3.3.1 (11 modes), `academic-paper-reviewer` 1.11.1 (6 modes),
`academic-pipeline` 3.23.2 (orchestrator plus `resume_from_passport`),
`sr-screener` 1.0.0 (8 modes); 35 modes total in `MODE_REGISTRY.md`. 43 agent prompt
files (`*_agent.md`): 14 in deep-research, 12 in academic-paper, 7 in
academic-paper-reviewer, 5 in academic-pipeline, 4 in sr-screener, 1 shared
(`compliance_agent`). 4 plugin-exposed agents under `agents/` are byte-identical
mirrors with a `tools: Read, Write, Edit, Grep, Glob` allowlist. 16 slash commands
under `commands/` (`ars-3w`, `ars-abstract`, `ars-cache-invalidate`,
`ars-citation-check`, `ars-disclosure`, `ars-format-convert`, `ars-full`,
`ars-lit-review`, `ars-mark-read`, `ars-outline`, `ars-plan`, `ars-rebuttal-audit`,
`ars-reviewer`, `ars-revision-coach`, `ars-revision`, `ars-unmark-read`). `skills/`
is a directory of symlinks to the five skill folders (Windows checkouts may not
preserve them; `marketplace.json` lists explicit paths).

**Hooks.** `hooks/hooks.json`: `SessionStart` → `scripts/announce-ars-loaded.sh`
(also runs a 24-hour update check against `raw.githubusercontent.com`, off with
`ARS_UPDATE_CHECK=0`); `PreToolUse` on `Write|Edit|MultiEdit|Bash` →
`hooks/run_guard.sh` → `scripts/ars_write_scope_guard.py`. The launcher states that
"ARS core needs no Python", degrades to pass-through without a real Python, and
needs Git Bash on Windows.

**Python.** 443 `.py` files under `scripts/`, most of them CI lints (`check_*`) and
tests (`test_*`). Runtime-relevant opt-in scripts: resolver clients
(`semantic_scholar_client`, `openalex_client`, `crossref_client`, `arxiv_client`,
plus `chinese_literature_client`), `verify_passport.py` and `verification_gate/`,
`verification_cache.py` (SQLite at `~/.cache/ars/verification.db`, 90-day TTL),
`ars_mark_read.py`, `ars_cache_invalidate.py`, `run_ledger.py`,
`pdf_read_preflight.py` (pypdf), `check_acronyms.py`,
`check_revision_token_conservation.py`, `evidence_rows.py`, `scripts/adapters/`
(folder scan, Zotero, Obsidian), `sr-screener/scripts/` (standard library),
`cross_model_codex_transport.py` (Codex app-server), `run_codex_audit.sh`.
`requirements-dev.txt`: pyyaml, ruamel.yaml, jsonschema[format], pypdf, defusedxml,
openpyxl, markdown-it-py, linkify-it-py; optional `pdf-inspector` for the PDF
classifier.

**Network touchpoints** (`docs/DATA_FLOWS.md`). Four key-free indexes (Semantic
Scholar, OpenAlex, Crossref, arXiv); optional Chinese-literature resolvers
(`doi.org`, `hdl.handle.net`, NCBI E-utilities); consent-gated cross-model providers
(`api.openai.com`, `generativelanguage.googleapis.com`, or a configured
OpenAI-compatible base URL); Codex CLI transports; the plugin update check. No
telemetry. Local stores: verification cache, update-check state, Material Passport
and ledgers, evidence-source folders, sr-screener folders.

**Pipeline.** 10 stages (1, 2, 2.5, 3, 4, 3', 4', 4.5, 5, 6); stage and global states
as listed in Section 10.1; checkpoint classes FULL / SLIM / MANDATORY; integrity
FAIL loop max 3 rounds; max 2 revision loops; Stage 4' max 1; Stage 3' never
Reject; Stage 6 terminal acknowledgement words `finish` / `end` / `done` / `confirm`.
Handoff schemas 1–13 in `shared/handoff_schemas.md` (RQ Brief, Bibliography,
Synthesis Report, Paper Draft, Integrity Report, Review Report, Revision Roadmap,
Response to Reviewers, Material Passport, …, R&R Traceability, Compliance Report,
Sprint Contract). About 100 JSON schemas under `shared/contracts/`.

**Reviewer panel.** Five seats (Journal-Fit, Methodology, Domain, Perspective,
Devil's Advocate), configured by `field_analyst_agent`, synthesized by
`editorial_synthesizer_agent` under a sprint-contract schema; decisions Accept /
Minor / Major / Reject; the repository forbids describing seat separation as
independence.

**Citation integrity.** Three-layer markers `<!--ref:slug-->` +
`<!--anchor:kind:value-->`; finalizer verdicts `ok` / `LOW-WARN` / `HIGH-BLOCK`;
contamination advisories; `terminal_policies` (`advisory` / `strict` /
`strict_articles_only`); `lookup_verified` ∈ {true, false, unresolvable}; claim
states and evidence rows versioned under `shared/contracts/evidence/`.

**Export in the repository.** Markdown native; DOCX via Pandoc when available; PDF via
LaTeX (Tectonic) when wanted; user-supplied `format_profile` layout schema.

**Model notes.** The repository calibrates prompts against Claude (Opus-class for
architecture and review, Sonnet-class for execution, never Haiku) and requires ports
to document model divergence. It already treats Codex CLI as a cross-model transport
(citation integrity only) and documents its version quirks.

**Environment flags seen** (configuration, not consent): `ARS_CROSS_MODEL`,
`ARS_CROSS_MODEL_TRANSPORT`, `ARS_CROSS_MODEL_REASONING_EFFORT`,
`ARS_OPENAI_COMPAT_BASE_URL`, `ARS_OPENAI_COMPAT_API_KEY`, `ARS_CLAIM_AUDIT`,
`ARS_MODEL_TIERING`, `ARS_VERIFICATION_CACHE_PATH`, `ARS_CACHE_REVALIDATE`,
`ARS_CACHE_STALE_ADVISORY_DAYS`, `ARS_UPDATE_CHECK`, `ARS_SOCRATIC_READING_PROBE`,
`ARS_SOCRATIC_ADJACENT_PROBE`, `ARS_INQUIRY_LEDGER`, `ARS_AUDIT_ARTIFACT_GATE`,
`ARS_RE_REVIEW_LEGACY`, `ARS_PANEL_CLAIM_CHECK`, and the passport reset flag
documented in `academic-pipeline/references/passport_as_reset_boundary.md`.

**Install channels and control availability** (`docs/CONTROL_AVAILABILITY.md`). The
plugin channel is the reference; slash commands, hooks, and the agent tools allowlist
exist only there; subagent orchestration needs a Claude Code session; Cowork and
claude.ai channels run skills standalone or read-only. ResearchDesk's engine-native
mode corresponds to the plugin channel.

## Appendix B. Engine capability matrix (fill during Phase 0 and keep current)

| Capability | Claude Code (docs checked 2026-10-25; installed version: __) | Codex (source checked 2026-10-25 at `openai/codex` commit `970111f6`; SDK 0.164.0; installed version: __) | Assist |
|------------|---------------------------------------------------------------|---------------------|--------|
| Non-interactive run with streamed JSON events | yes: `-p --output-format stream-json --verbose`; SDK `query()` | yes: `codex exec --json` (JSONL); SDK `runStreamed()`; app-server notifications | yes |
| Loads repository skills natively | yes: local plugin via `plugins` / `--plugin-dir`; skills namespaced `/plugin:skill` | partial: `SKILL.md` folders under `.agents/skills` (and other roots) plus `AGENTS.md` (32 KiB cap); no plugin manifest, commands, or hooks | no |
| Subagent dispatch in the repository's sense | yes (`Task` tool; `--forward-subagent-text` to see it) | no: experimental `multi_agent` feature (`collab_tool_call` items) is not the repository's model | no |
| Hooks (`SessionStart`, `PreToolUse`) | yes in non-bare mode when Git Bash is available (inferred; confirm with `--include-hook-events`) | no (Codex has its own experimental hooks feature; not used) | no |
| Permission callback or approval protocol | yes: SDK `canUseTool` + `PreToolUse` hook; CLI `--permission-prompt-tool`; `defer` for durable pauses | app-server only: `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, `item/tool/requestUserInput`; `exec` defaults `approval_policy = never` | n/a |
| Read-only tool policy | `tools` to restrict the set, `disallowedTools` to remove tools; `allowedTools` only auto-approves | `--sandbox read-only` (OS-enforced only where a sandbox backend is active; see Windows row) | n/a |
| Working-directory restriction | `cwd` + `additionalDirectories`; no OS sandbox on native Windows | `-C` + `--add-dir` + sandbox `writable_roots`; `--skip-git-repo-check` needed outside git repos | n/a |
| MCP client (for the bridge) | yes: stdio, http, sse, in-process `createSdkMcpServer`; `strictMcpConfig` | yes: `mcp_servers.<name>` stdio or streamable HTTP via `-c` overrides or `codex mcp add`; `required = true` aborts the run if the bridge fails | n/a |
| Structured output | yes: `outputFormat: {type:'json_schema'}` / `--json-schema` → `structured_output` | yes: `--output-schema <file>` / SDK `outputSchema`; final `agent_message` is a JSON string | per provider |
| Session resume | yes: `session_id` on init and result; `resume`; `--continue` skips `-p` sessions | yes: `thread_id` from `thread.started`; `codex exec resume <id>`; SDK `resumeThread(id)` | app-managed |
| Usage reporting | `usage`, `modelUsage`, `total_cost_usd` (estimate) on `result` | `turn.completed.usage` token counts (input, cached input, cache write, output, reasoning); no cost figure | per provider |
| Budget controls | `maxTurns`, `maxBudgetUsd` (client-side estimate), `--effort` | `model_reasoning_effort`; no turn or cost cap found: enforce wall-clock and token budgets in the engine host | per provider |
| Web search tool | built-in `WebSearch` / `WebFetch`; disable for reviewer runs | `web_search` set to `disabled`, `cached`, `indexed`, or `live` (`cached` is the default: set `disabled` for reviewer runs) | per provider |
| Windows native support | yes (Windows 10 1809+, x64/arm64); Bash tool needs Git for Windows; no sandbox outside WSL 2 | binaries ship for win32; OS sandbox only with `[windows] sandbox` set to `mxc`, `elevated`, or `unelevated`, otherwise **no enforcement**; install docs still mention WSL 2 in places | yes |
| Authentication | user's own CLI login (`claude auth status`) or user's own `ANTHROPIC_API_KEY`; app never handles credentials | user's own `codex login` (ChatGPT) or `codex login --with-api-key`; `codex login status`; `CODEX_API_KEY` / `OPENAI_API_KEY`; app never reads `auth.json` | API key |
| Config isolation | `settingSources: []`, `strictMcpConfig`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | `--ignore-user-config` plus explicit `-c` overrides; `history.persistence = none`; `check_for_update_on_startup = false` | n/a |
| Bare mode | do not use for engine-native (drops plugins, skills, hooks, MCP, CLAUDE.md, subscription auth); acceptable for app-orchestrated roles with API-key auth | n/a | n/a |
| Minimum supported version | at least 2.1.302 (Manual default in `-p`); set after contract tests | set after contract tests against the installed version (0.164.0 was current at the check) | — |

Replace every open item with a verified value and the documentation URL. Claude Code
sources: `code.claude.com/docs/en/headless`, `/cli-reference`,
`/agent-sdk/typescript`, `/agent-sdk/permissions`, `/agent-sdk/plugins`,
`/agent-sdk/sessions`, `/skills`, `/authentication`, `/legal-and-compliance`, `/setup`.
Codex sources: `github.com/openai/codex` at the commit above (`codex-rs/exec/src/cli.rs`,
`codex-rs/exec/src/exec_events.rs`, `sdk/typescript/`, `codex-rs/app-server/README.md`,
`codex-rs/app-server-protocol/schema/typescript/`, `codex-rs/config/src/`,
`codex-rs/core/src/agents_md.rs`, `codex-rs/ext/skills/`, `codex-rs/models-manager/models.json`)
and the npm registry; the hosted pages under `developers.openai.com/codex/` still need
to be read and cited.

## Appendix C. Plain-language labels for repository concepts

| Repository term | UI label |
|-----------------|----------|
| Material Passport | Project run record |
| Run ledger | Decision log |
| RQ Brief | Research question brief |
| Methodology Blueprint | Methods plan |
| Stage 2.5 / 4.5 integrity check | Citation and claim check (before review / final) |
| FULL / SLIM / MANDATORY checkpoint | Checkpoint / Quick checkpoint / Required decision |
| Editorial Decision Letter | Review decision |
| Revision Roadmap | Revision plan |
| Devil's Advocate | Skeptical reviewer |
| Journal-Fit Reviewer | Scope and fit reviewer |
| Socratic mode | Guided thinking |
| three-way-scan | WHY / HOW / WHAT paper scan |
| `literature_corpus[]` | Library import |
| Evidence rows | Evidence table |
| Claim-strength ladder | Claim strength scale |
| Process Summary / AI Self-Reflection Report | Run report |
| `lookup_verified` | Found in index / Not found by identifier / Could not resolve |
| Contamination advisory | Index-coverage warning |
| sr-screener | Study screening |

## Appendix D. Acceptance matrix seed

| Requirement | Implementation | Test | Result | Limitation |
|-------------|----------------|------|--------|------------|
| Engines detected with version and auth state | | engine detect unit + e2e | | |
| Engine below minimum version refused with explanation | | version-drift test | | |
| Engine-native run on Claude Code loads imported plugin from resources | | e2e step 7 | | |
| Structured checkpoint blocks until user decision; words recorded verbatim | | integration (bridge) + e2e step 8 | | |
| Reviewer run cannot write the manuscript on either engine | | security test | | |
| Cancel kills process tree within 5 s and persists partial transcript | | failure test | | |
| Engine environment contains no app secrets | | security test | | |
| Citation gate: identifier mismatch → false; title-only miss → unresolvable; offline → not checked | | unit | | |
| Draft export allowed with warnings; strict final export blocked without recorded override | | integration | | |
| App usable with neither engine installed | | e2e | | |
| No telemetry; data-flow table matches code | | lint over network imports | | |
| Imported content unmodified with hashes and license notice | | manifest test | | |

## Appendix E. Questions for the owner (answer before Phase 1 ends)

1. **Intended use and license.** Is ResearchDesk personal or noncommercial-only? CC BY-NC 4.0 binds any distribution that bundles the repository content. Commercial intent requires removing the content or a separate license from the author.
2. **Engine priority.** Claude Code is the reference engine for the repository's content; Codex is a second engine with disclosed gaps. Confirm the default, and whether the assist engine may ever run repository skills.
3. **Subscription terms and naming.** Anthropic's published terms (Section 7.7) forbid third-party apps from offering claude.ai login or handling subscription credentials, and forbid "Claude Code" or Anthropic names in a product name. Confirm the two-posture design (user's own CLI login, untouched by the app, or the user's own API key), confirm that the product name stays "ResearchDesk", and have counsel confirm the reading before release.
4. **Python.** Accept TypeScript ports of the opt-in scripts, or require an embedded Python runtime so engine-native runs can execute the repository's own scripts?
5. **Upstream relationship.** Do you want ResearchDesk to qualify as a recognized port under the repository's platform-ports policy (pinned commit, sync policy, named maintainer, end-to-end evidence)? If so, plan the evidence set from Phase 2.
6. **Languages.** English only at first, or the repository's Traditional Chinese and bilingual-abstract support from the start?
7. **Cross-model verification.** Leave it to Phase 6, or require consent-gated support earlier?
8. **Codex on Windows.** Without a configured Windows sandbox, Codex enforces no filesystem boundary (Section 7.5). Accept the brief's rule (writer roles only through app-server approvals or with a verified OS sandbox; otherwise read-only roles with a visible "no OS sandbox" notice), or restrict Codex to app-orchestrated review and integrity roles until the sandbox is verified on your machine?
