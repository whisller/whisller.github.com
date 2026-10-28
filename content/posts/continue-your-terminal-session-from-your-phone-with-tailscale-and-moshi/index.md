+++
author = "Daniel Ancuta"
title = "Same Terminal on Your Computer and Your Phone with Tailscale and Moshi"
date = "2026-10-28"
description = "Run your coding agents on a dedicated Mac and attach to the same terminal session from your desk or your phone"
tags = ["tailscale", "moshi", "mosh", "ssh", "ai", "claude-code", "herdr", "agents"]
+++

You start a long Claude Code task at your desk, then you want to get up. Make coffee, go for a walk, sit on the couch. The agent is halfway through a refactor and about to ask you a question you'd like to answer without walking back to the computer.

If the agent lives in a terminal on the computer you're working on, you have two options: close the lid or shut it down and kill the agent, or stay put. With a laptop you can also carry it open under your arm.

## Move the terminal off the thing you carry

Stop running agents on the machine you work on. Put them on a separate machine that stays on, and treat your main computer and your phone as two windows into it. The host can be anything that runs all the time: a Mac mini under the desk, an old MacBook, a small VPS.

I use a separate Mac that stays on and plugged in. Everything long-running happens there. From my main computer I attach to it, from the phone I attach to it, and both land in the same terminal session. Start something at the desk, walk away, answer the prompt from the phone, come back and the screen is exactly where I left it.

Four pieces make this work:

- **[Tailscale](https://tailscale.com)** gives the phone and the Mac a private network, so no port forwarding and nothing exposed to the internet
- **SSH** with key-only login gets you in
- **[Moshi](https://getmoshi.app)**, over [Mosh](https://mosh.org), keeps the connection alive when the phone changes networks or goes to sleep
- **[herdr](https://herdr.dev)** is the multi-agent workspace I run inside [Ghostty](https://ghostty.org/). It runs as a server on the Mac, so the agents keep working whether or not a client is attached

On the phone I use [Moshi](https://getmoshi.app), an iOS terminal app built for this. Mosh transport is a Pro feature there, plain SSH is free.

## The host Mac

Install the Tailscale app and sign in, then turn on Remote Login in System Settings under General and Sharing. Tailscale gives the Mac a stable name like `my-mac.<tailnet>.ts.net`, and it works from anywhere your phone has signal.

A Mac that goes to sleep can't answer your phone. A Mac mini or any desktop Mac only needs sleep turned off, and a laptop also needs to stay awake with the lid closed. This command does both while the Mac is on power, so an unplugged closed laptop still sleeps:

```bash
sudo pmset -c sleep 0 disablesleep 1 womp 1 powernap 1
```

Remote Login listens on every network interface, not just Tailscale, so the only thing between your Mac and anyone on the same Wi-Fi is how you authenticate. Turn off password login. Add your key to `authorized_keys` first (next section), then:

```bash
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/100-no-passwords.conf
```

Both lines matter. macOS runs sshd with PAM, which can still accept your account password through keyboard-interactive login if only `PasswordAuthentication` is off.

Install `mosh-server`:

```bash
brew install mosh
```

## Moshi on the phone

Install Tailscale on the iPhone and sign in to the same tailnet. In Moshi, generate a key and copy the public key. Then append it on the Mac:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... moshi' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Add the server in Moshi: the Tailscale name as host, port 22, your username, and the key you just generated. If MagicDNS doesn't resolve on the phone, the `100.x.y.z` Tailscale address works too.

## Make sure Moshi really uses Mosh

Moshi's Auto transport tries Mosh first and falls back to SSH without telling you, so a connection that works isn't proof that Mosh is running. Mosh starts `mosh-server` through a non-interactive SSH command, and on macOS that command's `PATH` doesn't include Homebrew's directory. It doesn't read your `.zshrc` either.

So set two things on the server entry in Moshi:

- Transport: **Mosh**, not Auto, so a failure shows an error
- Mosh server path: the absolute path to the binary, `/opt/homebrew/bin/mosh-server` on Apple Silicon or `/usr/local/bin/mosh-server` on Intel

Reconnect, then check on the Mac:

```bash
pgrep -fl mosh-server
```

You should see a `mosh-server new ...` process. If you see only `sshd-session` processes, Moshi is on plain SSH.

With herdr running, the `mosh-server` command Moshi starts ends with `exec herdr --session 'default'`, so opening the server on the phone drops you straight into the same herdr session you use at the desk.

## Leaving the Mac unattended

The Mac stays awake and logged in with your agents attached, so anyone with physical access gets a shell. Three settings cover it.

**FileVault.** Encrypts the disk and turns off automatic login. Turn it on under Privacy & Security in System Settings, or from the terminal, where it asks for your password and prints a recovery key. Keep that key off the Mac.

```bash
sudo fdesetup enable
```

After a reboot SSH is unreachable until someone unlocks the Mac locally. For planned restarts like updates, `sudo fdesetup authrestart` reboots once without that step.

**Screen lock.** Under Lock Screen, set "Require password after screen saver begins or display is turned off" to Immediately, then let the display sleep. SSH, Mosh and your agents keep running.

```bash
sudo pmset -c displaysleep 10
```

**Find My Mac.** Turn it on in your Apple Account settings, so you can lock or erase the Mac remotely.

## Why Mosh instead of plain SSH

Plain SSH is one TCP connection, so it dies when your IP changes or iOS suspends the app. Mosh runs over UDP, so switching from Wi-Fi to cellular or locking the phone doesn't end the session. It also echoes your typing locally, and Ctrl-C works while a command floods the screen.

Mosh has no scrollback, but herdr keeps the sessions alive, so I don't need tmux.

## What you have now

At this point you have a Mac that stays awake, reachable from anywhere through your tailnet and protected by key-only SSH, with your agents running in herdr. You can sit down at your main computer, start a task, pick up your phone and land in the same session, and keep going from the couch, the garden or the train. Network changes and a locked phone don't end the session.

## The End

That's it! Hope you enjoyed reading. If you want help setting up a remote workflow like this for your team, [get in touch]({{< ref "/contact" >}} "Get in touch").

---
> **Note:** This article was written with assistance from Claude (Anthropic). The experiences, code, and opinions are my own, but AI helped structure and articulate them.
