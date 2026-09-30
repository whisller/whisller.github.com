+++
author = "Daniel Ancuta"
title = "From Ticket to Green PR: Automating the Whole Dev Loop with an AI Agent Harness"
date = "2026-09-30"
description = "How an agent harness goes from ticket to PR, then watches CI and review comments and fixes them without the model doing the waiting."
tags = ["ai", "agents", "automation", "ci-cd", "code-review", "developer-experience"]
+++

Most AI coding workflows stop where the fun ends: the agent writes code, and you do the rest. Create the branch, write the commit, open the PR, refresh the checks page, read the review comments, fix them, push, refresh again.

That second half is where my time was going, so with my team I built an agent harness that does it.

## The Problem

Writing code is maybe a third of shipping a ticket. The rest is glue work that's easy to do badly: wrong branch names, "fix stuff" commits, PRs that don't mention the ticket, review comments sitting unresolved because nobody noticed the PR went red again.

Add AI reviewers (one session reviewing what another wrote) and there are even more comments to process. Something has to work through them.

## The Harness

"Agent harness" here means the layer around the model: the tools it may call, the skills (prompts and rules stored as files) it can load, and an orchestrator that decides what runs next.

The orchestrator is itself a skill. You say "implement this ticket", it picks a workflow template and runs other skills in order. It holds no logic of its own, only the order and what to pass along.

The pipeline has two halves. Before the push, it's a straight line:

1. Analyse the ticket, detect the stack
2. Create the branch
3. Write the code, using the skill for the stack
4. Review
5. Commit
6. Open a draft PR

After the push, it's a loop: wait for CI, fix failures, fix unresolved review threads, push again, until everything is green or a cap is hit. Then it keeps an eye out for new feedback.

{{< mermaid >}}
flowchart TD
    A[Ticket] --> B[Analyse + branch]
    B --> C[Write the code]
    C --> D[Review + commit]
    D --> E[Draft PR + push]
    E --> F{CI}
    F -->|pending| G[Tool waits for checks]
    G --> F
    F -->|failed| H[Fix failures and comments]
    F -->|green| I{Open review threads?}
    I -->|yes| H
    I -->|no| K[Ready for human, watch for feedback]
    K -->|new thread, comment or CI failure| H
    H --> J[Commit, push, resolve threads]
    J --> L{3 rounds?}
    L -->|no| F
    L -->|yes| M[Stop, report]
{{< /mermaid >}}

## Before the Push

You confirm the plan once, then it runs:

```text
Workflow: implement-ticket
Steps: ticket-analysis → branch → dev → review → commit → pr

Proceed? (y/n)
```

Skills can't pass structured data to each other, so the orchestrator summarises each step's output in plain text before the next one starts. Three rules keep it honest: nothing touches code until the branch step returns a branch name (otherwise the model may skip branching and commit to whatever is checked out), a failed step stops the run and asks what to do, and PRs are always drafts.

## After the Push

An open PR isn't done. It's waiting on CI and reviewers, and both generate work.

After each push the harness asks a tool for the check status of the pushed commit. If a check failed, the agent reads the failure, fixes and pushes. If everything is green but review threads are still open, it works through them instead. It doesn't matter who wrote them: a human, a bot, or an AI review session run separately. Fixed threads get resolved via the API.

One session reviews the branch and posts comments, another fixes them and closes the threads. They never talk to each other: the PR is the shared state.

### Waiting without polling

Every model turn re-reads the whole conversation, so a model that keeps asking "is CI done yet?" burns tokens for nothing. The harness does the waiting instead, in plain code:

- **CI:** one tool call waits for the checks of the pushed commit and returns when they finish. It copes with the awkward cases, like a newer push or a rewritten branch, instead of waiting for a commit that will never get checks.
- **Feedback:** a small background watcher follows the pull request (GitHub or Azure DevOps) and wakes the agent with a one-line summary when something new arrives: a comment, a review, a failing build, a merge. Waiting costs nothing until then.

### Caps

An unbounded fix-push-wait loop can burn tokens on a PR that never converges, so there are two limits. **4 waits per round** (about 18 minutes), then it reports the PR as still pending. **3 rounds per PR**, then it summarises what's still open and asks me.

Three rounds covers the normal cases: a lint failure, a missed edge case, a flaky test. If it's not converging by then, a human should look.

### Where the tokens go

Waits are kept short enough that the prompt cache stays warm, and watching costs nothing until something happens.

Superseded work is waste too: a review of an old commit keeps spending after a newer push. So the review tools notice when the PR moved on, stop, and post nothing.

## Let the Harness Fetch, Not the Model

The same idea applies to all GitHub data. Left alone, the model writes `gh` and `jq` one-liners, reads pages of JSON and gets the filtering slightly wrong. So the harness ships small deterministic tools that return only what the next decision needs: check status as counts plus failed names (about 170 bytes instead of 1,900 for 17 checks), unresolved review threads with capped bodies, file contents with line ranges.

Errors come back as errors, never as an empty result that looks like success. And since everything returned gets re-read on every later turn, keep it small. Bonus: ordinary code can be tested against a fake GitHub and a fake clock. A prompt can't.

## What Stays Manual

The fix step asks yes or no before applying each change. Review comments are concerns, not requirements, and some are wrong. An AI reviewer flagging something and another AI "fixing" it unseen gives you code that satisfies every comment and still does the wrong thing.

Waiting, thread bookkeeping, commits and pushes are automated. Judging whether a comment is right isn't.

## Final Thoughts

If you're building something similar, start with the post-push loop, not code generation. Coding agents are everywhere. One that watches the PR, fixes what it's told about and stops when it should is rare.

For the other end of the pipeline, see [error tracking and incident response on production](/posts/error-tracking-and-incident-response-on-production/).

If you want to hear more, feel free to [get in touch](https://whisller.dev/about/).

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
