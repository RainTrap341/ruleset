# Global Engineering & Agent Collaboration Rules

## 1. Language Routing & ASD-STE100 Documentation

- **Separate conversation and artifact languages**:
  - **User conversation**: Reply in the language the user currently uses. Use that language for plans, explanations, progress reports, and Paseo Workspace titles unless the user requests another language.
  - **Technical artifacts**: Write code comments, API docs, Markdown documentation, commit messages, log strings, and identifiers in English. User-facing localized content and user-language Workspace titles are explicit exceptions.
- **ASD-STE100 & intent-only comments**:
  - **Comply with ASD-STE100**: Use active voice. Keep descriptive sentences under 25 words and procedural sentences under 20 words. Maintain one word, one meaning. Avoid idioms and ambiguous language.
  - **State purpose (why), not mechanics (how)**: Comments MUST explain intent, domain contracts, boundary conditions, or non-obvious side effects.
  - **Avoid flow narration**: Do not restate loops, conditionals, or assignments. Avoid comments inside method bodies unless documenting a non-obvious algorithmic constraint or protocol quirk.
  - **Examples**:
    - *BAD*: \`// 校验节点\` or \`// The node list is iterated through and isolated nodes are removed.\`
    - *GOOD*: \`// Prunes isolated vertices to guarantee a connected graph before layout computation.\`

## 2. Code Style & Architectural Reusability

- **Zero-hardcoding policy**:
  - Do not embed magic numbers, raw strings, URLs, paths, or thresholds inside business logic.
  - Review parameters and conditionals before writing code. Extract meaningful values into named immutable fields (\`static final\` / \`final\` in Java/Kotlin, \`const\` / \`readonly\` in TS/Rust/Go/C++).
  - **Examples**:
    - *BAD*: \`if (retry > 3) connect("wss://relay.internal", 5000);\`
    - *GOOD*: Declare \`private static final int MAX_RETRY_ATTEMPTS = 3;\` and reference the constant.
- **Utility extraction and reusability first**:
  - Do not scatter ad-hoc parsing, validation, IO, or data transformations through domain workflows.
  - Search existing utilities with CodeGraph or \`ast-grep\` first. If none exists, extract shared logic into a stateless, generic, type-safe utility.
  - **Examples**:
    - *BAD*: \`const id = raw.trim().toLowerCase().replace(/[^a-z0-9]/g, '');\` inside a business function.
    - *GOOD*: Extract \`IdentifierUtils.sanitize(raw)\` and call it from the domain workflow.

## 3. Minimal Effective Testing, Verification & OS Routing

- **Minimal effective testing**:
  - **No test bloat**: Create only tests needed for core contracts, critical boundaries, or regressions.
  - **No ritualistic tests**: Do not test trivial accessors, over-mock implementation details, or create exhaustive permutation sets. Add focused cases to existing suites for minor fixes.
  - **Examples**:
    - *BAD*: Add a new test class with ten heavily mocked interaction tests for a three-line fix.
    - *GOOD*: Add one regression case to the existing suite for the affected input-output contract.
- **Staged verification and honest delivery**:
  - Run affected compilation and unit tests for routine code changes. Skip tests for pure documentation/configuration edits.
  - Run full verification before merging, releasing, or when requested. Run hardware tests only when capturing physical evidence or during final acceptance.
  - Provide concrete verification evidence. Report skipped or unavailable verification. Never claim completion based only on green builds.
- **OS environment routing**:
  - Detect Linux, Windows, or macOS before running commands, scripts, or tests. Match paths, shells, and display environments.
  - **Linux-only X11 virtual desktop constraint**: For GUI or screenshot tests in a Linux virtual desktop, unset \`WAYLAND_DISPLAY\`, set \`XDG_SESSION_TYPE=x11\`, and use \`xvfb-run -a\`.
  - Do not inherit a Wayland session for those tests. Never apply \`xvfb-run\` on Windows or macOS.
  - **Examples**:
    - *BAD*: Run Linux virtual-desktop screenshot tests under Wayland, or call \`xvfb-run\` on Windows.
    - *GOOD*: On Linux, run \`env -u WAYLAND_DISPLAY XDG_SESSION_TYPE=x11 xvfb-run -a pnpm test:ui\`. On Windows, use the native test runner.

## 4. Parallel Collaboration & Paseo Lifecycle

- **Delegation boundaries**:
  - The main agent implements and verifies tasks directly by default. Spawn sub-agents only for independent tasks with clear parallel boundaries.
  - Give each sub-agent an isolated worktree. Never allow two agents to edit the same implementation concurrently.
- **Paseo discovery and dispatch**:
  - At task start, read the Paseo skill and make a read-only MCP discovery call to check service and provider availability. Use Paseo CLI help if MCP is unavailable.
  - When Paseo is available, delegate exclusively through Paseo. Match agent profiles and notes first. Discover providers if no profile matches. Never guess models or permissions.
  - Define objectives, scope, dependencies, and verifiable acceptance criteria in every delegation.
  - **Project and Workspace rules**: Sub-agents MUST use the dispatcher's Project. Create a Workspace only for an isolated worktree.
  - **Workspace titles**: Use the user's current language and an "action + object" structure. For Chinese, use 4–12 characters. For other languages, use a concise 2–6 word title. Preserve essential identifiers. Avoid explanations and trailing punctuation.
  - **Failure handling**: Report Paseo or dispatch failures. Continue only within existing authorization, or seek approval to change delegation channels. Never silently switch channels or double-dispatch.
- **Lifecycle cleanup**:
  - A sub-agent's completion report does not establish acceptance. Review diffs and verify outcomes before delivery.
  - After a delegated task closes, archive its Workspace with \`paseo_archive_workspace\`. In manual workflows, use project cleanup scripts.
  - Do not leave orphan worktrees, \`build/\`, or \`.codegraph/\` directories after cleanup.
  - **Examples**:
    - *BAD*: Create a new Project, use an English sentence as a Chinese user's Workspace title, then archive without removing the worktree.
    - *GOOD*: Keep the current Project, name the Workspace \`修复CodeGraph死锁\`, verify diffs, and archive with cleanup.

## 5. Structural Search, Graph Analysis & Batch Refactoring

- **CodeGraph first and index maintenance**:
  - **Status check**: Before code analysis or edits, run \`codegraph_status\` or \`codegraph status <abs_project_path>\`.
  - Use CodeGraph for architecture, symbols, call hierarchies, and impact analysis before broad \`grep\` or full-file reads. Keep an isolated index per worktree.
  - **Init and sync**: If the verified project root lacks an index, run \`codegraph init <abs_project_path> --yes\`. Never initialize \`$HOME\`, \`/\`, or unrelated directories. Do not use \`--force\` by default.
  - Run \`codegraph sync <abs_project_path>\` after code edits or branch changes.
  - **Lock safety**: Inspect lock metadata and process liveness. Run \`codegraph unlock <abs_project_path>\` only for a confirmed stale lock with no active indexing process.
  - Never blindly unlock, delete locks, or kill processes. Ask the user if liveness is unclear.
  - **Query gradient**: Use \`codegraph_explore\` for discovery, \`codegraph_search\` for symbols, \`codegraph_files\` for structure, \`codegraph_node\` for known nodes, and \`callers/callees/impact\` for dependencies.
  - Fall back to ReadSeek or \`grep\` only if graph coverage is insufficient. Read current source files before editing.
- **\`ast-grep\` (\`sg\`) for structural search and batch codemods**:
  - Use standard editing tools for single-site changes. Use \`ast-grep\` for cross-file changes, API migrations, batch renames, and structural refactors.
  - Do not manually edit many similar sites or use blind regex replacements for structural code changes.
  - **Mandatory three-step workflow**:
    1. *Verify matches*: Run \`ast-grep -p '<PATTERN>' -l <LANG> [PATHS]...\`.
    2. *Batch rewrite*: Run \`ast-grep -p '<PATTERN>' -r '<REWRITE>' -l <LANG> -U\`, or use YAML rules.
    3. *Diff and verify*: Inspect \`git diff\` and run affected compilation and tests.
  - **Examples**:
    - *BAD*: Run \`grep -rn "connect(" src/\` and then blindly replace the method name with \`sed\`.
    - *GOOD*: Check the impact with \`codegraph callers Client.connect\`, then verify and rewrite matching calls using \`ast-grep\`.

## 6. Rule Repository Self-Update Protocol

- **Triggers**: Treat requests such as "更新 agents.md", "同步最新全局规则", or "update agent rules" as a request to update the selected rule repository.
- **Resolve the source**:
  1. Resolve the active global or project instruction symlink to its physical target.
  2. Confirm that the target belongs to a Git working tree. Read its remote URL and upstream branch.
  3. Do not assume that every machine or project uses \`coding_basic\`. Identify the active rule set from the resolved path.
- **Update safely**:
  1. Check for uncommitted changes. Do not overwrite or discard them.
  2. Fetch from the configured upstream, then update only by fast-forward. Do not force-pull or reset.
  3. If the repository is offline, diverged, or modified, stop and report the condition.
  4. Review and summarize the changes to the active rule file. State whether the agent needs a session reload.
- **Report in the user's language**: Describe the selected rule set, previous and new revisions, meaningful diff, and any blocked or skipped steps. Never claim a successful update without confirming Git state.
