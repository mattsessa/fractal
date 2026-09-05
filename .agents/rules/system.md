# Plasma Fractal (Hierarchical Agent Loops)

This repository provides `plasma-fractal` (`fractal`), an orchestration system that runs hierarchical trees of autonomous AI coding agents. Autonomous nodes iterate toward goals in isolated git worktrees, spawn child nodes for separable subtasks, and merge results back up the tree under hard caps (iterations, depth, children, cost, time).

## Core Architecture

1. **CLI Layer (`fractal/cli/`)**:
   - Built with Typer and Rich.
   - Entry point: `cli/main.py`. Sub-apps in `cli/cmd/` (`node`, `radio`, `plan`, `config`, `cost`, `time`, `db`, `event`, `channel`) and top-level commands in `cli/fractal.py` (`init`, `open`, `pause`, `resume`, `reset`, `destroy`, `commit`). Commands parse arguments and delegate directly to core logic.

2. **Core Engine (`fractal/core/`)**:
   - Central business logic: Node lifecycle (`node.py`), in-process iteration loop (`loop.py`), commit and work-product pipeline (`commit.py`), git worktree management (`worktree.py`), radio/messaging (`radio.py`), files facade (`files.py`), and cost/pricing accounting (`cost.py`, `pricing.py`).
   - Node iteration loop runs in-process Python (`fractal node _loop` executed inside tmux sessions).

3. **Agent Backends (`fractal/impl/`)**:
   - Provider-agnostic base class and registry in `core/agent.py`.
   - Implementations in `impl/`: `claude.py`, `codex.py`, `grok.py`, `opencode.py`, `omp.py`.
   - Defines invocation arguments, stream parsing, cost tracking, and transcript management.

4. **Terminal UI Cockpit (`fractal/tui/`)**:
   - Built with Textual (`fractal open`).
   - Displays live tree status, node inspect panes, chat, poller snapshots, and logs. Purely observational through core; holds no business logic.

5. **Storage & Data Discipline (Central SQLite & Worktrees)**:
   - Central SQLite database (`db.sqlite`) per tree in the root (user) node's data directory (`.fractal/<branch>/db.sqlite`), operating in WAL mode.
   - Isolated `git worktree` per node (`.fractal/worktrees/<branch>`), where dotted branch names encode hierarchy (`main.task.subtask`).

6. **Node Machinery Seeds (`fractal/_scripts/`, `fractal/_node/`, `fractal/_assets/`)**:
   - Shell scripts driving node lifecycle operations via `subprocess.run()`, node template files, and runtime git ignore assets.

## Core Rules

- **QUESTIONS ARE NOT EDIT REQUESTS**: When a message asks a question without explicitly requesting a change, answer and stop. Do not make unilateral changes.
- **WAIT FOR AN EXPLICIT GO-AHEAD**: Proposed actions remain pending until the user explicitly approves them.
- **READ THE WIKI FIRST**: The project wiki in `wiki/` is the descriptive architecture reference mirroring `fractal/`. Consult `wiki/` before answering questions or modifying components.
- **UPDATE THE WIKI ON CODE CHANGES**: When modifying, adding, or renaming modules, update the corresponding `wiki/` branch in the same change. Keep links and frontmatter intact using `wiki update` and `wiki lint`.
- **CONSISTENCY OVER SILENT IMPROVEMENTS**: Match existing patterns and conventions. Propose improvements explicitly rather than applying them silently. Do not remove comments.
- **SCOPE DISCIPLINE**: Do not add defensive code for impossible cases. Only validate at system boundaries. No unrequested features, shims, or premature abstractions.
- **NO DEVELOPMENT-HISTORY COMMENTS**: State rationales in present-tense design terms. Do not reference bug IDs, internal phase names, old defaults, or former implementations.
- **STRUCTURED BLOCK COMMENTS**: Use step-by-step `# verb noun` comments before logical blocks in long methods.
- **MODULE-LEVEL PUBLIC DATA**: Document constants and type aliases with `#: ` doc-comments for Sphinx autodoc.
- **NO ABSOLUTE PATHS IN PERSISTED DATA**: All stored paths must be relative to the repository root.
- **CLI COMMAND WRAPPING**: Use `@command(app, 'name')` from `fractal.cli.utils`. Define Typer options as local variables before command functions. Assign method calls before printing.
- **LIFECYCLE STATUS MODEL**: Adhere strictly to `fractal.constants.STATUSES`.

## Terminal Commands

