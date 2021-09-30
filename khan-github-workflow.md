# My GitHub workflow at Khan

A few folks have asked me to write up how I'm using GitHub and/or how I want to be using it, this note discusses both!  I don't think it's right for everyone; use what you like!

## What I'm doing now

### Regular diffs

For non-stacked diffs I'm mostly following the official workflow, i.e.:

```sh
git checkout <deploy-branch> -b <review-branch>
# hack hack hack
git commit
git pull-request
# wait for review
git land
```

I write my commit message, issue, reviewers, etc. all in `git commit`, because I prefer to write a single formatted message than to answer a bunch of prompts.

### Stacked diffs

To stack another diff on top I use the usual method:

```sh
git checkout <review-branch-1> -b <review-branch-2>
# hack hack hack
git commit
git pull-request
```

Where it gets interesting is in editing earlier diffs in the stack.  I find the official workflow for this very cumbersome and do the following (supposing I have a stack of `n` branches, and want to edit the `i`th):

```
git checkout <review-branch-n>

git rebase -i <deploy-branch>  # or git rebase -i <review-branch-i>^ is fine
# choose to edit the current head of <review-branch-i>

# we're now on the head of <review-branch-i>:
# hack hack hack
git commit
git push origin HEAD:<review-branch-i>
git rebase --continue

# now find the rewritten commit for the head of each
# of the subsequent branches, and:
git branch -f <review-branch-i+1> <new-head-of-i+1>
git push -f origin <review-branch-i+1>
# (repeat)
git branch -f <review-branch-n-1> <new-head-of-i+1>
git push -f origin <review-branch-n-1>
# the rebase has already updated the head of <review-branch-n>,
# so we skip the `git branch -f` and just:
git push -f origin <review-branch-n>
```

The idea here is that instead of doing a bunch of small rebases, we do one big rebase, of everything from our change on up, and then just fix up the branches as needed.  The result is equivalent.  It's fine to edit several commits in the same rebase; in that case if you are editing both `i` and `j` you want to fix up the heads between `i` and `j` before making your changes to `j` (or, most importantly, before pushing it), but the method is the same.

For landing, we do something similar, again as one big rebase.  We always land starting from the bottom, of course, so to land the first review:

```
git checkout <review-branch-n>

git rebase -i <deploy-branch>  # or git rebase -i <review-branch-1>^ is fine
# choose to edit the current head of <review-branch-1>

# we're now on the head of <review-branch-1>:
git land review-branch-1
# we're now on <deploy-branch>, which now contains a
# squashed version of <review-branch-1>:
git checkout --detach  # `git rebase` expects a detached head

git rebase --continue
# now update the heads of each subsequent branch, as above
```

We can land several branches at once this way; if we do we just need to edit all the relevant commits in our interactive-rebase, then at each step:
```
# fix up the HEAD on github (not necessary the first time)
git branch -f <review-branch-i> HEAD
git push -f origin <review-branch-i>
# optionally, wait for actions to finish
git land <review-branch-i>
git checkout --detach
git rebase --continue
# (repeat)
```

I have an alias for `git branch -f $1 ${2:-HEAD} && git push -f origin $1` to make that part easier.  At this time, I just keep track of the heads of each branch manually; I'd like to add some tooling to do that, but it's a bit tricky to do without amending commits more than necessary.

## What I still need to figure out

### The Lawful Good way

I still haven't gotten used to having a bunch of review branches all over the place.  I think the ideal here is to update my tooling; after talking to Craig he convinced me it's possible.  A few things I'd like to change, but haven't done yet:

1. In my [`git status` wrapper](https://github.com/benjaminjkraft/dotfiles/blob/main/bin/git-s), I print, in addition to ordinary `git status`, `git log --oneline main..` and `git diff --stat`.  With a patch-stack workflow, I used the `git log` to glance at all the other things I'm working on.  It may make sense to do `git branch` instead now.
2. If `git branch` is my new "what's sitting around", I need to clean it up, so that random old WIP branches don't clutter it.  Craig suggests just having a convention for branch-names (he prefixes an underscore which sorts them at the top; I might do `benkraft.wip.` and then have my wrapper remove or resort those).
3. I actually don't just do `git log --oneline`; I have [another wrapper](https://github.com/benjaminjkraft/dotfiles/blob/main/bin/git-l) which extracts information from the commit message and appends it to the line, so I get `sha (refs) Commit message (ISSUE-1 D12345)`, and the `ISSUE-1` and `D12345` are links.  I need to adapt that to `git branch`; one tricky bit is that I want to link the PR, but it's not clear where to get that from; I can't put it in the commit message without amending (which github never loves so I try to minimize).  Many things are possible -- the most likely one is to make sure I pull down `refs/pull/XXX/head` and use that.

For stacked diffs, I'd also like to have some tooling to support fixing up all the branch heads.  I think I'd want to additionally:

1. For my new `git branch` wrapper, make it aware of the stack structure, which of course `git log --oneline` naturally is.
2. Once I figure out how to link commits to PRs for my `git branch` wrapper, also use that metadata to make a tool to fix up the subsequent branch-heads after a rebase; this is the most annoying part of the workflow currently.

In general I feel that this does generate a lot of visual noise in the stacked diffs' changelog.  I don't see a way around this with any stacked-diff workflow (certainly not for one with squash-merges at the end).  This frustrates me, but in practice it doesn't seem to be a big problem.

### The Chaotic Neutral way

The other option is to do a patch-stack workflow locally, but then keep a "shadow" worktree that I push to github.  This would require a lot of very magical tooling, and it's probably a bad idea.  The idea is: if the commits are unrelated, then they cherry-pick cleanly past one another, so we can maintain a shadow worktree of what our commits would look like if we did them the review-branch way, and syncing our actual changes to that worktree can't have merge conflicts.  I wrote more about how to do this in a [separate note](github-patch-workflow.md).

For stacked diffs, we could have a version of the same that has to do some force-pushes (but so do all the stacked-diff workflows, at least in the presence of squash-merge).  Or, we could fall back to the above workflow, which isn't the same as its patch-stack cousin, but is at least closer.

I will remind you I didn't say this was a good idea.
