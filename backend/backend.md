# Backend Architecture

This document describes the Go rewrite of Agent Orchestrator (the `backend/` directory). It was written by reading the source — if something here contradicts a README or doc, the source wins.

---

## Entry points

There are two entry points, both under `backend/cmd/`:

| Binary | Source | Purpose |
|---|---|---|
| `ao` (CLI) | `cmd/ao/main.go` | User-facing binary. Calls `cli.Execute()` which builds a Cobra command tree and runs the matched subcommand. Most subcommands call the loopback daemon over HTTP. |
| `genspec` | `cmd/genspec/main.go` | OpenAPI spec generator, invoked by `npm run api:spec`. Writes `openapi.yaml`. |

A compatibility wrapper at `backend/main.go` calls `daemon.Run()` directly so `go run .` still starts the daemon during migration from the old TypeScript codebase.

### CLI subcommands (defined in `internal/cli/`)

All subcommands are registered in `NewRootCommand()` at `internal/cli/root.go:154`. The full list:

`ao daemon` (hidden), `start`, `stop`, `status`, `doctor`, `spawn`, `send`, `preview`, `hooks`, `launch`, `pty-host` (hidden), `import`, `project`, `session`, `orchestrator`, `review`, `completion`, `version`

Each subcommand lives in its own file under `internal/cli/` (e.g. `spawn.go`, `session.go`, `project.go`, `review.go`). The CLI is deliberately thin: it discovers the local daemon via `running.json`, calls its loopback HTTP API, and formats output. It never opens SQLite or calls adapters directly.

---

## Daemon startup sequence

The daemon lifecycle lives in `internal/daemon/daemon.go`, function `Run()`. Boot order:

1. **Config.Load()** (`internal/config/config.go`) — reads environment variables (AO_PORT, AO_DATA_DIR, AO_RUN_FILE, AO_AGENT, AO_ALLOWED_ORIGINS, telemetry vars). The bind host is hardcoded to `127.0.0.1` and is not configurable. Unset vars fall back to defaults.

2. **Stale daemon check** — `runfile.CheckStale()` (`internal/runfile/runfile.go`) checks whether `running.json` points at a live process. If so, an HTTP GET to `/healthz` on the recorded port confirms the predecessor is genuinely serving, and the daemon refuses to start. This prevents a split-brain scenario where a leaked `running.json` (common on Windows after a hard kill) falsely blocks a new daemon.

3. **sqlite.Open()** (`internal/storage/sqlite/db.go`) — opens the SQLite database file (`ao.db`) under the configured data directory, runs goose migrations from the embedded `migrations/*.sql` files. Uses **two connection pools**: one writer (MaxOpenConns=1, for read-your-writes) and a reader pool (MaxOpenConns=8, via WAL mode). Pure-Go `modernc.org/sqlite` driver — no CGO.

4. **Telemetry sink** (`internal/daemon/telemetry_wiring.go`) — builds a `ports.EventSink`. When `AO_TELEMETRY_EVENTS=on`, writes events to SQLite (local sink). When `AO_TELEMETRY_REMOTE=posthog`, also writes to PostHog via a fanout sink. NoopSink when disabled.

5. **CDC pipeline** (`internal/daemon/cdc_wiring.go`) — constructs a `cdc.Poller` that tails the `change_log` table and a `cdc.Broadcaster` that fans events out to live subscribers. The poller seeks to head so only new events are broadcast.

6. **Terminal manager** (`terminal.NewManager`) — wraps a runtime adapter (tmux on macOS/Linux, conpty on Windows) selected by `runtimeselect.New()`. The manager provides WebSocket-based terminal attach/detach.

7. **Messenger + Notifications** — builds `runtimeMessenger` (sends user messages to the live runtime pane), `notify.Hub` (in-memory pub/sub for dashboard notifications via SSE), `notificationsvc` (durable notification CRUD), and `notify.Writer` (writes notifications + publishes to hub).

