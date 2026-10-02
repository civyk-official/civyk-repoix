# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [3.0.0] - 2026-10-02

Two bodies of work since 2.1.1. This is a major release because an upgrade from 2.1.1 needs
action: the Jev settings and environment variables are renamed and `civyk_repoix.MCPServer` is
removed (see Upgrading). The daemon keeps answering while it is busy (design and
measurements: `docs/design/daemon-responsiveness.md`). The tools, the index, the Jev decision
model and the wiki got a quality overhaul (tasks and evidence: `docs/design/quality-overhaul-plan.md`).
The MCP surface stays at 14 tools.

### Upgrading

When upgrading from 2.1.1 to 3.0.0, do these before or right after installing 3.0.0:

1. **Rename your Jev settings and environment variables.** The config section `decision` is now
   `jev`, and the keys and `CIVYK_DECISION_*` variables have new names (table below). 3.0.0 reads
   only the new names: an old section, key or variable is ignored, with no alias and no warning,
   so its value silently stops applying. The repository (not the installed package) holds a
   one-time script that rewrites config files and copies the variables. It needs Python 3.10+,
   plus PyYAML for its parse check. From a checkout, in
   `scripts/migrations/2026-10-02-jev-rename/`:

   ```text
   python migrate_jev_rename.py selftest                 # must print PASS
   python migrate_jev_rename.py --root <dir>             # dry run: lists what would change
   python migrate_jev_rename.py --root <dir> --apply     # back up, rewrite configs, copy env vars
   python migrate_jev_rename.py --root <dir>             # every file should now say "unchanged"
   python migrate_jev_rename.py --root <dir> --apply --remove-old-env   # drop the old variables
   ```

   It finds every `memory/codebase-index/config.yaml` under `--root` (default `D:/work` on Windows,
   the home folder elsewhere) and the global `~/.config/civyk-repoix/config.yaml`. Each rewritten
   file gets a `config.yaml.before-jev-rename` backup. A value that equals the old default is
   replaced by the new default, so a 2.1.1 `enabled: false` becomes `enabled: true`. Environment
   variables are copied only at Windows user scope. On POSIX, and for the current process, the
   script only reports them: rename those yourself. It also reports, without changing them,
   other files that still name `CIVYK_DECISION_*` (MCP and agent configs, `.env` files, scripts).
   Its `README.md` covers the options, exit codes and rollback.

   | 2.1.1 name | 3.0.0 name |
   |---|---|
   | section `decision` | section `jev` |
   | `base_url` | `endpoint_url` |
   | `timeout_s` | `call_timeout_s` |
   | `max_retries` | `max_retries_per_call` |
   | `max_concurrency` | `max_parallel_calls` |
   | `max_calls_per_run` | `max_calls_per_batch` |
   | `max_tokens_per_run` | `max_input_tokens_per_batch` |
   | `send_source` | `allow_source_upload` |
   | `ask_filter` | `wiki_ask_filter_chunks` |
   | `resolve_edges` | `index_resolve_edges` |
   | `lint_pairs` | `wiki_lint_flag_pairs` |
   | `search_rerank` | `search_rerank_results` |
   | `CIVYK_DECISION_API_KEY` | `CIVYK_REPOIX_JEV_API_KEY` |
   | `CIVYK_DECISION_ENABLED` | `CIVYK_REPOIX_JEV_ENABLED` |
   | `CIVYK_DECISION_BASE_URL` | `CIVYK_REPOIX_JEV_ENDPOINT_URL` |
   | `CIVYK_DECISION_MODEL` | `CIVYK_REPOIX_JEV_MODEL` |
   | `CIVYK_DECISION_SEND_SOURCE` | `CIVYK_REPOIX_JEV_ALLOW_SOURCE_UPLOAD` |

   `enabled` and `model` keep their names. The script also maps the keys that only development
   builds had (for example `explore_rerank`, `dead_code_triage`, `max_cost_per_day`). The
   `status check` block and the `config` tool's field for the model are also named `jev` now.
   The full settings reference is the README's "Jev decision model" section.

2. **Jev now calls out by default once a key is set.** `jev.enabled` defaults to `true` (it was
   `false`). With `CIVYK_REPOIX_JEV_API_KEY` set, four features that make a few calls on index
   metadata are on: `search_rerank_results`, `explore_rerank_results`, `wiki_ask_filter_chunks`
   and `wiki_lint_flag_pairs`. The features that send source or make many calls stay off:
   `allow_source_upload`, `index_resolve_edges`, `wiki_check_citations` and
   `quality_triage_dead_code`. Paid calls stop for the rest of the UTC day once they cost
   `daily_cost_cap_usd` (1.0 USD per index; `0` allows none). Without the key nothing is sent.
   To turn everything off, set `CIVYK_REPOIX_JEV_ENABLED=0` or `jev.enabled: false`.
3. **`civyk_repoix.MCPServer` is removed.** Importing it raises `AttributeError`. Run
   `civyk-repoix mcp` instead. `tools/list` no longer takes a `profile` (`core`, `minimal`):
   every client gets all 14 tools.
4. **Remove config keys that are no longer read:** the `branches` and `context` sections,
   `daemon.idle_daemon_timeout_s` and `daemon.startup_mode`. They are ignored.
5. **Indexes.** A 2.1.1 index (schema 2.11.0) migrates to 2.25.0 in one step on first start and
   re-parses its files once, because the extractor versions changed. An index that a development
   build stamped 2.12.0 to 2.24.0 is not migrated; the daemon logs that it does not know the
   version. Delete that repository's `memory/codebase-index/index.db` and let it rebuild. This
   loses that repository's `remember` entries, wiki rows and Jev ledger.
