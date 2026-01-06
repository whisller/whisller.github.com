+++
author = "Daniel Ancuta"
title = "Claude Compose Sandbox: Flexible Docker Environment for AI Agents"
date = "2026-01-06"
description = "A Docker Compose-based sandbox for Claude Code with persistent workspaces"
tags = ["docker", "ai", "claude", "devops", "automation", "docker-compose"]
+++

Running AI agents locally is great until you realize they have full access to your machine. One wrong command and you could lose important files.

Docker sandboxes solve this, but the official one lacks flexibility - no persistent workspaces, no multi-instance support, limited customization.

So I built [claude-compose-sandbox](https://github.com/whisller/claude-compose-sandbox) - a Docker Compose-based environment that gives you:

- **Persistent workspaces** - installed packages and files survive between sessions
- **Multi-instance support** - run multiple Claude instances in parallel
- **Zero-config Git** - automatic SSH agent forwarding and config mounting
- **Full flexibility** - it's just Docker Compose, customize however you want

```bash
git clone https://github.com/whisller/claude-compose-sandbox.git
cd claude-compose-sandbox
docker compose up -d
docker compose exec workspace bash
claude-code
```

That's it. Secure, persistent, and flexible.

Check it out: [github.com/whisller/claude-compose-sandbox](https://github.com/whisller/claude-compose-sandbox)

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