8. **Lifecycle Manager + Reaper** (`internal/daemon/lifecycle_wiring.go`) — `startLifecycle()` creates `lifecycle.Manager` (synchronous reducer for session lifecycle facts) and `reaper.Reaper` (polls runtime liveness every 5s, feeds facts to LCM). The LCM is the canonical write path for session state.

9. **SCM Observer** (`internal/daemon/scm_wiring.go`) — `startSCMObserver()` wires a GitHub SCM provider (via `gh` binary or AO_GITHUB_TOKEN) into a polling observer that fetches PR facts and feeds them to the LCM for agent nudges. Missing credentials don't block startup; the observer logs a warning and disables itself.

10. **Session service + Review service + Session Manager** (`startSession()` in `lifecycle_wiring.go:79`) — wires together:
    - Agent resolver (`agentRegistry` over `adapters.Registry` — 25+ shipped adapters)
    - Workspace adapter (`gitworktree` — per-session git worktrees under `~/.ao/data/worktrees/`)
    - Session Manager (spawn, kill, restore, reconcile)
    - Session service (read-model assembly, status derivation, PR claiming)
    - Review engine (`reviewcore.Engine` + `reviewsvc.Service`)

11. **Preview Poller** (`internal/preview/poller.go`) — background goroutine that checks session workspaces for index.html-type entry points, so the desktop browser panel can auto-detect preview targets.

12. **HTTP Server** (`httpd.NewWithDeps()` in `internal/httpd/server.go`) — binds the configured loopback port (with fallback to ephemeral if the port is held by a non-AO process), mounts the chi router, and starts serving.

13. **Reconcile** — `sessMgr.Reconcile(ctx)` runs after the server is wired but before `srv.Run()`. Three passes: adopt crash-surviving sessions, reap leaked runtimes, restore shutdown-saved sessions.

14. **Supervisor listener** (`internal/daemon/supervisor/`) — optional Unix socket so the Electron desktop app can signal "frontend died" to trigger daemon shutdown.

15. **srv.Run()** — blocks until SIGINT/SIGTERM or POST /shutdown. On exit: cancels the context (stops all background goroutines), waits for them to drain, then returns. Sessions survive the daemon exit — the next boot's Reconcile adopts them.

---

## Package map