- **Install dev dependencies**: `./install.sh --all-extras --groups=test,lint,type` or `uv sync --all-extras --group test --group lint --group type`
- **Run test suite**: `uv run --no-sync pytest`
- **Run pre-commit checks**: `uv run --no-sync pre-commit run [--all-files]`
- **Run type checks**: `uv run --no-sync pyright`
- **Wiki operations**: `uv run wiki search <pattern>`, `uv run wiki map`, `uv run wiki update`, `uv run wiki lint`
- **Run fractal CLI**: `uv run fractal --help`, `uv run fractal open`

## Core Technologies

- **Language**: Python 3.11+
- **Package & Environment Management**: `uv`
- **CLI Framework**: Typer, Rich
- **TUI Framework**: Textual
- **Database**: SQLite3 (WAL mode)
- **Session & Process Management**: `tmux`, `subprocess`
- **Version Control**: Git (branches, worktrees)
- **Testing**: `pytest` with doctest modules
- **Linting & Formatting**: `ruff`, `pre-commit`

## File Structure

```plaintext
fractal/
├── _assets/                        # Seed assets (git exclude configurations)
├── _node/                          # Node instance scaffolding and seed files
├── _scripts/                       # Lifecycle shell scripts invoked by Node via subprocess
├── cli/                            # Typer CLI application
│   ├── cmd/                        # Sub-app modules (node, radio, plan, config, cost, etc.)
│   ├── fractal.py                  # Top-level commands (init, open, pause, resume, etc.)
│   ├── main.py                     # Typer CLI app assembly
│   └── utils.py                    # Shared CLI decorators, formatters, and helpers
├── core/                           # Core domain logic
│   ├── agent.py                    # Agent base class, invocation hooks, and provider registry
│   ├── commit.py                   # Work-product staging, commit, and squash pipeline
│   ├── config.py                   # Node configuration parser and validator
│   ├── cost.py                     # Cost tracking, aggregation, and budget enforcement
│   ├── db.py                       # SQLite database connection, WAL pragmas, queries
│   ├── loop.py                     # In-process iteration loop runner
│   ├── node.py                     # Node model, lifecycle transitions, hierarchy methods
│   ├── radio.py                    # Inter-node messaging, mailbox, and dispatch
│   ├── record.py                   # Run, iteration, and step row ledger accounting
│   ├── schema.sql                  # Central database DDL schema
│   └── worktree.py                 # Git worktree creation, validation, and branch topology
├── impl/                           # Agent provider backends
│   ├── claude.py                   # Claude Code backend integration
│   ├── codex.py                    # Codex backend integration
│   ├── grok.py                     # Grok CLI backend integration
│   ├── omp.py                      # OMP backend integration
│   └── opencode.py                 # OpenCode backend integration
├── skills/                         # Plugin skill definitions (fractal skill)
├── tui/                            # Textual terminal dashboard (fractal open)
│   ├── app.py                      # Textual application cockpit
│   ├── app.tcss                    # Textual styling rules
│   ├── chat.py                     # Operator-to-node live chat widget
│   ├── panes/                      # Inspection and log panes
│   └── snapshot.py                 # State capture layer reading from central DB
├── util/                           # Reusable low-level utilities
│   ├── duration.py                 # Duration parser and formatter
│   ├── filesystem.py               # Path and filesystem helpers
│   ├── git.py                      # Git command wrappers
│   └── tmux.py                     # Tmux session management helpers
├── constants.py                    # Global statuses and system constants
└── exceptions.py                   # Domain exception hierarchy
shim/                               # Metadata-only pointer dist for PyPI
tests/                              # Pytest test suite (unit and worktree integration tests)
wiki/                               # Descriptive knowledge base and architectural docs
```

## Nomenclature

- **Python Classes**: PascalCase (e.g., `Node`, `Loop`, `Files`)
- **Functions & Methods**: snake_case (e.g., `resolve_user`, `_run_script`)
- **Event Hooks**: `on_<event>` (e.g., `on_start`, `on_finish`) with `logging_level: int = logging.<LEVEL>` default
- **Constants & Type Aliases**: UPPER_SNAKE_CASE; document with `#: ` doc-comments
- **Public CLI Commands**: lowercase single words / kebab-case (e.g., `init`, `open`, `reset`)
- **Private CLI Commands**: leading underscore (e.g., `_loop`, `_get`, `_set`)
- **Shell Scripts**: `set -euo pipefail`, uppercase variables, options formatted as `--opt=value`
- **Node Branch Naming**: Dotted hierarchy (`root.child.subtask`); dot is strictly reserved as separator
- **Lifecycle Statuses**: Use exact values defined in `fractal.constants.STATUSES`
- **No Abbreviations**: Use clear, descriptive names rather than obscure acronyms
