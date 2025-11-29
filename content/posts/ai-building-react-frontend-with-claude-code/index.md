+++
author = "Daniel Ancuta"
title = "From Bug Fixes to Full Features: Building React Applications with Claude Code"
date = "2025-11-29"
description = "How an automated Jira-to-Claude workflow evolved from fixing UI bugs to enabling rapid feature development"
tags = ["claude code", "ai", "automation", "development-workflow", "productivity", "react", "llm"]
+++

So here's the situation: our frontend team had this growing backlog of UI tweaks. 

Buttons slightly off, wrong font sizes, missing translations, components that didn't quite match what the designers put in Figma. 

Every ticket meant jumping between Jira, Figma, and the editor, figuring out what actually needs to change...

I kept thinking: most of this is pretty mechanical. What if we could automate it?

I'm not a frontend developer, but I wanted to help clear some of these tickets.

## Building the Workflow

I ended up building a workflow that takes a Jira ticket ID and handles most of the repetitive work:

1. Grabs the ticket details from Jira (REST API)
2. Pulls the linked Figma designs (also REST API)
3. Analyzes what needs to change
4. Creates a git branch with the right naming
5. Implements the fix (with me reviewing and giving feedback)
6. Commits everything with a proper message

The trick was getting the prompt right and making sure the tickets were structured in a way Claude could actually parse.

## What Actually Worked

**Standardizing the Jira tickets**: Clear "Expected Result" and "Actual Result" sections. Direct Figma links to components. XPath selectors. 

This made tickets easier for humans to read too.

**Iterating on the prompt**: Prompts are like code - you don't get it right the first time, you refactor until it works. By defining required information, patterns that needs to be used, and anti-patterns to avoid.

**QA collaboration**: None of this would've worked without the QA team helping redesign the ticket structure and testing workflow. That was crucial.

## Results

As someone who doesn't do frontend work regularly, I fixed 13 bugs in 2 days using this workflow. Most were simple - text alignment, translations, typography. But it also handled trickier stuff like building a new filter component that needed GraphQL query changes and different UI behavior.

But, that's not all!

### It works great for feature development too

The workflow turned out to be useful for building new features as well. One person on the team used these same prompts to build an entire React app's frontend in a few days.

Other folks took the prompt concept and split it into specialized tools: one for creating Jira tickets from code diff, another for generating PRs with descriptions, one for setting up git branches.

Those prompts we extracted to separated repository and now can be used by other teams as well!

## Where This Could Go

Right now the workflow is AI-assisted but human is required in the loop for every step. The next evolution could be more autonomous agents that handle entire workflows and only check in at key decision points.

## Limitations

Look, Claude Code isn't magic. I reviewed and iterated on basically everything it produced. 

You need to understand your codebase - if you can't tell good solutions from bad ones, you're gonna have problems. 

Skills matter too. Without solid development fundamentals, you'll just create a mess faster.

## Wrapping Up
The key isn't just "use AI" - it's building the structure around it (clear tickets, good prompts, proper review processes) that makes it actually useful.

Want help setting up something similar for your team? [Get in touch](/contact)