---
layout: post
title: "Why Git Submodule Pointers Silently Drift When You Merge"
date: 2026-05-29
categories: git
tags: [git, submodules, workflow]
---

If you work on a repo with submodules, you've probably pushed a PR and had a reviewer ask: *"Why are you changing the enterprise submodule?"* — even though you never intentionally touched it. Here's exactly why it happens and how to fix it every time.

## What a submodule pointer actually is

A submodule is not a folder inside your repo — it's a single-line file that records a commit hash from another repo:

```
backend/src/main/java/com/skapp/enterprise  →  commit abc123...
```

Git tracks it like any other file (mode `160000`). That means it can be merged, conflicted, and committed just like `Validation.java` or `package.json`. Most people never notice because it doesn't show up as a normal file diff.

## Why the drift happens

When teammates push updates to develop, they sometimes advance submodule pointers (the enterprise plugin gets a new version, config changes, etc.). Your feature branch was cut from an older state, so it holds older pointers:

```
develop:        enterprise → eeeee297  (newer, teammates updated this)
your branch:    enterprise → 236e1647  (older, from when you branched)
```

When you run `git merge origin/develop` to sync your branch, git sees **two different values** for the submodule pointer — exactly like a text conflict on any file. If git auto-resolves it (or you hit "accept ours" during a conflict), it keeps your branch's older pointer.

Your branch now records a *downgrade* relative to develop. That shows up in your PR diff as a submodule change you never intended.

## Why it goes unnoticed

`git status` shows it like this:

```
modified: backend/src/main/java/com/skapp/enterprise (new commits)
```

Most developers scan for `.java` and `.ts` changes and skip past the gitlink entries. It only gets caught when a reviewer looks at the PR diff and sees a stray submodule line.

## How to fix it

**Step 1 — Find which submodules differ from develop:**

```bash
git diff origin/develop HEAD -- \
  backend/src/main/java/com/skapp/enterprise \
  frontend/pages/enterprise \
  frontend/src/enterprise
```

If you see any differences you didn't intend, proceed to step 2.

**Step 2 — Reset them to develop's versions:**

```bash
git checkout origin/develop -- \
  backend/src/main/java/com/skapp/enterprise \
  frontend/pages/enterprise \
  frontend/src/enterprise
```

This stages the correct pointer without touching any other file.

**Step 3 — Verify the staged diff looks right:**

```bash
git diff --cached -- backend/src/main/java/com/skapp/enterprise
```

You should see your old hash on the `-` line and develop's hash on the `+` line.

**Step 4 — Commit and push:**

```bash
git commit -m "fix: revert accidental submodule pointer changes"
git push
```

## How to catch it before it happens

After every `git merge origin/develop`, run a quick check:

```bash
git diff HEAD -- $(git config --file .gitmodules --get-regexp path | awk '{print $2}')
```

If any submodule paths show up and you didn't intend to bump them, reset immediately with `git checkout origin/develop -- <path>` before committing.

## The rule of thumb

> Your feature branch should never own submodule pointer changes unless updating a submodule version is literally the feature you're building.

If you see them in a PR, it's always drift from a merge — and it's always safe to reset to develop's version.
