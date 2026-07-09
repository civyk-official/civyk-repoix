# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.8.0] - 2026-07-09

### Security

- **Public-distribution hardening: Nuitka build artifacts can never leak into the repo or sdist.**
  `cli.build/`, `*.build/`, and `*.dist/` are now gitignored, and `MANIFEST.in` prunes
  `src/civyk_repoix/cli.build`/`cli.dist` and excludes `scons-*.py`. Nuitka's scons scaffolding can
  embed the build environment (env vars), so this keeps it out of both `git` and source
  distributions. (Released wheels are Nuitka-compiled and were already unaffected.)

### Fixed

- **Claude Code hooks now reach the model.** Every civyk-repoix hook emitted its
  text as a top-level `systemMessage`, which Claude Code shows to the user's
  terminal only and **never** sends to the model (per the Claude Code hooks
  reference, `additionalContext` is the field that reaches the model). A
  transcript audit across all local projects found thousands of hook fires with a
  **0% honor rate** — the agent never saw the "check cache" / "remind to store" /
  session-start nudges — while the per-Read hooks added multiple seconds of
  subprocess latency to every file read. `cli_hooks._output_claude` now emits
  `hookSpecificOutput.additionalContext` (with the correct `hookEventName`, and
  `permissionDecision`/`permissionDecisionReason` for PreToolUse denials) for all
  events that support it, falling back to `systemMessage` only for events that
  cannot inject context (e.g. PreCompact).
- **Hook ownership detection no longer deletes a user's own civyk-repoix hooks.**
  The merge that runs on `civyk-repoix init` identified "our" hooks by the bare
  `civyk-repoix` substring, so a user hook that merely invoked the CLI for another
  purpose (e.g. `civyk-repoix query understand` in a Stop hook) was silently
  removed on the next init. Ownership is now matched on `civyk-repoix hooks`
  (every installed command is `civyk-repoix hooks <subcommand>`), preserving user
  hooks that call `civyk-repoix query`/`status`/`serve`.

### Changed

- **Redesigned the Claude per-file cache hints to be model-visible and quiet.**
  `check-cache` (PreToolUse:Read) now injects a one-line recall hint via
  `additionalContext` **only when the file has a model-authored cached
  understanding** — the indexer's auto-generated structural stubs and un-analysed
  files stay silent (no more nudge on every read). A new `refresh-cache`
  (PostToolUse:Edit|Write) fires **only when you edit a file that has a
  model-authored cached understanding** your edit may have outdated, suggesting
  `understand(store)` to refresh it. (Both are advisory, non-blocking hints — the
  model receives them but is free to act or not.) The old always-on `remind-store`
  nudge, the `persist-learnings`
  (Stop) reminder, and the `save-compact` (PreCompact) hook were removed — they
  emitted user-terminal `systemMessage` the model never received (and PreCompact
  cannot inject context at all). Session-start / restore-compact (AI-cache status +
  restored conversation context) and conversation logging / finalize are retained.
  The CLI handlers remain available and Cursor/Windsurf/Copilot installs are
  unchanged.

### Added

- **GitHub Copilot as an LLM provider.** Set `generation.provider: copilot` (or
  `CIVYK_LLM_PROVIDER=copilot`) to drive the whole deep-wiki pipeline — page generation,
  business-context, sequence diagrams, and `ask` — with a GitHub Copilot subscription, a drop-in
  alternative to MiniMax/OpenAI. An in-process adapter exchanges a GitHub OAuth token for the
  short-lived Copilot token (auto-refreshed) and sends the editor headers, talking directly to
  `api.githubcopilot.com` (no separate proxy). The GitHub credential is reused from an editor that's
  already signed in to Copilot (`~/.config/github-copilot/{apps,hosts}.json`), or obtained via a
  one-time `civyk-repoix copilot login`; it is never written to config. New CLI:
  `copilot login | models | status`. Pick a model with `generation.model` (e.g. `gpt-4o`,
  `claude-3.7-sonnet`); list ids via `civyk-repoix copilot models`.
- **Streaming (SSE) responses for all LLM providers** (`generation.stream`, default `true`). The
  client now accumulates streamed deltas over both transports (stdlib `urllib`/Copilot and the
  `openai` SDK). This lifts GitHub Copilot's **16k non-streaming output cap** (`choices: []` on
  cap-hit) up to each model's full output limit (e.g. 64k for Claude Sonnet 4.6), so large wiki
  pages generate completely instead of failing. Hardened for cross-provider use: tolerates
  non-numeric usage fields, surfaces in-band `data: {error}` events as retryable, falls back to a
  buffered JSON body when a provider ignores `stream`, gracefully degrades to non-streaming on a
  `stream_options` 400, and bounds total wall-clock per call (the socket timeout only bounds each
  read). Set `generation.stream: false` for endpoints without SSE support.