```
backend/
├── main.go                        # Compatibility wrapper → daemon.Run()
├── go.mod / go.sum                # Module: github.com/aoagents/agent-orchestrator/backend
├── sqlc.yaml                      # sqlc config for query generation
├── .golangci.yml                  # Linter config
├── cmd/
│   ├── ao/main.go                 # CLI entry → cli.Execute()
│   └── genspec/main.go            # OpenAPI spec generator
└── internal/
    ├── cli/                       # Cobra CLI commands
    │   ├── root.go                # Root command, Deps struct, Execute()
    │   ├── start.go               # `ao start` — fetches & opens desktop app
    │   ├── stop.go                # `ao stop` — POST /shutdown
    │   ├── status.go              # `ao status` — daemon info
    │   ├── doctor.go              # `ao doctor` — diagnostics
    │   ├── spawn.go               # `ao spawn` — POST /api/v1/sessions
    │   ├── send.go                # `ao send` — POST .../send
    │   ├── preview.go             # `ao preview` — POST .../preview
    │   ├── hooks.go               # `ao hooks` — agent hook callback handler
    │   ├── launch.go              # `ao launch` — orchestrator spawn
    │   ├── session.go             # `ao session` — get/list
    │   ├── project.go             # `ao project` — add/list/remove
    │   ├── review.go              # `ao review` — trigger/submit
    │   ├── import.go              # `ao import` — migrate from legacy
    │   ├── orchestrator.go        # `ao orchestrator` — list/spawn
    │   ├── process.go             # Cross-platform process start
    │   ├── process_unix.go        # Unix process start (setsid)
    │   ├── process_windows.go     # Windows process start (detached)
    │   ├── ptyhost.go             # Hidden conpty-helper host command
    │   ├── completion.go          # Shell completion generator
    │   ├── version.go             # Version stamp
    │   ├── output.go              # Output formatting helpers
    │   ├── pr_ref.go              # PR reference parsing
    │   └── client.go              # HTTP client helpers for daemon calls
    │
    ├── daemon/                    # Process bootstrap & wiring
    │   ├── daemon.go              # Run() — full startup sequence
    │   ├── stale.go               # Stale run-file detection via /healthz probe
    │   ├── lifecycle_wiring.go    # startLifecycle, startSession, buildAgentResolver
    │   ├── cdc_wiring.go          # startCDC (poller + broadcaster)
    │   ├── scm_wiring.go          # startSCMObserver (GitHub provider)
    │   ├── telemetry_wiring.go    # Telemetry sink construction
    │   ├── wiring_test.go         # Wiring integration tests
    │   ├── stale_test.go
    │   ├── telemetry_wiring_test.go
    │   └── supervisor/            # Unix socket for Electron frontend death detection
    │       ├── supervisor.go      # Supervisor struct (grace period, shutdown request)
    │       ├── listen_unix.go     # Unix socket listener on Linux/macOS
    │       └── listen_windows.go  # Named pipe listener on Windows
    │
    ├── config/                    # Environment-based configuration
    │   ├── config.go              # Load(), Config struct, env var definitions
    │   └── config_test.go
    │
    ├── domain/                    # Shared domain types (no external dependencies)
    │   ├── doc.go                 # Package doc
    │   ├── session.go             # SessionID, SessionRecord, SessionMetadata, Session
    │   ├── activity.go            # ActivityState, Activity
    │   ├── status.go              # SessionStatus enum (derived, never stored)
    │   ├── project.go             # ProjectRecord, WorkspaceRepoRecord, SessionWorktreeRecord
    │   ├── pr.go                  # PullRequest, PRFacts, CIState, ReviewDecision, Mergeability
    │   ├── review.go              # Review, ReviewRun, ReviewRunStatus, ReviewVerdict
    │   ├── notification.go        # NotificationRecord, NotificationType, NotificationStatus
    │   ├── projectconfig.go       # ProjectConfig, RoleOverride, AgentConfig
    │   ├── agentconfig.go         # AgentConfig, PermissionMode
    │   ├── harness.go             # AgentHarness, ReviewerHarness types
    │   ├── tracker.go             # TrackerID, Issue, NormalizedIssueState
    │   ├── text.go                # SanitizeControlChars
    │   ├── reviewerharness.go     # ReviewerHarness
    │   └── ...
    │
    ├── ports/                     # Boundary interfaces (what the core needs from outside)
    │   ├── doc.go
    │   ├── agent.go               # Agent, AgentResolver, LaunchConfig, RestoreConfig
    │   ├── session.go             # SpawnConfig
    │   ├── outbound.go            # Runtime, Workspace, AgentMessenger, PRWriter, SCMWriter, PRClaimer
    │   ├── runtime_observations.go # ProbeResult, RuntimeFacts, ActivitySignal
    │   ├── scm_observations.go    # SCMObservation and nested types
    │   ├── pr_observations.go     # PRObservation, PRCheckObservation, PRCommentObservation
    │   ├── tracker_observations.go# TrackerObservation and nested types
    │   ├── reviewer.go            # Reviewer, ReviewInvocation, ReviewerResolver
    │   ├── notifications.go       # NotificationIntent
    │   ├── tracker.go             # Tracker (issue tracker port)
    │   ├── telemetry.go           # EventSink, TelemetryEvent
    │   ├── agent_test.go
    │   └── ...
    │
    ├── adapters/                  # Pluggable implementations of ports
    │   ├── registry.go            # Generic adapter Registry
    │   ├── agent/                 # Agent adapter implementations
    │   │   ├── registry/          # Constructor map (registry.Build)
    │   │   ├── activitydispatch/  # Activity dispatch/discovery
    │   │   ├── hookutil/          # Shared hook utilities
    │   │   ├── claudecode/        # Claude Code adapter
    │   │   ├── codex/             # Codex adapter
    │   │   ├── aider/             # Aider adapter
    │   │   ├── ... (25+ total)    # Each subdirectory is one agent adapter
    │   │   └── vibe/              # Vibe coding adapter
    │   ├── runtime/               # Runtime adapters
    │   │   ├── runtimeselect/     # Platform-selected runtime (tmux vs conpty)
    │   │   ├── tmux/              # tmux on macOS/Linux
    │   │   └── conpty/            # conpty on Windows
    │   ├── workspace/
    │   │   └── gitworktree/       # Per-session git worktree manager
    │   ├── scm/
    │   │   └── github/            # GitHub SCM provider (via gh CLI + REST API)
    │   ├── reviewer/              # Code reviewer resolver
    │   ├── telemetry/             # Telemetry sinks
    │   │   ├── localsqlite.go     # Local SQLite event store
    │   │   ├── posthog.go         # PostHog remote exporter
    │   │   ├── fanout.go          # Fanout sink (local + remote)
    │   │   └── noop.go            # No-op sink
    │   └── tracker/               # Issue tracker adapters
    │
    ├── service/                   # Business logic layer
    │   ├── session/               # Session read-model assembly, status derivation, PR claiming
    │   │   ├── service.go         # Service struct, Spawn/Kill/Get/List/SetPreview
    │   │   ├── stack.go           # PR stack detection (for multi-PR sessions)
    │   │   ├── status.go          # deriveStatus() — display status from facts
    │   │   ├── claim_pr.go        # PR claiming logic
    │   │   └── pr_summary.go      # PR summary assembly
    │   ├── review/                # Review service (triggers, delivery tracking)
    │   ├── project/               # Project service (CRUD for registered projects)
    │   ├── pr/                    # PR service (action manager)
    │   ├── notification/          # Durable notification CRUD
    │   └── importer/              # Legacy TypeScript state importer
    │
    ├── session_manager/           # Session lifecycle orchestration
    │   ├── manager.go             # Spawn, Kill, Restore, Reconcile, Cleanup, Send
    │   ├── provision.go           # Workspace provisioning (symlinks, postCreate)
    │   └── manager_test.go
    │
    ├── lifecycle/                 # Lifecycle Manager (durable session state reducer)
    │   ├── manager.go             # Manager: ApplyActivitySignal, MarkSpawned, MarkTerminated
    │   ├── reactions.go           # ApplyPRObservation, ApplyReviewBatch, ApplyTrackerFacts, sendOnce
    │   └── runtime.go             # hasRecentActivity, runtimeClearlyDead
    │
    ├── cdc/                       # Change Data Capture
    │   ├── poller.go              # Poller — tails change_log table
    │   ├── broadcast.go           # Broadcaster — in-process fan-out
    │   └── event.go               # Event, EventType types
    │
    ├── observe/                   # OBSERVE layer (polling background loops)
    │   ├── reaper/                # Runtime liveness polling (every 5s)
    │   │   └── reaper.go
    │   └── scm/                   # SCM polling observer
    │       └── observer.go
    │
    ├── httpd/                     # HTTP delivery mechanism
    │   ├── server.go              # Server (bind, run, graceful shutdown)
    │   ├── router.go              # Router (middleware stack, route mounting)
    │   ├── api.go                 # API struct, APIDeps, resource registration
    │   ├── events.go              # SSE event stream (/api/v1/events)
    │   ├── cors.go                # CORS middleware (app://renderer, configurable origins)
    │   ├── log.go                 # Request logging middleware
    │   ├── logger.go              # Structured logger
    │   ├── recover.go             # Panic recovery middleware
    │   ├── telemetry.go           # Request telemetry middleware
    │   ├── terminal_mux.go        # Terminal WebSocket mux (/mux)
    │   ├── controllers/           # REST resource controllers
    │   │   ├── dto.go             # Request/Response DTOs
    │   │   ├── sessions.go        # Session CRUD + activity + preview + PR claim
    │   │   ├── projects.go        # Project CRUD
    │   │   ├── prs.go             # PR routes
    │   │   ├── reviews.go         # Review trigger/submit
    │   │   ├── notifications.go   # Notification list/read
    │   │   └── imports.go         # Legacy import routes
    │   ├── envelope/              # JSON response envelope (WriteJSON, WriteAPIError, WriteError)
    │   ├── apierr/                # Typed API errors (NotFound, Conflict, Invalid)
    │   └── apispec/               # Code-first OpenAPI spec generation
    │       ├── specgen/           # Spec builder (operation registry, schema generation)
    │       └── openapi.yaml       # Generated spec (committed artifact)
    │
    ├── storage/sqlite/            # Persistence layer
    │   ├── db.go                  # Open(), migration setup, Store type alias
    │   ├── migrate.go             # goose migration runner
    │   ├── migrations/            # SQL migration files (goose-managed)
    │   ├── queries/               # sqlc query source files (*.sql)
    │   ├── gen/                   # sqlc-generated Go code (DO NOT HAND-EDIT)
    │   │   ├── db.go, models.go   # Generated DB handle + model types
    │   │   ├── sessions.sql.go    # Session queries
    │   │   ├── projects.sql.go    # Project queries
    │   │   ├── pr.sql.go          # PR queries
    │   │   ├── changelog.sql.go   # change_log queries
    │   │   └── ...                # Per-entity generated queries
    │   └── store/                 # Typed CRUD methods per entity
    │       ├── store.go           # Store struct (writer+reader, inTx)
    │       ├── session_store.go   # Session CRUD
    │       ├── project_store.go   # Project CRUD
    │       ├── pr_store.go        # PR CRUD
    │       ├── pr_facts.go        # PR facts (display PRFacts)
    │       ├── review_store.go    # Review/ReviewRun CRUD
    │       ├── notification_store.go
    │       ├── telemetry_store.go
    │       ├── changelog_store.go
    │       └── ...
    │
    ├── notify/                    # Server-Sent Events notification stream
    │   ├── hub.go                 # In-memory pub/sub hub
    │   ├── manager.go             # Subscription management
    │   └── enrich.go              # Notification enrichment
    │
    ├── terminal/                  # Terminal multiplexer
    │   ├── manager.go             # Terminal manager (attach/detach via WebSocket)
    │   └── ...
    │
    ├── preview/                   # Browser preview auto-detection
    │   ├── poller.go              # Background poller for index.html discovery
    │   └── entry.go               # Entry point discovery helpers
    │
    ├── review/                    # Code review engine
    │   ├── planner.go             # Review planner (what to review, when)
    │   ├── launcher.go            # Reviewer process launcher
    │   ├── prompt.go              # Review prompt generation
    │   └── review.go              # Core review orchestration
    │
    ├── config/                    # (already covered above)
    ├── runfile/                   # running.json handshake file (PID, port, owner)
    │   └── runfile.go
    ├── process/                   # Cross-platform process execution
    ├── processalive/              # Cross-platform PID liveness check
    ├── daemonmeta/                # Service name constant
    ├── telemetrymeta/             # Telemetry error classification
    ├── legacyimport/              # Legacy TypeScript state importer
    └── agentlaunch/               # Agent launch orchestration helpers
```

