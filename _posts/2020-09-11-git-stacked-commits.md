---
layout: post
title:  "Git: Stacked Commits"
date:   2020-09-10 10:26:23 -0800
categories: git
---

A few years ago, [Keith Smiley](https://twitter.com/smileykeith) and I just had
read [Stacked Diffs Versus Pull
Requests](https://jg.gg/2018/09/29/stacked-diffs-versus-pull-requests/) by
Jackson Gabbard. It's a good write up. This got us discussing the shortcomings
in git and GitHub, and brainstorming what a better workflow might look like.

In the next couple weeks we messed around with some scripts I wrote and quickly
found an approach that worked quite well for us, that we would use regularly
from then on. This blog post gives some details, provides some starter scripts,
and covers the weaknesses in the workflow.

### tl;dr

1. All commits are done on `main` (aka `master`) - see below for some benefits
2. Everything is done through regular old commits, branches are a necessary implementation detail
3. Branch creation and updates are automatically managed
4. Simple interface, `git newpr` and `git updatepr`

The motivating goal is to not have work spread out across many branches. Branch
switches are context switches, for you and your tools (see: slow build times).
Some people use multiple repos/worktrees to support multiple branches, which is
a not exactly lightweight. Instead, this workflow puts all work in progress on
your main branch.

### Benefits

1. Local integration, ensures all development work builds together, tests together
2. Less build system thrash, which is time saved (unless you have a [build cache](https://bazel.build/))
3. Reduced cognitive overhead of naming, managing, and remembering branches
4. Rebase all work in one shot

There's more to say about this, but the goal of this writing is not to sell the
idea of stacked diffs. I'm hoping you're reading because you already want a
stacked diff style workflow for git/GitHub. Again, [Stacked Diffs Versus Pull
Requests](https://jg.gg/2018/09/29/stacked-diffs-versus-pull-requests/) by
Jackson Gabbard is a good read on the subject. This workflow is common at some
big tech companies, many engineers know and want this workflow, but doing it
with git/GitHub requires some supplementary tools, and that's what this post is
for.

### Step 1: Creating Pull Requests from a Commit

The mismatch between GitHub and a stacked commit workflow is that GitHub Pull
Requests require a branch. If all your working commits are on your `main`
branch, then you can't use that branch to create a PR. What's needed here is a
tool to take a single commit, and make a PR out of that.

Let's call this tool `git newpr`, and here is a basic implementation of it:

```sh
#!/bin/bash

set -euo pipefail

readonly pr_commit="${1:-main}"

# Autogenerate a branch name based on the commit subject.
readonly branch_name="$(git show --no-patch --format="%f" "$pr_commit")"

# Create the new branch and switch to it.
git branch --no-track "$branch_name" origin/main
git switch "$branch_name"

# Cherry pick the desired commit.
if ! git cherry-pick "$pr_commit"; then
    git cherry-pick --abort
    git switch main
    exit 1
fi

# Create a new remote branch by the same name.
git -c push.default=current push

# Use GitHub's cli to create the PR from the branch.
# See: https://github.com/cli/cli
gh pr create

# Go back to main branch.
git switch main
```

With this implementation, a PR for the latest commit on `main` can be created by
running `git newpr`. To create a PR from some other commit, run `git newpr <sha>`.

In words, here's the process of what's happening:

1. The new branch name is generated from the commit's subject line (`%f`)
2. Create the new branch, then switch to it
3. Cherrypick the desired commit onto the new branch
4. Push the new branch to the remote
5. Create the actual PR using GitHub's [cli](https://github.com/cli/cli)
6. Return back to the main branch

Please note this doesn't handle every case. It's optimized to be simple starting
point, and for easier reading. This implementation leaves out some nice
features:

* creating branches from a commit range instead of a single commit
* stashing before and after
* handling cherry-picks that can't be performed cleanly
* etc

### Part 2: Updating a Pull Requests from a Commit

Of course, most Pull Requests have one or more updates. Sometimes CI finds
issues, reviewers requests changes, or we see that we made a boneheaded mistake.
The stacked commits workflow wouldn't be all that useful if there wasn't a
convenient way to update PRs too. As mentioned above, a stacked commit workflow
is centered on commits not branches, so what we need is a tool that can take a
commit, and add it to a PR.

Let's call this tool `git updatepr`. Here is a basic implementation of it:

```sh
#!/bin/bash

set -euo pipefail

if [[ $# -ne 1 ]]; then
    echo "usage: $0 <pr-commit>" 2>&1
    exit
fi

readonly pr_commit=$1

readonly branch_name="$(git show --no-patch --format="%f" "$pr_commit")"

git switch "$branch_name"

# Cherrypick the latest commit to the PR branch.
if ! git cherry-pick main; then
    git cherry-pick --abort
    git switch main
    exit 1
fi

# Push the updated branch.
git push

# Go back to main.
git switch main

# This allows for scripted (non-interactive) use of interactive rebase.
export GIT_SEQUENCE_EDITOR=/usr/bin/true

# In two steps, squash the latest commit into its PR commit.
# 1. Mark the commit as a fixup
git commit --amend --fixup="$pr_commit"
# 2. Use the autosquash feature of interactive rebase to perform the squash.
git rebase --interactive --autosquash "${pr_commit}^"
```

(You're probably wondering what these wild commands are, they'll be explained shortly.)

To reduce the complexity of this example implementation (between bash and the
wtf commands at the end, this short script is already somewhat complex), this
version has a restriction. The restriction only the latest commit, `HEAD` on
`main` can be treated as an update to a PR. Even with this limitation, both GUI
tools and `git rebase -i` will allow you to move the commit you want to use as
update to the top of the branch.

To use this script, run `git updatepr <pr_sha>`. This will update a PR
previously created using `git newpr`, using the latest commit as the update.

#### What's Really Going ON

Ok, this needs more explaining. While `git newpr` is hopefully fairly
straightforward, `git updatepr` is not.

The core idea is this, in this stacked commits workflow PRs live two parallel
lives:

1. As an autogenerated side-branch
2. As a squashed commit on `main`

On `main`, you see each PR as an atomic whole, a single (squashed) commit. This
is convenient, and matches the convention of stacked diff workflows.

Conversely, the branch created for you by `git newpr` does not use squashing.
Each commit update is 'picked over to the branch. This way, the branches
preserve full history. This way is reviewer friendly, updates to the PR can be
seen through each of its commits.

Now that the mental model has been described, here's how it's implemented. The
initial steps should require little explanation:

1. Change to the PR branch
2. Cherry-pick the update commit
3. Push the now updated branch
4. Switch back to the `main` branch

Now this is where it gets interesting. To squash the update commit into the PR
commit, two jargon-heavy commands are run:

1. `git commit --amend --fixup="$pr_commit"`
2. `GIT_SEQUENCE_EDITOR=true git rebase --interactive --autosquash "${pr_commit}^"`

To understand this, let's step through an example. Let's assume we have the
following local commits on `main`:

```
* 4a1fc694f1e (HEAD -> main) update for PR1
* 24f2ef346fb [OtherFeature] This is PR2
* dea5dc6818b [SomeFeature] This is PR1
```

From this state, let's run `git commit --amend --fixup=dea5dc6818b`. Now the
latest commit has had its commit message changed:

```
* 4a1fc694f1e (HEAD -> main) fixup! [SomeFeature] This is PR1
* 24f2ef346fb [OtherFeature] This is PR2
* dea5dc6818b [SomeFeature] This is PR1
```

At this point, by running `git rebase -i --autosquash origin/main` an EDITOR
would open and show us the following contents:

```
pick dea5dc6818b [SomeFeature] This is PR1
fixup 4a1fc694f1e fixup! [SomeFeature] This is PR1
pick 24f2ef346fb [OtherFeature] This is PR2
```

Notice that the order has changed, the fixup for PR1 has been positioned just
after the commit for PR1. And, it's marked as `fixup`, which causes it to be
squashed into PR1.

To state the obvious, interactive rebase typically involves actual user
interaction. We the user reorder commits, etc. However squashing can be arranged
via non-interactive scripting using the two commands, `commit --fixup` first,
followed by `rebase --autosquash`. This is where `GIT_SEQUENCE_EDITOR=true`
comes in. Git reads this environment variable to determine which editor will be
used for interactive rebasing. Since no interactive editing is needed,
`/usr/bin/true` acts as an editing no-op.

### Epilog: How to Use, How Not to Use

First, to use these scripts, save them as `git-newpr` and `git-updatepr`
respectively. Save them somewhere in your `$PATH`. This allows them to be called
via `git` (ex `git newpr`).

Secondly, as mentioned these scripts could be improved in a few ways. There are
common workflows that they could be expanded to cover, and there are less common
workflows that you might want to customize for.

Finally, there's an elephant in the room that I haven't mentioned. In this
article I've written "stacked commits" instead of "stacked PRs", and that's
because GitHub doesn't provide the features required to really do stacked PRs,
not in the way that Phabricator allows. Hopefully GitHub improves functionality
for dependent PRs. The best way to use this workflow and these tools is to avoid
PR depdenencies where possible, and otherwise manage order dependencies
manually. In truth, this workflow is more like a "pile of PRs" than a stack.
