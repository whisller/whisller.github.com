+++
author = "Daniel Ancuta"
title = "Give Your AI Agent Tools, Not Just Instructions"
date = "2026-10-09"
description = "Branch names, commit footers, PR titles: stop asking the model to follow conventions and move the routine steps into tools it calls through an internal MCP server."
tags = ["ai", "agents", "mcp", "automation", "developer-experience", "git"]
+++

Your agent follows your conventions. Until it doesn't.

The context fills up and the rules get compacted away. A subagent starts fresh and never saw them. Two skills say different things. Then the branch is `feat/add-export` on Monday and `add-export-feature` on Tuesday, a commit has no footer, a `.env` file gets staged.

We were in the same boat: branches like `task/abc-104-add-export` and `chore/abc-107-fix-timeout` showed up side by side, and only one of those prefixes is valid. Our conventions lived in skill text, so nothing caught a wrong one.

## The Challenge

Skills are prompts stored as files. They're great for deciding what to do, and pretty bad at doing it the same way every time.

> A wrong branch name or commit footer **can't be fixed after the push**. A human would amend and force push. Our agents aren't allowed to.

Tightening the wording in the prompt doesn't fix this. A prompt is a suggestion, and the model is free to read it a bit differently on Tuesday.

## Let Code Do the Routine

Look at what the agent does on every ticket: create a branch, write a commit, open a PR, run the tests, move the ticket along.

None of that needs judgment. It needs the same result every time. So I took those steps out of the prompts and put them into code that the agent calls as tools.

{{< mermaid >}}
flowchart LR
    S[Skill decides what] --> T[Tool call]
    T --> V{Rules OK?}
    V -->|no| R[Refuse and say what to fix]
    R --> S
    V -->|yes| X[Do it]
    X --> G[git / GitHub / ticket tracker]
{{< /mermaid >}}

The skill still says when to start work on a ticket. The tool builds the name and checks it, so the model has no reason to type a branch name.

It isn't a wall. The agent can still run `git checkout -b` in a shell if it wants to. The tool makes the right path the easy one, and the next sections cover what catches the rest.

## An Internal MCP Server

MCP (Model Context Protocol) is how you expose tools to an agent. You can run your own server inside the harness with tools that only make sense for your workflow.

**The agent passes as little as possible.** This is the whole call for starting a branch:

```text
call:   branch_start(ticket="ABC-104")
result: started feat/abc-104-add-export from main, ticket moved to In Progress
```

The tool does the rest. The same idea for a commit: the agent sends type, summary and body, the tool adds the footer and signs. The model can't forget what it never writes.

**One builder, not one prompt copied twice.** Starting work on a ticket and suggesting a branch name for a new ticket both call the same function. One rule, one place to change it.

**Errors are instructions.** A vague refusal makes the model guess:

```text
Error: commit validation failed
```

A good one tells it exactly what to do:

```text
Error: commit 3f2a1c9 has no ticket footer.
Expected last line of the message: Refs: ABC-104
Nothing was pushed. Fix the commit and call the push tool again.
```

It names the commit, what's missing, what it should look like and that nothing is half-done. The reader that has to act on the error is a model, so write it for one.

## What a Tool Can Do

Whatever you want. It's regular code, so it can do anything your workflow needs. Take branch creation as an example. One tool call could:

- read the ticket
- pick the prefix from the issue type
- build a short slug from the summary
- find the default branch and branch from it
- refuse on a dirty working tree
- move the ticket to In Progress

Six steps the model used to do (or skip) by hand, now one call that behaves the same every time.

And that's one tool. Others in the same spirit:

- **Commit:** builds the message, adds the footer, signs, refuses secret-looking files
- **Pull request:** checks branch name and title, always opens a draft
- **Push:** re-checks the rules and refuses `main`, release and hotfix branches
- **Tests:** detects the project's stack and runs the right command
- **Review comments:** returns only unresolved threads, trimmed to what the next decision needs

It's about the idea, not the full list.

## Testing the Part That Used to Be a Prompt

This is the bonus I hadn't planned for. A prompt can't be unit tested. A tool can, with a fake GitHub, a fake clock and an ordinary test runner.

The tests we run on the branch tool:

- the invalid prefix that started all this (`task/...`) is refused
- each issue type maps to the right prefix
- a summary with filler words and special characters becomes the expected slug
- a dirty working tree is refused and nothing is changed

The error messages are tested too. The test checks that the footer error contains the expected format, because that text is what the agent acts on. A wrong convention is now refused locally, before anything reaches GitHub, and the tests keep it that way.

## What Stays With the Model

Not everything should become a tool. The model still decides what the code should look like, whether a review comment is right and what the commit summary says. The tool only decides what the *name* of the branch is and whether the push is allowed.

My rule of thumb: if two correct runs could differ and nobody would care, leave it to the model. If a difference would break a convention, a script or a pipeline, write a tool.

## Final Thoughts

If you want to try this, don't start with all of it. Take the one convention your agent breaks most often and move just that into a tool. For us it was the branch name.

One thing I skipped on purpose: a tool only helps if the agent can't go around it with a plain shell command. That deserves its own post.

The earlier post on [From Ticket to Green PR: Automating the Whole Dev Loop with an AI Agent Harness](/posts/from-ticket-to-green-pr-with-an-ai-agent-harness/) shows where these tools sit in the flow, and the one on [AI Agents in Your CI/CD: Why GitHub Rulesets Matter Now More Than Ever](/posts/ai-agents-in-your-ci-cd-why-github-rulesets-matter/) covers what stops a bad push when the tools are bypassed.

## The End

That's it! Hope you enjoyed reading. If you're building an agent harness and want a second pair of eyes, feel free to [get in touch](https://whisller.dev/about/).

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