---

## Architecture pattern: Ports & Adapters (Hexagonal)

The backend follows a hexagonal architecture with explicit port interfaces in `internal/ports/` and adapter implementations in `internal/adapters/`. Domain types in `internal/domain/` are pure Go structs with no external dependencies.

```
┌─────────────────────────────────────────────────────┐
│                    CLI (thin client)                 │
│              (internal/cli/)                         │
└──────────────┬──────────────────────────────────────┘
               │ HTTP (loopback)
               ▼
┌─────────────────────────────────────────────────────┐
│              HTTP Server (internal/httpd/)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ Router   │ │Middleware│ │Controller│ │ SSE    │ │
│  │ (chi)    │ │ stack    │ │   layer  │ │ Events │ │
│  └──────────┘ └──────────┘ └────┬─────┘ └────────┘ │
└─────────────────────────────────┼───────────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
   ┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │ Service Layer  │  │ Lifecycle Mgr    │  │ Notification Hub │
   │ (internal/     │  │ (internal/       │  │ (internal/       │
   │  service/)     │  │  lifecycle/)     │  │  notify/)        │
   └────────┬───────┘  └────────┬─────────┘  └────────┬─────────┘
            │                   │                      │
            ▼                   ▼                      │
   ┌────────────────┐  ┌──────────────────┐            │
   │ Session Mgr    │  │ Reaper / SCM     │            │
   │ (internal/     │  │ Observer         │            │
   │  session_mgr/) │  │ (internal/       │            │
   └────────┬───────┘  │  observe/)       │            │
            │          └────────┬─────────┘            │
            │                   │                      │
            ▼                   ▼                      ▼
   ┌──────────────────────────────────────────────────────┐
   │              Port interfaces (internal/ports/)        │
   │  Agent │ Runtime │ Workspace │ SCM │ Tracker │ Telemetry│
   └──────────────┬───────────────────┬────────────────────┘
                  │                   │
        ┌─────────┴─────────┐  ┌─────┴──────────────┐
        │ Adapters          │  │ SQLite Store        │
        │ (internal/        │  │ (internal/storage/  │
        │  adapters/)       │  │  sqlite/store/)     │
        │ - Agent adapters  │  │                     │
        │ - tmux / conpty   │  │ sqlc-generated      │
        │ - git worktree    │  │ queries             │
        │ - GitHub SCM      │  │ (internal/storage/  │
        │ - Telemetry sinks │  │  sqlite/gen/)       │
        └───────────────────┘  └──────────┬──────────┘
                                          │
                                          ▼
                                  ┌──────────────┐
                                  │  SQLite DB   │
                                  │  (ao.db)     │
                                  │  + WAL       │
                                  └──────────────┘
```

