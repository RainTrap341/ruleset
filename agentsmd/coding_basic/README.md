# coding_basic — Agent Rule Deployment

[View the rules](AGENTS.md) · [Back to the rule catalog](../../README.md)

This repository stores independent agent rule profiles. Each machine can select a global default. Each project can select another profile.

The `coding_basic` profile defines engineering standards, verification rules, agent collaboration, and self-update behavior. Install only the profiles you need. All profiles share one Git clone.

```text
~/.config/agents/
└── agentsmd/
    ├── coding_basic/
    │   ├── AGENTS.md      # Active rules
    │   └── README.md      # Deployment instructions
    └── other_profile/
        └── AGENTS.md      # Optional independent profile
```

## 1. Clone the Central Repository

Clone this repository into `~/.config/agents` on each development machine. Skip cloning if the directory already contains this repository.

### Linux / macOS

```bash
mkdir -p "$HOME/.config"
git clone https://github.com/RainTrap341/ruleset.git "$HOME/.config/agents"
```

### Windows (PowerShell)

**Prerequisite:** Enable **Developer Mode** in Windows Settings to create symbolic links without elevation. Alternatively, open PowerShell as Administrator.

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.config" | Out-Null
git clone https://github.com/RainTrap341/ruleset.git "$HOME\.config\agents"
```

## 2. Bind Global Agent Configurations

Select a rule profile for this machine. Change `coding_basic` to any existing folder under `agentsmd/`. These links do **not** point to a root-level `AGENTS.md`.

Existing agent configuration files are **not** overwritten. Back them up and remove or migrate them manually before linking.

Agent-specific instruction discovery may vary by version and configuration. Verify each tool's effective instruction files after setup. Existing sessions may need a restart.

### Linux / macOS

```bash
AGENT_PROFILE="coding_basic"
MASTER_RULE="$HOME/.config/agents/agentsmd/$AGENT_PROFILE/AGENTS.md"
test -f "$MASTER_RULE" || { echo "Missing profile: $MASTER_RULE"; return 1 2>/dev/null || exit 1; }

link_global_rule() {
  local path="$1"
  mkdir -p "$(dirname "$path")"
  if [ -e "$path" ] || [ -L "$path" ]; then
    printf 'Preserved existing file: %s\n' "$path"
    return
  fi
  ln -s "$MASTER_RULE" "$path"
}

# Pi Coding Agent
link_global_rule "$HOME/.config/pi/AGENTS.md"
link_global_rule "$HOME/.pi/agent/AGENTS.md"

# Claude Code
link_global_rule "$HOME/.claude/CLAUDE.md"

# OpenAI Codex CLI
link_global_rule "$HOME/.codex/AGENTS.md"

# Gemini CLI
link_global_rule "$HOME/.gemini/GEMINI.md"
```

**Aider (optional):** Add the selected rule file to the existing `read` setting in `~/.aider.conf.yml`. Do not create a second `read` key. For a new configuration file:

```bash
printf 'read:\n  - %s\n' "$MASTER_RULE" > "$HOME/.aider.conf.yml"
```

Only run the Aider command if `~/.aider.conf.yml` does not exist.

### Windows (PowerShell)

```powershell
$AgentProfile = "coding_basic"
$MasterRule = Join-Path $HOME ".config\agents\agentsmd\$AgentProfile\AGENTS.md"
if (-not (Test-Path -LiteralPath $MasterRule -PathType Leaf)) {
    throw "Missing profile: $MasterRule"
}

function New-GlobalAgentLink {
    param([string]$Path)
    $Existing = Get-Item -LiteralPath $Path -Force -ErrorAction SilentlyContinue
    if ($null -ne $Existing) {
        Write-Host "Preserved existing file: $Path"
        return
    }
    $Parent = Split-Path -Parent $Path
    New-Item -ItemType Directory -Force -Path $Parent | Out-Null
    New-Item -ItemType SymbolicLink -Path $Path -Target $MasterRule | Out-Null
}

# Pi Coding Agent
New-GlobalAgentLink "$HOME\.config\pi\AGENTS.md"
New-GlobalAgentLink "$HOME\.pi\agent\AGENTS.md"

# Claude Code
New-GlobalAgentLink "$HOME\.claude\CLAUDE.md"

# OpenAI Codex CLI
New-GlobalAgentLink "$HOME\.codex\AGENTS.md"