- **Incremental wiki edits — delta builds stop reword-churning pages** (`wiki.incremental_edits`,
  default `true`). A delta rebuild previously re-synthesized every stale page from scratch, so an
  unrelated edit produced large reword/rephrase diffs. Now each page is gated on a hash of its
  actual LLM input (grounding snippets + deterministic diagrams): a page whose input is unchanged
  **reuses its prior prose with no LLM call** (and the build is a true no-op when the rendered
  markdown is identical), while a page whose input changed is **minimally revised** — the model
  edits the prior page instead of rewriting it. Citations are re-validated against the index in
  both paths, so accuracy is preserved while diffs stay small and token use drops. `force=true`
  (and the enrichment correction pass) still do a full re-synthesis. Cached per page in a new
  `wiki_pages.build_cache` column (schema `2.5.0`); set `wiki.incremental_edits: false` to restore
  full re-synthesis of every stale page. A malformed/missing cache degrades to a full rebuild of
  that page (it never aborts a build), and a no-op reuse still rewrites the page file so a deleted
  page self-heals.

### Changed

- **Deep-wiki is now locked to the default branch by default** (new `wiki.default_branch_only`,
  default `true`). Every build path — delta/file-change, scheduled, and manual
  `wiki(action="generate")` including `force=true` — only runs on the repo's git-detected default
  branch (main/master). Auto-triggers on other branches are silently skipped; a manual `generate`
  on another branch builds nothing and returns `{"status": "skipped", ...}` with a message naming
  the default branch. When on, `wiki.branches` is ignored. The gate is enforced at every entry
  point — the daemon's auto-triggers and MCP `generate`, plus the standalone server / CLI
  `generate` — so it can't be bypassed. Branch names are normalized before comparison
  (`main`/`origin/main`/`refs/heads/main` are treated as the same branch), and a detached HEAD is
  refused rather than silently building as the default branch. **Behavior change:** setups that used
  `wiki.branches` to build on additional branches, or that force-generated on feature branches,
  must set `wiki.default_branch_only: false` to restore that. Changing the key restarts the repo's
  worker so it takes effect immediately.
- **Default `wiki.planning` is now `structural`** (was `auto`). The page set is derived
  deterministically from the module/directory tree, so the wiki no longer churns across rebuilds —
  same code produces the same page set, and diffs reflect real content changes only (no
  split/merge/rename thrash or fluctuating page counts). This is the better default because the
  deep-wiki is normally committed to git and rebuilt automatically. Set `wiki.planning: auto` (or
  `llm`) to restore LLM-driven semantic grouping for one-off or human-reviewed generation, where
  grouping quality matters more than diff stability.
- **Default LLM is now GitHub Copilot + `claude-opus-4.8`** (`generation.provider: copilot`,
  `generation.model: claude-opus-4.8`). In head-to-head deep-wiki comparisons across the Copilot
  model menu, Opus 4.8 produced the highest-quality pages (most accurate citations, deepest, no
  truncation); `claude-haiku-4.5` is the fast alternative (~6× quicker, slightly shallower). The
  default `generation.timeout_s` is raised to `1200` because verbose models stream large pages
  slowly and the timeout now bounds total wall-clock per call. Switch back any time with
  `provider: openai`/`minimax` + `base_url`/`CIVYK_LLM_API_KEY`. The Copilot-token-exchange fallback
  model also moved from the retired `gpt-4o` to `claude-opus-4.8`.
  **Note:** `claude-opus-4.8` is a *premium* Copilot model — it requires an entitled plan with
  available premium-request quota; when unavailable Copilot returns `model_not_supported` and the
  wiki degrades to structural pages. Set `generation.model: claude-haiku-4.5` for an always-available,
  fast default.
- **Deep-wiki is opt-in** (`wiki.enabled` defaults to `false`) — a safe default for public
  distribution, since generation calls out to an LLM. Enable it explicitly (`wiki.enabled: true`)
  **and** configure a `generation` provider/model (or `provider: copilot`) to build the wiki. With
  the wiki disabled, indexing, search, and the AI cache all work normally; only wiki build/`ask`
  are inert.
