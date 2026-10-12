+++
author = "Daniel Ancuta"
title = "Short Sentences Make Better Docs, and a Lint Keeps Them Short"
date = "2026-10-12"
description = "A few writing rules from Simplified Technical English, plus a test that fails the build, cut the longest sentence in our docs from 80 words to 25."
tags = ["documentation", "ai", "agents", "technical-writing", "developer-experience"]
+++

Look for the longest sentence in the docs of any project that's a few years old. It's usually a monster: three clauses, a parenthesis, an exception tucked in at the end. Every fix added a clause instead of a sentence, because that's the smallest diff.

Add an agent that writes docs and it gets worse. Models write long, polished sentences.

## The Challenge

Reviewers read the diff, not the paragraph, so a clause added today looks fine. A year later the paragraph needs two passes to parse. A reader who isn't a native English speaker, or who runs the page through a translator, gives up even sooner.

## Simplified Technical English

There's an old fix for this: Simplified Technical English, a controlled language written for aircraft maintenance manuals, where a misread sentence has consequences. The full specification is big, so we use a small subset. It lives in a skill the agent loads whenever it touches a README or a docs page:

- At most 20 words per sentence in procedures, 25 in descriptions
- One instruction per sentence, in the imperative
- Active voice
- At most 6 sentences per paragraph
- One term per concept
- Vertical lists for steps and for sets of 3 or more items

Two guardrails: **change prose only** (code, links and headings stay as they are) and **keep every fact** (if a shorter sentence would drop one, split it in two).

Here's what a change looks like. Before:

```text
The service retries failed requests up to three times, waiting longer after each
attempt (starting at one second), unless the response is a client error, in which
case it gives up immediately and logs the failure.
```

After:

```text
The service retries a failed request up to three times. It waits longer after
each attempt, starting at one second. If the response is a client error, the
service does not retry. It logs the failure.
```

And a set of things becomes a list. Before:

```text
The config accepts a host, a port and a timeout.
```

After:

```text
The config accepts:

- a host
- a port
- a timeout
```

### Adding a lint

Same lesson as in my [post about agent tools](/posts/give-your-ai-agent-tools-not-just-instructions/): the skill alone isn't enough.

It tells the agent how to write, but docs drift back the moment someone types a long sentence by hand. So we added a test to the repo's own suite. It **fails the build** on a sentence over 25 words or a paragraph over 6 sentences, and names the file, line and sentence.

It can only count, so passive voice, noun clusters and one-term-per-concept stay as review rules.

## The Numbers

The lint is also where the numbers come from. I ran the same check twice over the same 14 files (a README and the docs pages): once on the docs as they were, once after rewriting them with the rules above.

| | Before | After |
|---|---|---|
| Sentences over 25 words | 207 | 0 |
| Longest sentence | 80 words | 25 words |
| Paragraphs over 6 sentences | 6 | 0 |

The files got longer, with about 950 lines added and 315 removed. Splitting sentences and making lists takes more lines. I don't mind, since each line is quicker to read.

## The End

That's it! Hope you enjoyed reading. If you want help with agent skills or doc checks in your repos, feel free to [get in touch](https://whisller.dev/about/).

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
