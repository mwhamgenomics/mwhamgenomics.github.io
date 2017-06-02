---
layout: post
title:  "Git branching"
date:   2015-07-16 16:22:59 +0000
category: Programming
tags: ['version control', 'environment']
---

If multiple people are working on a Git project, they should ideally not work directly on the master branch. Instead, when they want to add something to the project, they should create a separate branch on which they make changes, test, debug, etc, and and then merge that branch back to master when ready. This way, the master is always fully functional. Modern development environments often have functionality to aid version control and branching, but this post aims to discuss how to use branches via the plain command line.

## Create a branch
This can be done on the command line with:

    git checkout -b my_branch

or:

    git branch -d my_branch

You can have multiple local branches and switch between them with `git checkout`.

There are a few different ways of using branches in Git. You might choose to use feature branches, where you open a branch specifically for adding or modifying one part of the project, push to it, then merge it. Alternatively, you might have a 'develop' branch, where you make whatever changes and merge into master again and again over time. There are different reasons for using each, and it really comes down to the preferences of the person/team.

To push a local branch to the remote for the first time, run:

    git push -u origin my_branch

You can work on this branch, committing and pushing changes. Then when you think it's ready to be put into staging/production, you can request to have it merged. This would be a pull request on GitHub, a merge request on Gitlab, or whatever equivalent your Git server may have.

## Pull request
It's usually a good idea to assign the request to someone else for peer review. The person reviewing the request can then discuss it with you, request further changes, and then close or approve it.

## Merge conflicts
Merging of two branches can usually be done automatically by Git, and is done automatically when a pull request is approved. However, it can get complicated if there are merge conflicts, which usually arise when your branch and the master has diverged (e.g. someone else merges their branch to the master just before you). A GitHub pull request should show whether you have merge conflicts. To fix merge conflicts, while in your branch, run:

    git pull origin master

Git should complain about conflicts and give you a list of offending files. The next step is to go in and (very carefully!) fix these files, which Git should have annotated with details of each conflict. Then add/commit/push the changes. This must be done carefully, because it is perfectly possible to overwrite legitimate changes from other people.

## Delete branch
Once the branch has been merged and you don't intend to use it further, it can be deleted. This should be done both on GitHub and locally. To do it locally:

    git checkout master
    git pull  # The master should now have all the changes
    git branch -d my_branch  # does nothing if branch hasn't been merged. Use -D to force-delete

## Things that should not be in Git repos

- Passwords. 'Nuff said.
- Command histories from interactive shells (.Rhistory, .python_history, etc.).
- Large data files. The amount of data stored locally is multiplied per user per branch, and you can end up taking up a lot of space.
  - You may have to store some things though, like images.

## .gitignore
This is a file that you can create in the repo's root directory. A .gitignore tells Git what files to ignore when detecting changes to commit. Syntax uses Linux-style paths and wildcards. Use a trailing slash to denote a directory. For example:

    .python_history
    __pycache__/
    this/that/other.txt
    one/two/*/*.sh
