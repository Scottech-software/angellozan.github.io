---
lang: en
page_id: dev_setup_ai_tools
permalink: /posts/developer-setup-ai-tools
title: "Optimizing My Developer Setup with Conductor, Headroom, and Obsidian"
date: 2026-07-17
categories: [Developer Tools, AI, Workflow]
tags: [Tutorial, Conductor, Headroom, Obsidian, Git, AI Tools]
author: Angel
description: An instructional guide on setting up a local developer environment designed for AI coding tools like Codex and Claude, utilizing Headroom for token savings and Obsidian for context memory.
image: /assets/img/deepseek.png
---

## Optimizing My Developer Setup with Conductor, Headroom, and Obsidian

Integrating AI assistants into daily coding workflows can significantly boost productivity. However, token consumption, context limits, and model amnesia often present challenges.

This post outlines my developer environment, which addresses these hurdles using **Conductor**, **Claude**, **Codex**, **Headroom**, and **Obsidian** (with the Git community plugin). Below is an instructional guide to replicate this highly optimized local AI coding environment.

---

## The Core Stack & Architecture

My local development environment centers on five main components:

1. **Conductor**: The command-line orchestration layer and IDE assistant that integrates LLMs directly into the coding environment.
2. **Claude & Codex**: The primary AI engines used for reasoning, code generation, and pair-programming.
3. **Headroom**: A local proxy/caching layer that intercepts LLM requests to optimize payload structures and save valuable tokens.
4. **Obsidian**: Acts as my "second brain," hosting a repository graph, documentation, and architecture notes.
5. **Obsidian Git Plugin**: Continually tracks changes in the codebase and knowledge base to provide the AI models with structured context and memory.

Here is how these components communicate:

```text
[ Conductor (VS Code / CLI) ] ──> [ Headroom (Caching Proxy) ] ──> [ Claude & Codex APIs ]
              │
              └─> [ Obsidian (Second Brain / Repo Graph) ] <── [ Obsidian Git (Memory Store) ]
```

---

## Step 1: Setting Up Conductor

**Conductor** coordinates the AI interactions in your workspace.

1. Install Conductor via your package manager or the official installer. Refer to the Conductor documentation for your platform.
2. Configure Conductor to recognize your project workspace. This usually involves creating a configuration file (e.g., `conductor.config.json`) in your project's root folder:
   ```json
   {
     "default_model": "claude-3-5-sonnet",
     "api_endpoint": "http://localhost:8080/v1"
   }
   ```
   *Note: We point the API endpoint to our local Headroom proxy (`localhost:8080`) instead of direct Anthropic or OpenAI servers.*

---

## Step 2: Routing Traffic through Headroom

To keep API costs manageable and speed up repeated queries, **Headroom** acts as a middleman that caches contexts and optimizes payload size.

1. Download and run Headroom locally:
   ```bash
   headroom start --port 8080
   ```
2. Configure Headroom with your provider credentials (API keys for Anthropic and OpenAI) inside its secure config file:
   ```yaml
   # ~/.config/headroom/config.yaml
   providers:
     anthropic: "your-claude-api-key"
     openai: "your-codex-api-key"
   cache:
     enabled: true
     strategy: "semantic"
   ```
3. Verify the proxy is running. Conductor will now route all LLM requests through Headroom, which detects overlapping system prompts or files and serves them from cache where possible.

---

## Step 3: Configuring Obsidian as a Second Brain

An AI assistant is only as good as the context it receives. By using Obsidian, we maintain a Markdown-based repository graph and project notes.

1. Install [Obsidian](https://obsidian.md/) and open your project directory (or a dedicated `docs` folder) as an Obsidian Vault.
2. Structure your vault with:
   - **`Arch/`**: Architecture decision records (ADRs).
   - **`Context/`**: Key entities, business logic rules, and API contracts.
   - **`Tasks/`**: Active development logs.
3. Use internal Markdown links (`[[Linked Note]]`) to establish a repository graph. This structure allows Conductor to parse relationships between files quickly.

---

## Step 4: Adding the Obsidian Git Plugin for AI Memory

The **Obsidian Git** community plugin keeps your notes versioned. This version history serves as a time-machine memory for your AI models.

1. Inside Obsidian, go to **Settings** -> **Community Plugins** -> **Browse** and search for `Obsidian Git`.
2. Install and enable the plugin.
3. Configure the plugin to auto-backup every 5 or 10 minutes.
4. The plugin will run `git commit` and `git push` automatically in the background, keeping a clean timeline of how your thoughts and codebase evolve.
5. In Conductor, configure the context gathering command to pull recent git diffs and Obsidian-linked notes to pass to Claude and Codex:
   ```bash
   conductor context register --plugin obsidian-git
   ```

---

## The Outcome

With this setup active:
- **Zero redundant token costs**: Headroom caches recurring system prompts and unchanged files, cutting monthly API spend significantly.
- **Deeper reasoning capability**: Claude and Codex can read Obsidian's structured notes and Git histories to understand *why* certain code decisions were made, not just *what* the current code says.
- **Frictionless workflow**: You write code and document notes normally, while the background tooling handles proxying, caching, and context synchronization.
