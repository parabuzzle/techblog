---
layout: post
title:  Fun with Git Aliases
date:   2014-10-13 12:00:00
categories: Git
#preview: A simple step-by-step guide to setting up Codeship.io to test and deploy Chef changes to your Chef server automatically.
tags: [Tech, Git, Tips, Tricks, Aliases, Rebasing]
og_type: article
og_description: Git is awesome! That goes without saying, but with aliases you can make it more awesome. These are some of the aliases I'm using to make my life with Git a little easier.
disqus_id: 10
---

Git is awesome! That goes without saying, but with aliases you can make it more awesome. These are some of the aliases I'm using to make my life with Git a little easier.

Let's first talk about aliases and what they are. The concept of aliases is as old as Bash. The idea is simple, you create a shortcut command for yourself so you don't have to remember what incantation of flags you need for that thing you need all the time. Here are some examples of common Bash aliases:

~~~ sh
# Use '..' to change to parent directory
alias ..="cd .."

# Use 'll' to do a list all long
alias ll="ls -la"

# List only directories
alias lsd="ls -lF ${colorflag} | grep --color=never '^d'"
~~~

These examples are Bash aliases, but it illustrates what an alias is and how they can make your command line life easier.

But did you know... Git has the same functionality?!

The easiest way to add aliases to Git is using the git config. (This can be per repo using `.git/config` in your repo or globally using `~/.gitconfig`)

Git configs use `key = value` syntaxing separated into sections like `[section]`. This is a basic git config:

~~~ sh
#
# default .gitconfig
#

[user]
        name       = "Mike Heijmans"
        email      = "parabuzzle@gmail.com"
        editor     = vim
        signingkey = 24BA2EBA6076278C

[core]
        whitespace = trailing-space,space-before-tab,indent-with-non-tab,cr-at-eol
~~~

So all we need to do is add an `[alias]` section with some basic aliases:

~~~ sh
[alias]
        s = status
~~~

So with this Git config we can use `git s` just like `git status`.

That saves 5 key strokes!! Look at that! We're already saving time!

By default, Git aliases are based off of the Git command. (as you can see in the above example) But you can have it run other commands using a shebang (let's say shell commands):

~~~ sh
[alias]
        cleanup = !sh -c 'git branch | grep -v master | grep -v staging | xargs git branch -D'
~~~

This alias gives you the command `git cleanup` and will remove all branches that are not "master" or "staging". (use carefully.. there is no confirmation) As you can see, its just shelling out.

Pretty neat yea? ...Let's take it further.

You can use aliases in other aliases to create compounding aliases. This is where it gets fun and super helpful.

In my current team, we try to keep our master branch as clean as possible. We do this by using rebase to squash our pull request changes after review and before we merge our feature branch into master. (you can read about how this works [here](https://github.com/edx/edx-platform/wiki/How-to-Rebase-a-Pull-Request))

If you don't want to read the linked article.. It boils down to this:

~~~ sh
# Get SHA for rebasing
git merge-base my-branch master

# Start the rebase
git rebase --interactive ${HASH}

# Force a push to the branch on github
git push -f
~~~

What this does is allow us to squash extra commits (like styling or typo fixes) into useful history chunks for future ages. Then we force an update to the feature branch on Github. (caveat: Don't do this if someone else is using that branch) Then we merge to master using the button on Github.

So now let's use compounding Git aliases to make this process much easier!

~~~ sh
[alias]
        # rebase off of master
        cb = rev-parse --abbrev-ref HEAD
        mergesha = !sh -c 'git merge-base `git cb` master'
        rb = !sh -c 'git rebase -i `git mergesha`'
~~~

So what do we have happening here? We're setting up 3 aliases:

  * `git cb` : Returns the current branch (cb = Current Branch)
  * `git mergesha` : Returns the merge-base SHA of the current branch and master
  * `git rb` : Runs the interactive rebase with the SHA

The thing to note here is that `git rb` uses `git mergesha` which uses `git cb`... This is a compounding alias. Sure, we could just make `git rb` shell out and do all this work in a bash oneliner but splitting it like this makes it easier to read and understand what's happening, plus it gives you shortcuts for the intermediate steps so you could run `git mergesha` to get the SHA only... etc

Let's bring this full circle with a Bash alias:

~~~ sh
alias rebase!="git rb"
~~~

Now I can just run `rebase!` in my Git repo and it'll give me the interactive rebase of my branch against master.

You can checkout my aliases file and my global gitconfig in my public dotfiles repo for more alias ideas: [https://github.com/parabuzzle/dotfiles](https://github.com/parabuzzle/dotfiles)
