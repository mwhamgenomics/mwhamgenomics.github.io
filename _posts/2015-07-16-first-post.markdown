---
layout: post
title:  "Git branching"
date:   2015-07-16 16:22:59 +0000
categories: programming, version control
---

### Git branching
If multiple people are working on a project, they should generally not work directly on the master branch. Instead, when they want to add something to the project, they should create separate branch, make changes, test, debug, etc. and merge the branch to the master. This way, the master is always fully functional. Often, modern development environments have their own version control support. This article mainly has the plain command line in mind, but may help understand how it works in other environments.

#### Create a branch
This can be done on the command line with:

{% highlight bash %}
    git checkout -b my_branch
{% endhighlight %}
or:
{% highlight bash %}
    git branch -d my_branch
{% endhighlight %}
It’s a good idea to use a descriptive name for the branch, such as the issue/ticket you're working on. You can have multiple local branches and switch between them with git checkout.

You can keep the branch local, or push it with:
{% highlight bash %}
    git push -u origin my_branch
{% endhighlight %}
(You'll have to do this eventually to merge it)

You can work on this branch, committing and pushing changes. Then when you think it's ready to be put into staging/production, you can open a pull request on GitHub (or its equivalent on GitLab, etc.).

#### Pull request
It's usually a good idea to assign the request to someone else for peer review. The person reviewing the request can then discuss it with you, request further changes, and then close or approve it. Merging can usually be done automatically by Git, unless there are merge conflicts, which have to be resolved manually. Once the branch has been merged, it can be deleted.

#### Delete branch
This should be done both on GitHub and locally. To do it locally:

{% highlight bash %}
    git checkout master
    git pull  # The master should now have all the changes
    git branch -d my_branch  # does nothing if branch hasn't been merged. Use -D to force-delete
{% endhighlight %}

#### Merge conflicts
If two branches diverge (e.g. someone else merges their branch to the master just before you), you may have to do some fiddly manual merging. You should be able to see on GitHub that automatic merging is not possible, in which case you can either resolve conflicts on GitHub, or locally by pulling changes to the master back into your branch:

{% highlight bash %}
    git pull origin master
{% endhighlight %}

Git may complain at this point about conflicts, in which case it should give you a list of offending files. You can then go in and fix these files, which Git should have annotated with details of each conflict. Then add/commit/push the changes. Just make sure everything's resolved properly. If you're not careful, you can end up overwriting other people's changes. Minimise this risk by keeping your change list small.

### Things that should not be in Git repos

- Passwords. 'Nuff said.
- Command histories from interactive shells (.Rhistory, python_history, etc.).
- Large binary files. The amount of data stored locally is multiplied per user per branch, and you can end up taking up a lot of space.
  - You may have to store some things, though, like images.

### .gitignore
This sits in the repo's root directory and tells Git what files to ignore when detecting changes to commit. Syntax uses Linux-style paths and wildcards. Use a trailing slash to denote a directory. For example:
{% highlight bash %}
    .python_history
    __pycache__/
    this/that/other.txt
    one/two/*/*.sh
{% endhighlight %}
