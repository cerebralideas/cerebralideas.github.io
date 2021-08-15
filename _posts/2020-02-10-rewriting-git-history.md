---
layout: post
title: 'Rewriting Git History'
tags: [Git, Quick Explainer]
featured_image: assets/images/posts/2020/octocat-roman-synkevych-unsplash.webp
author: justin
---

Understand how rewriting history with `git amend` and `git rebase` can be crucial in keeping your Git logs clean and simple.

## Explainer

Say you have a code base that you just created and you have two commits locally:

```text
commit a
commit b
```

This is your “version history”. You then push these commits to your remote. Now you have two repos with the same history:

```text
localhost          remote
=========          =========
commit a           commit a
commit b           commit b
```

Unfortunately, you notice a simple typo in your code. Ugh, you fix it and write a new commit. You now have two Git repos with the respective histories:

```text
localhost          remote
=========          =========
commit a           commit a
commit b           commit b
commit c
```

When you want to push the change to your remote repo, Git easily knows what to do:

```text
localhost          remote
=========          =========
commit a           commit a
commit b           commit b
commit c    -->    commit c
```

Nice and simple. Since the only difference between your localhost and remote is the new commit, Git can easily add it to your remote’s history. You can think of this as writing new history.

But, let’s say you don’t want to add a whole new commit to fix a silly typo, and you’d like to just update your previous commit. You can do that, but it’s called “rewriting history”, and this is where things get a bit more complicated.

