---
layout: post
title: "How to Restore Only Selected Files From a Git Stash"
date: 2026-09-07
categories: git
tags: [git, stash, workflow]
---

`git stash pop` is all-or-nothing: it brings back every file the stash touched, staged and unstaged alike. Most of the time that's fine. But sometimes a stash is a grab-bag — a bit of work-in-progress mixed in with a handful of files you actually want back right now, unstaged, without disturbing anything else. Here's how to pull exactly those files out.

## The situation

Say you stashed a pile of changes on another branch weeks ago, and it contains both some in-progress server code *and* a set of API docs you now want back on your current branch — nothing else.

```
$ git stash list
stash@{0}: On develop-temp-for-ui: admin-changes
stash@{1}: On feature/x: wip before rebase
```

`git stash show -p --stat stash@{0}` only shows **tracked** files that were modified. If the stash also contains brand-new (untracked) files — like a whole new docs folder — they won't show up there by default. You need `-u`:

```bash
git stash show -p -u --stat stash@{0}
```

That's the key detail: a stash is not one blob, it's up to three commits glued together — one for the index, one for the working tree, and (only if you stashed with `-u`) a third one purely for untracked files. Knowing which parent holds what determines which command actually works.

## Finding the files you want

Grep the stat output for whatever identifies the files you're after:

```bash
git stash show -p -u --stat stash@{0} | grep -i bruno
```

```
bruno/collection-agency/agency/acknowledge.bru     |  22 ++
bruno/collection-agency/agency/dispute.bru         |  27 ++
...
docs/collection-agency-testing-bruno-compass.md    | 392 +++++++++++++++++++
```

That confirms exactly which paths you need, and nothing else.

## Restoring just those paths

The first instinct is:

```bash
git checkout stash@{0} -- bruno/collection-agency docs/collection-agency-testing-bruno-compass.md
```

If the files are new (untracked in the working tree when you stashed them), this fails:

```
error: pathspec 'bruno/collection-agency' did not match any file(s) known to file
```

That's because `stash@{0}` by itself only resolves to the working-tree-changes commit — untracked files live on a *third* parent, `stash@{0}^3`. Point at that instead:

```bash
git checkout stash@{0}^3 -- bruno/collection-agency docs/collection-agency-testing-bruno-compass.md
```

(If the files you want *are* tracked, modified files, `stash@{0}` — or explicitly `stash@{0}^2` for the working-tree commit — works directly, no `^3` needed.)

## Keeping it unstaged

`git checkout <commit> -- <path>` always stages what it restores. If you want it sitting as an unstaged, uncommitted change instead — say, because you want to review it before adding anything — just unstage it afterward:

```bash
git reset -- bruno/collection-agency docs/collection-agency-testing-bruno-compass.md
```

End result:

```
$ git status --short
?? bruno/
?? docs/collection-agency-testing-bruno-compass.md
```

Exactly the files you wanted, nothing staged, nothing committed, and the original stash entry still sitting untouched at `stash@{0}` in case you need anything else out of it later.

## The pattern, condensed

```bash
# 1. See what's in the stash, tracked and untracked
git stash show -p -u --stat stash@{0}

# 2. Pull only the paths you need
#    - tracked/modified files:   stash@{0}  (or stash@{0}^2)
#    - untracked/new files:      stash@{0}^3
git checkout stash@{0}^3 -- <path> [<path> ...]

# 3. Unstage if you want it left as a working-tree change
git reset -- <path> [<path> ...]
```

No `pop`, no `apply`, no risk to the rest of what's sitting in the stash — just the files you actually asked for.