6. **Changed defaults:** `daemon.cleanup_interval_s` is 60 s (was 300 s). With
   `daemon.embedding_backend: auto` (the default) the backend is still chosen local, then API,
   then TF-IDF: TF-IDF is used when `sentence-transformers` is not installed, or the local
   backend cannot be created, and no API endpoint is configured. What changed is that a local model that
   fails to load is reported and retried (after 30 s, doubling up to 600 s) instead of switching
   to TF-IDF silently. Set `daemon.embedding_backend: tfidf` to use TF-IDF always.

### Added

- **Jev features:** `explore_rerank_results` reranks the top 12 hits of an `explore` question.
  `wiki_check_citations` checks each cited sentence against its cited lines and needs
  `allow_source_upload`. `quality_triage_dead_code` labels `dead_code` rows. Calls a tool makes
  while someone waits end by `tool_deadline_s` (3 s), and the tool then answers without the model.
- **Jev ledger and spend control:** every answer is recorded with the feature that asked and
  replayed from cache. `daily_cost_cap_usd` caps paid calls. After a refusal (HTTP 401, 402, 403,
  520 or HTML) the endpoint is paused for `refusal_pause_s`, doubling up to `refusal_pause_max_s`.
  `civyk-repoix restore-edge` undoes an edge the resolver deleted. `status check` shows the
  model's state and spend per feature.
- **Retrieval:** a hybrid retriever (full-text, embeddings, exact names) behind `explore`, which
  routes a question by its shape (identifier, "tests for X", path). `context(task)` is seeded
  from the retriever. A miss in `symbol`, `file` or `search definition` answers with the nearest
  names. An FTS5 index of code chunks backs `search code` and `explore`.
- **Index content:** Markdown is indexed as nested sections. Relative links and `[[wiki links]]`
  become `doc_link` edges, and `file related` shows `links_to` and `linked_from`. Doc links never
  enter code caller, impact or provenance counts. Architecture components come from project units
  (workspace manifests, `src/<pkg>`).
- **Memories are searchable by meaning:** the top 2 are attached to `explore`, `context` and
  `wiki ask`, labelled as untrusted.
- **MCP:** `initialize` returns instructions that map needs to calls. The report and each wiki
  page are MCP resources (`repoix://report`, `repoix://wiki/<page>`). Read-only tools carry
  `readOnlyHint`.
- **Tool arguments are normalized:** common parameter and action synonyms are accepted, values
  sent as text are coerced, out-of-range numbers are clamped (and the answer says so), and an
  unknown parameter gets a did-you-mean. `--action` is optional on the CLI when a tool has a
  default action.
- **Status:** `status check` counts index errors, lists partial parses with their first error
  line, shows real progress during a forced reindex, and lists installed repoix skills older than
  the package.
- **Wiki:** planner v2 gives every page a type and a descriptive title. `save_page` takes an
  optional `title`. Citations are validated on save, degraded pages are reported, and
  `plan` and `page_context` report the stale sections and changed files.
- **Daemon resilience:** a loop watchdog writes every thread's stack to `daemon-fault.log` after
  `loop_stall_warn_s` (5 s) and exits with status 70 after `loop_stall_exit_s` (180 s). A
  heartbeat stamps the state file every `heartbeat_interval_s` (30 s). New `daemon` settings:
  `loop_stall_warn_s`, `loop_stall_exit_s`, `health_probe_timeout_s`,
  `health_failures_before_restart` and `heartbeat_interval_s`.
- **`civyk-repoix status`** shows calls running on the tool lane, slow calls waiting, and the
  embedding model's state.
- **Logs:** each MCP gateway writes its own `mcp-<pid>.log`, and every log line carries the process
  id and the request it serves. Pruning skips the log of a process that is still running.
- **Config keys:** `tools.duplicates_budget_s` and `tools.duplicates_max_symbols`. The `index.*`
  keys (for example `max_file_size_mb`, `languages`) now take effect.
- **Developer tooling** (in the repository, not the package): a real-repository smoke matrix
  (`scripts/smoke_tools.py`), a frozen decision-model evaluation (`scripts/eval_decisions.py`), a
  local stand-in for the decision endpoint (`scripts/decision_standin.py`), contract tests for every
  tool and action, a daemon load test (`tests/integration/test_responsiveness.py`, marker
  `responsiveness`), and guards for module size, layers and annotation resolution on Python 3.10.

### Changed

- **One answer shape:** no null fields at any depth, hints at most 15% of an answer, and compact
  JSON over MCP. MCP gets terse tool descriptions; the CLI keeps the full ones. Results carry the
  code lines that matter (referencing lines, matching lines with one line of context). `file
  symbols` is compact by default.
- **`wiki ask`** answers are smaller: at most 2 chunks per page and 8 in all.
- **`quality impact`** pages callers and affected files by `limit` and `offset`.
  `tests recommended` defaults to `limit` 20 and ranks its rows with a `reason` per row.
- **`git` defaults mean what they say:** an omitted `since` is 7 days for `changes` and 30 days
  for `hotspots` (before, `hotspots --since 7d` ran 30 days). `diff` compares against the
  repository's default branch (or `origin/<name>` when there is no local copy) instead of a fixed
  `main`. `source_only` is on by default for `hotspots` and now also drops tests, documents and
  generated files. A `since` that is not a positive number with `m`, `h`, `d` or `w`, nor an ISO
  date, is an error that names the accepted forms, not a silent 7 days. Each row says whether its
  file was added, modified, deleted or renamed. `context delta` gives real line numbers and counts
  what it leaves out.
- **Symbol-name queries are literal:** `_` is an ordinary character (it used to match any one
  character), `*` or `%` matches any run of characters, and `|` separates alternatives. Hits are
  ranked exact name, whole word, prefix, then part of a name, and each carries a `match` and a
  `score`. A query that used `_` as a one-character wildcard must use `*` instead; a query
  holding a literal `|` is now split into alternatives.
