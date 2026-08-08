---
lang: en
page_id: dev_setup_ai_tools
permalink: /posts/developer-setup-ai-tools
title: "The Ultimate Local AI Coding & Memory Stack: Conductor, Claude Code, RTK, Headroom, and Obsidian"
date: 2026-07-17
categories: [Developer Tools, AI, Workflow]
tags: [Tutorial, Conductor, Claude, Codex, Obsidian, Git, AI Tools, RTK, Graphify, Headroom]
author: Angel
description: A comprehensive instructional guide on building a local, memory-backed, and token-optimized AI development setup using Claude Code, Conductor, Graphify, Headroom, Obsidian, and RTK.
image: /assets/img/deepseek.png
---

## Building a Local AI Coding & Memory Stack

In modern software engineering, AI assistants like Claude and Codex have transitioned from basic copy-paste utilities to fully active agents operating in our development environments. However, scaling these agents across complex, multi-project workflows introduces friction: sky-high API token bills, lack of persistent cross-session memory, and "context overload" where models get lost in codebase noise.

To address these hurdles, I have wired together a **seven-layer local AI coding and memory stack**. It turns Claude Code into a project-aware, memory-backed agent capable of executing complex code tasks securely, efficiently, and with context preservation.

All layers run locally on your machine, with no cloud interaction beyond direct, secure model API calls. Below is a detailed, instructional blueprint on how to configure and run this exact setup.

---

## High-Level Architecture

Here is how the seven layers cooperate to optimize reasoning, control costs, and capture context:

```text
                  ┌──────────────────────────────────────────────┐
                  │       Claude Code (Primary Agent)            │
                  │  instructions · hooks · skills · plugins     │
                  └──┬──────────┬──────────┬──────────┬──────────┘
                     │          │          │          │
            (Hooks)  │  (MCP)   │  (Sub-   │  (Skill) │
                     │          │  agent)  │          │
                     ▼          ▼          ▼          ▼
  ┌─────────────┐ ┌──────────┐ ┌────────┐ ┌─────────────────────┐
  │ RTK         │ │ Obsidian │ │ Codex  │ │ Skills              │
  │ (PreToolUse │ │ Vault    │ │ CLI    │ │ graphify, ponytail, │
  │  hook)      │ │ (MCP +   │ │ plugin │ │ rails-*, pr-review  │
  └──────┬──────┘ │ hooks)   │ └────┬───┘ └─────────────────────┘
         │        └────┬─────┘      │
         ▼             │            ▼
  ┌─────────────┐      │     ┌───────────┐
  │ Headroom    │      ▼     │  OpenAI   │
  │ Proxy :8787 │ ┌────────┐ │  API      │
  └──────┬──────┘ │Obsidian│ └───────────┘
         │        │Git     │
         ▼        └────────┘
  ┌─────────────┐
  │ Anthropic   │
  │ API         │
  └─────────────┘

         ┌──────────────────────────────────────┐
         │  Conductor                           │
         │  Orchestrates parallel worktrees,    │
         │  each with its own agent session     │
         └──────────────────────────────────────┘
```

Let's break down each layer and construct the environment step-by-step.

---

## Layer 1: Claude Code with Layered Instructions, Hooks, and Skills

**Claude Code** is the primary agent. Configuration is not monolithic — it loads in a strict cascade from most-global to most-local, and a hook system allows you to intercept and transform agent behavior at runtime.

### Step 1: Establish Your Instruction Cascade

Claude Code loads instructions in this order, each layer overriding or augmenting the previous:

1. **`~/.claude/CLAUDE.md`** — Machine-global rules. Apply everywhere, every project.
2. **`<REPO>/CLAUDE.md`** — Team-visible project conventions, checked into version control.
3. **`<REPO>/AGENTS.md`** — Repository structure, style conventions, and gotchas (referenced via `@AGENTS.md` from the project CLAUDE.md).
4. **Subdirectory `AGENTS.md`** (e.g. `<REPO>/app/javascript/AGENTS.md`) — Activates only when working inside that directory. Use this to scope frontend-specific rules so they don't pollute backend tasks.

Use `@filename` imports to compose instructions from external files rather than maintaining huge, duplicated configs:

```markdown
# ~/.claude/CLAUDE.md

@RTK.md

# Global Directives
- Always run `/graphify query` for any architectural or behavioral question
  before opening code files.
- Favor using `rtk` commands over native shell utilities to keep payload
  sizes optimized.
```

