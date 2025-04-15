+++
author = "Daniel Ancuta"
title = "Serving static pages on AWS with Amplify"
date = "2025-04-15"
description = "When you think about creating an easy page, the first thing that comes to mind is WordPress - but there's an easier way. Check out the combination of Amplify + Hugo for a cost-efficient solution."
tags = ["aws", "amplify", "static page", "hugo", "hugo go", "cost efficiency", "cost", "github static pages"]
+++

After creating branding for your static website - layout, logo, typography, content, and so on - serving it should be easy, right?

But often, it's overcomplicated. You think about a custom solution for the engine, look at WordPress, and realize it's overkill for what you need.

If that's the case, I have two solutions you might want to consider:
1. [GitHub Pages](https://pages.github.com/) + [Hugo](https://gohugo.io/) + [Cloudflare](https://www.cloudflare.com)
2. [AWS Amplify](https://aws.amazon.com/amplify/) + [Hugo](https://gohugo.io/)

Both are great setups that will allow you to serve your static page (blog, simple company website, portfolio...) at little to no cost.

Let's break them down!

## Hugo

In both options, I mentioned [Hugo](https://gohugo.io/).  
Hugo is a really great static site generator that fits most requirements for a static page.  
Even better - it's supported by both GitHub Pages and Amplify. 🙂

## GitHub Pages + Hugo

This great duo allows you to serve your site with no hosting cost at all. All you really need is a GitHub account.

The page is built using GitHub Actions. After a successful build, it will be available at `{your-nickname}.github.io`.

You might ask, "Well, that's great, but what about my domain?" - don't worry!  
GitHub Pages supports [custom domains](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site).

A great example is this very website you're reading: [ancuta.org](https://ancuta.org) - built with GitHub Pages + Hugo.

If you want to add an SSL certificate, check out [Cloudflare](https://www.cloudflare.com), which offers a free SSL option.

Look at source code of this website [whisller/whisller.github.com](https://github.com/whisller/whisller.github.com)

## AWS Amplify + Hugo

[AWS Amplify](https://aws.amazon.com/amplify/) is a great tool when building web/mobile apps that connect to a backend.  
I mentioned it in [another article]({{< relref "posts/extending-amplify-on-example-modify-resolvers/index.md" >}}).

But even for simpler setups, Amplify works really well with static site generators like Hugo!

The setup is very simple:
1. Use an [AWS Route 53 Hosted Zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html) to handle your domain.
2. [Grant access](https://docs.aws.amazon.com/amplify/latest/userguide/setting-up-GitHub-access.html) for Amplify to your GitHub repository.
3. [Configure amplify.yml](https://gohugo.io/host-and-deploy/host-on-aws-amplify/)
4. Push your static page to GitHub. 🎉

That’s it! In most cases, you’ll only pay for the hosted zone - about **$0.50/month**.

No complicated setup needed. You can get the whole thing done in an evening.

## Final Thoughts

As you can see, setting up your blog or a simple static site is easier than ever.  
No more excuses - start your blog today! :)