- **Symbol FQNs can change:** a file that shares its stem with another (`foo.py` and `foo.ts`)
  keeps its extension in the FQN prefix, and a repeated name inside one file gets `#n`. Before,
  the second file's symbols were dropped with "UNIQUE constraint failed: symbols.fqn"; a
  collision that still occurs is renamed and recorded as an `fqn_collision` index error. FQNs
  stored outside the index (`remember` entries, agent notes) may need updating.
- **`symbol` and `search code`** accept a partial fqn when it resolves uniquely.
  `search symbols` hides Markdown headings and data keys unless a `kind` asks for them.
- **Endpoint detection** no longer lists UI reducers or every file under an `api` directory.
- **Extraction:** scope-aware references with resolved imports; edges carry the reference line
  and kind; TypeScript and JavaScript declarations, signatures and exports are complete; data
  files no longer produce symbols for every entry. A change of extractor version re-parses the
  affected files at daemon start.
- **Queries do not wait for writes:** they run on read-only WAL connections
  (`REPOIX_DB_READERS`, 6 by default).
- **Parsing runs in worker processes:** `REPOIX_PARSE_WORKERS` (default half the cores, `0` parses
  in the daemon) now counts processes, not threads.
- **Shorter interpreter switch interval:** the daemon sets Python's thread switch interval to
  0.5 ms so that reads of many rows are not starved by a computing thread, and restores the old
  value at stop. `REPOIX_SWITCH_INTERVAL_S` (new) sets it in seconds (kept between 0.0001 and
  0.005); `0` leaves Python's own value.
- **Slow calls share three slots:** full-text and semantic search, duplicates, git, context delta,
  the report and LLM-backed wiki actions. Once 16 wait, the next one is answered as busy.
- **Embedding model:** a model that fails to load is reported and retried (after 30 s, doubling
  to ten minutes) instead of falling back to TF-IDF. While it loads, semantic search and
  `wiki ask` answer at once with a note. A cached model loads without contacting the model hub.
- **The daemon runs on a selector event loop** (Windows included) and is built from the loaded
  configuration; the `daemon` section never took effect before.
- **New databases use incremental auto-vacuum** and give free pages back after a swap or delete.
- **Internal layout:** the indexer, worker and manager are split into a core plus mixins; tool
  handlers are one module per tool family; one registry describes every tool, action and
  parameter, and the docs' tool tables are generated from it (`scripts/gen_registry_docs.py`).
  The `civyk_repoix` public names are unchanged apart from `MCPServer`.

### Fixed

- **The daemon outlives its client:** it is launched outside the client's job object and process
  tree, so a host that kills its MCP servers' job no longer ends the daemon.
- **`civyk-repoix mcp` survives daemon restarts:** it answers `initialize` and `tools/list`
  itself and reconnects. A lost worker fails only the calls in flight; a read-only call is retried
  once.
- **The event loop is never blocked** by tool handlers, indexing, housekeeping, the control plane
  or logging. Previously a 1,200-file re-index stalled every request for up to 17 s.
- **Reads on a network share or mapped drive work again:** every read on such a repository failed
  with "invalid uri authority" while indexing went on. The reader pool now builds
  `file:////server/share/...` URIs, and a read connection that cannot be opened falls back to the
  writer's connection.
- **Healthy workers are not restarted:** a restart needs a closed listening socket or three
  unanswered probes in a row.
- **The daemon no longer loses its listening socket on Windows** when a client aborts during
  accept. A failed health check or idle cleanup restarts after a pause.
- **The embedding model no longer degrades to TF-IDF after a slow start** and purges its
  embeddings, which had broken `wiki ask` (dimension mismatch).
- **`quality duplicates`** prunes candidate pairs and has a time budget: 217 s became about 1 s on
  a large repository, with the same top groups. A cut scan says `scan_truncated`.
- **Edges into a re-indexed file come back as they were**, also after a restart. A `unique`,
  `ambiguous` or `model` edge is bound again by name; a kept import edge whose target lost its
  symbol is bound only through the file's own imports.
- **The edge resolver** applies an answer only to an edge whose source is still in the same file.
- **A failed first index pass** shows as `error` in `status`, and the watcher and delta check still
  start; unbounded `IN` lists no longer fail with `too many SQL variables`.
- **Without `allow_source_upload`, no source reaches Jev:** code blocks (fenced or indented) are
  cut from wiki prose before it is sent, and the wiki citation check requires the consent.
- **Extractor and binder gaps:** value uses, Java, C#, Go, SQL naming, `.mts` and `.cts`, deep
  nesting, and pending doc links dropped when the target was indexed later.
- A stop request reaches indexing, the embedding pass and every LLM-driven wiki action, including
  `wiki lint`.
- `symbol hierarchy` explains an empty answer.
- Torch thread use is bounded where the model is called, so a query no longer adds 16 threads.

### Removed

- `civyk_repoix.MCPServer`, the standalone `mcp_server.py` and `service.py`.
- The `tools/list` profile tiers (`core`, `minimal`).
- Config: the `branches` and `context` sections, `daemon.idle_daemon_timeout_s`,
  `daemon.startup_mode`, and the `decision` section with its `CIVYK_DECISION_*` variables
  (replaced by `jev`; see Upgrading).
- The unused `GitWatcher`.

### Security