# Gemini CLI
New-GlobalAgentLink "$HOME\.gemini\GEMINI.md"
```

**Aider (optional):** Add the absolute `$MasterRule` path to the `read` list in `$HOME\.aider.conf.yml`. Preserve any existing settings.

## 3. Link Project Repositories (`link-agent`)

Project agents may read `AGENTS.md` from the workspace root. Use `link-agent` to bind a project to a selected profile.

The helper ignores only the **local, untracked** symlink through Git's `info/exclude`. It does not modify the project's shared `.gitignore`. It refuses to replace a file or a tracked `AGENTS.md`.

### Linux / macOS (`~/.zshrc` or `~/.bashrc`)

```bash
link-agent() {
  local profile="${1:-coding_basic}"
  local target="$HOME/.config/agents/agentsmd/$profile/AGENTS.md"
  local exclude_file
  if [ ! -f "$target" ]; then
    printf 'Missing rule profile: %s\n' "$target" >&2
    return 1
  fi
  if [ -e ./AGENTS.md ] || [ -L ./AGENTS.md ]; then
    echo "Existing ./AGENTS.md preserved" >&2
    return 1
  fi
  if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    if git ls-files --error-unmatch -- AGENTS.md >/dev/null 2>&1; then
      echo "Tracked AGENTS.md must not be replaced" >&2
      return 1
    fi
    exclude_file="$(git rev-parse --git-path info/exclude)" || return 1
    mkdir -p "$(dirname "$exclude_file")"
    grep -qxF "/AGENTS.md" "$exclude_file" 2>/dev/null ||
      printf '/AGENTS.md\n' >> "$exclude_file"
  fi
  ln -s "$target" ./AGENTS.md
  printf 'Linked ./AGENTS.md -> %s\n' "$target"
}
```


### Windows (PowerShell `$PROFILE`)

```powershell
function link-agent {
    param([string]$ProfileName = "coding_basic")
    $Target = Join-Path $HOME ".config\agents\agentsmd\$ProfileName\AGENTS.md"
    if (-not (Test-Path -LiteralPath $Target -PathType Leaf)) {
        throw "Missing rule profile: $Target"
    }
    if ($null -ne (Get-Item -LiteralPath ".\AGENTS.md" -Force -ErrorAction SilentlyContinue)) {
        throw "Existing AGENTS.md must not be replaced"
    }
    if (git rev-parse --is-inside-work-tree 2>$null) {
        git ls-files --error-unmatch -- AGENTS.md 2>$null | Out-Null
        if ($LASTEXITCODE -eq 0) { throw "Tracked AGENTS.md must not be replaced" }
        $ExcludeFile = git rev-parse --git-path info/exclude
        $ExcludeDir = Split-Path -Parent $ExcludeFile
        if ($ExcludeDir) {
            New-Item -ItemType Directory -Force -Path $ExcludeDir | Out-Null
        }
        if (-not (Select-String -Path $ExcludeFile -Pattern '^/AGENTS\.md$' -Quiet -ErrorAction SilentlyContinue)) {
            Add-Content -Path $ExcludeFile -Value "/AGENTS.md"
        }
    }
    New-Item -ItemType SymbolicLink -Path ".\AGENTS.md" -Target $Target | Out-Null
    Write-Host "Linked .\AGENTS.md -> $Target"
}
```

**Usage:** Open a new shell inside the **project root**.

```bash
link-agent                 # Select coding_basic
link-agent other_profile   # Select a different installed profile
```

For PowerShell, use the same two commands. Only profiles that exist in the central clone can be selected.

**Note:** Exclude rules cannot hide a file Git already tracks. Projects that already track `AGENTS.md` must keep their existing file or use a different integration strategy.

## 4. Enable Automated Background Sync

Global and project symlinks reference files inside the same Git clone. A fast-forward pull updates all selected profiles without changing their links.

Use `git pull --ff-only` to avoid unintended merge commits. A pull can stop when the clone has local changes or a divergent history.

### Linux (`systemd --user` — every 5 minutes)

```bash
mkdir -p "$HOME/.config/systemd/user"

cat > "$HOME/.config/systemd/user/agents-sync.service" <<'EOF'
[Unit]
Description=Sync Central Agent Rule Repository

[Service]
Type=oneshot
ExecStart=/usr/bin/env git -C %h/.config/agents pull --ff-only --quiet
EOF

cat > "$HOME/.config/systemd/user/agents-sync.timer" <<'EOF'
[Unit]
Description=Periodic Central Agent Rule Sync

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now agents-sync.timer
```

Check sync status with `systemctl --user status agents-sync.timer` and `journalctl --user -u agents-sync.service`.

### macOS (`launchd` — every 5 minutes)

```bash
mkdir -p "$HOME/Library/LaunchAgents"
cat > "$HOME/Library/LaunchAgents/dev.ruleset.sync.plist" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>dev.ruleset.sync</string>
  <key>ProgramArguments</key>
  <array>
    <string>$(command -v git)</string>
    <string>-C</string>
    <string>$HOME/.config/agents</string>
    <string>pull</string>
    <string>--ff-only</string>
    <string>--quiet</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>StartInterval</key><integer>300</integer>
</dict>
</plist>
EOF
launchctl bootstrap "gui/$(id -u)" "$HOME/Library/LaunchAgents/dev.ruleset.sync.plist"
```

If you already loaded this LaunchAgent, use `launchctl bootout` before loading it again.

### Windows (PowerShell `$PROFILE` asynchronous pull)

Add this line to your PowerShell profile to pull when a terminal opens:

```powershell
Start-Job -ScriptBlock { git -C "$HOME\.config\agents" pull --ff-only --quiet } | Out-Null
```

For periodic sync without opening a terminal, register a Scheduled Task instead:

```powershell
$GitExe = (Get-Command git.exe -ErrorAction Stop).Source
$AgentDir = Join-Path $HOME ".config\agents"
$Action = New-ScheduledTaskAction -Execute $GitExe -Argument "-C `"$AgentDir`" pull --ff-only --quiet"
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) -RepetitionInterval (New-TimeSpan -Minutes 5)
$Settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -DontStopIfGoingOnBatteries -AllowStartIfOnBatteries -Hidden
Register-ScheduledTask -TaskName "SyncGlobalAgentsMd" -Action $Action -Trigger $Trigger -Settings $Settings -Force
```

Choose either scheduled sync or terminal-start sync if you do not need both.

## 5. Agent-Triggered Self-Update

The selected profile's `AGENTS.md` defines a self-update protocol in Section 6.

Ask an active agent:

> `更新 agents.md` / `同步最新全局规则` / `update agent rules`

The agent must identify the active profile from the resolved symlink path. It then checks the central clone, pulls only with `--ff-only`, reviews the diff, and reports changes in the user's language.

For a manual update, run:

```bash
git -C "$HOME/.config/agents" pull --ff-only
```

PowerShell:

```powershell
git -C "$HOME\.config\agents" pull --ff-only
```

Restart an existing agent session if it caches instruction files. Automatic Git sync does not guarantee that a running agent reloads instructions.