The `@RTK.md` import pulls in a separate file (`~/.claude/RTK.md`) with RTK-specific usage rules, keeping the main file focused.

### Step 2: Wire Up the Hook System

Hooks are the key mechanism that transforms Claude Code from a passive assistant into a self-optimizing agent. They are configured in `~/.claude/settings.json` and fire at specific lifecycle events:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/rtk-rewrite.sh"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/graphify-nudge.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/skills/obsidian-second-brain/hooks/load_vault_context.py"
          }
        ]
      }
    ],
    "PostCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ]
  }
}
```

Each hook serves a distinct purpose:

| Hook Event | What It Does |
|---|---|
| **PreToolUse** (Bash) | Intercepts every shell command and rewrites it through RTK for token optimization — completely transparent |
| **UserPromptSubmit** | Scans your prompt for behavior/architecture questions and injects a reminder to query Graphify first |
| **SessionStart** | Loads Obsidian vault context (`_CLAUDE.md`) into the session when working inside the vault directory |
| **PostCompact** | When the context window is compacted, spawns a background agent that harvests vault-worthy items from the transcript |

### Step 3: Install Plugins

Claude Code supports a plugin system for integrating external tools. Enable plugins in `settings.json`:

```json
{
  "enabledPlugins": {
    "headroom@headroom-marketplace": true,
    "codex@openai-codex": true,
    "obsidian@obsidian-skills": true,
    "discord@claude-plugins-official": true
  }
}
```

Plugins add MCP tools, hooks, skills, and subagent types. The Codex plugin, for example, registers the `codex:codex-rescue` subagent and several slash commands (`/codex:rescue`, `/codex:review`, `/codex:status`).

### Step 4: Set Up Skills

Skills are packaged instruction sets that activate on slash commands. They live in `~/.claude/skills/` as directories containing a `SKILL.md` file:

```
~/.claude/skills/
├── graphify/SKILL.md        # /graphify — code-to-knowledge-graph
├── obsidian-second-brain/   # /obsidian-* — vault management
│   ├── SKILL.md
│   └── hooks/               # SessionStart + PostCompact hooks
├── pr-review/SKILL.md       # /pr-review — parallel multi-reviewer PR review
├── pull-request/SKILL.md    # /pull-request — create PR with Linear issue
├── resolve-fault/SKILL.md   # /resolve-fault — Honeybadger fault → PR
├── review-fix/SKILL.md      # /review-fix — self-review and fix
├── ponytail -> ~/code/agent-skills/skills/ponytail  # symlink
├── rails-best-practices-core -> ...                 # symlink
└── ...
```

Symlinked skills point to a shared `~/code/agent-skills/` repository, so updates to the skills repo propagate to all projects without manual copying.

---

## Layer 2: Headroom (Context Compression Proxy)

**Headroom** is the single biggest token saver in this stack. It runs as a local HTTP proxy that compresses everything flowing between your agent and the model API — tool outputs, file contents, conversation history — before it reaches the LLM.

### How It Works

Headroom intercepts API calls by sitting between Claude Code and the Anthropic API. You point Claude Code at the proxy instead of the real API:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8787"
  }
}
```

This single environment variable routes all model calls through Headroom's compression pipeline. No code changes, no wrapper scripts — Claude Code thinks it's talking directly to Anthropic.

### Step 1: Install and Start

```bash
pip install "headroom-ai[all]"
headroom proxy --port 8787
```

Or wrap your agent directly:

```bash
headroom wrap claude
```

### Step 2: Understand the Compression Pipeline

Headroom applies multiple compression strategies based on content type:

```text
 Agent prompt / tool output
        │
    ┌───┴──────────────────────────────────────────┐
    │  Headroom (runs locally)                     │
    │  CacheAligner  →  ContentRouter  →  CCR      │
    │                    ├─ SmartCrusher   (JSON)   │
    │                    ├─ CodeCompressor (AST)    │
    │                    └─ Kompress-base  (prose)  │
    └───┬──────────────────────────────────────────┘
        │  compressed prompt + retrieval tool
        ▼
    LLM provider
```

- **SmartCrusher** — Compresses JSON (tool outputs, API responses) by removing structural noise
- **CodeCompressor** — Uses AST-level analysis to strip non-essential code while preserving semantic meaning
- **Kompress-base** — A HuggingFace model that compresses natural language while preserving key claims
- **CacheAligner** — Stabilizes prompt prefixes so provider KV caches actually hit, reducing costs further
- **CCR (Compressed Context Retrieval)** — Stores originals locally; the LLM can call `headroom_retrieve` if it needs the full version of something that was compressed