- **A path the model names is judged from its text before the disk is read.** The gateway admits
  a `repo_root` only inside the session's workspace (`--repo-root`, the host's roots or the working
  directory). A network share or device path (`\\host\share`, `\\?\`, `\\.\`) is never opened
  unless the user named that share, since opening one on Windows offers the user's credentials to
  that host. A drive letter, mapped or not, still works.
- `status(action="reindex", paths=[...])` refuses a path on another share or a device path before
  it touches the path.
- Jev sends index metadata (names, kinds, paths, docstrings, wiki prose with code removed, the
  query) unless `allow_source_upload` is set. The key is read only from
  `CIVYK_REPOIX_JEV_API_KEY`, never from a config file.

## [2.1.1] - 2026-09-29

### Fixed

- **The daemon no longer dies when a worker is stopped.** A worker stop (health
  restart, `config set`, idle cleanup) closed the shared SQLite connection while
  passes in worker threads were still running statements on it, which ended the
  whole daemon with an access violation and took every repo's worker with it.
  `Database.close()` now takes the connection lock like every other access and
  runs off the event loop; a thread that arrives afterwards gets `RuntimeError`.
- **The worker answers while symbols are embedded.** The embedding pass (up to
  8,000 symbols) ran on the event loop, so requests, pings and health checks
  waited until it ended; the health check then restarted the busy worker. The
  pass runs in a worker thread and ends when the worker stops.
- **Background passes no longer pile up.** Every batch of file events started
  its own embedding and edge-resolution pass. One pass of a kind runs at a time,
  and requests that arrive meanwhile are covered by one further run.

### Added

- **Fault log.** The daemon writes the traceback of a fatal native fault to
  `daemon-fault.log` in the log directory; its standard streams are on the null
  device, so such a fault left nothing in `daemon.log`.

## [2.1.0] - 2026-09-19

### Added

- **Optional decision model (TypeSafe Jev via OpenRouter).** A new `decision:`
  config block and `engines/decision.py` client for a System One decisions API:
  typed answers (choice / score / noul) with calibrated probabilities, no text
  generation. Everything defaults off; the client is unconfigured unless
  `decision.enabled` and `CIVYK_DECISION_API_KEY` are set, each feature has its
  own switch, and `decision.send_source` is a separate consent for features that
  upload source bodies. Answers are cached by content hash in the index database
  (`decision_cache`, schema 2.10.0) so repeated questions and re-indexes replay
  without spend; `scripts/eval_decisions.py` is the vendor gate against this
  repo's live index. Design and measurements: `docs/design/jev-decision-model-plan.md`.
- **`wiki ask` retrieval gate** (`decision.ask_filter`): answer and deep modes
  screen retrieved chunks for relevance and prompt injection and report whether
  the wiki appears to cover the question; the response carries a `decisions`
  summary (calls, tokens, cost, kept/dropped).
- **`search(action="semantic")`**: meaning-based symbol search over the stored
  symbol embeddings from a natural-language query, in the daemon and the
  standalone server. Returns an honest empty result when no embeddings exist for
  the current backend. With `decision.search_rerank` the 30-candidate shortlist
  is reordered by the decision model and a "no confident match" note is added
  when nothing fits.
- **Wiki lint pair pre-selection** (`decision.lint_pairs`): page pairs that share
  source files are scored for contradiction and duplication in one decision call;
  only flagged pages reach the LLM (which still writes every finding), and the
  LLM is skipped when nothing is flagged.
- **Ambiguous-edge resolution** (`decision.resolve_edges` + `send_source`): a
  daemon post-index pass re-binds `ambiguous` reference edges to the definition
  the model judges correct (`resolution='model'`, confidence on the edge, schema
  2.11.0) or deletes them when the name is external (builtins such as `.close()`
  and `.clear()`). `model` edges are excluded from rankings; the evidence set has
  one definition (`EVIDENCE_RESOLUTIONS` in `models/edge.py`) shared by the
  repository and the report generator.

### Fixed

- **`wiki ask` code context was keyword junk.** The symbols attached to answers
  came from exact keyword-name matching and were off-topic for ordinary
  questions. `ContextBuilder` gains `seed_symbols`; ask seeds the pack from
  embedding search when symbol embeddings exist and says so in the note when it
  cannot.
- **Reasoning stripper discarded answers that quote a think tag.** An answer
  that documents `<think>` handling quotes the close tag inside a code fence; the
  stripper treated it as stray reasoning and dropped everything before it. Tags
  inside fenced or inline code are now ignored.

### Changed

- README no longer claims "100% offline": the product is local by default, and
  the LLM and decision-model features are documented as explicit opt-ins with
  what each one sends.

## [2.0.1] - 2026-08-15

### Fixed

- **Daemon instances no longer pile up.** Three compounding lifecycle defects
  let orphaned `daemon start --foreground` processes accumulate (observed: 10
  live daemons for 2 IDE windows):
  1. A start that found the daemon already running fell through to
     `manager.stop()` in the foreground CLI's cleanup, overwriting the shared
     state file with `"stopped"` while the real daemon kept serving — blinding
     the singleton guard for every later start.
  2. With the state file lying, a duplicate start proceeded to bind the control
     port; the Windows port-exclusion fallback then silently bound an
     OS-assigned port, so the duplicate served forever instead of exiting.
  3. The availability probe used a 1-second identity-handshake timeout, which
     false-negatives whenever the daemon is busy (e.g. mid-index), so each new
     MCP session or CLI query spawned another duplicate.
  The singleton is now an OS-level exclusive file lock (`daemon.lock`) held
  for the daemon's lifetime and released by the OS on process death — it can
  neither go stale like the state file nor time out like a socket probe. A
  start that cannot take the lock raises `DaemonAlreadyRunningError` and the
  CLI exits without teardown (previously that teardown corrupted the state
  file). Defense-in-depth for daemons predating the lock:
  `start_server(exclusive=True)` probes a conflicting control port for an
  identity-matched daemon and raises `AlreadyServingError` instead of falling
  back to an OS-assigned port (the fallback remains for genuine Windows port
  exclusions and foreign conflicts). The daemon binds its control socket
  before touching the state file, `stop()` refuses to flip the shared state
  file unless the calling process is the recorded daemon, and daemon-liveness
  probes now allow 5 seconds for the identity greeting. A side effect of the
  pile-up is also gone: many daemons sharing one `daemon.log` broke Windows
  rename-based log rotation, freezing the log at its size cap.
- **The daemon self-heals a dead control listener.** Live soak-testing the
  singleton fix surfaced a pre-existing co-contributor to the pile-up: on
  Windows, a client aborting mid-accept (`WinError 64`, easy to hit while the
  event loop is starved by indexing) makes asyncio's proactor close the
  LISTENING socket permanently — the daemon process keeps running but is
  unreachable, looking "not running" to every probe (pre-fix: infinite
  duplicate spawns; post-fix: the singleton lock would block replacements).
  `SocketTransport.is_serving` now detects the closed listener (the `Server`
  object still reports serving), and the daemon's health loop rebinds it;
  after 3 consecutive rebind failures the daemon shuts down and releases the
  singleton lock so a fresh daemon can take over. Workers already self-healed
  through the existing health checks; the control socket was the only
  unsupervised listener.
- The MCP `initialize` handshake reports the real package version in
  `serverInfo` instead of a hardcoded `1.0.0`.

## [2.0.0] - 2026-07-13

A ground-up re-architecture. 2.0 removes three speculative subsystems — hooks,
conversation history, and the AI understanding cache — and rebuilds the core
around what agents demonstrably use: a deterministic, evidence-graded code
index, agent skills for every skill-capable agent, and a self-healing deep
wiki. Every ranking, diagram, and coverage number now traces to verifiable
evidence; upgrades heal themselves; and per-session LLM cost drops because
nothing generates speculative content nobody reads. Simple, deterministic,
low-maintenance — by design.

### Removed

- **The AI understanding cache** (`understand` tool with store/recall/stats/
  invalidate, the indexer's auto-generation tier, understanding embeddings,
  and the wiki mirror; 15 tools → 14). Measured on this repo: every one of its
  207 rows was indexer-generated boilerplate (first docstring paragraphs),
  none was agent-authored, and recalls were effectively zero — pure storage,
  embedding, and maintenance cost with no reads. Durable knowledge now lives
  where it is actually consumed: the committed deep wiki (plus Research Notes)
  for prose, `remember` for decisions and conventions, and
  `file(action="symbols")` for cheap file orientation. Schema 2.9.0 drops the
  `ai_understanding` and `understanding_embeddings` tables on first open.
- **The hooks subsystem.** Its only two useful handlers fired solely for
  model-authored understanding entries that never existed; the rest were
  nudges agents ignored. Skills do the one job hooks did well — surfacing
  repoix at decision time — in every skill-capable agent, not just Claude.
- **The `conversation` tool and its history database.** It was fed entirely by
  hook data, so without hooks it could only return empty results.
- **Config keys read by nothing**: `generation.enabled`,
  `generation.max_concurrency`, `wiki.deep_research`. Old config files
  carrying them still load (unknown keys are ignored).

### Added

- **Agent skills for every skill-capable agent** — Claude, Cursor, Copilot,
  and Windsurf all get `repoix`, `repoix-map`, and `repoix-wiki`, installed
  into directories chosen so each configured agent discovers each skill
  exactly once (overlapping read paths collapse; `init` prints the
  directory → agent map). `skill install`/`skill status` take `--agent`, and
  `status` reports every directory an agent actually reads.
- **A rules file for every agent, in the form that agent loads** — `.mdc`
  with `alwaysApply` for Cursor, a `trigger` for Windsurf. The always-loaded
  rules block itself shrank to a fifth of its size (~4,000 → ~2,300 chars):
  routing rules only, with the manual in the `repoix` skill.
- **Edge provenance — a guess is never presented as a fact.** Every reference
  edge records how it was resolved (`local`/`import`/`unique`/`ambiguous`);
  rankings, diagrams, and impact analysis use evidence-backed edges only.
- **`memory/codebase-index/graph.json`** — the file-level dependency graph as
  a machine-readable artifact beside REPORT.md: weighted, resolved,
  provenance-tagged file→file edges over production code, tests excluded. The
  rules block and skills route "what depends on what, file by file" to it.
- **Wiki pages grounded in the reference graph** — each page carries its own
  file→file edges in both directions with the resolution mix stated.
- **`wiki.include_tests`** (default `false`) — first-class wiki pages for
  repos whose product genuinely is a test suite.

### Fixed

The index, the report, the wiki, and the upgrade path were audited end to end;
the defects below shipped in 1.x and are all covered by regression tests.

- **Index correctness.** One reference now creates one edge (previously every
  same-named symbol in the repo got one, inflating fan-in/fan-out, component
  dependencies, and impact analysis). Module-style imports (`import utils`)
  contribute resolution evidence; module keys are anchored so `import config`
  cannot resolve into an unrelated `config.py`; relative imports resolve
  against the importer's package; every module in `import os, pkg.mod` is
  recorded. Test detection is one implementation for all callers —
  `is_test_path` is registered as a SQLite function, ending three-way drift
  between Python, SQL `LIKE`, and `fnmatch` semantics across OSes.
- **Report and rankings.** Schema 2.7.0 force-re-resolves edges that predate
  provenance (on this repo: 15,471 evidence-free edges → 0, ~6,900 phantom
  edges gone), so REPORT.md no longer crowns `close`/`get`/`set` as the
  codebase's most central symbols. Rankings count production references only
  (91–97% of "refs" were test calls); entry points admit attribute-dispatched
  methods; homonyms keep separate evidence; production files whose names
  merely contain "test" are no longer silently dropped. Edge classification
  is 7.3× faster at scale.
- **Wiki integrity.** A decomposed module family is recognized by its
  children, so re-chunked modules retire superseded pages instead of
  accumulating six overlapping ones; the agent build path now prunes, writes
  `digests.json`, honors `steering.yaml` exclusions, and plans structurally
  and offline — deterministically, with no LLM call — ending plan churn and
  duplicate pages. Coverage has one definition across build, `wiki status`,
  and lint. The write lock heartbeats (a long build is no longer judged
  stale), detects dead holders correctly on Windows, and survives two writers
  racing over a crashed holder's lock. A half-built index no longer prunes
  pages whose files are still on disk; the dead-link lint now resolves links
  for real (it previously skipped exactly the links that were dead); an
  embedding-backend change is reported with the fix instead of masquerading
  as an empty wiki; `plan` is gated by `default_branch_only` like `generate`;
  `wiki.enabled` no longer flips to `true` when a config file loads; agent
  pages load the persisted business context; and the wiki no longer documents
  its own test suite as the product's largest subsystem.
- **Upgrade safety.** A 1.x index opens cleanly (index creation no longer
  references a column only the migration adds). Stale hook entries
  self-remove: `civyk-repoix hooks …` short-circuits before argparse and
  exits 0, so a leftover hook can never block an agent action. The rules-file
  refresh proves the civyk block's extent before replacing it — a hand-edited
  block is left byte-for-byte untouched — is fence-aware, retrofits missing
  frontmatter, and cleans up retired 1.x rules paths. The hook prune reports
  only removals it actually made.

### Changed

- **`profile="minimal"` is an alias of `"core"`** and the "specialist" tier is
  gone; the 14 tools are `core` (8) and `extended` (6).
- **`wiki.retrieval_top_k` is now actually used** — the tool's `top_k` falls
  back to the configured value instead of a hardcoded 20.

### Migration

- **Nothing to do.** `init` strips stale hook entries (and a stale hook that
  fires first removes them itself), refreshes the rules block in place, and
  the index re-resolves itself once on the next pass. Schema 2.9.0 drops the
  understanding tables automatically on first open. `memory/conversations/`
  is left on disk untouched — delete it if you want the space back. An
  existing wiki heals its plan of record on the next ordinary build; no
  `--replan` needed.

## [1.12.0] - 2026-07-12

### Added

- **Pinned wiki plan of record** — the stored page set is now authoritative
  across rebuilds: pages keep their `page_id` (hysteresis), new subsystems
  arrive as *add* amendments, pages with vanished sources are *retired*, and
  every structural change is logged to `memory/deep-wiki/plan-log.jsonl`
  with a drift signal when the structure churns. `replan=true`
  (generate/plan) is the escape hatch: fresh structure wins, old pages are
  remapped to successors by source overlap so prose carries forward. Under
  LLM planning, page identity is now derived from the dominant directory of
  the assigned files — LLM names survive only as display titles, ending the
  page-rename churn that defeated the reuse cache.
- **Agent-session wiki generation (no API key)** — new `wiki` actions
  `plan` (pinned pages + amendments + dependency-ordered stale build list),
  `page_context` (per-page grounding, citation seeds, prior markdown), and
  `save_page` (citation-validated, atomically written, hash-stamped,
  re-embedded). The calling agent writes the prose; the tools own identity,
  staleness, grounding, and storage. Works without `wiki.enabled` or an LLM
  credential.
- **`repoix-wiki` skill** — user-invoked `/repoix-wiki` flow over those
  actions: plan → report amendments/drift → write or surgically edit stale
  pages bottom-up → save → lint, capped per invocation with explicit
  continuation. The API-LLM `generate` path remains for headless/CI use.
- **`repoix` playbook: artifact routing** — new "Which artifact for which
  question" section (REPORT.md / wiki ask / graph tools / grep) plus
  staleness advice pointing at `/repoix-map` and `/repoix-wiki`.
- **`skill show --skill <name>`** — inspect any embedded skill; skill CLI
  help now documents that installs cover all three skills.
- **REPORT.md test-suite overview** — a `## Tests` section (test file/function
  counts, per-directory layout, largest test files, pointer to
  `tests(action="recommended")`) so agents working on tests orient from the
  same one-read artifact.
