---
layout: post
title: "LF vs CRLF: How Linux and Windows End a Line"
date: 2026-10-07
categories: fundamentals
tags: [line-endings, lf, crlf, windows, linux, git]
---

Open a text file and it looks like lines stacked on top of each other. But a file is just a long row of bytes. There is no "line" in there. Something has to mark **where one line ends**, and Linux and Windows chose different marks.

## The short answer

| OS | Line ending | Characters | Bytes (hex) |
|---|---|---|---|
| **Linux / macOS** | **LF** (Line Feed) | `\n` | `0A` |
| **Windows** | **CRLF** (Carriage Return + Line Feed) | `\r\n` | `0D 0A` |
| Classic Mac OS (before 2001) | CR | `\r` | `0D` |

So your memory is right: Linux uses `\n`, Windows uses `\r\n`. The "r" is **Carriage Return**.

I checked it on a real machine by writing the same word both ways and dumping the bytes:

```
$ printf 'hi\n'   | od -An -tx1c
  68  69  0a
   h   i  \n

$ printf 'hi\r\n' | od -An -tx1c
  68  69  0d  0a
   h   i  \r  \n
```

Same text. Windows style has **one extra byte** on every line.

## Picture: what the file really looks like

```
 What you see:            What's stored (Linux)        What's stored (Windows)

 Hello                    H e l l o ⏎                  H e l l o ␍ ⏎
 World                    W o r l d ⏎                  W o r l d ␍ ⏎

                          ⏎ = 0A (LF)                  ␍ = 0D (CR),  ⏎ = 0A (LF)
```

## Why two different answers? History.

The names come from **typewriters and teletype machines**, which did two separate physical jobs at the end of a line:

```
 ┌──────────────────────────────┐
 │ Hello World▮                 │   print head is at the right edge
 └──────────────────────────────┘

 1. CARRIAGE RETURN (CR)  ─  slide the print head back to the LEFT edge
 2. LINE FEED (LF)        ─  roll the paper UP one line
```

- **CR** = "go back to the start of the line."
- **LF** = "move down one line."

Mechanical machines needed both, and moving the heavy carriage back took time, so the usual order was CR first and then LF, which gave the head a moment to get there.

When computers arrived, storage and bandwidth were expensive, and designers split into camps:

- **Unix** (and so Linux) decided one character was enough: **LF alone**. The terminal driver can add the carriage return on screen if it needs to.
- **CP/M**, and then **MS-DOS and Windows**, kept the old teletype behaviour: **CR + LF**, both stored.
- **Old Mac OS** picked **CR alone**. (Modern macOS is Unix-based, so it uses LF.)

Nobody was "wrong." They were different choices that stuck, and now we all live with the mismatch.

## Why it matters: what goes wrong

Mixing the two usually fails quietly, which is exactly why it's annoying.

**1. Scripts that refuse to run on Linux**

Write a shell script on Windows and copy it over:

```
$ ./deploy.sh
bash: ./deploy.sh: /bin/bash^M: bad interpreter: No such file or directory
```

Linux read the first line as `#!/bin/bash` **plus a stray CR** (`^M`). There is no program called `bash\r`, so it fails.

**2. Invisible characters in your data**

Read a Windows file on Linux and every line ends with a hidden `\r`:

```
 "alice\r"  ≠  "alice"      ← string comparisons fail, login/lookup "mysteriously" breaks
```

**3. Giant, noisy git diffs**

If one developer saves with LF and another with CRLF, git sees **every line changed**, even though no visible text did. Reviews become unreadable.

**4. Files that look squashed**

Some old Windows tools (older Notepad versions) didn't understand a bare LF. A Linux file opened there showed as **one long line**. Newer Windows versions handle it fine, but the habit of hitting this problem remains.

## Where each format is *required*

Not everything follows your OS. Many network protocols demand CRLF regardless of where you run:

- **HTTP** headers end each line with `\r\n`.
- **SMTP** (email) and many other internet text protocols do too.
- **CSV** files, per [RFC 4180](https://datatracker.ietf.org/doc/html/rfc4180), use CRLF between records (though lots of tools accept LF).

So a program that builds an HTTP request by hand must write `\r\n` even on Linux.

## Reading and writing: how languages cope

Most languages try to hide the difference, but it helps to know they do:

- **C on Windows, text mode:** writing `\n` is translated to `\r\n` on disk, and reading `\r\n` becomes `\n`. In binary mode, nothing is translated.
- **Python:** `open()` in text mode uses "universal newlines" and reads `\r\n`, `\n` and `\r` all as `\n`. When writing, it uses the OS default unless you pass `newline=''` or `newline='\n'`.
- **Java:** use `System.lineSeparator()` instead of hard-coding `"\n"`.
- **Byte-level tools** (`cat`, `diff`, checksums) see the raw bytes, so the `\r` shows up as a real difference.

The safe habit: **be explicit when the format matters** (protocols, scripts, data files), and **let the language handle it** when you're only writing for people.

## How to see and fix line endings

See them:

```bash
file deploy.sh            # "ASCII text, with CRLF line terminators"
cat -A deploy.sh          # CRLF lines end with ^M$ ; LF lines end with just $
```

Convert them:

```bash
dos2unix deploy.sh        # CRLF -> LF
unix2dos notes.txt        # LF -> CRLF
sed -i 's/\r$//' file     # strip CR without extra tools
```

## Keep a team sane with git

Tell git what you want so nobody's editor decides for you. Add a `.gitattributes` file to the repo:

```
* text=auto eol=lf
*.bat text eol=crlf
*.sh  text eol=lf
```

- Files are stored with **LF** in the repo.
- Windows batch files keep **CRLF**, because `cmd.exe` expects it.
- Shell scripts are always **LF**, so they run everywhere.

On your own machine, `git config core.autocrlf` is the older per-user setting that converts on checkout and commit. A `.gitattributes` file is better because it travels with the repo and applies to everyone.

Most editors (VS Code, for instance) show `LF` or `CRLF` in the status bar. Click it to switch.

## The rule of thumb

> **Linux: `\n` (LF). Windows: `\r\n` (CRLF).** The CR comes from typewriter carriages, the LF from rolling the paper up.
> When files cross between systems, or you write a protocol or script, **decide the line ending on purpose** instead of letting your OS pick.
