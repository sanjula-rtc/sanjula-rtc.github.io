---
layout: post
title: "Connect Claude Desktop (Windows) to WSL Over SSH"
date: 2026-10-15
categories: tools
tags: [claude, wsl, ssh, windows, openssh]
---

Your code lives inside WSL, but Claude Desktop runs on Windows. Here's how to connect them: set up an SSH server in WSL, then add it as an SSH connection in the Code tab.

## First: do you even need SSH?

Claude Desktop has a **built-in WSL option** that needs no SSH at all. Try it first:

1. Open the **Code** tab and start a new session.
2. Open the environment picker. Your WSL 2 distributions are listed in a **WSL** section.
3. Pick one, choose a Linux folder like `/home/you/project`, and trust the folder.

That's all. It needs Windows 10/11 with WSL 2, one installed distro, and `git` inside it.

So why would you use SSH instead? According to the docs, WSL sessions don't yet support the **integrated terminal, connectors and plugins, the file browser pane, and `@` file suggestions**. SSH sessions list the terminal, connectors, plugins, MCP servers and the file editor as supported. If you need those, or you simply want the same setup as any remote server, use SSH.

```
 Route A: native WSL                    Route B: SSH

 Claude Desktop ──▶ WSL distro          Claude Desktop ──▶ ssh ──▶ sshd (in WSL)
 (nothing to set up)                    (a few minutes of setup, more features)
```

The rest of this guide is Route B.

## The big picture

```
 ┌─────────────── Windows ───────────────┐      ┌────────── WSL 2 ──────────┐
 │                                       │      │                           │
 │  Claude Desktop                       │      │   sshd  (port 2222)       │
 │   └─ SSH client ── 127.0.0.1:2222 ────┼─────▶│    └─ Claude Code engine  │
 │                                       │      │       (auto-installed)    │
 └───────────────────────────────────────┘      │    your project files     │
                                                └───────────────────────────┘
```

Claude Desktop installs Claude Code on the remote side automatically the first time you connect, so you don't install it yourself. Every file read, edit and command then runs **inside WSL**.

## Step 1: Install the SSH server in WSL

Open your WSL terminal (Ubuntu shown here):

```bash
sudo apt update
sudo apt install openssh-server
```

## Step 2: Pick a port (avoid clashes)

Port 22 can collide with Windows' own SSH server or with WSL's networking. Use **2222** to be safe. Make a small config file instead of editing the main one:

```bash
sudo tee /etc/ssh/sshd_config.d/wsl.conf > /dev/null <<'EOF'
Port 2222
ListenAddress 127.0.0.1
PasswordAuthentication no
EOF

sudo sshd -t        # prints nothing if the config is valid
```

What these lines do:

- `Port 2222`: avoids port 22 conflicts.
- `ListenAddress 127.0.0.1`: only accept connections from the same machine, so nobody on your network can reach it. In the default WSL networking mode that's the Windows host.
- `PasswordAuthentication no`: allow keys only. Set this *after* your key works, or you'll lock yourself out of SSH (see Step 4). If you want to be careful, delete that line for now and add it back at the end.

## Step 3: Start sshd (and keep it starting)

The simplest way for a first test:

```bash
sudo service ssh start
ss -ltn | grep 2222        # you should see a LISTEN line
```

To have it start automatically every time WSL boots, turn on systemd. Edit `/etc/wsl.conf`:

```ini
[boot]
systemd=true
```

Then, in **PowerShell**, restart WSL and enable the service:

```powershell
wsl --shutdown
```

```bash
# back in WSL
sudo systemctl enable --now ssh
```

(On some distros the service is called `sshd`.)

## Step 4: Create a key on Windows and authorize it

Passwords work poorly with this setup, so use a key. In **PowerShell**:

```powershell
ssh-keygen -t ed25519
```

Press Enter to accept the default path (`C:\Users\YOU\.ssh\id_ed25519`). Add a passphrase if you want one. The **private** key (`id_ed25519`) never leaves your PC. Only the **public** key (`id_ed25519.pub`) is copied.

Now copy the public key into WSL. In your **WSL terminal** (replace `YOU` with your Windows username):

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
tr -d '\r' < /mnt/c/Users/YOU/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Why `tr -d '\r'`? Windows can add a hidden carriage return at the end of lines, and SSH rejects a key line that has one. We covered that exact CRLF problem in [LF vs CRLF]({% post_url 2026-10-07-line-endings-lf-vs-crlf %}).