- **REPORT.md layout fallback** — when keyword-based component detection
  covers <50% of source files, the report renders a directory map derived
  from actual paths instead of a misleading near-empty components table.

### Fixed

- **Windows-style path lookups on POSIX** — `_normalize_path` converts
  backslashes explicitly, so cached-understanding checks (and any DB path
  query) match rows regardless of the caller's path separators.
- **CLI query dropped long-running responses** — `civyk-repoix query` read a
  single protocol frame, so on calls longer than the keepalive interval the
  worker's `internal.ping` was mistaken for the response ("No response from
  worker" while the worker completed). The CLI now skips keepalive frames
  under a 300s overall deadline.
- **Wiki plan of record now pins on first contact** — `wiki(action="plan")`
  persists skeleton rows for not-yet-stored module pages (and, after a
  replan, for the replanned structure), so page ids handed out by `plan`
  can never vanish before `page_context`/`save_page`. Decomposed families
  are pinned too: the parent stores its children in `related_pages` and
  reconciliation reconstructs stored families instead of letting the
  planner re-decompose differently on every call. The drift signal now
  counts only module-level churn, not first-appearance records of
  deterministic pages.
- **REPORT.md ranked test scaffolding and homonyms** — entry points and
  fan-in/fan-out excluded nothing: thousands of exported test defs drowned
  the real entry points, and name-resolved reference edges repeated the
  same symbol name at multiple files with inflated counts. Test paths are
  now excluded, only function/class/method kinds rank, homonyms collapse to
  the highest-degree definition, and component dependencies skip the test
  layer and count distinct referencing symbols.

