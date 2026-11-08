# Global Engineering & Agent Collaboration Rules

## 1. Language Routing & ASD-STE100 Documentation

- **Strict Language Separation**:
  - **User Conversation (User's Language)**: Always use the language the user currently uses when chatting, explaining plans, reporting status, and naming Paseo Workspaces, unless the user requests another language.
  - **Technical Artifacts (English ONLY)**: Write all code comments, API docs, Markdown documentation, commit messages, log strings, and identifiers strictly in **English**. User-facing localized content and Paseo Workspace titles follow the user's language.
- **ASD-STE100 & Intent-Only Comments**:
  - **Comply with ASD-STE100**: Use **Active Voice** only. Keep descriptive sentences under **25 words** and procedural sentences under **20 words**. Maintain **One Word, One Meaning** with zero idioms or ambiguous phrasing.
  - **State Purpose (Why), Never Mechanics (How)**: Comments MUST explain why a method exists, its domain contract, boundary conditions, or non-obvious side effects.
  - **Zero Flow Narration**: Restating internal control flow, loops, conditionals, or variable assignments is **STRICTLY FORBIDDEN**. Do not write inline comments inside method bodies unless documenting a non-obvious algorithmic constraint or protocol quirk.
  - **Examples**:
    - *BAD*: `// 校验节点` or `// The node list is iterated through and isolated nodes are removed.` (Chinese comment, passive voice, or narrates internal flow)
    - *GOOD*: `// Prunes isolated vertices to guarantee a connected graph before layout computation.` (Active voice, states purpose and contract)

## 2. Code Style & Architectural Reusability

- **Zero Hardcoding Policy**:
  - Hardcoding magic numbers, raw strings, URLs, paths, or thresholds inside business logic or method bodies is **STRICTLY FORBIDDEN**.
  - Scan all parameters and conditionals before writing code. Extract all literals into semantically named immutable fields (`static final` / `final` in Java/Kotlin, `const` / `readonly` in TS/Rust/Go/C++).
  - **Examples**:
    - *BAD*: `if (retry > 3) connect("wss://relay.internal", 5000);`
    - *GOOD*: Declare `private static final int MAX_RETRY_ATTEMPTS = 3;` and reference the constant.
- **Utility Extraction & Reusability First**:
  - Embedding scattered, ad-hoc helper logic (data transformation, parsing, validation, IO) inside domain workflows is **STRICTLY FORBIDDEN**.
  - Search existing utilities via CodeGraph or `ast-grep` before writing helper code. If none exists, extract the logic into a stateless, generic, type-safe utility module decoupled from caller state.
  - **Examples**:
    - *BAD*: `const id = raw.trim().toLowerCase().replace(/[^a-z0-9]/g, '');` (Inlined inside a business function)
    - *GOOD*: Extract to `IdentifierUtils.sanitize(raw)` and call it from the domain workflow.

## 3. Minimal Effective Testing, Verification & OS Routing

- **Minimal Effective Testing**:
  - **No Test Bloat**: Create only the **minimal effective test set** required to verify core business contracts, critical boundaries, or bug regressions.
  - **No Ritualistic Tests**: Writing trivial getter/setter tests, over-mocking internal implementation details, or generating exhaustive permutation bloat is **STRICTLY FORBIDDEN**. Do not create new test classes for minor changes; append focused cases to existing test suites.
  - **Examples**:
    - *BAD*: Creating a new test class with 10 heavily mocked interaction tests for a 3-line fix.
    - *GOOD*: Adding 1 regression test case to the existing test class targeting the exact input-output contract.
- **Staged Verification & Honest Delivery**:
  - Run only affected compilation and unit tests for daily changes (skip tests for pure doc/config edits). Run full verification gates only before merging, releasing, or when explicitly requested. Execute host/hardware tests only when capturing physical evidence or during final acceptance.
  - Delivery requires passing verification with concrete evidence. Report missing verification honestly; **NEVER** claim completion based solely on green builds or self-assessment.
- **OS Environment Routing**:
  - Detect the host OS (Linux, Windows, or macOS) before running commands, scripts, or tests. Route paths, shell syntax, and display environments accordingly.
  - **Linux-Only X11 Virtual Desktop Constraint**: When running GUI or screenshot tests inside a virtual desktop **on Linux**, you **MUST** enforce a pure X11 environment: unset `WAYLAND_DISPLAY`, set `XDG_SESSION_TYPE=x11`, and run via `xvfb-run -a`. Running screenshot tests under an inherited Wayland session is **STRICTLY FORBIDDEN** (prevents screen-cast authorization hangs). **NEVER** apply X11/`xvfb-run` commands on Windows or macOS.
  - **Examples**:
    - *BAD*: Running GUI screenshot tests directly in a Linux Wayland session, or calling `xvfb-run` on Windows.
    - *GOOD*: On Linux: `env -u WAYLAND_DISPLAY XDG_SESSION_TYPE=x11 xvfb-run -a pnpm test:ui`; on Windows: use native test runners.

## 4. Parallel Collaboration & Paseo Lifecycle

- **Delegation Boundaries**:
  - The main Agent implements and verifies tasks directly by default. Spawn sub-agents **ONLY** when work can be split into independent, parallelizable tasks with clear boundaries. Do not delegate merely because a task is complex. Each parallel sub-agent MUST use an isolated worktree and **NEVER** modify the same implementation concurrently.
- **Paseo Discovery & Dispatch**:
  - At task start, read the Paseo skill and run a read-only MCP discovery call to confirm service and provider availability (fall back to Paseo CLI help if MCP is unavailable).
  - When Paseo is available, route all sub-agent delegation exclusively through Paseo. Match tasks against agent profiles/notes first; use provider discovery if no profile matches (**NEVER** guess models or permissions). State the objective, scope, dependencies, and verifiable acceptance criteria in the prompt.
  - **Project & Workspace Rules**: Sub-agents MUST stay under the same Project as the dispatcher (**NEVER** create a new Project). Create a new Workspace only when assigning an isolated worktree.
  - **Workspace Title Format**: Write Workspace titles in the user's language using an "Action + Object" structure (preserve essential identifiers, e.g., `重构CodeGraph索引锁`). Use **4–12 characters for Chinese** or a concise **2–6 words for other languages**. Full explanations or trailing punctuation are **STRICTLY FORBIDDEN**.
  - **Failure Handling**: Report Paseo unavailability or dispatch failures explicitly. Continue within existing authorization or request user confirmation before switching channels. **NEVER** degrade silently or double-dispatch ambiguous tasks.
- **Lifecycle Cleanup**:
  - Sub-agent self-reported completion does not equal acceptance. The main Agent MUST review diffs and run verification before delivery.
  - Once a delegated task finishes or closes, the main Agent **MUST** clean up its isolated worktree: call `paseo_archive_workspace` in Paseo environments, or run the project cleanup script in manual workflows. Leaving orphan worktrees, `build/`, or `.codegraph/` directories on disk after archiving an agent is **STRICTLY FORBIDDEN**.
  - **Examples**:
    - *BAD*: Creating a new Project, naming the Workspace `"Refactor lock logic."` for a Chinese-speaking user, and archiving the agent without removing its worktree.
    - *GOOD*: Creating an isolated Workspace under the current Project named `修复CodeGraph死锁`, verifying diffs, and calling `paseo_archive_workspace` to purge disk artifacts.

## 5. Structural Search, Graph Analysis & Batch Refactoring

- **CodeGraph First & Index Maintenance**:
  - **Status Check**: Before analyzing or editing code, run `codegraph_status` or `codegraph status <abs_project_path>`. Prioritize CodeGraph for architecture exploration, symbol lookup, call hierarchies, and impact analysis over broad `grep` or full-file reads. Each worktree maintains its own isolated index.
  - **Init & Sync**: If no index exists for the verified project/worktree root, run `codegraph init <abs_project_path> --yes` (**NEVER** run on `$HOME`, `/`, or unrelated directories; do not pass `--force` by default). Run `codegraph sync <abs_project_path>` after code edits or branch switches.
  - **Lock Safety**: When encountering an index lock, inspect lock metadata and process liveness first. Run `codegraph unlock <abs_project_path>` **ONLY** after confirming the lock is stale and no indexing daemon is active. Blindly unlocking, deleting lock files, or killing processes is **STRICTLY FORBIDDEN**; pause and ask the user if liveness is unclear.
  - **Query Gradient**: Use `codegraph_explore` for broad discovery, `codegraph_search` for symbols, `codegraph_files` for structure, `codegraph_node` for known nodes, and `callers/callees/impact` for blast radius. Fall back to ReadSeek/`grep` only when graph coverage is insufficient. Always read fresh source files before editing.
- **`ast-grep` (`sg`) for Structural Search & Batch Codemods**:
  - Use standard edit tools for single-site edits. For cross-file changes, API migrations, batch renames, or pattern refactoring, using `ast-grep` is **MANDATORY**. Manual file-by-file editing or blind regex replacement (`sed`) is **STRICTLY FORBIDDEN**.
  - **Mandatory 3-Step Workflow**:
    1. *Verify Matches*: Run `ast-grep -p '<PATTERN>' -l <LANG> [PATHS]...` to confirm accurate AST hits.
    2. *Batch Rewrite*: Apply changes via `ast-grep -p '<PATTERN>' -r '<REWRITE>' -l <LANG> -U` (or YAML rules for complex transforms).
    3. *Diff & Verify*: Inspect `git diff` and run minimal affected compilation/tests.
  - **Examples**:
    - *BAD*: Running `grep -rn "connect(" src/` followed by `sed -i 's/connect(/openSession(/g'`.
    - *GOOD*: Checking impact via `codegraph callers Client.connect`, then executing `sg run -p '$C.connect($URL, $T)' -r '$C.openSession($URL, $T)' -U`.

## 6. Meta: Self-Update & Distribution Protocol

- **Canonical Source**:
  - **Repository**: `https://github.com/RainTrap341/ruleset`
  - **Rule Set**: `coding_basic` (other profiles may coexist under `agentsmd/`)
  - **Raw Endpoint**: `https://raw.githubusercontent.com/RainTrap341/ruleset/main/agentsmd/coding_basic/AGENTS.md`
  - **Local Central Clone**: `~/.config/agents`
  - **Active Rule File**: `~/.config/agents/agentsmd/coding_basic/AGENTS.md`
- **Agent-Triggered Self-Update**:
  - When the user asks to update, sync, or refresh `AGENTS.md` (e.g., "更新 agents.md" or "同步最新全局规则"), update the central clone first (`git -C ~/.config/agents pull --ff-only`).
  - **Selected Profile**: Resolve the active symlink and identify its profile under `agentsmd/<profile>/AGENTS.md`. Do not assume all machines or projects use `coding_basic`.
  - **Preserve Symlinks**: Never replace a project symlink with a regular file. Do not run `curl -o ./AGENTS.md` from a project directory.
  - **Safety**: Inspect the working tree and upstream before pulling. Never discard local changes or force-reset a diverged branch. Report offline or blocked updates.
  - **Verification**: Inspect the diff after updating and summarize the changes to the user in their current language.
  - **Examples**:
    - *BAD*: Running `curl -o ./AGENTS.md <URL>` directly in a project folder and breaking the symlink to a shared rule file.
    - *GOOD*: Running `git -C ~/.config/agents pull --ff-only`, resolving the selected rule file, and reporting updated sections in the user's language.

---
**CRITICAL REMINDER**: Write technical rules and artifacts in English. Always reply to the user in the user's current language, unless the user requests otherwise.
