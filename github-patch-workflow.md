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

TODO: what happens when the patches are dependent?  Basically we just do the
best we can, according to previous proposals, but it'd be worth working out
some of the tricky cases.
TODO: can we actually do this via merges, such that even stacked commits will
work right (and thereby better than the canonical rebase workflow for the
same)?