Instead of writing new history, you can [rewrite _existing_ history](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History) with things like [the `rebase` command](https://git-scm.com/docs/git-rebase), or [the `commit` command with the `--amend` option](https://git-scm.com/docs/git-commit#Documentation/git-commit.txt---amend). Let’s take the earlier example above, but instead of adding a new commit, you modify your previous one. This is done with `commit --amend`, and the thing to keep in mind is that commits are all based off of SHAs, which in turn are based off of your code. So, let’s add the SHAs (abbreviated, of course) to our original history of commits:

```text
localhost                  remote
=========                  =========
commit a | ab4ir86d        commit a | ab4ir86d
commit b | bkudgei8        commit b | bkudgei8
```

Okay, so, you make the change, save your file and run `git commit —amend`. Let’s see what it does to our histories:

```text
localhost                  remote
=========                  =========
commit a | ab4ir86d        commit a | ab4ir86d
commit b | mfdkg97r        commit b | bkudgei8
```

Locally, this goes without a hitch, so now you're ready to push the changes to your remote, excited that you’ve kept everything nice and clean. Ah oh, Git’s now confused. Knowing that Git ignores your commit messages, and focuses _only_ on SHAs, it doesn’t know all you did was modify your previous commit. To Git, it’s a brand new commit, so it’s trying to be helpful and let you know that you're missing the following commit locally:

```text
commit b | bkudgei8
```

It’s wanting to recommend this:

```text
localhost                   remote
=========                   =========
commit a | ab4ir86d         commit a | ab4ir86d
commit b | bkudgei8   <--   commit b | bkudgei8
commit b | mfdkg97r   -->   commit b | mfdkg97r
```

Wait, that’s not right! Why does Git suggest that? Well, when you amended the previous commit, you re-generated the SHA under slightly different conditions, so the SHA is now different, regardless of how simple the code change. You rewrote history, and essentially deleted the previous commit and created a new one in its place. Git is trying to recommend a non-destructive way of keeping the two repos in sync, but that’s not what we actually want.

Now, to push these changes the way we actually want, you need to tell Git that the history of the remote is no longer correct. You do that by [forcing the push](https://git-scm.com/docs/git-push#Documentation/git-push.txt---force): “Hey Git, ignore your built in safety checks, and just trust me that this is right!”.

The reason this is dangerous is because rewriting history is destructive and cannot be corrected without a [`reflog`](https://git-scm.com/docs/git-reflog), and that’s only stored locally where the change took place. That doesn’t mean force pushing is wrong, but that you just want to make sure you are indeed doing what you intend.

## Rebasing

Now, rebasing with another branch, say `main`, is similar to the `amend` command above in that it "rewrites history", but rebase is much more powerful and complex. So, we'll take this a bit slower.

When you rebase, you are doing a one-way sync on two branches, commonly from `<source branch>`, what will call `feature-one` for the example, to `<target branch>`, what will call `develop`. Let’s take a look at the steps `rebase` takes to accomplish this goal:

1. Comparing the commit history of `develop` to `feature-one`, essentially verifying what commits are missing on `feature-one` that exist on `develop` (if there are none, there’s nothing to do, so exit)
2. Compare the commit history of `feature-on` to `develop`, essentially verifying what commits are new (if there are none, both branches are identical)
3. Remove any new commits from the `feature-one` that are not present on the `develop` to get them out of the way
4. Move all the missing commits from `develop` to the `feature-one` we found at step 1.
5. Now that `feature-one` is caught up with `develop`, “replay” the commits we removed in step 3 on the new commit history
6. If there are conflicts, report them and pause the `rebase`; if there are no conflicts, exit successfully

Let’s look at this visually:

### Step One

Compare `develop` (the source) to `feature-one` (target) and verify commits that `feature-one` is missing.

```text
develop                      feature-one
=========                    =========
commit alpha | jlsdgb97      commit alpha | jlsdgb97
commit foo   | dgh9g76e ✔︎    commit baz   | kjdfjgh8
commit bar   | fglj75bd ✔︎
```

Okay, we’ve identified two that do not exist on `feature-one`, and we’ve check marked them.

### Step Two

Compare `feature-one` to `develop` and verify commits that `develop` is missing.

```text
develop                        feature-one
=========                      =========
commit alpha | jlsdgb97        commit alpha | jlsdgb97
commit foo   | dgh9g76e        commit baz   | kjdfjgh8 ✔︎
commit bar   | fglj75bd
```

We’ve identified one that does not exist on `develop` , and we’ve check marked it.

### Step Three

Remove the commits identified in Step Two.

```text
develop                        feature-one
=========                      =========
commit alpha | jlsdgb97        commit alpha | jlsdgb97
commit foo   | dgh9g76e
commit bar   | fglj75bd
```

### Step Four

Move the missing commits from `develop` too `feature-one`.

```text
develop                        feature-one
=========                      =========
commit alpha | jlsdgb97        commit alpha | jlsdgb97
commit foo   | dgh9g76e        commit foo   | dgh9g76e
commit bar   | fglj75bd        commit bar   | fglj75bd
```

Now, your `feature-one` branch is now in sync with your develop branch!!!! Just a few more steps to go to finish this off.

### Step Five

Replay any `feature-one` commits found in Step Two.

```text
develop                        feature-one
=========                      =========
commit alpha | jlsdgb97        commit alpha | jlsdgb97
commit foo   | dgh9g76e        commit foo   | dgh9g76e
commit bar   | fglj75bd        commit bar   | fglj75bd
                               commit baz   | vjkfng71
```

It’s now VERY important that we notice the new SHA for `commit baz`. This is because the underlying code changes in `commit baz` are applied to a different code environment since we added commits that weren’t there before the `rebase`. Because we reapplied `commit baz`, as well see shortly, Git sees it as a completely different commit.

### Step Six

Any conflicts?

```text
develop                        feature-one
=========                      =========
commit alpha | jlsdgb97        commit alpha | jlsdgb97
commit foo   | dgh9g76e        commit foo   | dgh9g76e
commit bar   | fglj75bd        commit bar   | fglj75bd
                               commit baz   | vjkfng71
```

If no conflicts result in replaying the new commit, then we exit successfully and you now have successfully ensured that `feature-one` is in sync with `develop`. If we did get conflicts, it will pause the `rebase` and ask for you to fix the problems, but that’s a different lesson :)

### Pushing your newly rebased branch

Now that you’ve successfully rebased your local branch (feature-one), let’s push it to our remote repo.

```text
local                           remote
=========                       =========
commit alpha | jlsdgb97         commit alpha | jlsdgb97
commit foo   | dgh9g76e    ✖︎    commit baz   | kjdfjgh8
commit bar   | fglj75bd
commit baz   | vjkfng71
```

Oops, notice how the histories between local and remote are now out of sync. Git will do here what it did when we tried to push an amended commit: it will tell us that our remote repo has a commit that is missing in our local, asking you to pull the changes. Remind you have the `commit --amend` problem from above? Yup, that’s not what we want. The old `commit baz` on our remote is out-dated, so we have to force push in order to tell Git, “Hey, ignore your built in safety checks, and just trust me that my local is the one true way!”.

With all that being said, you can now see why people often just use “merge commits”, and that’s to avoid having to learn/teach/understand the above. Merge commits essentially preserve the existing history by adding new history, rather than rewriting old. No “force pushing” necessary. But, if you dedicate the time to properly learning how to safely rewrite history, the power that you can wield is awesome!

Be safe out there!
