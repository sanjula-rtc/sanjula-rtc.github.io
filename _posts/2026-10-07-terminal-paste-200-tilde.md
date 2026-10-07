---
layout: post
title: "Why Ctrl+V Pastes ^[[200~ in Your Terminal (and Ctrl+Shift+V Doesn't)"
date: 2026-10-07
categories: terminal
tags: [terminal, shell, bracketed-paste, readline, linux]
---

You copy a command, press **Ctrl+V** in the terminal, and instead of your text you get this:

```
$ ^[[200~git status^[[201~
```

Press **Ctrl+Shift+V** and everything is fine. Same clipboard, different result. Here's what's going on, in plain words.

## First: what is `^[[200~`?

It's not corrupted text. It's a **signal** the terminal wraps around whatever you paste. This feature is called **bracketed paste**.

```
 what you copied:      git status

 what the terminal sends to the program:

   ESC [ 200 ~   git status   ESC [ 201 ~
   └── "paste starts" ┘          └── "paste ends" ┘
```

- `ESC` is an invisible control character (code 27). Terminals show it as `^[`.
- So `^[[200~` is really `ESC [ 200 ~`: "a paste is starting now".
- `^[[201~` means "the paste is done".

A program that understands these markers (bash, zsh, vim…) swallows them silently. A program that **doesn't** understand them prints them on screen. That's what you saw.

## Why does this feature exist? A little history

In the early days, a terminal had no idea what "paste" was. It only knew **keystrokes**. Pasting was just the terminal typing your text very fast, as if you had a very quick pair of hands.

That caused real problems:

```
 You paste:                      The shell sees:

 echo hello                      echo hello⏎      ← Enter pressed! runs immediately
 rm -rf ./build                  rm -rf ./build⏎  ← runs too, before you could read it
```

- **Newlines ran commands instantly**, with no chance to review. Copy a command from a web page with a hidden extra line and you could be running something you never saw (an attack known as *paste-jacking*).
- **Tabs triggered auto-complete**, mangling pasted code.
- **Editors like vim auto-indented** every line, so pasted code staircased to the right.

The fix: let the terminal **tell the program** "this chunk is a paste, not typing". The terminal wraps the paste in the `ESC[200~ ... ESC[201~` markers, and the program decides what to do. For example, bash now inserts the whole thing on the command line and waits for **you** to press Enter. Terminals like xterm introduced this, and others copied it. Modern shells use it by default (bash turned it on by default in version 5.1).

The program asks for it by sending a switch-on code, and turns it off the same way:

```
 program  ──  ESC[?2004h  ──▶  terminal   "please wrap my pastes"
 program  ──  ESC[?2004l  ──▶  terminal   "stop wrapping"
```

## So why does Ctrl+V break it?

**Ctrl+V and Ctrl+Shift+V are handled by two different parts of the system.**

```
 Ctrl+Shift+V                       Ctrl+V (in many terminals)

 keyboard                           keyboard
    │                                  │
    ▼                                  ▼
 TERMINAL APP handles it            TERMINAL passes it straight
 "that's paste!"                    to the program as a keystroke
    │                                  │
    ▼                                  ▼
 sends wrapped text                 program gets the key 0x16
 ESC[200~ text ESC[201~             (= "quote next character")
    │                                  │
    ▼                                  ▼
 shell: "bracketed paste, got it" ✔  shell: "next char is literal!" ✘
                                    (the ESC gets printed as ^[ )
```

Here's the old, historical part. **Ctrl+V already had a job in Unix long before "paste" existed.** In a terminal it means *"insert the next character literally"* (called `lnext` in the tty driver and `quoted-insert` in readline). It's how you type a real Tab or a real Escape into a command line.

Ctrl+C couldn't be "copy" either, because Ctrl+C has meant *"stop this program"* (SIGINT) since the 1970s. So terminal makers chose **Ctrl+Shift+C / Ctrl+Shift+V** for copy and paste: the Shift keeps them from colliding with these old shortcuts.

What happens when you press Ctrl+V on such a terminal:

1. The terminal does **not** paste. It sends `Ctrl+V` to the shell.
2. The shell says "OK, the next character is literal".
3. The terminal's wrapped paste then arrives, starting with `ESC`.
4. The shell prints that `ESC` literally instead of treating it as a marker. You see `^[[200~`.

(Terminals that bind Ctrl+V to paste, like Windows Terminal, avoid step 1. If you see this there, it's usually the next cause.)

## The other cause: a stale "paste mode"

Sometimes you do use the right shortcut and still see `^[[200~`. This usually means a program switched bracketed paste **on** and then died before switching it **off**. Common culprits: a crashed `ssh` session, a killed `vim`, or a Docker/remote shell that doesn't understand the markers. The terminal keeps wrapping, but nobody is listening.

## How to fix it

**Right now**, switch the mode off in this terminal:

```bash
printf '\e[?2004l'
```

Or reset the whole terminal state:

```bash
reset
```

**For good**:

- Use **Ctrl+Shift+V** (or right-click → Paste) in terminals where Ctrl+V isn't paste.
- Check your terminal's settings and bind Ctrl+V to paste if it offers that.
- If a remote or old shell can't handle it, you can turn the feature off for readline-based shells in `~/.inputrc`:

```
set enable-bracketed-paste off
```

(That turns off the safety feature, so only do this if you really need to.)

## The rule of thumb

> **`^[[200~` = "a paste starts here."** If you can see it, the program you're typing into didn't understand the message, or Ctrl+V sent it the wrong signal.
> **Ctrl+Shift+V asks the terminal to paste. Ctrl+V often asks the shell to quote a character.**
