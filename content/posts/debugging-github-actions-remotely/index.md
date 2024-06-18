---
author: "Daniel Ancuta"
title: "Debugging GitHub Actions Remotely"
date: "2023-05-01"
description: "Debugging GitHub Actions remotely"
tags: ["github", "tools", "github actions", "ci/cd", "devops"]
---

If you use GitHub to store your repositories, you may also be utilizing GitHub Actions. I've been using it for several years across multiple projects. There have been a few instances where I had to debug a failing workflow.

So, what do you do when everything works as expected on your machine but fails on a remote one? Well, you might be tempted to don your "Works on my side" hat and move along... ;)

Or you could debug it on the remote machine. That's how I discovered [lhotari/action-upterm](https://github.com/lhotari/action-upterm), an action that allows you to connect through [upterm](https://upterm.dev/) to your GitHub Actions runner remotely.

It's as easy as adding this snippet just before the step you want to start debugging from:
```yaml
- name: Setup upterm session
  uses: lhotari/action-upterm@v1
```

You can also limit access to specified GitHub users, SSH keys, as well as use your own upterm instance.

After successfully initializing a reverse SSH tunnel with upterm, you will receive an SSH address that you can connect to from your local machine.

Enjoy your debugging!