## Step 5: Test from a plain Windows terminal first

Before touching Claude Desktop, check that ordinary SSH works. In **PowerShell**:

```powershell
ssh -p 2222 -i $env:USERPROFILE\.ssh\id_ed25519 YOUR_WSL_USER@127.0.0.1
```

Use your **Linux** username (the one in the WSL prompt). The first time, SSH asks whether to trust the host fingerprint. Type `yes`. If you land at a Linux prompt, SSH works. **If this fails, Claude Desktop will fail too**, so fix it here first.

Now it's safe to add `PasswordAuthentication no` if you left it out earlier, then run `sudo systemctl restart ssh` (or `sudo service ssh restart`).

## Step 6: Add the connection in Claude Desktop

1. Open the **Code** tab.
2. Open the **environment dropdown** and choose **SSH > Add SSH connection…**
3. Fill in:

| Field | Value |
|---|---|
| **Name** | anything, e.g. `WSL Ubuntu` |
| **SSH host** | `YOUR_WSL_USER@127.0.0.1` |
| **SSH port** | `2222` |
| **SSH key** | `C:\Users\YOU\.ssh\id_ed25519` |

4. Save, then pick the new connection from the dropdown and choose a project folder like `/home/you/project`.

The first connection takes a little longer, because Claude Desktop installs Claude Code under `~/.claude/remote/` in WSL. Later connections reuse it.

The docs also describe using `~/.ssh/config` host aliases. Their note says that works on macOS and Linux, and doesn't say the same for a Windows client, so filling in the fields above is the safe route on Windows.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `Connection refused` | sshd isn't running. In WSL: `sudo service ssh start`. Check `ss -ltn \| grep 2222`. |
| Works for a while, then stops | WSL shut down when idle. Keep a WSL terminal open, or look up the `vmIdleTimeout` setting in the WSL docs. Also your setup may have restarted WSL, so run Step 3 again. |
| `Permission denied (publickey)` | The key line didn't land correctly. Check `~/.ssh/authorized_keys` has your public key, with permissions 700 on `~/.ssh` and 600 on the file. Look for the `\r` problem from Step 4. |
| Terminal SSH works, Claude Desktop doesn't | Re-check Host, Port and key path in the connection. Make sure the key path is the **private** key, not `.pub`. |
| Port already in use | Another program has 2222. Pick another port, change it in `wsl.conf` **and** in Claude Desktop. |
| Fails right at the start | The server needs the SFTP subsystem. Ubuntu enables it by default. Test with `sftp -P 2222 USER@127.0.0.1`. |
| Can't create PRs from a session | The remote machine needs `git` and `gh` installed and logged in. Do `gh auth login` inside WSL. |

## Good to know

- **Networking modes.** In the default NAT mode, Windows can reach WSL through `localhost`, so `127.0.0.1` normally works. Newer Windows 11 versions (22H2 and later) also offer *mirrored* mode (`networkingMode=mirrored` in `%USERPROFILE%\.wslconfig`). Mirrored mode shares ports with Windows, which is another reason to avoid port 22. If you use mirrored mode, `ListenAddress 127.0.0.1` matters even more.
- **Disconnects.** Tasks run as a process inside WSL, so they can survive a brief network loss or sleep. With a Claude account, the sign-in token only renews while Desktop is connected, so very long jobs can stop. Keep your PC awake for overnight work.
- **Dev servers.** The Browser pane can't open an app running on an SSH host. Forward the port yourself, e.g. `ssh -N -L 3000:localhost:3000 -p 2222 USER@127.0.0.1`, then open `localhost:3000` in your normal browser. (In WSL NAT mode `localhost` forwarding often makes this unnecessary.)
- **File speed.** Keep your project **inside** the WSL filesystem (`/home/you/...`), not under `/mnt/c/...`. Cross-OS file access is slow.

## The rule of thumb

> **Try the native WSL option first. Use SSH when you want the terminal, plugins and connectors.**
> Always make plain `ssh` work in a Windows terminal *before* adding it to Claude Desktop.

*Sources: the Claude Code docs for [Desktop in WSL](https://code.claude.com/docs/en/desktop-wsl) and [Desktop SSH sessions](https://code.claude.com/docs/en/desktop). I haven't tested every step on a live machine, and Claude Desktop's menus may change between versions.*
