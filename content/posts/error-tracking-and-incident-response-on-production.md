+++
author = "Daniel Ancuta"
title = "Error tracking and incident response on production with Sentry + Squadcast + Linear + Slack"
date = "2023-07-01"
description = "Error tracking and incident response on production with Sentry + Squadcast + Linear"
tags = ["sentry", "squadcast", "linear", "slack", "production incident", "error monitoring"]
+++

There is multiple strategies to mitigate number of bugs and problems with the app on production.
Some teams focus on unit tests, functional tests, integration tests, manual tests, multiple environments 
to run those and build artifact of the app. Processes to review code, static code analysis, CI/CD pipelines and so on and so on. 

There is dozens of different approaches you can take, all are dependent on project you're building, company's budget or people's skills.

But regardless on your approach or skills, there is quite likely that sooner or later your application will face some problems. It can be unhandled exception, third party API that is not working as it used to or bug in your app.

That's why very important is to:
> Be aware of your platform's health and react to incidents that occur, ideally before your users will notice, to maintain trust with your customers.

## Alerting & Monitoring
As with assuring quality of your application, there is dozens of different ways to handle alerting and monitoring.

CloudWatch, PagerDuty, Datadog, New Relic, Sentry just to name a few. Dozens of possible configurations and combinations of tools. 

Today I would like to focus on three tools that together, or separately, can help you with Alerting & Monitoring.

[Sentry](https://sentry.io/), [squadcast](https://www.squadcast.com/), [slack](https://slack.com/), and [Linear](https://linear.app).

## How to integrate Sentry + Squadcast + Slack + Linear

All those tools separately brings a lot of value for your project. But the true power sits in integrating them together. 

{{< mermaid >}}
flowchart TD
    A["1. App (Throws Exception)"] -->|Exception| B[2. Sentry]
    B -->|Create Ticket in Linear| C[4. Linear Ticketing System]
    B -->|Send Notification to Slack| D[3. Slack]
    D -->|Write message to channel| G[#alerts-production]
    C -->|Notify through webhook| E[5. Squadcast]
    E -->|Create Incident| F[Escalation policy is triggered]
{{< /mermaid >}}

So what really happened in here?

1. Your App (microservice, docker container, frontend app etc.) throws an exception
2. Sentry catches it and based on project's alert rule sends notification about it to Slack and Linear
3. Slack API receives HTTP request and sends message to channel that your team follows, e.g. `#alerts-production`
4. Linear creates ticket from sentry issue in separated team e.g. `issues:prod`
5. Squadcast is being notified through webhook about new Linear ticket and creates incident which triggers configured escalation policy

This way we connected 4 systems that will help you improve awareness of your platform's health, and, hopefully, maintain trust with your customers.

Let us go through specified components of this flow and its, selected, features and configuration. That can be useful during process of setting up whole flow. 

### Sentry
Sentry is great error tracker, but not only, it also has profiling, replays of sessions that caused problems (on frontend), cron monitoring, dashboards, alerting and few extra things more.
Those components are available for multiple languages and frameworks.

But what we're really interested in during this exercise are two components, error tracking and alerting.

Setting up alerts is very easy, you can find it in "Alerts -> Create Alert" section. It should look similar to:
![Sentry Alerting](/img/sentry-linear-squadcast/sentry-alerting.png)

This way every new or regression issue will trigger alert. As you can see setting up it in Sentry is really easy. 

One obstacle you might find is when you have microservice environment with plenty of sentry projects.
You need to setup those alerts on every single project separately, as sentry does not allow you to have global rules. And as all of us know, number of microservices can grow rapidly. 

What worked for me is to write wrapper on top of Sentry REST API and automate the process. Which can be done as a part of your CI/CD.

For that I wrote python library [sentry-api-python](https://github.com/epsylabs/sentry-api-python), that should help you with this process.

### Slack
Slack is great team collaboration tool. With its API it's also great tool that you can integrate with. That's something that we can use in our "Alerting & Monitoring" flow.

I suggest to have, separated channel, where all production problems can be found and people can be notified. Place where first conversations about severity and solutions for the problem starts.

That's how example of the message will look like:
![Slack Message](/img/sentry-linear-squadcast/slack-message.png)

### Linear
I use [Linear](https://linear.app/) in the flow for multiple reasons:
1. Being a bridge between Sentry and Squadcast
2. Have a single place where you can prioritise issues (not all problems on production are equally important)
3. Keep a track of historical issues (for reporting)
4. Easier to share and be transparent with product owners/project managers for allocation of people to fix those issues
5. Keep a track of status of work for specified issue. Any change to status of ticket made in Linear is reflected in both Sentry and Squadcast

### Squadcast
Last, but not least, [Squadcast](https://squadcast.com). That's relatively new tool for me, which look super promising!

It allows you to gather alerts from different sources, currently they have more than 166 integrations available. What I really liked about this tool, it has good "on call" schedule module, with flexible escalation policy.
With those two modules of Squadcast you get powerful tool that notifies your team about platform's health.

I could write post on its own about only squadcast, so if you haven't used or heard about it, I recommend checking it.

### Final thoughts
Every tool used in this flow can be replaced or removed, in fact you could change the whole flow completely.
But what is important is to understand tools you're using and how you can connect them together to give your users/customers better experience, and to make you **aware of your platform's health** and **react quicker than your users**.

## The end
That's it! Let's keep your platform healthy :) If your company needs some help with Alerting & Monitoring, [get in touch]({{< ref "/contact" >}} "Get in touch").