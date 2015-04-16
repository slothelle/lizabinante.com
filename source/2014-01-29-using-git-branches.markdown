---
title:  "Using Git repositories alongside tutorials"
date:   2014-01-29 20:17:39
category: tutorial
tags: tutorial, git, basics
---

Every tutorial has a corresponding Github repo that contains branches for different sections to make it easier to follow along. At the beginning of every post, there will be a link to the repository.

You can either:

* View the branches on Github, or
* Clone the repo locally

No matter what you decide to do, each section of the tutorial will tell you the name of the branch that contains the code being explained.

## Viewing branches locally

If you choose to clone the repo locally, you can either pull all of the branches down at once, or pull them down one at a time.

### To grab all branches at once

Run `git fetch --all` to set up local branches without remotes.

If you want remotes (you don't really need them unless you're trying to push stuff up to my repo, which you can't): `git pull --all`.

### To grab select branches

Clone the repo like normal, and then run `git branch -a` to see all remote branches.

To checkout a branch from that list without creating a local branch, run:

`git checkout origin/branch_name`

To checkout a branch from that list *and* create a local branch, run:

`git checkout -b origin/branch_name`