Key principle: each consumer defines its own narrow interface, not a shared god interface. For example:
- `internal/service/session/service.go` defines `Store` (read-only), `commander`, `scmProvider` — each exactly the methods it needs.
- `internal/session_manager/manager.go` defines `Store`, `runtimeController`, `lifecycleRecorder`.
- `internal/httpd/controllers/sessions.go` defines `SessionService` and `ActivityRecorder`.

---

## Data flow: request lifecycle

### Example: spawning a worker session (`POST /api/v1/sessions`)

```
CLI                          Daemon HTTP                  Service                  Session Manager
 │                            │                            │                        │
 ├─ POST /api/v1/sessions ───►                            │                        │
 │                            │  ┌─────────────────────┐   │                        │
 │                            │  │ chi middleware stack │   │                        │
 │                            │  │ - RequestID          │   │                        │
 │                            │  │ - RealIP             │   │                        │
 │                            │  │ - requestLogger      │   │                        │
 │                            │  │ - recoverTelemetry   │   │                        │
 │                            │  │ - CORS               │   │                        │
 │                            │  │ - Timeout (60s)      │   │                        │
 │                            │  └─────────────────────┘   │                        │
 │                            │  SessionsController.spawn()│                        │
 │                            │  decode + validate JSON    │                        │
 │                            ├─ sessionSvc.Spawn() ──────►│                        │
 │                            │                            │  requireProject()      │
 │                            │                            │  (checks DB for project)│
 │                            │                            ├── manager.Spawn() ────►│
 │                            │                            │                        │  loadProject
 │                            │                            │                        │  resolve harness
 │                            │                            │                        │  validate agent binary
 │                            │                            │                        │  store.CreateSession (seed row)
 │                            │                            │                        │  workspace.Create (git worktree)
 │                            │                            │                        │  provisionWorkspace (symlinks, postCreate)
 │                            │                            │                        │  agent.GetLaunchCommand (argv)
 │                            │                            │                        │  runtime.Create (tmux pane)
 │                            │                            │                        │  lcm.MarkSpawned (store UpdateSession)
 │                            │                            │  ◄─── SessionRecord ───│
 │                            │                            │  toSession()           │
 │                            │                            │   └─ ListPRFactsForSession
 │                            │                            │   └─ deriveStatus()
 │                            │  ◄── domain.Session ────── │                        │
 │                            │  envelope.WriteJSON(201)   │                        │
 │  ◄─── HTTP 201 ────────────┤                            │                        │
```