### Step 3: Verify Savings

```bash
headroom perf            # one-shot savings report
headroom dashboard       # live dashboard (proxy must be running)
```

On real agent workloads, Headroom typically delivers **47–92% token reduction** depending on content type:

| Workload | Before | After | Savings |
|---|---:|---:|---:|
| Code search (100 results) | 17,765 | 1,408 | **92%** |
| SRE incident debugging | 65,694 | 5,118 | **92%** |
| GitHub issue triage | 54,174 | 14,761 | **73%** |
| Codebase exploration | 78,502 | 41,254 | **47%** |

Headroom also reduces **output tokens** — it trims ceremony and restated code from what the model writes back, not just what you send.

### Step 4: Persistent Deployment with launchd

For a "set and forget" proxy, deploy it as a macOS LaunchAgent:

```xml
<!-- ~/Library/LaunchAgents/com.user.headroom-proxy.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.headroom-proxy</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/headroom</string>
        <string>proxy</string>
        <string>--port</string>
        <string>8787</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/headroom-proxy.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/headroom-proxy.err</string>
</dict>
</plist>
```

Load it once:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.headroom-proxy.plist
```

Now Headroom starts automatically on login and restarts if it crashes.

---

## Layer 3: Conductor (Parallel Worktree Agents)

**Conductor** is an orchestration app that spawns parallel git worktrees under a canonical path, such as `~/conductor/workspaces/<project>/<workspace-name>/`. Each workspace hosts an isolated, non-interfering Claude Code session. This is how you scale from "one agent on one task" to "many agents on many tasks" without branch conflicts or context pollution.

### Step 1: Global Conductor Configuration

Conductor's global settings live at `~/.conductor/settings.toml`. This configures which model each agent uses, git conventions, and paths to agent executables:

```toml
"$schema" = "https://conductor.build/schemas/settings.schema.json"
claude_code_executable_path = "~/.local/bin/claude"
codex_executable_path = "/opt/homebrew/bin/codex"

[git]
branch_prefix_type = "github_username"
delete_branch_on_archive = true

[models]
default = "opus-4-6-1m"
review = "opus-4-6-1m"

[models.codex]
default_thinking_level = "high"
review_thinking_level = "high"
```

Key settings:
- **`branch_prefix_type = "github_username"`** — Branches auto-prefix with your GitHub handle for easy identification
- **`delete_branch_on_archive`** — Cleaned up branches don't litter your remote
- **Model selection** — Different models for different tasks. The 1M context variant is essential for large codebases

### Step 2: Per-Project Setup Scripts

Each project has a `.conductor/settings.local.toml` file (gitignored, private to your machine) that defines a **setup script**. This script runs automatically when Conductor creates a new worktree:

```toml
# <REPO>/.conductor/settings.local.toml
"$schema" = "https://conductor.build/schemas/settings.repo.schema.json"
file_include_globs = ".env*\n.gitignore\n"