### Reliability (wiki deep-review hardening)

- **Cross-process wiki write lock** — generate, save_page, and replan
  application hold an advisory lock file (stale-detected by pid/age), so a
  CLI/standalone-server writer can no longer interleave with a daemon
  background build; in-daemon writers additionally serialize on the build
  lock for the build's full duration.
- **Agent-maintained wikis are protected** — automatic (threshold and
  scheduled) API-LLM builds skip a wiki whose last writer was an agent
  session; an explicit `wiki(action="generate")` hands it back.
- **Agent saves keep artifacts truthful** — save_page refreshes the wiki
  index/manifest and wiki_state (page count, last-built, model), computes
  its reuse-cache hash with the same inputs as the generator (so unchanged
  agent pages take the free reuse path in later builds), and re-flags
  synthesis pages stale via a sentinel that works even for zero-source
  pages.
- **Replan application is atomic** — remap upserts and prunes commit in one
  transaction (Database.transaction now supports join-outer nesting), file
  ops follow the commit; a mid-replan crash can no longer pin both an old
  page and its successor.
- **Replan audit completeness** — two old pages merging into one successor
  now record a retire amendment for the loser; an emptied catch-all page is
  retired instead of living forever; the plan audit log is branch-scoped
  (memory/deep-wiki/<branch>/plan-log.jsonl) and rotates.