### Example: status derivation (read path)

```
GET /api/v1/sessions/{id}

1. Controller → sessionSvc.Get(id)
2. Service → store.GetSession(id) → SessionRecord
3. Service → store.ListPRFactsForSession(id) → []PRFacts
4. Service → deriveStatus(record, prFacts, now, signalCapable)
   Logic in internal/service/session/status.go:
   - If terminated: "merged" (if any PR merged) or "terminated"
   - If waiting_input: "needs_input"
   - If open PRs exist: worst-wins aggregate of PR pipeline statuses
     (ci_failed > changes_requested > draft > review_pending > pr_open > approved > mergeable)
   - If active: "working"
   - If no hook signal after grace (90s): "no_signal"
   - Otherwise: "idle"
```

### Example: CDC event flow

```
1. Any store write → DB trigger (from migration 0001) writes to change_log
2. cdc.Poller (every 100ms): SELECT from change_log WHERE seq > lastSeq
3. For each new event: broadcaster.Publish(event)
4. Subscribers receive the event synchronously:
   - Terminal mux (session-state updates for WebSocket clients)
   - SSE endpoint (/api/v1/events) fans out to connected browsers
   - Tests subscribe for assertions
```

### Example: SCM observation lifecycle

```
1. scmObserver (every N seconds): polls GitHub for PR changes
2. Normalizes response into ports.SCMObservation
3. Writes PR row via store.WriteSCMObservation()
   → DB triggers fire CDC events
4. Calls lcm.ApplySCMObservation(sessionID, observation)
5. LCM reacts:
   - PR merged + no open PRs remain → MarkTerminated(session)
   - CI failing → sendOnce() sends nudge to agent
   - Changes requested → sendOnce() sends nudge
   - Merge conflict → sendOnce() sends "rebase" nudge
   - Ready to merge → emit notification
   - Each nudge uses sendOnce() for dedup (in-memory + persisted to pr.last_nudge_signature)
```