[scripts]
run = "dev"
run_mode = "concurrent"
setup = """#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
MAIN_REPO="${CONDUCTOR_ROOT_PATH:-$HOME/code/my-project}"

# Ensure PATH includes language version managers
export RBENV_ROOT="${RBENV_ROOT:-$HOME/.rbenv}"
export PATH="$RBENV_ROOT/shims:$RBENV_ROOT/bin:$PATH"

if [[ -s "$HOME/.nvm/nvm.sh" ]]; then
  export NVM_DIR="${NVM_DIR:-$HOME/.nvm}"
  source "$NVM_DIR/nvm.sh"
  nvm use 22 --silent || nvm install 22 --no-progress
fi

echo "Root: $ROOT"

# Symlink graphify-out from the main repo into this worktree
if [[ "$ROOT" != "$MAIN_REPO" ]]; then
  GRAPHIFY_BIN="$HOME/.local/bin/graphify"
  if [[ ! -d "$MAIN_REPO/graphify-out" && -x "$GRAPHIFY_BIN" ]]; then
    echo "graphify-out missing - rebuilding (AST only, no LLM)"
    ( cd "$MAIN_REPO" && "$GRAPHIFY_BIN" update . ) \\
      || echo "graphify rebuild failed - worktree will have no graph"
  fi
  if [[ -d "$MAIN_REPO/graphify-out" && ! -e "$ROOT/graphify-out" ]]; then
    ln -s "$MAIN_REPO/graphify-out" "$ROOT/graphify-out"
    echo "Linked graphify-out from $MAIN_REPO"
  fi
fi

# Install dependencies
bundle install
yarn install

# Copy credentials from the main repo (if applicable)
SRC_KEY="$MAIN_REPO/config/credentials/development.key"
DEST_KEY="$ROOT/config/credentials/development.key"
if [[ -f "$SRC_KEY" ]]; then
  mkdir -p "$(dirname "$DEST_KEY")"
  cp "$SRC_KEY" "$DEST_KEY"
  echo "Copied development.key"
fi
"""
```

Key design decisions:
- **Inline script in TOML** — The setup script is embedded directly in the settings file, not a separate bash file. This keeps the entire workspace config self-contained.
- **Graphify symlink** — Every worktree shares one canonical `graphify-out/` from the main repo. If the graph is missing, it rebuilds automatically (AST-only, no LLM required).
- **Auto-rebuild** — If the graphify-out directory is missing from the main repo, the setup script runs `graphify update .` to rebuild it before symlinking.
- **`file_include_globs`** — Conductor copies `.env*` and `.gitignore` files into new worktrees, so environment config is always present.

### Step 3: Working with Workspaces

Conductor names each workspace after a city. When you open the Conductor app and create a new workspace for a project, you get something like:

```
~/conductor/workspaces/my-project/
├── tokyo/       ← feature branch A
├── berlin/      ← bug fix B
├── nairobi/     ← refactor C
└── amsterdam/   ← review task D
```

Each has its own branch, its own Claude session, and its own running dev server. The setup script runs once on creation, ensuring every workspace is immediately ready for work with no manual setup.

---

## Layer 4: Graphify (Code to Knowledge Graph)

Instead of feeding hundreds of raw code lines to the model to search for a bug, we use **Graphify** — a CLI tool that constructs a local knowledge graph of your codebase with community detection, an audit trail (EXTRACTED/INFERRED/AMBIGUOUS edges), and multiple query modes.

### Step 1: Build the Graph

```bash
graphify .                              # full pipeline on current directory
graphify . --mode deep                  # thorough extraction, richer edges
graphify . --update                     # incremental - only new/changed files
graphify . --obsidian --obsidian-dir ~/vaults/my-project-graph  # export to Obsidian
```

Graphify produces two key artifacts:
1. **Machine Graph** (`graphify-out/graph.json` + `GRAPH_REPORT.md`) — Queried directly by Claude Code
2. **Visual Export** — An Obsidian vault with interlinked markdown files and a `graph.canvas` showing relationships between controllers, models, jobs, and services

### Step 2: Query the Graph from Claude

Claude can query the graph directly via the `/graphify query` skill:

```bash
/graphify query "How does authentication work in this app?"     # BFS - broad context
/graphify query "Trace the payment flow" --dfs                  # DFS - follow one path deep
/graphify query "What calls the UserMailer?" --budget 1500      # cap answer at N tokens
/graphify path "AuthModule" "Database"                          # shortest path between concepts
/graphify explain "BackgroundWorker"                            # plain-language node explanation
```

### Step 3: Auto-Enforce Graphify-First with a Hook

The real power is making Claude use Graphify automatically. The `graphify-nudge.sh` hook (registered as a `UserPromptSubmit` hook) scans every prompt you submit for behavior/architecture questions:

```bash
#!/bin/bash
# ~/.claude/hooks/graphify-nudge.sh
set -u

INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // empty')
CWD=$(echo "$INPUT" | jq -r '.cwd // .working_directory // empty')

# Only fire when graphify-out exists in the current project
if [ ! -f "$CWD/graphify-out/graph.json" ]; then
  exit 0
fi

# Pattern-match behavior/flow/architecture questions
if echo "$PROMPT" | grep -qiE \
  'how (does|do|is|are|can)|what calls|where is .+ (used|called)|why does|walk me through|explain (the|how)|architecture|data ?flow|control ?flow|trace the|life ?cycle|end-to-end'; then

  jq -n '{
    "hookSpecificOutput": {
      "hookEventName": "UserPromptSubmit",
      "additionalContext": "GRAPHIFY REMINDER: A behavior/flow question was detected and graphify-out/graph.json is present. Your FIRST tool call MUST be /graphify query. Only Read/Grep the files it surfaces."
    }
  }'
