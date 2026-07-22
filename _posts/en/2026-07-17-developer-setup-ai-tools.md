---
lang: en
page_id: dev_setup_ai_tools
permalink: /posts/developer-setup-ai-tools
title: "The Ultimate Local AI Coding & Memory Stack: Conductor, Claude Code, RTK, and Obsidian"
date: 2026-07-17
categories: [Developer Tools, AI, Workflow]
tags: [Tutorial, Conductor, Claude, Codex, Obsidian, Git, AI Tools, RTK, Graphify]
author: Angel
description: A comprehensive instructional guide on building a local, memory-backed, and token-optimized AI development setup using Claude Code, Conductor, Graphify, Obsidian, and RTK.
image: /assets/img/deepseek.png
---

## Building a Local AI Coding & Memory Stack

In modern software engineering, AI assistants like Claude and Codex have transitioned from basic copy-paste utilities to fully active agents operating in our development environments. However, scaling these agents across complex, multi-project workflows introduces friction: sky-high API token bills, lack of persistent cross-session memory, and "context overload" where models get lost in codebase noise.

To address these hurdles, I have wired together a **six-layer local AI coding and memory stack**. It turns Claude Code into a project-aware, memory-backed agent capable of executing complex code tasks securely, efficiently, and with context preservation.

All layers run locally on your machine, with no cloud interaction beyond direct, secure model API calls. Below is a detailed, instructional blueprint on how to configure and run this exact setup.

---

## High-Level Architecture

Here is how the six layers cooperate to optimize reasoning, control costs, and capture context:

```text
                  ┌──────────────────────────────────────────────┐
                  │          Claude Code (Primary Agent)         │
                  └──────┬────────────────────┬───────────┬──────┘
                         │                    │           │
                         │ (Hooks proxy)      │ (MCP)     │ (Subagent)
                         ▼                    ▼           ▼
  ┌──────────────────────────────┐     ┌───────────┐     ┌───────────────────────┐
  │  RTK (Rust Token Killer)     │     │ Obsidian  │     │   Codex CLI           │
  │  Transparent Proxy           │     │ Vault     │     │   (codex:codex-rescue)│
  └──────────────┬───────────────┘     └─────┬─────┘     └───────────────────────┘
                 │                           │
                 ▼                           ▼
  ┌──────────────────────────────┐     ┌───────────┐
  │  Model APIs (Claude/OpenAI)  │     │ Obsidian  │
  └──────────────────────────────┘     │ Git       │
                                       └───────────┘
```

Let's break down each layer and construct the environment step-by-step.

---

## Layer 1: Claude Code with Layered Instructions

**Claude Code** is the primary agent. To make configuration rules "just apply" depending on where the agent is operating, it loads instructions in a strict, layered cascade from most-global to most-local.

### Step 1: Establish Your Composition Rules
By using `@filename` imports, we can compose instructions from external files rather than maintaining huge, duplicated configs. Create your machine-global instructions:

```markdown
# ~/.claude/CLAUDE.md

@import "~/.claude/RTK.md"

# Global Directives
- Always run `/graphify query` for any architectural or behavioral question before opening code files.
- Favor using `rtk` commands over native shell utilities to keep payload sizes optimized.
```

### Step 2: Configure Scoped Agents
In your project repository, define specific guidelines that check-in to version control. This ensures team members also benefit from standard AI conventions:

- `<REPO>/CLAUDE.md`: Configured as the team-visible entry point, referencing `@AGENTS.md`.
- `<REPO>/AGENTS.md`: Outlines repository structure, style conventions, and gotchas.
- Subdirectory `AGENTS.md` (e.g. `<REPO>/app/javascript/AGENTS.md`): Activates only when working inside that directory. For example, specify React/Vite-specific rules here to prevent polluting backend Rails tasks with frontend noise.

---

## Layer 2: Conductor (Parallel Worktree Agents)

**Conductor** is an orchestration app that spawns parallel git worktrees under a canonical path, such as `~/conductor/workspaces/<project>/<workspace-name>/`. Each workspace hosts an isolated, non-interfering Claude Code session.

### Step 1: Configure Private Workspace Automation
Each project has a `.conductor/settings.local.toml` file (ignored, private to your machine). Define a **setup script** that automates the construction of self-healing workspaces when a new worktree is spun up:

```toml
# .conductor/settings.local.toml
[workspace]
setup_script = "~/.conductor/scripts/setup-worktree.sh"
```

### Step 2: Write the Self-Healing Script
Create the bash setup script to run inside your workspace. This script resolves the new worktree root, verifies the code graph, and creates a symlink so every parallel worktree shares one canonical reference:

```bash
#!/bin/bash
# ~/.conductor/scripts/setup-worktree.sh

set -e

# Resolve current worktree and main repo paths
WORKTREE_ROOT=$(pwd)
MAIN_REPO_ROOT="$ARTSVP_MAIN_REPO" # Environment variable passed by Conductor

echo "Initializing parallel worktree at: $WORKTREE_ROOT"

# Ensure the shared graphify folder exists in the main repository
if [ ! -d "$MAIN_REPO_ROOT/graphify-out" ]; then
  mkdir -p "$MAIN_REPO_ROOT/graphify-out"
fi

# Symlink the canonical graph into the new worktree
ln -sfn "$MAIN_REPO_ROOT/graphify-out" "$WORKTREE_ROOT/graphify-out"

# Run project setup steps (e.g., install dependencies)
bundle config set --local path 'vendor/bundle'
bundle install
yarn install
```

---

## Layer 3: Graphify (Code to Knowledge Graph)

Instead of feeding hundreds of raw code lines to the model to search for a bug, we use **Graphify**—a CLI tool installed at `~/.local/bin/graphify` that constructs a local dependency graph of our codebase.

