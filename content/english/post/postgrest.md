---
title: "PostgREST - Write Your REST Server in SQL"
date: 2023-04-11
description: "PostgREST is a standalone web server that transforms your PostgreSQL database directly into a RESTful API."
tags: ["postgresql", "rest", "tools"]
---

## What is PostgREST?
As the [authors themselves say](https://postgrest.org/en/stable/index.html):
> PostgREST is a standalone web server that transforms your PostgreSQL database directly into a RESTful API. The structural constraints and permissions in the database determine the API endpoints and operations.

[PostgREST](https://postgrest.org/) is written in Haskell, and it aims to replace any CRUD tool you have ever used with PostgreSQL.

You write your REST API in plain SQL (PL/SQL) – sounds like fun, right?

## What it Offers
In fact, it offers quite a lot!

Obviously, it allows simple CRUD operations, so HTTP `GET`, `POST`, `PUT`, `PATCH`, and `DELETE` are supported. But not only that, you also have out-of-the-box support for bulk updates or upserts.

All your operations are authenticated with JWT tokens, and gradual access is supported through PostgreSQL user and group definitions.

On top of that, there is OpenAPI documentation generated based on your schema. There is also an extensive list of possible data filtering (for `GET`), support for JSON columns, and more.

Check out [their documentation](https://postgrest.org/en/stable/index.html) – it's very informative.

## My Experience with PostgREST
I had the chance to look at and work with a complex application in PostgREST that is a few years old.

A few things that stood out:

* You need to be very proficient in SQL and PL/SQL to understand what's going on, especially in complex applications.
* The learning curve is rather steep.
* Debugging is not that easy.
* Deployment of the app requires a third-party solution, like [pressly/goose](https://github.com/pressly/goose).
* There is duplication of code between structured code and migration files.
* If your app requires integration with some new third-party API or tool, the solution on how to achieve that might not be obvious.

## Final Thoughts
PostgREST is definitely an interesting but niche tool that can be used as an alternative for some parts of your business logic. However, the programming language used, SQL (PL/SQL), puts it in a position where you might end up with a complex project that no one really wants to maintain.
