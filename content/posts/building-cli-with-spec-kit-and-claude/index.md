+++
author = "Daniel Ancuta"
title = "Building a full CLI tool with spec-kit (spec-driven development) and Claude (AI)"
date = "2026-01-02"
description = "From loose ideas to working CLI + TUI + Web in 3 days using spec-driven development with Claude Code (AI)."
tags = ["claude code", "ai", "spec-kit", "cli", "golang", "productivity"]
+++

Prompt engineering has become a fundamental skill for developers. There are different ways to approach it.

Some are more manual, some are way more automated with semi or fully autonomous agents.

Today I would like to share my experience with [spec-kit](https://github.com/github/spec-kit) and Claude Code (Anthropic's AI coding assistant). 

`spec-kit` is a toolkit for spec-driven development. You define the project's principles (its "constitution"), what you want to build (its "specification"), and technology implementation plan (its "planning").

Based on all of that `spec-kit` will generate a list of tasks that need to be completed to achieve the goal.

That's how I built [pkit](https://github.com/whisller/pkit), a fully functional CLI tool with TUI and web interfaces, in less than 3 days.
Not a single line of code is written by hand.

## What's Spec-Kit?

Instead of throwing random prompts at an AI and hoping for the best, spec-kit gives you a structured workflow:

1. **/speckit.constitution** - Define your project principles
2. **/speckit.specify** - Describe what you want to build
3. **/speckit.plan** - Create technical implementation plans
4. **/speckit.tasks** - Generate actionable task lists
5. **/speckit.implement** - Execute the build

It's designed to replace ad-hoc prompting with structured, intent-driven development.

You can then implement these tasks:
1. one by one, this way you will have most impact on implementation during development
2. in batches, or all in one go, this is the fastest way, especially with "bypass permissions on" mode (which lets Claude make changes without asking for approval each time)

Have a look at its documentation at [spec-kit](https://github.com/github/spec-kit).

## The Experiment

I wanted to build **[pkit](https://github.com/whisller/pkit)** - a multi-source AI prompt bookmark manager. 
The tool lets you subscribe to GitHub repos full of prompts (your private collections or public like Fabric patterns), search them, bookmark favorites, and pipe them to AI tools.

My approach was intentionally hands-off:
- Let spec-kit organize my loose ideas
- Run everything in Docker sandbox with bypass permissions enabled for Claude
- At first, I was reviewing execution carefully, after few iterations I decided to let it run wild with "bypass permissions on" mode
- Do not write any code by hand

## What worked really well
### 1. Spec-kit structured my thoughts

I had these scattered thoughts about what the tool should do. Spec-kit helped me to think through:
- What problem am I actually solving?
- What principles should guide the implementation?
- What are the core features vs nice-to-haves?

That structure alone made the project clearer, even before any code existed.

### 2. Architectural changes were easy
Some of the initial architecture decisions weren't ideal. Like allowing only bookmarked prompts to have "tags".

Still, introducing changes during the planning phase, or later during implementation, for such implementation details was easy.

I believe this particular problem with tags could have been avoided if I spent more time reading through `spec-kit` generated model.

{{< alert >}}
**Keep in mind**
{{< /alert >}}

> Design phase with `spec-kit` is a critical part of the process.

### 3. TUI visual fixes are amazing now

This part genuinely impressed me.

6-7 months ago, I tried building TUI apps with Claude. It was painful - lots of back-and-forth fixing layout issues.

Now? I'd just describe where something was cut off or misplaced, maybe provide a copy of the broken UI, and Claude would fix it correctly. The improvement in handling visual aspects of terminal UIs is massive.

### 4. Full stack in less than 3 days

CLI commands, interactive TUI with real-time filtering, web interface, all working in less than 3 days.

Yeah, the scope is small. Yeah, there's code that'll need refactoring. But still, without Claude and spec-kit, this would've taken way longer.

```bash
# What I ended up with:
pkit subscribe fabric/patterns
# Interactive TUI
pkit find
# Web interface
pkit web --port 8080
pkit search "code review"
# Save with custom alias
pkit save fabric:code-review --as review --tags dev,security
# Later, use the alias to pipe to Claude
pkit get review | claude -p "analyse this code"
```

## What didn't work so well

### 1. Claude tries to build not specified parts from scratch

This was the biggest friction point.

If something was described in the `spec-kit`, Claude would follow it. But if you have left some field for interpretation, Claude often would try to build a solution from scratch.

Good examples of this were model validation, terminal interaction, config management. 
Instead of just using existing libraries, Claude tried to build everything from scratch.

Maybe this can be fixed by modifying the `spec-kit` constitution with point like: 
> "Always prefer well-established libraries over custom implementations. Research and use standard tools for the target language."

This again shows how important the architectural thinking and planning phase is.

I haven't tested this yet, but worth exploring.

### 2. Spec-kit is overhead for small changes

Spec-kit shines when you're building something from scratch or adding significant features.

But if you just want to fix a bug or tweak one thing? It's complete overkill. The whole constitution → specify → plan → tasks → implement flow feels heavy for "make this button blue."

## The Results

After ~15 hours over 3 days, Claude generated:
- **~10k lines** of Go code (some require simplification and refactoring)
- **27 CLI commands** (subscribe, search, find, get, show, serve, etc.)
- **Interactive TUI**
- **Web interface**
- **Full-text search** powered by Bleve index
- **Multi-format parser** (Fabric patterns, awesome-chatgpt CSV, markdown)
- **Private GitHub repositories** support

The tool works. I've been using it daily to manage prompts from  multiple sources. It does exactly what I designed it to do.

## Not a single line of code was written by me

The most important takeaway? I didn't write a single line of code for version 0.1.0.

Everything you see in the [pkit repository](https://github.com/whisller/pkit) came from iterating on prompts and specs.

## When to use spec-kit

**Use it when:**
- Starting a new project from scratch
- Adding multiple related features
- You want structured, documented decision-making
- The project has clear boundaries and requirements
- You're building something with a defined scope

**Skip it when:**
- Fixing small bugs
- Making quick tweaks
- The change touches 1-2 files
- You already know exactly what to do
- The overhead of spec → plan → tasks is more work than just doing the change

## What I'll do differently next time

1. **Spend more time on the constitution** - The architectural principles you set upfront matter a lot. You need to list ALL things you want to avoid or make sure are in place.
2. **Review the data model carefully** - Changes are easy, but getting it right first is easier
3. **Use bypass mode sooner** - Once the foundation is solid, let AI implement everything.

If you want to try this yourself, I've open-sourced both the tool and the specs that generated it. 
Check out the [pkit repo](https://github.com/whisller/pkit) to see the final result, or start with the spec-kit documentation to build your own.

## Final thoughts

Spec-kit isn't magic. You still need to:
- Understand what you're building
- Review what the AI produces
- Know enough about the tech stack to spot bad decisions
- Iterate on prompts until they produce quality output

But as a framework for turning ideas into working code? It's pretty effective.

The structure it provides - constitution, specification, planning, tasks - keeps both you and the AI focused. You're not just throwing prompts into the void. You're building systematically.

And if you're willing to accept that some code will need refactoring, that some architectural decisions will need adjustment, that you'll spend time on prompt iteration instead of coding...

Well, you can build a lot faster than you'd expect.

That's it!

Want to talk about your experience with spec-kit or Claude Code? I'm curious how others are approaching this. [Get in touch](/contact).