- **Gates** — plan(replan=true) honors default_branch_only like the other
  mutating actions; the skill documents targeted single-page rebuilds and
  the one-time --replan migration for wikis predating plan pinning.

## [1.11.0] - 2026-07-12

### Added

- **Static repo report (`REPORT.md`)** — a pre-digested structural overview
  (components, dependencies, likely entry points, fan-in/fan-out symbols,
  change hotspots) written to `memory/codebase-index/REPORT.md` and refreshed
  automatically after every full or delta index pass. Any agent can orient
  itself with a single file read, in any client, with no MCP setup.
  Regenerate on demand with `status(action="report")` (MCP) or the new
  `civyk-repoix report` CLI command (`--print` echoes it to stdout).
- **`repoix-map` skill** — a user-invoked `/repoix-map` orientation flow:
  index if needed, refresh the report, summarize it, and set report-first /
  graph-second defaults for the session. Installed alongside `repoix` by
  `civyk-repoix skill install` and `civyk-repoix init`.
- **OR + substring symbol search** — `search(action="symbols")` now treats
  bare terms as substrings (`"UserService"`), honors SQL wildcards when
  present (`"get_%"` — expert mode, used verbatim and never split), and
  supports `|` OR alternatives (`"save|store"`), removing the LIKE-syntax
  sharp edge that pushed agents back to grep. Normalization lives in the
  agent-facing handler; internal callers keep exact LIKE semantics, and the
  repository accepts an explicit pattern list.
- `civyk-repoix report` CLI command (`--print` echoes the report), plus a
  shared `report_result` helper used by both MCP dispatchers.

### Changed

- **Instruction surface collapsed and reframed** — the CLAUDE.md template is
  now a short report-first directive; the `repoix` skill's mandatory shouting
  ("MANDATORY", "you MUST") became positively framed defaults; the session
  brief, the init-installed instruction block, and the grep nudge now lead
  with `REPORT.md` before tool routing.

### Reliability (deep-review hardening)

- Report generation runs off the daemon event loop (asyncio.to_thread) and
  writes atomically (tmp + rename), so MCP requests never stall behind git
  churn analysis and readers never see a torn file.
- Report regeneration is serialized per repo with coalescing (single writer,
  bounded trailing regens), rate-limited to one unforced regen per 60s, and
  deferred/failed refreshes are flushed by the periodic delta pass — the
  last indexed change always lands in the report.
- Staging force-rebuilds publish the report only after the atomic index swap
  commits; watcher-indexed changes (the primary live path) refresh the
  report too; a missing REPORT.md materializes on the next delta pass.
- Report content is source-filtered (no docs/lockfile noise in hotspots or
  entry points), stamped with generation time vs index freshness honestly,
  and uses pre-aggregated edge scans plus a bounded dependency query.
- CLI `report` gives a specific database-is-locked hint instead of a
  traceback; `status(action="report")` returns structured errors.

## [1.10.1] - 2026-07-11

### Changed

- **`repoix` skill hardened to a mandatory playbook** (skill-creator standards
  review): the trigger description is now deliberately pushy ("MANDATORY for any
  multi-file code investigation … even when the user never mentions repoix"),
  and the advisory guidance became six binding rules (route semantic tasks
  through repoix, recall-before-read, tests-before-commit, store-after-analysis,
  CLI fallback when the MCP is down, and where grep/Read stays the required
  choice) — each with a one-line rationale. Re-run `civyk-repoix skill install`
  to pick it up (version-gated installs upgrade automatically).

## [1.10.0] - 2026-07-10

### Added

- **Embedded `repoix` agent skill + `civyk-repoix skill` command** — the adoption layer
  for agents that under-use the semantic index. The skill (decision matrix vs grep,
  scenario recipes, `civyk-repoix query` CLI fallback, freshness rules) is embedded in
  the executable as a constant (works in wheels and Nuitka builds alike) and installs
  with `civyk-repoix skill install --scope user|project` (`show`/`status` included).
  Installs are version-stamped: re-running upgrades older installs and leaves
  same-or-newer ones untouched. `civyk-repoix init` now installs the project-scope
  skill for Claude by default (`--no-skill` to opt out).
- **Decision-time semantic nudge (`suggest-repoix` hook)** — a PreToolUse `Grep|Glob`
  hook that, when a search pattern looks like SYMBOL discovery (identifier-shaped, no
  regex metacharacters), injects one `additionalContext` line pointing at
  `search(action="symbols")`/`symbol(action="callers")`/`explore` and the skill.
  Rate-limited to once per 10 minutes per repo, silent without an index, and never
  fired for literal/regex searches — a rare, high-signal reminder rather than noise.
- **Permission friction removal** — installing Claude hooks now additively allows the
  `mcp__civyk-repoix` server in the project `.claude/settings.json` permissions, so
  semantic calls never cost a permission prompt that a plain grep doesn't (user
  entries are preserved verbatim; the entry is added only when absent).