- **Deep Wiki — RAG grounding overhaul (documents HOW code works, not just what exists).**
  - Retrieval: the candidate search now matches keywords as substrings (was exact-name-only,
    which starved pages into the same generic top-50 exports); token accounting charges
    docstring + snippet before the budget gate; relevance score outranks kind (an on-topic
    function is no longer buried under an off-topic class); snippet caps scale with the budget.
    All opt-in via `build_context_pack(wide_match=…, deep_fqns=…)`, so the `context` MCP tool's
    behaviour is unchanged.
  - Bodies where logic lives: the few highest-fan-in / churned functions per page get their
    FULL body (not a 50-line head); a module's own symbols now carry snippets (were
    signature-only); previously-discarded enrichment (params/usage/related/has_tests) is rendered.
  - Structure & flow: each deep-treatment function gets an ordered `calls (static)` slice
    (call-graph), the Flows page is anchored on the real entrypoint and its callees, and the
    grounding is assembled block-by-block within a budget split that never bisects a code fence.
  - Extra signals: git hotspots ("Recent activity" + churn-aware deep-treatment), tests as
    behavioural grounding, curated `recall_understanding`, and dependency cycles for the
    architecture/dependencies pages. Prompt: a "Key Logic & Algorithms" section directs the
    model to explain the bodies now provided.
  - Index: each function/method now stores a `body_digest` (control-flow skeleton) at index
    time (schema 2.2.0 → 2.3.0), surfaced as a logic-skeleton fallback when a full body isn't
    shown.
  - Semantic retrieval: symbols are embedded at index time into a new `symbol_embeddings`
    table (schema 2.3.0 → 2.4.0); the wiki merges the symbols nearest the page topic into the
    candidate pool (hybrid lexical + semantic), so descriptive topics recall code that
    exact-name matching misses.

### Fixed

- **Deep-wiki no longer silently ships body-less "stub" pages.** When the LLM returned an empty
  completion (e.g. a verbose model whose huge-page generation was dropped), the generator fell
  through to a diagram+sources-only structural page and counted it as `built` — so the build tally
  reported success while the page had no prose. Such a page now fails loudly (logged + counted in
  `failed`) instead of masquerading as built, so the tally is honest and the page can be
  regenerated. (Raising the per-call `generation.timeout_s` also prevents the empty-completion case
  for slow/verbose models on the largest pages.) The same guard runs in the enrichment/correction
  pass, so a failed correction can never overwrite a good page with a stub.
- **Gap markers no longer leak into published pages inside code fences.** Some models emit the
  internal `<<NEED: …>>` grounding marker as a code comment (e.g. `import X  # <<NEED: path>>`);
  the stripper skipped fenced code, so those leaked into examples. It now removes the marker (and a
  comment it leaves empty) inside fences too, keeping the code line intact.
- **Auto-understanding embeddings are no longer left missing.** The index-time
  auto-understanding pass skipped files that already had an understanding row *regardless of
  whether its embedding existed*, so understanding written when the embed failed (model still
  downloading at first index) or after an embedding-backend purge was never re-embedded. The
  pass now backfills orphaned understanding rows, so semantic recall stays populated.
- **The `WikiConfig` dataclass default and the bundled `DEFAULT_CONFIG_YAML` template are kept in
  sync** (both seed `wiki.enabled: false`), so a fresh repo's `config.yaml` and the in-code default
  agree instead of silently disagreeing at load time.
- **Docs now show the real generation defaults.** The README config example and the configuration
  reference advertised the retired MiniMax default (`provider: minimax`, `model: MiniMax-Text-01`,
  stale `timeout_s`/`max_output_tokens`); they now show the actual defaults (`provider: copilot`,
  `model: claude-opus-4.8`) and stop calling MiniMax "the default."

## [1.7.0] - 2026-06-14

A deep-wiki generation overhaul plus a codebase-wide hygiene & resilience pass. No changes to the
MCP/CLI tool surface — public API preserved.

### Added

- **Deep Wiki — aspect-specific pages.** Every page type now has its own section contract (with
  table-shaped hints) instead of one shared skeleton, so overview/architecture/getting-started/data
  models/API/module pages each read differently. Grounding retrieval is biased per aspect.
- **Deep Wiki — new detector-gated pages:** Configuration, Dependencies, Errors & Exceptions, Key
  Flows, and Examples. Each is emitted only when the repository actually has that aspect (config
  modules, a dependency manifest, exception classes, an entry point, a test suite) and stays
  grounded and citation-first, with aspect guardrails (e.g. config never echoes secret values).
- **Deep Wiki — business-context pre-pass.** One grounded LLM call distills product purpose, domain
  entities, and business rules into `business-context.json`, prepended to the overview/getting-started
  grounding. Gated by `wiki.business_context` (default on); reused on incremental builds.
- **Deep Wiki — README landing page + quality score.** A human `README.md` (grouped by section, with
  quick-nav and a per-page table) is now the single landing page (`index.md` is no longer generated),
  plus a 0–100 quality score in the manifest — both computed with no extra LLM cost.
- **Deep Wiki — optional claim verification.** `wiki.enrich` (default off) records a per-page
  verification score (the fraction of backticked identifiers that resolve in the index).
- **Deep Wiki — shared glossary anchor.** A compact project-framing + glossary anchor (from
  `business-context.json`) is fed into every page's grounding so terminology stays consistent.