### Step 1: Understand Graphify Artifacts
Graphify scans your project files and exports two assets:
1. **Machine Graph** (`<REPO>/graphify-out/graph.json` + `GRAPH_REPORT.md`): Queried directly by Claude Code using `/graphify query "..."`.
2. **Visual Export** (`<VAULT_STORAGE>/<project>-graph/`): An Obsidian vault consisting of interlinked markdown files and a `graph.canvas` showing the relationship between controllers, models, and background jobs.

### Step 2: Automate with a Weekly launchd Agent
To ensure the graph never goes stale, configure a background launchd job on macOS. Create a plist file:

```xml
<!-- ~/Library/LaunchAgents/com.user.graphify-project-update.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.graphify-project-update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/admin/.local/bin/graphify-project-update.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>1</integer> <!-- Monday -->
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>37</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/admin/.local/state/graphify-project-update.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/admin/.local/state/graphify-project-update.err</string>
</dict>
</plist>
```

And write the companion script `~/.local/bin/graphify-project-update.sh`:

```bash
#!/bin/bash
# ~/.local/bin/graphify-project-update.sh

REPO_PATH="/Users/admin/code/my-project"
VAULT_PATH="/Users/admin/vaults/my-project-graph"

cd "$REPO_PATH"
/Users/admin/.local/bin/graphify update .
/Users/admin/.local/bin/graphify export obsidian --dir "$VAULT_PATH"

echo "Graphify update completed at $(date)"
```

Load and start the background agent:
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.graphify-project-update.plist
```

---

## Layer 4: Obsidian as a "Second Brain"

A key design decision in this stack is that **nothing durable is written directly to Claude's built-in memory**. Built-in model memory is transient and difficult to version. Instead, a dedicated Obsidian vault located at `<VAULT_STORAGE>/second-brain/` hosts all durable cross-project memory.

### Step 1: Enforce the AI-First Operating Manual
We optimize the vault structure for LLM parsers. Create a root manual file inside your Obsidian vault:

```markdown
# second-brain/_CLAUDE.md

## AI-First Note Writing Rules:
1. **Self-Contained**: Ensure each note is completely understandable without requiring other context.
2. **Context Preamble**: Every note must begin with a 2-3 sentence summary meant for a future AI session.
3. **Rich Frontmatter**: Include metadata in every note:
   ```yaml
   type: logic-explainer
   date: 2026-07-17
   topic: Authentication
   confidence: high
   ai-first: true
   ```
4. **Time Markers**: Reference external events with precise dates: "As of 2026-07...".
5. **Wiki Links**: Link entities with `[[wikilinks]]` so Conductor/Claude can parse the notes graph easily.
```

### Step 2: Set Up Obsidian Git Plugin for Version Control
1. In Obsidian, go to **Community Plugins**, search for **Obsidian Git**, and click install.
2. Configure backup intervals to trigger every 5 to 10 minutes in the background. This provides a transparent, timestamped memory log of how your notes and architecture reasoning evolve over time.

---

## Layer 5: Codex CLI as a Second-Opinion Subagent

Sometimes, Claude gets stuck on a recursive loop or needs a second perspective on a complex algorithmic challenge. For this, we hook a separate **Codex CLI** as a Claude Code subagent called `codex:codex-rescue`.

1. Wire up custom skills inside `~/.claude/skills/codex/SKILL.md`.
2. Define how Claude communicates with the subagent using specialized tool specifications (`codex:rescue`, `codex:setup`).
3. Claude delegates complex tasks with:
   ```bash
   /subagent run codex:codex-rescue "Explain this segmentation fault"
   ```
4. This keeps your workflows flowing without forcing you to manually copy-paste errors between different browser browser tabs.

---

## Layer 6: RTK (Rust Token Killer) — The Transparent Proxy

**RTK** is a high-performance CLI wrapper written in Rust. It intercepts common shell utilities and strips out useless logs, metadata, and excessively verbose code files before passing them to the AI agent.

### Step 1: Configure Claude Code Hook
RTK is wired into Claude Code via a `PreToolUse` Bash hook in `~/.claude/CLAUDE.md` or as wrapper commands:

```bash
# Intercept git and file read commands transparently
alias git="rtk git"
alias cat="rtk read"
alias grep="rtk grep"
```

### Step 2: Use RTK Rules to Drastically Save Tokens
- For files exceeding **50KB**, use `rtk read <path>` to summarize the structural definitions instead of reading the whole code file.
- Check token savings statistics using:
  ```bash
  rtk gain
  ```
  *Typically, developers see a **40–90% reduction in tokens** on common operations like directory listing and system diffs.*

---

## Put It All Together: The AI Development Flow

When you ask Claude a complex question like *"How does authentication work inside this microservice?"*, here is how your stack operates seamlessly:

1. Claude reads the instructions from the hierarchical `CLAUDE.md` -> `AGENTS.md` cascade.
2. The rules force Claude to run `/graphify query` as its **first** operation.
3. Graphify reads the shared `graphify-out/graph.json` (symlinked in automatically by Conductor when the workspace was created) and names the target files.
4. Claude reads only the named files, relying on `rtk read` to keep the context payload clean and cheap.
5. If Claude encounters an unexpected blocker, it triggers `codex:codex-rescue` to get a second opinion.
6. Once the solution is found, Claude automatically uses the `/obsidian-save` skill to write a highly formatted, permanent record in the `second-brain/` Obsidian Vault.
7. Obsidian Git auto-saves the new record, preserving your progress for future sessions.

By combining **transparent token optimization** with a **structured local knowledge graph and memory**, this setup creates a developer workspace that is incredibly fast, context-aware, and highly cost-efficient.