### Changed

- **Slimmer session-start context** — the SessionStart hook now injects a compact
  (~15-line) decision brief instead of the full ~2k-token tool manual; the manual
  content moved into the installable skill. Small blocks survive long-session context
  decay; nothing is lost.
- `civyk_repoix.__version__` re-synced with the package version (was stale at 1.8.0).

## [1.9.0] - 2026-07-10

### Added

- **Compounding research notes** — `wiki(action="ask", mode="deep", save=true)` files the
  deep-research answer as a durable "Research Notes" page under
  `memory/deep-wiki/<branch>/notes/`, chunked + embedded so future asks retrieve it instead of
  re-deriving the research. Notes are exempt from build pruning, appear in the wiki README and
  manifest under their own section, and surface as stale (status + lint) when their cited files
  change. Sources are capped to the 20 paths the answer actually cites (frequency-ranked);
  re-saving the same question refreshes the note; long distinct questions get hash-disambiguated
  ids so truncation can never overwrite a different note; a per-branch cap (200) fails loudly
  instead of flooding retrieval; `save` outside `mode="deep"` returns an explanatory `save_note`
  instead of silently doing nothing.
- **Owner steering (`memory/deep-wiki/steering.yaml`)** — Devin `.devin/wiki.json`-style human
  guidance the generator honors: `notes` (repo context injected into planning/synthesis/business
  prompts; bounded at 10 notes × 2000 chars), `pages` (owner-pinned pages whose path prefixes are
  claimed away from auto-planning), `emphasis` (per-page-id or `"*"` global), and `exclude_paths`
  (hidden from the wiki — including flow anchors and research-note sources — and surfaced by lint
  as an `excluded-paths` finding so exclusions are never silent). Lenient validation (bad steering
  never fails a build); the steering hash participates in both the staleness sentinel AND the
  incremental grounding hash, so a steering-only change re-synthesizes pages exactly once (never
  reuses stale cached prose). Gate: `wiki.steering` (default on; also `CIVYK_WIKI_STEERING`),
  honored consistently by generate, status, and lint.
- **Wiki lint (`wiki(action="lint")`)** — cross-page health checks the per-page pipeline cannot
  see: broken inter-page links, dead repo links, orphan pages, dead citations, stale pages/notes,
  dead notes (every cited source left the index — high severity), unverifiable notes (no indexed
  sources at all), coverage gaps, deliberately excluded paths, duplicate titles; findings are
  capped per kind with a visible rollup, links inside code fences are ignored, and repo-link
  probes refuse to leave the repository root. An optional LLM tier (`wiki.lint_llm`, default off;
  `CIVYK_WIKI_LINT_LLM`) checks page digests for contradictions/duplicated coverage, accepting
  only findings grounded in real page ids. The free structural tier also runs at the end of every
  changed build (no-op builds carry the previous summary forward); its summary lands in
  `manifest.json` stats. Status and lint share one staleness predicate, so they can never
  disagree.
- **Append-only wiki log** — `memory/deep-wiki/<branch>/log.md` records one grep-able
  `## [timestamp] kind | summary` line per build, filed research note, and lint pass (O(1)
  appends; rotates past 500 entries).
- **Architecture-aware planning** — the LLM structure planner now sees the subsystem dependency
  edges, top-level (zero in-degree) subsystems, and entry-point files, so module grouping follows
  the code's wiring rather than folder shape.
- **PageRank importance ranking** — pure-Python PageRank over the file dependency graph (Aider
  repo-map style), fed by one bulk edge query per build. It upgrades module-page importance
  (central-or-large, not just large) and drives deep-treatment selection (global centrality
  breaks fan-in ties); the page-set cut itself stays size-primary with rank as tie-break, keeping
  the structural planner's documented stable-page-set contract.
- **Adaptive module decomposition** — module pages above `wiki.max_files_per_page` (default 40;
  `CIVYK_WIKI_MAX_FILES_PER_PAGE`) split into child pages (by directory, or alphabetical chunks
  for one flat directory); the parent becomes a synthesis page grounded in its children's digests
  via the existing bottom-up phases, and rebuilds only when one of its own children changed. The
  low-importance "Additional Modules" catch-all is exempt (explicit flag, not an id match).

### Fixed

- `wiki(action="status")` staleness no longer misreads non-file sentinel keys in
  `source_hashes` (e.g. the steering sentinel) as missing files, and now runs on one bulk
  file-hash query instead of one point query per cited path.
- `civyk-repoix rebuild` now actually reindexes. It was sending an unregistered
  top-level `force_reindex` RPC (the worker routes reindex through the `status`
  tool's `action="reindex"`), so every invocation failed with "Method not found:
  force_reindex". Now sends the correct `status`/`reindex` request.

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

- Reorganized into `docs/design/` with indexes
- Architecture docs with diagrams (lifecycle, MCP flow, cache, schema)
- Conversation tools and AI cache hooks documentation
- MCP server handler docstrings, Sigstore verification docs

### Tests

- 49 MCP protocol compliance tests (daemon worker + SDK server)
- 10 conversation handler tests, hook config tests (97% coverage)
- Hook verification suite, cp1252 encoding safety tests
- Fixed 384 mypy errors across 21 test files