fi
```

When you ask "How does authentication work?", the hook injects a context reminder into the prompt that forces Claude to query the graph first, then only read the specific files the graph identifies. No more reading 30 files to answer a cross-cutting question.

### Step 4: Automate with a Weekly launchd Agent

To ensure the graph never goes stale, configure a background launchd job on macOS:

```xml
<!-- ~/Library/LaunchAgents/com.user.graphify-update.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.graphify-update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>~/.local/bin/graphify-update.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>1</integer>
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
</dict>
</plist>
```

The companion script:

```bash
#!/bin/bash
# ~/.local/bin/graphify-update.sh

REPO_PATH="$HOME/code/my-project"
VAULT_PATH="$HOME/vaults/my-project-graph"

cd "$REPO_PATH"
graphify update .
graphify export obsidian --dir "$VAULT_PATH"

echo "Graphify update completed at $(date)"
```

---

## Layer 5: Obsidian as a "Second Brain"

A key design decision: **nothing durable is written to Claude's built-in memory**. Built-in model memory is transient and difficult to version. Instead, a dedicated Obsidian vault hosts all durable cross-project memory, and a pair of hooks keep it synchronized with your agent sessions automatically.

### Step 1: Enforce the AI-First Operating Manual

Create a root `_CLAUDE.md` file in your vault. This file is automatically loaded into Claude sessions via the `SessionStart` hook when working inside the vault:

```markdown
# _CLAUDE.md (vault root)

## AI-First Note Writing Rules:
1. **Self-Contained**: Each note must be understandable without requiring other context.
2. **Context Preamble**: Every note begins with a 2-3 sentence summary for future AI sessions.
3. **Rich Frontmatter**:
   ```yaml
   type: logic-explainer
   date: 2026-07-17
   topic: Authentication
   confidence: high
   ai-first: true
   ```
4. **Time Markers**: Reference events with precise dates: "As of 2026-07...".
5. **Wiki Links**: Link entities with `[[wikilinks]]` so Conductor/Claude can parse the graph.
```

### Step 2: The SessionStart Hook (Vault Context Loader)

The `load_vault_context.py` script fires at session start and checks whether the current working directory is inside the Obsidian vault. If so, it injects the vault's `_CLAUDE.md` into the agent's context:

```python
# ~/.claude/skills/obsidian-second-brain/hooks/load_vault_context.py

import json, os, sys
from pathlib import Path

def main():
    vault = os.environ.get("OBSIDIAN_VAULT_PATH", "")
    if not vault:
        return 0

    payload = json.load(sys.stdin)
    cwd = payload.get("cwd", "")

    # Only inject context when working inside the vault
    if not cwd.startswith(vault):
        return 0

    claude_md = Path(vault) / "_CLAUDE.md"
    if not claude_md.is_file():
        return 0

    content = claude_md.read_text(encoding="utf-8")
    output = {
        "hookSpecificOutput": {
            "hookEventName": "SessionStart",
            "additionalContext": f"Vault operating manual:\n\n{content}"
        }
    }
    json.dump(output, sys.stdout)
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

Configure the vault path in `settings.json`:

```json
{
  "env": {
    "OBSIDIAN_VAULT_PATH": "/path/to/your/vault",
    "OBSIDIAN_BG_AGENT_ENABLED": "1"
  }
}
```

### Step 3: The PostCompact Hook (Background Vault Agent)

This is the most powerful hook in the system. When Claude's context window fills up and gets compacted (summarized), the `PostCompact` hook spawns a **background Claude session** that reads the compacted transcript and propagates everything worth preserving to the vault:

