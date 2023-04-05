# hot take on how to do a patch workflow on github

Disclaimer: I haven't tested this at all, I'm just quick jotting it down.

The question here is: how do you make it feel like locally you're acting on a
patch-stack, but upload things to github in a reasonable fashion?

It seems the way most of the tools for this workflow try to handle this is to
provide tooling such that you can operate on a chain of branches like a
patch-stack, namely updating all the refs in github whenever you make a change
to any commit, and allowing you do locally do what is essentially a branch-wise
rebase like you would with a patch stack.  My hot take is: what if we actually
just keep a patch-stack locally?

Now, the problem with the naive approach is the same as the problem with using
a patch-stack in github when you don't need one: github doesn't handle rebases
well.  It especially doesn't handle *contentful* rebases well; one proposed
approach has been to separate contenful and contentless rebases but this
doesn't totally solve the problem on the reviewer side.

The key insight is that we can handle two cases separately separately:
1. The commits rebase cleanly past one another, and stacking is just a
   convenience for branch management.  In this case, our tool is free to do
   exactly that!
2. The commits in fact must be stacked because they depend on one other.  In
   this case, we're already stuck with rebases; the best we can do is to split
   up contentful and contentless rebases.

So the idea is going to be, we keep a patch-stack locally, and then (perhaps in
a separate worktree) we do all the cherry-picking.  Let's look at an example:
suppose we have reviews, A, B, and C, stacked in that order and based on
`main`, which are currently at commits A1, B1, and C1, and which don't depend
on each other.  To upload them to github, the tool will do:
```sh
git co main -b patch-A
git cherry-pick A1
git push origin patch-A
gh pr patch-A  # or `git pull-request patch-A` for OLC users

# repeat for B and C
```
The cherry-picks are by construction clean, because the commits have no
dependencies.

Now, suppose you make some changes, such that the three are at commits A2, B2,
and C2; for now assume that the parent of A2 is the same as that of A1.  The
tool will do:
```sh
git co --detach main
git cherry-pick A2   # create commit A2.1
git co patch-A
git restore -s A2.1 .
# if any changes:
git commit -a        # prompt user for commit message
git push origin patch-A

# repeat for B and C
```

So the author keeps their patch-stack, and the reviewers see only commits!

If the commits have dependencies, then we stack our branches in that order in
the side-worktree, and do basically the same thing, but now we may have to
rebase the B and C when A is updated; this seems to be a necessary evil for
dependent pull-requests in all workflows.  Of course, we can try to force-push
the rebases separately from the contentful changes, and importantly, we keep
the commit-by-commit history.  (A potential problem here is that that means we
need to rebase each rev separately; we'd have to have an option to squash them
if that will cause conflicts.)

Ideally, we can do the switching between the two modes under the hood (perhaps
with a warning to the user); it'll just tell you "I noticed this dependency"
and do the right thing.


TODO: flesh out the dependent-patches cases, there are a lot of details to
understand and where this might break down.
TODO: can we actually do this via merges, such that even stacked commits will
work right (and thereby better than the canonical rebase workflow for the
same)?  It's a bit hard when you land the bottom patch (into, say, commit AS),
but maybe you can merge AS into B (and in principle you should be able to know
the correct resolution to what git will presumably see as conflicts).  Still,
there's no way to reorder or split reviews, so it's not clear that it's worth
the effort.

## prior art

TODO: look at more, explain what I don't like about these. generally many of them are too visible to reviewers for my taste

- https://graphite.dev/stacking 
- https://github.com/ezyang/ghstack
- https://github.com/ejoffe/spr
- https://github.com/keith/git-pile

## appendix: what I do right now

This part is not particularly original, but I do have a few tricks.

The basic idea is you have a stack of branches, each of whose upstream is the
previous branch. When you make a change to one branch, you rebase all the ones
above it.

The first set of tricks is around how to know what to rebase onto. The basic
idea is when you are updating `branch-n+1`, you want to check it out, then `git
rebase old-branch-n --onto new-branch-n` so that you don't have to deal with
replaying duplicate commits. When you've already rebased `branch-n` (and it's
set as your upstream), you can just use `git rebase --fork-point` which will
automatically figure that out. When you've just squash-merged the bottom PR
(through the UI/merge queue), what you want is:
```
git fetch origin
git checkout branch-2
# origin/main is new branch-1
git rebase branch-1 --onto origin/main
```

The other annoying thing is you really have to merge them into each branch
every time you update any lower branch, otherwise you get spurious changes (and
merge conflicts) in the dependent PRs. The new `git rebase --update-refs` may
help for this but I haven't tried it. The alternate solution is to use merges;
GitHub understands those so you can just merge into whichever branches you feel
like. (The tangled history will of course disappear with the final squash.) The
problem incorporating updates from squash-merged branches is now even more
painful. The trick is to create a fake merge commit between our local
`branch-1` and main that actually just takes main's version, and then merge
*that* into `branch-2`:

```
git fetch origin
git checkout branch-1
# clobber our version with their version
# (TODO: this isn't strictly right; it only clobbers in case of conflicts.
# figure out how to *really* clobber just in case.)
git merge origin/main -X theirs
git checkout branch-2
git merge branch-1
```

Look, I didn't say this was a good workflow!
