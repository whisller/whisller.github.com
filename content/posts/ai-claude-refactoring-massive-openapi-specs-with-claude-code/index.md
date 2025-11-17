+++
author = "Daniel Ancuta"
title = "Refactoring Massive OpenAPI Specs with Claude Code"
date = "2025-11-17"
description = "How I used Claude Code to transform 3000-line monolithic OpenAPI files into maintainable modular structures in just four days"
tags = ["ai", "prompts", "prompt engineering", "productivity", "llm", "best practices", "api"]
+++

I was recently asked to help improve OpenAPI specs for a project where the documentation had gotten completely out of hand. Thousands of lines of OpenAPI specs scrambled in a few files.

This was the reality I walked into.

## The challenge and the solution

- The largest OpenAPI specification file had over 3,000 lines of code.
- Schema definitions were duplicated everywhere, creating potential inconsistencies in APIs.
- Code reviews? Pretty much impossible. You could review the specific change someone made, but understanding the file as a whole was just too much cognitive load.
- Finding anything meant using Ctrl+F and wishing for the best.

Luckily, a senior engineer on the team had already refactored one of the APIs. They had designed a modular structure that separated components into logical groups. The result was elegant, maintainable, and actually readable.

But we had several other APIs still stuck in monolith hell. Applying this pattern manually to each one would take weeks.

So I decided to try using Claude Code to speed up the refactoring work.

## How I approached it

First, I had Claude identify and extract common components across all the schemas: pagination parameters, response patterns, shared data structures. This created a foundation of reusable components that every API could reference.

The next step was to find a way to refactor other specs. Instead of doing it manually, I asked Claude:

> "Based on the structure of src/xyz, prepare me a prompt that will allow me to refactor other monolithic schemas."

Claude generated a prompt that captured all the patterns from the well-structured example: directory layouts, naming conventions, step-by-step instructions, common mistakes to avoid.

## The implementation

With the reusable prompt ready, I tackled the remaining API specs one by one. For each API, I had feed Claude the prompt along with the monolith API path, let it analyze it and propose a structure, review what it came up with, then apply the changes incrementally. Testing after each step was crucial.

I finished all the refactorings in about four days. These were APIs with thousands of lines that would have taken weeks to refactor manually. Most of my time went into verification, opening generated OpenAPI in side-by-side browser windows and checking that everything was correct.

## What I learned

Could I have done this without Claude? Sure. But it would've taken a lot longer. The output was not perfect. I still had to review everything and make tweaks, especially with the biggest APIs. But getting a 90% solution that I could reuse was a huge productivity boost.

The senior engineer's work was essential. Everything, the prompt, the execution, all of it was based on their initial structure. That's the real takeaway: you need someone who knows what good looks like to create that first example. Then you can use AI to scale it.

Token limits were the main technical hiccup. The largest APIs ate through available tokens fast. If you're dealing with truly massive files, you'll probably need to break the work into smaller chunks.

The reusable prompt approach is what made this work. Once I had that prompt based on the expert design, I could apply it to multiple APIs without starting from scratch each time.

## Try it out yourself!

The approach here works for any repetitive refactoring-spec files, config files, whatever.

- Start with human expertise. Get someone experienced to create one really good example of the target structure. (You can use Claude to help with this part too, but you need someone who can evaluate whether the result is actually good.)

- Once you have that gold standard, create a reusable prompt. Ask Claude to help you document all the patterns, conventions, and common mistakes from your example. This is what you'll use to apply the same refactoring elsewhere.

- Work incrementally and test frequently. Don't try to refactor everything at once. One API, one resource, one part at a time.

- And verify everything. AI-generated code needs review. Use diffs, run tests, manually check the output.

## Final thoughts

AI coding assistants multiply whatever you give them, good or bad. This is why having that senior engineer's work as the foundation was so important.

What could've been weeks of tedious work got done in four days. The APIs are now maintainable, reviewable, and actually pleasant to work with. All because we had a good pattern to replicate.

If you need help with API design, OpenAPI specs, or figuring out how to use AI tools effectively in your workflow, [get in touch](/about).