```bash
#!/bin/bash
# ~/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh

VAULT="${OBSIDIAN_VAULT_PATH:-}"
[[ -z "$VAULT" ]] && exit 0
[[ "${OBSIDIAN_BG_AGENT_ENABLED:-0}" != "1" ]] && exit 0

INPUT=$(cat)
TRANSCRIPT=$(printf '%s' "$INPUT" | jq -r '.transcript_path // ""')
[[ -z "$TRANSCRIPT" || ! -f "$TRANSCRIPT" ]] && exit 0

# Extract the compaction summary from the transcript
SUMMARY=$(jq -rc 'select(.isCompactSummary == true) | .message.content' \
  "$TRANSCRIPT" | tail -n 1)
[[ -z "$SUMMARY" ]] && exit 0

TODAY=$(date +%Y-%m-%d)

# Build the prompt for the background agent
PROMPT="You are an autonomous Obsidian vault agent. The Claude session was just
compacted. Propagate everything worth preserving from the summary to the vault.

VAULT: $VAULT
TODAY: $TODAY

SESSION SUMMARY:
$SUMMARY

INSTRUCTIONS:
1. Read _CLAUDE.md at the vault root - follow its rules exactly.
2. Identify vault-worthy items: decisions, tasks, people, projects, dev work,
   ideas, learnings.
3. Before creating any note, search for an existing one. Never duplicate.
4. Update today's daily note with links to everything you touched.
5. Propagate: nothing saved in isolation - every write ripples to the daily
   note, boards, and linked notes."

# Spawn the background agent (fire-and-forget)
(cd "$VAULT" && claude --dangerously-skip-permissions -p "$PROMPT" \
  >> /tmp/obsidian-bg-agent.log 2>&1) &

exit 0
```

This means every long Claude session automatically captures decisions, code changes, and project context into the vault — without you doing anything.

### Step 4: The Obsidian Skill Catalog

The `obsidian-second-brain` skill exposes a rich set of slash commands for manual vault interaction:

| Command | Purpose |
|---|---|
| `/obsidian-save` | Save conversation highlights to the vault |
| `/obsidian-find <topic>` | Smart vault search with context |
| `/obsidian-log` | Log a dev session to the vault |
| `/obsidian-daily` | Create/update today's daily note |
| `/obsidian-project` | Create/update a project note |
| `/obsidian-person` | Create/update a person note |
| `/obsidian-task` | Add a task to the right kanban board |
| `/obsidian-decide` | Record a decision (lightweight or formal ADR) |
| `/obsidian-recap` | Summarize a time period from vault history |
| `/obsidian-health` | Vault health check — contradictions, gaps, staleness |
| `/obsidian-synthesize` | Auto-scan for unnamed patterns, write synthesis pages |

### Step 5: Set Up Obsidian Git Plugin for Version Control

1. In Obsidian, go to **Community Plugins**, search for **Obsidian Git**, and install.
2. Configure backup intervals to trigger every 5 to 10 minutes. This provides a timestamped memory log of how your notes and reasoning evolve over time.

---

## Layer 6: Codex CLI as a Second-Opinion Subagent

Sometimes Claude gets stuck in a recursive loop or needs a second perspective. For this, the **Codex CLI** (OpenAI) runs as a Claude Code plugin that registers a subagent called `codex:codex-rescue`.

### Step 1: Install the Plugin

The Codex plugin is installed via Claude Code's plugin marketplace system. Once enabled, it registers:

- **`codex:codex-rescue`** — A subagent that forwards tasks to Codex CLI for independent investigation
- **`/codex:rescue`** — Slash command for manual delegation
- **`/codex:review`** — Ask Codex for an adversarial code review
- **`/codex:status`** — Check on a background Codex task

### Step 2: Configure Codex in Conductor

Conductor's global settings specify the Codex executable path and thinking levels:

```toml
# ~/.conductor/settings.toml
codex_executable_path = "/opt/homebrew/bin/codex"

[models.codex]
default_thinking_level = "high"
review_thinking_level = "high"
```

### Step 3: How the Rescue Agent Works

The `codex-rescue` agent is a thin forwarding wrapper. When Claude encounters a blocker, it spawns a Codex task through the shared runtime:

```
Claude (stuck on a bug)
  → spawns codex:codex-rescue subagent
    → subagent calls: node codex-companion.mjs task "<prompt>" --write
      → Codex CLI runs independently, reads/writes code
        → result returns to Claude's context
```

Selection guidance built into the agent:
- **Proactive** — Don't wait for the user to ask; trigger when Claude is stuck
- **Bounded** — Don't grab simple tasks the main thread can finish quickly
- **Write-capable** — Default to `--write` mode so Codex can apply its fix directly
- **Resume-aware** — If continuing prior work, adds `--resume-last` automatically

---

## Layer 7: RTK (Rust Token Killer) — The Transparent Proxy

**RTK** is a high-performance CLI wrapper written in Rust. Unlike Headroom (which compresses at the API level), RTK operates at the **shell command level** — it intercepts `git`, `cat`, `grep`, `find`, and other CLI tools, strips noise from their output, and returns a token-optimized result.