---

## External integrations

| Integration | How | Code location |
|---|---|---|
| **SQLite (embedded)** | Pure-Go `modernc.org/sqlite` driver, WAL mode, two connection pools (1 writer + 8 readers) | `internal/storage/sqlite/db.go` |
| **Schema migrations** | `goose/v3` with embedded SQL files | `internal/storage/sqlite/migrations/*.sql` |
| **GitHub SCM** | `gh` CLI binary or `AO_GITHUB_TOKEN` env var for REST API calls | `internal/adapters/scm/github/` |
| **Agent CLIs** | ~25 agent adapters each resolve their own binary (claude-code, codex, aider, etc.) | `internal/adapters/agent/*/` |
| **tmux** | Terminal multiplexer on macOS/Linux for session runtimes | `internal/adapters/runtime/tmux/` |
| **conpty** | Windows pseudo console for session runtimes | `internal/adapters/runtime/conpty/` |
| **PostHog** | Remote telemetry export (optional, off by default) | `internal/adapters/telemetry/posthog.go` |
| **Git operations** | Shells out to `git` for worktree management | `internal/adapters/workspace/gitworktree/` |
| **Electron supervisor** | Unix socket / named pipe for frontend death detection | `internal/daemon/supervisor/` |
| **GitHub release downloads** | HTTP download of desktop app binaries (.zip, .exe, .AppImage) | `internal/cli/start.go` |

---

## Conventions and patterns