- **Deep Wiki — prompt-prefix caching.** The page-synthesis system prompt is page-independent
  (identical across every call in a build), and the user message is ordered stable-content-first,
  so OpenAI-compatible providers can serve the shared prefix from their automatic cache.

### Fixed

- **Concurrent DB access is serialized** — the shared SQLite connection (reused by the
  daemon's indexing/query threads and the deep-wiki build) is now guarded by a reentrant
  lock, so a long wiki build can no longer interleave `BEGIN`/`COMMIT` with other threads
  and corrupt a transaction.
- **Deep Wiki — page text is never corrupted by post-processing** — the gap/hedge cleanup
  (markers, hedge scrubbing, empty-section removal, blank-line normalization, chunking) is now
  code-fence-aware (fenced code is left intact, incl. its blank-line runs), the hedge detector
  is anchored to "context" so it no longer
  deletes grounded prose like "could not be found in the cache" (while also catching
  context-first "the provided context does not …", "supplied snippets", and "signaled as
  gaps" RAG-meta phrasings), and inline-citation scrubbing runs before diagrams are appended
  (never touches Mermaid). The page prompt forbids referencing the retrieval mechanism.
- **Force-reindex no longer empties the index on failure** — `force_reindex` with
  `clear_existing=true` builds the new index in a staging DB and swaps it onto the live
  connection in a single transaction, so a failed rebuild leaves the existing index fully
  intact instead of empty. A reentrancy guard prevents a second reindex from clearing the
  index out from under an in-flight one, and failures now surface `status="error"`.
- **Windows: daemon identity handshake** — the daemon announces its socket identifier on
  connect and clients verify it before use, so a deterministic-port collision or stale port
  file can no longer silently wire a client to a *different* repository's daemon. The bind
  fallback now also handles a colliding in-use port (previously only Windows port exclusions).
- **Conversation auto-cleanup no longer wipes all history** — `cleanup_by_size` accounts for
  the SQLite freelist, so size-based pruning stops at the configured budget instead of
  deleting every retained session.
- **Pagination & results** — `get_api_endpoints` applies `offset` before `limit` and reports a
  true `total_count`; `get_related_files` now returns the actual related files.
- **Robustness** — atomic (temp + `os.replace`) writes for service/daemon/setup state (a crash
  mid-write can no longer corrupt them, and setup no longer silently destroys an existing
  `.claude/settings.json` with invalid JSON); fire-and-forget daemon tasks are tracked and
  cancelled on shutdown; the dedup cache no longer drops a concurrently-stored result; the
  hook file-path guard uses `os.path.commonpath` (sibling-prefix paths no longer pass);
  `parse_time_spec` falls back to 7 days on malformed input; embedding API backends gained a
  configurable timeout + bounded retry on transient (429/5xx) errors; the file watcher cancels
  pending debounce tasks on stop; `daemon start` reports failure if the child exits immediately.
- **FIPS hosts** — the request-dedup hash uses `hashlib.md5(..., usedforsecurity=False)`.

### Changed

- **Deep Wiki — bottom-up (map-reduce) synthesis.** Module/detail pages are built first and emit
  compact digests (cached to `digests.json`); the overview and architecture pages are then
  synthesized from those digests plus the dependency graph, carrying real source citations up for
  better consistency and coverage. The grounding token budget now scales with repository size.
- **Deep Wiki — diagrams are relevance-gated.** Trivial graphs (a single node, no edges, or
  degenerate labels) and short sequence diagrams are dropped instead of cluttering pages; module
  dependency graphs now show real file names (fixed a bug that labelled every node `py`).
- **`TimingCollector` is now genuinely thread-safe** (lock around all stat access) and uses
  `inspect.iscoroutinefunction` (removes a Python 3.16 deprecation warning).
- **Observability** — module loggers and exception detail added at previously-silent error
  paths; destructive operations (clear-all, schema migrations, embedding purge) log audit
  trails; slow tool calls emit a latency warning. Reuses the existing logging/timing facilities.
- **civyk-repoix no longer indexes its own `memory/` workspace** (`memory/codebase-index`,
  `memory/deep-wiki`), avoiding self-indexing and write contention.
- Internal de-duplication refactors (shared git/DB/transport/indexer helpers) and comment
  hygiene across the source tree. No behavior change.

### Removed

- Confirmed-dead internal code and abandoned, production-unwired capabilities (public API and
  behavior preserved; net ~2,400 lines removed): ~16 unused `mcp_responses` builders, the
  `languages` SQL-dialect/MongoDB detection and the unused tree-sitter `SYMBOL_QUERIES`, the
  unused `EmbeddingIndex` reload protocol, the `get_external_dependencies` /
  `ExternalDependency` `NotImplementedError` API, the dead `QueryCache` prefix-invalidation
  machinery, and assorted unreferenced helpers.

## [1.6.0] - 2026-06-14

### Added

#### Deep Wiki — Devin DeepWiki-style docs + grounded Q&A

A new `wiki` MCP/CLI tool that generates an auto-maintained knowledge base for the
repository using any **OpenAI-compatible** LLM (OpenAI, Minimax, OpenRouter, local) and
answers questions over it. Off by default; degrades gracefully when no LLM is configured.

- **`wiki` tool** (Extended tier) — actions `ask`, `generate`, `status`, `list`, `read`,
  `export`. `ask` supports `rag` (sections + citations, no LLM cost), `answer`
  (LLM-synthesized), and `deep` (bounded multi-step research). Drop-in DeepWiki MCP aliases
  `read_wiki_structure` / `read_wiki_contents` / `ask_question` are also exposed.
- **OpenAI-compatible provider layer** — `engines/llm.py` (`LLMClient`: `openai` SDK with a
  stdlib `urllib` fallback, bounded retries) and `OpenAIEmbeddingBackend` (`/embeddings`).
  API keys are read from the environment only and never serialized.
- **RAG-grounded generator** (`workers/wiki_generator.py`) — hierarchical page synthesis,
  `[path:Lstart-Lend]` citations validated against the index (≥5 source files/page),
  inline **Mermaid diagrams**, chunk-level embeddings for retrieval, `index.md` TOC +
  `manifest.json`, and incremental rebuilds. Output under `memory/deep-wiki/<branch>/`,
  branch-aware.
- **LLM-driven module planning** — the generator sends the real source-tree structure to the
  configured LLM, which proposes semantic module groupings *and* depth; code reconciles every
  source file onto a page (longest-prefix match + catch-all) so coverage stays ~100% regardless of
  the plan, with a deterministic directory-planner fallback. Configurable via `wiki.planning`
  (`auto` | `llm` | `structural`).
- **Diagrams** (inline, per page):
  - *Deterministic, from the edge graph:* component dependencies; a whole-system **data flow**
    with upstream/downstream **external systems** (CLI, MCP client, LLM API, embeddings, SQLite,
    git, filesystem); per-module data flow (providers → module → consumers); intra-module
    dependency graphs (class-diagram fallback); and class diagrams on data pages.
  - *LLM-proposed, then validated* against indexed symbols: **sequence diagrams** for the
    architecture page and high-importance module pages (`wiki.sequence_diagrams`, default true;
    hallucinated diagrams are dropped).
- **`ask` grounded in code, not just prose** — answers attach budgeted code symbols + snippets from
  the index (`wiki.code_context_token_budget`) on top of the wiki excerpts, with larger retrieval
  (`retrieval_top_k` 30, `answer_token_budget` 8000). When the current branch has no wiki, `ask`
  falls back to a configured/default branch's wiki.
- **Parallel page generation** (`wiki.concurrency`, default 4) — independent pages build
  concurrently (LLM network waits overlap; DB writes serialized), ~3-5x faster.
- **Triggers** — changed-file threshold and/or schedule (debounced, idle-gated, background),
  configurable and off by default.
- **Config** — new `generation` and `wiki` sections in `config.yaml`; new env vars
  `CIVYK_LLM_API_KEY`, `CIVYK_LLM_BASE_URL`, `CIVYK_LLM_MODEL`, `CIVYK_LLM_PROVIDER`,
  `CIVYK_LLM_EMBEDDING_API_KEY`, `CIVYK_LLM_EMBEDDING_MODEL`, `CIVYK_WIKI_ENABLED`,
  `CIVYK_WIKI_FILE_CHANGE_THRESHOLD`; `CIVYK_EMBEDDING_BACKEND` now also accepts `openai`.
- **Optional extras** — `pip install civyk-repoix[llm]` (openai SDK) and
  `civyk-repoix[embeddings]` (`sentence-transformers` for local semantic retrieval).
- **Schema** — bumped to `2.2.0` with new `wiki_pages`, `wiki_chunks`, and `wiki_state`
  tables (idempotent `2.1.0 -> 2.2.0` migration).

### Changed

- **15 consolidated tools** (was 14) — `wiki` joins the Extended tier
  (Core 9 / Extended 5 / Specialist 1). Profile listing: `full` (15), `core` (9),
  `minimal` (9 minus conversation).
- Deep-wiki `generation` defaults to **MiniMax-M3** @ `https://api.minimax.io/v1` (still off by
  default; API key read only from `CIVYK_LLM_API_KEY`). Reasoning models emit a `<think>…</think>`
  block that the client strips automatically, and `max_output_tokens` defaults to 16000 to leave
  headroom for the reasoning pass.
- **Recommended setup uses local semantic embeddings** — `pip install civyk-repoix[embeddings,llm]`
  installs `sentence-transformers`; with `embedding_backend: auto` the local `all-MiniLM-L6-v2`
  model is used (offline, free), sharply improving `ask` retrieval over the tf-idf fallback.
- The daemon now reads a **per-repo** config at `memory/codebase-index/config.yaml`
  (auto-created from the global config/template on first run, including the
  `generation`/`wiki` sections; falls back to the global `~/.config/civyk-repoix/config.yaml`).
  `memory/config.json` remains the separate hooks/conversation config.
- Documentation updated across README, `docs/`, and `wiki/` for the deep-wiki feature.
- **Diagrams are inline-only** — embedded in each page's markdown; no separate
  `memory/deep-wiki/<branch>/diagrams/*.mmd` files are written.
- `wiki` status now reports `total_sources` alongside `files_covered`.

### Fixed

- **JSON-mode portability** — providers that reject `response_format=json_object` (e.g. MiniMax,
  HTTP 400) now transparently retry without it, and structure plans are parsed tolerantly
  (fenced/prose-wrapped JSON). Previously this silently defeated LLM planning.
- **Orphan pruning** — pages (and their on-disk `.md` files) that drop out of the plan between
  builds are removed from both the DB and disk; an obsolete `diagrams/` directory is cleaned.
- **Coverage accounting** — coverage % is a true fraction of indexed source files, so it no longer
  exceeds 100% when a page cites a non-source file like `README`.

## [1.5.0] - 2026-03-05

### Changed

#### Tool Consolidation: 44 → 14 Tools

Consolidates 44 MCP tools into 14 using action/mode parameter dispatching.
Reduces cognitive load for AI agents while preserving 100% functionality.

- **14 consolidated tools** replace 44 individual tools via `action` parameter
  dispatching (e.g., `search(action="symbols")` replaces `search_symbols`)
- **~50% token reduction** in tool definitions (~18,700 → ~9,800 tokens)
- **Core** (9 tools): status, search, symbol, file, files, git, understand,
  explore, remember
- **Extended** (4 tools): architecture, quality, context, tests
- **Specialist** (1 tool): conversation
- **Old tools removed** — no backward compatibility layer; old 44 tool names
  are no longer registered in schemas, descriptions, worker, or MCP server
- **Profile-based tool listing** — `tools/list` supports `profile` param:
  `full` (14 tools), `core` (9 tools), `minimal` (9 tools minus conversation)
- **`tier` field** added to `ToolDescription` dataclass for programmatic
  profile filtering (core/extended/specialist)
- **CLI dispatch updated** — `cli_tools.py` routes consolidated tool names
  with `--action` parameter alongside old kebab-case names
- Dynamic `next_steps` hints include concrete FQNs from results
- MCP server name shortened from "civyk-repoix" to "repoix"
- Hook output text updated to reference consolidated tool names
- All 2310 tests pass with updated tool references

### Fixed

- Missing `repo_root` parameter in dispatcher handlers, bad kwargs passthrough
- mypy errors and stale imports from consolidation
- Hooks calling old CLI subcommands (e.g., `index-status` → `status --action
  check`)
- CLI dispatch routing, explore `next_steps` generation, and old tool name
  hints
- Markdown lint warnings in documentation
- Claude Code hooks using wrong output format (`hookSpecificOutput` with
  `additionalContext` → standard `systemMessage` JSON); fixed SessionStart
  compact hook error
- Hooks crash-proofing: wrap `main()` and `cmd_hooks` dispatcher with
  try/except so any crash returns exit 0 (Claude Code treats exit 1 as hook
  error); add `_hook_debug_log()` for diagnostics via `CIVYK_HOOK_DEBUG=1`;
  skip instruction injection on compact (already loaded via CLAUDE.md)
- Outdated stats in README (152→178 files, 6443→7677 symbols), stale
  AGENTS.md references updated to CLAUDE.md, Tool Health Tracker location
  corrected in architecture docs, wiki tool counts updated
- mypy error in `cli_hooks.py` (`Callable` type annotation on handlers dict)

### Documentation

- Rewrote `docs/mcp-tools.md` as 14-tool API reference with action tables,
  parameter docs, and JSON examples
- Updated `README.md` tool tables, CLI examples, performance benchmarks, and
  cache/conversation sections
- Updated `docs/architecture.md` diagrams (mindmap, sequence, flowcharts) for
  consolidated tool names
- Updated `docs/quickstart.md` tool table (45 tool+action rows), workflows,
  CLI examples, and agent instruction templates
- Updated `docs/claude-instructions-template.md` workflows and decision guide
- Updated `setup/config.py` instruction template for hook-injected content

## [1.4.0] - 2026-03-04

### Added

#### Open Brain Integration

Evolves civyk-repoix from a 42-tool MCP server into a 44-tool "Open Brain for
code" with reduced tool surface, vector embeddings, server-side auto-logging,
and smart meta-tools. Clean-slate schema — no backward compatibility with v1.x
databases.

#### Tiered Tool Profiles (Phase 1)

- Tool tier metadata: `core` (19), `extended` (15), `specialist` (10)
- Profile-based filtering in `tools/list`: `core`, `full` (default), `minimal`
- Next-step suggestions appended to tool responses
- Inline understanding cache hints on file-returning tools
- Competitive "INSTEAD OF grep" positioning on 6 core tool descriptions

#### Embedding Engine & Semantic Search (Phase 2)

- `EmbeddingEngine` with 3-backend fallback: `LocalEmbeddingBackend`
  (sentence-transformers), `APIEmbeddingBackend` (HTTP), `TFIDFEmbeddingBackend`
  (stdlib-only, always available)
- `EmbeddingIndex` for in-memory vector similarity search (numpy fast path +
  pure-Python fallback)
- `understanding_embeddings` table for vector storage
- Backend migration detection via `check_backend_migration()`
- Optional dependency: `pip install civyk-repoix[embeddings]`
- **Multi-embedding per file** — symbol-level embeddings alongside file-level
  for files with 20+ symbols (up to 10 per file); `fragment_key` column added
  to `understanding_embeddings`, schema 2.1.0; improved average similarity
  from 0.45 to 0.63 across test queries
- Config-driven backend selection via `daemon.embedding_backend` in config YAML
  or `CIVYK_EMBEDDING_BACKEND` env var (`auto`/`local`/`api`/`tfidf`)
- Background model warm-up via `warm_up_async()` in `_init_components()`

#### Server-Side Auto-Logging (Phase 2)

- Automatic tool call logging to conversation history (no client action needed)
- Per-client log buffers with delayed flush via `contextvars`
- `conversation_tool_calls` table for structured tool call records
- `SKIP_LOG_METHODS` for protocol methods (ping, initialize, tools/list, etc.)

#### AI Understanding Enhancements (Phase 2-3)

- `source` field on `ai_understanding` (agent/indexer/user) — multi-source
  understanding without cross-contamination
- `quality_tier` field (standard/auto/verified)
- UNIQUE constraint updated to `(scope, target_path, source)`
- Auto-understanding generation during indexing (`source="indexer"`)
- Semantic search for understanding cache and conversation history

#### Tool Health & Reliability (Phase 3)

- `tool_health` table tracking call counts and error rates
- Auto-disable after 5 consecutive errors, re-enable after 10-minute cooldown
- Health check before tool dispatch in worker

#### Meta-Tools (Phase 4)

- `explore` — Multi-strategy deep-dive: orchestrates symbol search, code search,
  file listing, callers, components, and conversation history in one call.
  Supports `depth` (quick/standard/deep), `scope`, `token_budget`. Handles
  meta-queries about available tools.
- `remember` — Cross-session key-value memory store with store/recall/list/forget
  actions. Category-based organization.

#### Request Deduplication (Phase 4)

- `ToolContext` frozen dataclass passed through handler chain
- 500ms MD5-keyed dedup cache per (client_id, tool_name, args_hash)
- Auto-prune at 200 entries

### Changed

- Tool count: 42 → 44
- `ai_understanding` schema: added `source`, `quality_tier` columns
- `ai_understanding` UNIQUE constraint: `(scope, target_path)` →
  `(scope, target_path, source)`
- `_add_understanding_cache_hint()` renamed to `_add_response_hints()` with
  next-step suggestions
- `batch_recall_understanding()` uses single SQL IN clause (true batch)

### Fixed

- **Docstring extraction byte-offset bug** — tree-sitter uses byte offsets but
  `_extract_docstring` was slicing a Python `str` (char-indexed); multi-byte
  UTF-8 chars caused offset drift, corrupting 265 docstring endings and 182
  starts; fix: pass content as bytes through extraction chain, decode after
  byte-slicing
- **Shallow auto-understanding** — auto-understanding only used first line of
  docstring; fix: new `_extract_first_paragraph()` includes second paragraph
  when first is < 80 chars, giving richer purpose strings for vector search
- **Startup-blocking race condition** — `check_backend_migration()` was called
  synchronously after `warm_up_async()`, blocking `_init_components()` for up
  to 30s; fix: deferred to background thread via
  `_schedule_embedding_migration_check()`
- **P1-P10 tool response issues** from deep testing: config-driven backend
  auto-detection, backend migration wiring on startup, `include_related`
  fallback to `get_related_files()`, `total_count` in `get_file_symbols`,
  scope-restricted explore, richer auto-understanding from docstrings
- Non-deterministic `hash()` replaced with `hashlib.md5` in TF-IDF embedding
- Store understanding: `INSERT OR REPLACE` → true UPSERT (preserves embedding
  FKs)
- Dedup cache: deepcopy on store/return, ContextVar race for concurrent clients
- Embedding deserialization: validate byte length (no silent truncation)
- Thread-safe model loading with double-checked locking
- Atomic `is_disabled` check in ToolHealthTracker (single UPDATE then SELECT)
- Silent failure upgrades: `logger.debug` → `warning` for 22 operational
  failures in worker, conversation manager, and explore

## [1.3.0] - 2026-02-07

### Added

#### Conversation History

Complete conversation history system for AI agents with SQLite storage and
semantic search. Persists sessions across restarts and context compactions.

- SQLite with WAL mode, FTS5 full-text search, automatic session cleanup
- 6 MCP tools: `log_conversation_turn`, `get_conversation_session`,
  `search_conversations`, `build_conversation_context`, `cleanup_conversations`,
  `export_conversation`
- Context strategies: `balanced`, `recent`, `important`
- CLI: `civyk-repoix conversation list|show|search|context|cleanup|export`
- Importance scoring, sensitive data sanitization, session compaction
- Hook integration with Bash/PowerShell scripts for all 4 supported agents
- Hybrid session ID (agent-provided or generated), `.current_session`
  persistence for Cursor/Copilot, agent type detection from payload
- Config via `memory/config.json`, `max_turns_per_session` enforcement

#### AI Cache Hooks

- Deterministic `recall_understanding`/`store_understanding` via hooks
  (no reliance on AI following instructions)
- Agent-specific output formatters (Claude, Copilot, Cursor, Windsurf)
- `--no-hooks` flag to skip hook installation during setup

#### Multi-Agent Support

- `--agent` CLI flag for all hook commands and config generators
- Dynamic instruction injection via SessionStart hook
- Agent aliases (`augment`->`auggie`, `cursor`->`cursor-agent`)
- MCP support added for Codex, Amazon Q, Gemini, Roo Code, Augment
- Instruction paths use rules directories (avoids overriding user files)
- Agent-specific setup notes during installation

#### Security and Signing

- Sigstore signing, GitHub attestations (SLSA provenance), OpenSSF Scorecard
- SECURITY.md with vulnerability reporting and verification guide

#### CLI Improvements

- Renamed `setup` to `init` (aligned with speckitadv)
- Interactive mode with "All supported agents" option
- Renamed `amazonq` agent key to `q`

### Changed

- Consolidated ~800 lines of Bash/PowerShell hook scripts into ~300 lines
  of Python CLI commands (`civyk-repoix hooks session-start|check-cache|...`)
- GitHub URLs migrated to civyk-official organization
- `importance_threshold` default raised from 0.5 to 0.7
- `retention_days` default changed from 30 to 10

### Fixed

#### Security

- ReDoS in regex patterns (bounded quantifiers, DB URI regex)
- Path traversal, symlink bypass, Unix-style path validation on Windows
- FTS5 syntax injection, session ID validation with length limits
- Subprocess arg limit reduced to 30K (Windows CreateProcess limit)
- AWS credential pattern, error message sanitization, input validation

#### Hooks

- Windows cp1252 encoding (ASCII replacements for Unicode)
- Stop hook path expansion and JSON validation
- Hook merging: deep copy, first-pass filtering, nested matcher support
- Tool-specific markers to prevent conflicts with speckitadv
- `hook_restore_compact` delegates to `hook_session_start`
- Skipped agents now return `success=True`, warning field on config errors
- Cursor/Windsurf timeouts, debug logging for silent failures

#### Conversations

- Turn number race condition (atomic `BEGIN IMMEDIATE` transaction)
- VACUUM loop, connection leak, TOCTOU race in `get_stats()`
- Deduplicated `build_context` logic, O(1) session lookup
- Atomic session file writes, WAL size in cleanup estimation
- `max_storage_mb` enforcement (previously dead code)

#### General

- Branch validation in `build_delta_context_pack`
- Agent alias resolution across all path/config functions
- Instruction paths for OpenCode, KiloCode, Antigravity
- Tool names in `CIVYK_INSTRUCTION_CONTENT`
- CLI subcommand format, flaky health check test

### Documentation

- Reorganized documentation with indexes
- Architecture docs with diagrams (lifecycle, MCP flow, cache, schema)
- Conversation tools and AI cache hooks documentation
- MCP server handler docstrings, Sigstore verification docs

### Tests

- 49 MCP protocol compliance tests (daemon worker + SDK server)
- 10 conversation handler tests, hook config tests (97% coverage)
- Hook verification suite, cp1252 encoding safety tests
- Fixed 384 mypy errors across 21 test files
