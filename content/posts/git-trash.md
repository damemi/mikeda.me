---
title: "\"git trash\""
date: 2019-03-05T19:18:20
draft: false
slug: git-trash
categories:
  - Uncategorized
tags:
  - git
wordpress_id: 180
---

I often end up doing test bumps of dependencies that leave my repo
with a ton of trash changes that I just want to be able to undo easily
and reset my repo including any new or deleted files and directories,
so I added this to my .gitconfig :

```
[alias]
  trash = !git stash && git clean -fd
```

This lets me run git trash to quickly stash all my unstaged changes
*and* delete all my untracked files at once, which I couldn't find any
other solution for.
