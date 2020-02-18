---
title: Cleaning up local git branches deleted on a remote
date: 2020-02-17
tags:
  - git
  - branch
  - cleanup
---

# Introduction

When using [git](https://git-scm.com/), local branches can track remote branches that no longer exist (the remote branch is _gone_). To identify these branches, we first have to cleanup (prune) the remote's branches:

```
$ git fetch -p
From https://test.com
 - [deleted]         (none)     -> origin/disable-feature-x
 - [deleted]         (none)     -> origin/fix-typo
 - [deleted]         (none)     -> origin/grammar-fix
```

In this case, three remote branches were deleted. Let's see if we have local branches that are tracking deleted branches:

```
$ git branch -v
  fix-typo            7b57d4f [gone] Fix typo in README
  grammar-fix         01257bd [gone] Fix some bad grammar
* master              477010d Bump to version 1.1
```

There are three local branches, of which two (`fix-typo` and `grammar-fix`) are marked with `[gone]`. This indicates that these branches are indeed tracking remote branches that have been deleted.

Unfortunately, git does not have built-in functionality to cleanup these local branches. Our only option is to manually delete them through `git branch -d <branch-name>`. But what if there are many of these branches? Things would get tedious quickly, so let's try to automate this!

# Identifying the gone branches

The first step in our automation is to identify the branches to delete. A naive approach would be to parse the aforementioned output of `git branch -v`. However, this is problematic for the following reasons:

1. The output could change in the future. See [this blog post](https://git-blame.blogspot.com/2013/06/checking-current-branch-programatically.html) by Junio C Hamano (git maintainer).
1. The output can be modified by the user. See [this StackOverflow post](https://stackoverflow.com/a/26152574/2071395).

Hmmm, let's try a different approach. It so happens that the [`git for-each-ref` command](https://git-scm.com/docs/git-for-each-ref) lets us specify its output format, neatly circumventing the aforementioned problems.

We can use `git for-each-ref` to list all branches _and_ their upstream's branch's status in our desired format:

```
$ git for-each-ref --format '%(refname) %(upstream:track)'
refs/heads/bugfix-1
refs/heads/fix-typo [gone]
refs/heads/grammar-fix [gone]
refs/heads/master [behind 15]
refs/heads/new-function [ahead 2, behind 9]
refs/remotes/origin/HEAD
refs/remotes/origin/master
refs/remotes/origin/new-function
refs/stash
```

Now this is something we can work with! As we're only interested in our local branches (heads), we'll filter them by appending `refs/heads` to our command:

```
$ git for-each-ref --format '%(refname) %(upstream:track)' refs/heads
refs/heads/bugfix-1
refs/heads/fix-typo [gone]
refs/heads/grammar-fix [gone]
refs/heads/master [behind 15]
refs/heads/new-function [ahead 2, behind 9]
```

Nice. We can do one last optimization here, and that is to return the branches in their shortened format by using `refname:short`:

```
$ git for-each-ref --format '%(refname:short) %(upstream:track)' refs/heads
bugfix-1
fix-typo [gone]
grammar-fix [gone]
master [behind 15]
new-function [ahead 2, behind 9]
```

Great! We now have a reliable, consistent way to retrieve our local branches and their remote tracking status.

# Deleting the gone branches

The next step is to filter the branches which remote branch is gone. We can do this by piping the output to `awk`, which can filter the branches and print their name (removing the remote tracking status):

```
$ git for-each-ref --format '%(refname:short) %(upstream:track)' |
  awk '$2 == "[gone]" {print $1}'
fix-typo
grammar-fix
```

Sweet! The final step is to pipe this output to `xargs` to delete the branches:

```
$ git for-each-ref --format '%(refname:short) %(upstream:track)' |
  awk '$2 == "[gone]" {print $1}' |
  xargs git branch -D

Deleted branch fix-typo (was 7b57d4f).
Deleted branch grammar-fix (was 01257bd).
```

And that's it! We now have a single command to delete all local branches which remote tracking branches have been deleted.

# Integrating with git

Our final step is to add our command as a [git alias](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases), which allows one to define custom commands that can be called as if they were built into git. Aliases in git are defined in the `.gitconfig` file found in your home dir. Within that file, find (or add) the `[alias]` section, and add an alias named `gone`:

```
[alias]
  gone = ! "git fetch -p && git for-each-ref --format '%(refname:short) %(upstream:track)' | awk '$2 == \"[gone]\" {print $1}' | xargs git branch -D"
```

The `gone` alias does two things:

1. Run `git fetch -p` (to remove any deleted remote branches).
1. Run our custom command (to remove local branches with a deleted remote branch).

Having added our alias, we can now run `git gone` as if it was a built-in command:

```
$ git gone

Deleted branch fix-typo (was 7b57d4f).
Deleted branch grammar-fix (was 01257bd).
```

The behavior is exactly the same as manually running `git fetch -p` followed by our custom command. Nice, isn't it?

Interestingly, due to the way git is implemented on Windows, the above alias also works on Windows.

# Conclusion

In this blog post, we've shown how to cleanup local git branches that are tracking remote branches that no longer exist. We did this by combining the `git for-each-ref` command with the `awk` and `xargs` commands. As a bonus, we added a git alias for our cleanup command that allows us to cleanup our local branches using `git gone`.
