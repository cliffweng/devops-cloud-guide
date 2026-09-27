---
title: "01. Linux/SSH & the day-1 box"
layout: default
nav_order: 2
---

# Linux/SSH & the day-1 box
{: .no_toc }

*~8 min read*

**Interview occasional**

## Why it matters

Every side project and internship eventually puts you on a remote box with no GUI — a `t3.micro` you spun up, a lab server, a Raspberry Pi. If you can't navigate the filesystem, read a log, check what's using memory, and get in over SSH without help, you can't do anything else in this guide. This topic is the floor everything else stands on.

## Core concepts

- **The filesystem is the API.** Config lives in `/etc`, logs in `/var/log`, your app probably runs from `/home/<user>` or `/opt`. `ls`, `cd`, `cat`, `less`, and `find`/`grep` are how you explore a box you've never seen before.
- **Processes and resources.** `ps aux`, `top`/`htop`, `df -h` (disk), `free -h` (memory) answer "what's running and what's this box's problem right now" — the first thing you check when something's slow or crashed.
- **Permissions are the security model.** `chmod`/`chown` and the `rwx` bits for owner/group/other decide who can read, write, or execute a file. Getting `sudo` wrong (or overusing it) is a top source of "why doesn't this work" for beginners.
- **SSH is public-key auth over an encrypted channel**, not a password prompt with extra steps. You generate a keypair (`ssh-keygen`), put the **public** key on the server (`~/.ssh/authorized_keys`), and keep the **private** key only on your machine — never copy or share it. The server proves you hold the private key without you ever sending it.
- **`~/.ssh/config` is how you stop retyping flags.** A `Host` block with `HostName`, `User`, and `IdentityFile` turns `ssh -i ~/.ssh/mykey.pem ubuntu@1.2.3.4` into `ssh myserver`. This matters the moment you're juggling more than one box.
- **`scp`/`rsync` move files over the same SSH connection** — no separate FTP setup needed. `rsync -avz` is the one worth memorizing since it only transfers what changed.
- **Environment variables and shell profiles** (`.bashrc`/`.zshrc`, `export PATH=...`) control what commands are found and what config an app reads at startup — a common source of "works on my machine, not on the server" confusion.
- **Background processes survive your SSH session ending** only if you detach them (`nohup`, `screen`, `tmux`, or — properly — a systemd service). Closing your terminal on a plain foreground process kills it; this trips up almost everyone the first time they deploy something by hand.

## Mental model

```
you (laptop)                         remote box
   |-- ssh-keygen -----> private key (stays here, never leaves)
   |                      public key ------> ~/.ssh/authorized_keys
   |
   |-- ssh myserver ---------------------->  proves you hold the private key
   |                                          drops you into a shell
   |
   |   ls / cat / grep --------------------> explore filesystem (/etc, /var/log)
   |   ps aux / top / df -h ---------------> what's running, what's full
   |   tmux / systemd ---------------------> keep a process alive after you disconnect
```

Think of the box as a room you're given a key to, not a service you call — everything you do (files, processes, permissions) is local state on that one machine until you set up something (like CI, in [CI with GitHub Actions](../03-ci-github-actions/)) to manage it for you.

## Interview questions

1. **Walk me through what happens when you run `ssh user@host`.**
   Answer: Your client and the server negotiate an encrypted channel, then the server checks whether your public key (already stored in its `authorized_keys`) matches a private key you can prove you hold — it does this via a challenge your client signs with the private key, without the private key ever being transmitted. If that verification succeeds, you get a shell as `user`.

2. **Why is it bad practice to SSH in as root and just work from there?**
   Answer: Root can do anything, including irreversibly breaking the system with a typo (`rm -rf` in the wrong directory) — running as a limited user and using `sudo` only for specific commands limits the blast radius of a mistake and creates an audit trail of what actually needed elevated privileges.

3. **You start a long-running script over SSH, then your laptop goes to sleep and the connection drops. What happens to the script, and how would you prevent it?**
   Answer: By default the script is a child of your shell session, so it receives `SIGHUP` and dies when the SSH connection drops. To prevent that, run it under `tmux`/`screen` (so the session persists independently of your connection) or `nohup command &` (so it ignores the hangup signal), or — for anything that should really survive reboots — set it up as a systemd service instead of a manual foreground command.

4. **A teammate says "the app works locally but not on the server." What are the first three things you'd check?**
   Answer: Environment variables/config differences (`.env` files, `PATH`, missing secrets), whether the process is actually running and listening on the expected port (`ps aux`, `ss -tlnp`), and resource limits (`df -h` for disk, `free -h` for memory) — a shockingly large fraction of "it works locally" bugs are one of these three.

5. **What's the difference between a public key and a private key in SSH auth, and what's the actual security failure mode if you mix them up?**
   Answer: The private key is the secret that proves your identity and must never leave your machine or be shared; the public key is safe to distribute and goes on every server you want to access, added to `authorized_keys`. If you ever copy your private key onto a server or share it, anyone with access to that key can now impersonate you on every system where the matching public key is trusted — so treat a leaked private key the same as a leaked password and rotate it immediately.

## Watch

- [How SSH Really Works](https://www.youtube.com/watch?v=rlMfRa7vfO8) — ByteByteGo. A concise breakdown of the SSH handshake and key-based auth.
- [60 Linux Commands you NEED to know (in 10 minutes)](https://www.youtube.com/watch?v=gd7BXuUQ91w) — NetworkChuck. Fast tour of the filesystem, process, and permission commands you'll use daily.

## Further reading

- [OpenSSH: Key management](https://www.openssh.com/manual.html) — official OpenSSH documentation.
- [DigitalOcean: SSH Essentials](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys) — practical walkthrough of keys, config files, and agent forwarding.