### Step 1: The PreToolUse Hook (Not Aliases)

RTK is wired into Claude Code via a `PreToolUse` hook — not shell aliases. This is critical: the hook intercepts the Bash tool at the Claude Code level, rewrites the command through `rtk rewrite`, and returns the optimized version with an automatic permission grant:

```bash
#!/bin/bash
# ~/.claude/hooks/rtk-rewrite.sh

# Read the tool input from Claude Code
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
[ -z "$CMD" ] && exit 0

# Ask RTK to rewrite the command
REWRITTEN=$(rtk rewrite "$CMD" 2>/dev/null) || exit 0

# If unchanged, let it pass through
[ "$CMD" = "$REWRITTEN" ] && exit 0

# Return the rewritten command with auto-allow
ORIGINAL_INPUT=$(echo "$INPUT" | jq -c '.tool_input')
UPDATED_INPUT=$(echo "$ORIGINAL_INPUT" | jq --arg cmd "$REWRITTEN" '.command = $cmd')

jq -n \
  --argjson updated "$UPDATED_INPUT" \
  '{
    "hookSpecificOutput": {
      "hookEventName": "PreToolUse",
      "permissionDecision": "allow",
      "permissionDecisionReason": "RTK auto-rewrite",
      "updatedInput": $updated
    }
  }'
```

What this means in practice: when Claude runs `git status`, the hook rewrites it to `rtk git status`. When Claude runs `cat schema.rb`, the hook rewrites it to `rtk read schema.rb`. The agent never knows it happened — it just gets smaller, cleaner output.

### Step 2: Use RTK Rules to Save Tokens

The `~/.claude/RTK.md` file (imported via `@RTK.md`) teaches Claude when to prefer RTK commands:

- For files exceeding **50KB**, use `rtk read <path>` to get a structurally filtered view
- For pattern search: `rtk grep <pattern> <path>` instead of `cat | grep`
- For file discovery: `rtk find` instead of raw `find`

### Step 3: Monitor Your Savings

```bash
rtk gain              # Show cumulative token savings
rtk gain --history    # Show per-command savings history
rtk discover          # Analyze Claude Code transcripts for missed RTK opportunities
```

Typical results: **40–90% reduction in tokens** on common operations like directory listing, git diffs, and file reads.

---

## Put It All Together: The AI Development Flow

When you ask Claude a complex question like *"How does authentication work inside this microservice?"*, here is how the full stack operates:

1. **Conductor** created this workspace as an isolated worktree with its own branch. The setup script ran automatically, symlinking `graphify-out/` and installing dependencies.

2. **Claude reads the instruction cascade**: global `~/.claude/CLAUDE.md` → project `CLAUDE.md` → `@AGENTS.md` → any subdirectory `AGENTS.md` files.

3. **The UserPromptSubmit hook fires.** `graphify-nudge.sh` detects that your question is a behavior/architecture question and injects a reminder: "Your FIRST tool call MUST be `/graphify query`."

4. **Claude queries Graphify.** `/graphify query "How does authentication work?"` does a BFS traversal of `graphify-out/graph.json` and returns the specific files and relationships involved.

5. **Claude reads only the named files.** The `PreToolUse` hook intercepts every `cat` and `grep` command, transparently rewriting them through **RTK** for token-optimized output.

6. **Headroom compresses the entire exchange.** Every API call flows through the local proxy on port 8787, where SmartCrusher compresses tool outputs and CodeCompressor strips non-essential code. The model sees 50–90% fewer tokens without losing semantic content.

7. If Claude encounters an unexpected blocker, it **triggers `codex:codex-rescue`** — spawning a Codex CLI task for a second opinion, which runs independently and returns its findings.

8. Once the solution is found, Claude uses **`/obsidian-save`** to write a permanent record in the vault.

9. When the context window eventually fills and gets compacted, the **PostCompact hook** fires — spawning a background agent that extracts decisions, code changes, and learnings from the transcript into the vault automatically.

10. **Obsidian Git** auto-commits the vault changes every few minutes, preserving your progress across sessions.

By combining **transparent token optimization at two levels** (Headroom at the API layer, RTK at the shell layer) with a **structured local knowledge graph** (Graphify) and **persistent, hook-driven memory** (Obsidian), this setup creates a developer workspace that is fast, context-aware, and cost-efficient — with knowledge compounding across sessions instead of evaporating.