### Dependency Injection
Every major component receives its dependencies via a `Deps` struct or functional options (`Option` pattern). Tests substitute fakes for real implementations. Examples:
- `sessionmanager.Deps` at `internal/session_manager/manager.go:124`
- `lifecycle.Option` at `internal/lifecycle/manager.go:38`
- `cli.Deps` at `internal/cli/root.go:61`

### Narrow consumer-defined interfaces
Each package defines the interface it needs from collaborators, not a shared interface it implements. This means the same concept (e.g. "store") has different interfaces in different packages (session manager's `Store` vs session service's `Store` vs lifecycle's `sessionStore`).

### No stored display status
User-facing session status (`SessionStatus`) is **derived at read time** from durable facts (`activity_state`, `is_terminated`, PR facts). Never stored. See `internal/service/session/status.go`.

### Sessions survive daemon restart
The daemon intentionally does not terminate sessions on shutdown. The next boot's `Reconcile()` adopts crash-surviving runtimes, captures uncommitted work from dead ones, and restores shutdown-saved sessions. See `internal/session_manager/manager.go:736` (`Reconcile`).

### CDC via DB triggers, not application code
Database triggers (migration 0001) write to `change_log` on session/PR/check mutations. The storage layer never writes change_log directly. This ensures every write produces a CDC event, even from future code paths. See `internal/cdc/` and migration files.

### SQLite writer serialisation
Single writer connection (`MaxOpenConns=1`) guarantees read-your-writes within the CDC trigger subqueries. Write methods use `inTx()` with a mutex for read-modify-write operations like session creation (which uses a sequence counter).

### CLI exit codes
- Exit 0: success
- Exit 1: runtime/daemon failure
- Exit 2: CLI usage error (bad flag, wrong args), marked with `usageError{}`

### No network calls in unit tests
Tests use `httptest`, fake implementations, and injected dependencies. Integration tests (in `internal/integration/`) use the real store but mock/gate external calls.

### Agent adapter registration
New agent adapters register themselves in `internal/adapters/agent/registry/` by adding to the constructor map. The daemon validates the default agent at startup against this registry.

### Hard rules (from AGENTS.md, confirmed by code)
- Daemon binds only `127.0.0.1` (not configurable)
- CLI never opens SQLite or calls adapters directly
- No derived status stored in DB
- Failed/unknown runtime probes are never treated as proof of death
- Dirty registered worktrees are never force-deleted
- Already-merged SQLite migrations are never modified
- sqlc-generated code is never hand-edited
- All app state resolves under `~/.ao` (or `AO_DATA_DIR`)
- Tests do not add network calls unless the package already has an integration/e2e pattern

---

## Things that may be unclear / inconsistent

- **`internal/agentlaunch/`** — small utility package used by the `ao launch` hidden subcommand (`cli/launch.go`). It persists launch argv to a temp JSON file so the conpty trampoline (pty-host) can read and exec the real agent command. This is separate from the session_manager's spawn path.
- **`internal/domain/tracker.go`** defines `TrackerID`, `Issue`, and `NormalizedIssueState`. The lifecycle manager has `ApplyTrackerFacts` for reacting to tracker observations. A full `ports.Tracker` adapter exists at `adapters/tracker/github/` (Get, List, Preflight) but is not yet wired into a polling observer — that loop is tracked by issue #35.
- **`internal/adapters/tracker/github/`** — 4-file implementation of `ports.Tracker` against the GitHub REST API. Used by `ao spawn` to hydrate the agent prompt from an issue. Write-side (Comment, Transition) is deferred to issue #40; the polling observer loop is deferred to issue #35. Not yet wired at daemon startup.
- **`internal/storage/sqlite/store/pr_facts.go`** contains display-oriented PR fact queries separate from the raw PR CRUD in `pr_store.go`. The split reflects the read-model boundary (facts for status derivation vs raw PR records).
- **`internal/review/`** has its own sub-packages (planner, launcher, prompt) parallel to `internal/service/review/`. The `internal/review/` package is the engine; `internal/service/review/` is the HTTP-facing service layer.
