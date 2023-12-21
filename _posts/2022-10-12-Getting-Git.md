---
pin: true
layout: post
title: "Getting Git"
date: 2022-10-12
author: shawn
categories:                                         
  - code
tags:
  - security
  - script
  - code
image:
  path: /static/img/Git-logo.svg
---

There are lots of articles and books that cover Git and I’m not one to reinvent the wheel - I always try to bring something new to the table. First, lets look at what’s out there so we have a good understanding of what makes up the wheel and then I’ll build on that.

Long ago we all realized that manually copying and pasting text was slow and error prone so we made the patch command. Patch is quite simple and still used in some systems today. There are even facilities in Git to "apply a/path/to/a/patch/file". Patches are a simple text file that can be emailed or handled the same as you would any other simple file.

But we also realized it's nicer to work within a system and soon after patch, CVS was made (and is widely used even today). Between CVS and Git, there was SVN, DARCS, and Mercurial. IBM made ClearCase, and Microsoft made Team Foundation Server; there are others. Wikipedia has an extensive list here: [https://en.wikipedia.org/wiki/Comparison_of_version-control_software](https://en.wikipedia.org/wiki/Comparison_of_version-control_software). 

The best history on the subject I could find is here: [https://www.oreilly.com/library/view/mercurial-the-definitive/9780596804756/ch01.html](https://www.oreilly.com/library/view/mercurial-the-definitive/9780596804756/ch01.html). I feel one should know what has happened in the past so you know why things are done a certain way and where mistakes were made so that we can move forward with a common understanding of the status of things. I may go back and dig into this history and write a post about that later, but that's not what this post is about. So let’s move on...

## Git Basics

I look at Git as having five parts - three data stores and two transitory areas. The data stores are our work tree, Git directory, and remote(s). When we clone or fetch or push or do any other remote operation, we are syncing parts of our Git directory to a remote, which does not impact or sync from our work tree. When we do a git add, checkout, or commit (and other commands), we are moving data between our work tree and the Git directory, which does not impact any remote sources.

We can store interesting artifacts in this Git directory that don’t get synced such as the config data and hooks. However, if it is to get synced or checked out, it is stored in our .git directory.

At the core of all change systems is the update or write. CRUD (create, read, update, delete) only has one operation that isn't a write operation of some type (i.e., read). Every versioning system I'm aware of does this write with a commit - Git is no different. Commit writes your changes to its Git directory. Unlike other systems, however, a commit doesn’t care about your file changes; it cares about staged changes. Think of the stage or index as a protocol between your work tree and Git directory. Stage isn’t a separate area (it’s in the index of the object store in the Git directory) but you need to use it to make Git store changes. So, stage (add/apply) before a commit, and then a push for remote changes, which handles our protocol layers. In other words, I let the Git directory know I have something for it (add/apply) and then give it that information (commit). After that, maybe I’ll push (up) or pull (down) to sync my Git directory with a remote (and possibly update my work tree as well). The Git directory can be further broken down into two parts: configuration data and object store, but we generally don’t need to consider this when working with the system.

Last, Git subcommands are actually just git-<command> executables in your path but normally stored along with git - we’ll see this more later. As such, the manpage for a git command is accessed with: man git-<command> (i.e., man git-fetch).

With those concepts explained, there are a few notable articles showing how this actually functions: 

Drupel explains common Git workflow very well: [https://www.drupal.org/docs/installing-drupal/building-a-drupal-site-with-git](https://www.drupal.org/docs/installing-drupal/building-a-drupal-site-with-git). Most of this is about the work tree, stage, and Git directory/object store layers.

This writeup shows more advanced work with Git objects: [https://wildlyinaccurate.com/a-hackers-guide-to-git/](https://wildlyinaccurate.com/a-hackers-guide-to-git/)

I put other articles covering technical details of the object store in my references. However, if you read and can fully comprehend the two articles above, you’ll have a better understanding of Git than most users.

## Git config

Git has an INI type configuration file you may interact with by the git config command. The properties you pass are `<heading>.<option>` but other than being able to create strings that are easier to write about or search for, I don’t see much point in interacting with Git’s configs like that (from the command line). I’d recommend using your favorite editor, open (or create) a `~/.gitconfig` file and type stuff in it. You should create this file and move it around with you, but you shouldn’t put anything proprietary in this file. Your employer should allow you to copy it onto and off of their systems (and you should consider taking your .gitconfig file when you change jobs). Each repo will also have its own config file within the Git directory for repo specific options (or for override options), but those options should normally be different than your global config so I will only cover the network/object (remote/branch) aspect of it later.

You’ll need a [user] section in your global `~/.gitconfig` at a minimum with your name and email or Git will complain as soon as you try to commit something. The config may have other things like a signingkey option too. But this section isn’t very interesting either. The aliases section starts to get interesting as you can augment Git commands with your own. However, as I’ll show later in the Subcommands section below, aliases are not that different than a script called git-<alias> somewhere in your path, which is probably the preferred approach because you can have nicer line breaks and quoting. So let’s move onto the other more interesting config options that I’ll show some examples of now.

Though I may like having a green and white terminal as default, I definitely like things to be colorized where possible. I want text with important information to cue my eye with colors. Git is quite configureable here. Maybe you won’t like the colors I use, but this should give a good idea of what is possible:

```
[color]
  ui = true
[color "diff"]
  plain = normal
  meta = bold
  frag = cyan red
  old = red
  new = green blue bold
  commit = yellow
  whitespace = normal red
[color "branch"]
  current = green
  local = normal
  remote = red
  plain = normal
[color "status"]
  header = normal red
  added = white bold
  updated = green bold
  changed = red bold
  untracked = yellow bold
  nobranch = red bold
[color "grep"]
  match = normal
[color "interactive"]
  prompt = normal
  header = normal red bold
  help = normal
  error = normal
[color "log"]
  header = normal red bold
```
  
Git commands that produce output also have a format that accept things like: `%C(bold blue)`, which you may even consider implementing as aliases, but the above is a nice start.

If you work on large projects, there are probably standards you need to adhere to. This means, when you create a new project, you generally want it to act a certain way. You do this by defining an `init.templatedir`, which is basically like using /etc/skel for Git. When you start (init) a Git repo, those files get put into your Git directory. You may also be asked to format commit messages a certain way - this can be accomplished by defining a `commit.template` file. For more about commit templates, see: [https://nulab.com/learn/software-development/git-commit-messages-bold-daring/](https://nulab.com/learn/software-development/git-commit-messages-bold-daring/)

The other options I have and may recommend further reading on are:

```
[commit]
  status = true
  gpgsign = true
[push]
  default = matching
[core]     
  pager = less -FRSX
[help]     
  autocorrect = true
[advice]
  pushNonFastForward = false
  statusHints = false
[diff]     
  renames = copies
  mnemonicprefix = true
[branch]
  autosetupmerge = true
[rerere]
  enabled = true
[merge]
  stat = true
```

It’s probably best to read on rerere before blindly enabling it (but it’s kind of godly when it works, which has been most of the time for me). And I’m probably going to replace my pager with bat soon ([https://github.com/sharkdp/bat](https://github.com/sharkdp/bat)). The gpgsign will probably break you if you don’t use gpg, as well. (I plan to look into s/mime x509 signing with smartcards to replace it soon). I have also disabled advice as it clutters up my terminal and doesn’t add anything for me. Any program “advise” is probably like Microsoft Clippy - you either love it or hate it.

In the end, these are settings I’ve discovered that are useful for my workflow. You should think about whether they work for you. 

## Looking at Git

Git offers us quite a few trace options. I find it’s much more useful to just wrap them in convenience commands than needing to set them. Also, when I make a wrapper function, I start looking for more debug data (like strace). As such, I’ve written two commands to look at Git further - one that gives the basic git output when entering a git command and another that shows the rest of the trace data:

```shell
tgit () {
  opts="$@"
  tgit_log="$( \
    strace -f -e open,access,connect,recvfrom,sendto,network -- \
      bash -c ' set -vx ; \             
        GIT_TRACE=1 \
        GIT_TRACE_PACK_ACCESS=1 \
        GIT_TRACE_PACKET=1 \
        GIT_TRACE_PERFORMANCE=1 \
        GIT_TRACE_SETUP=1 \
        GIT_SSH_COMMAND="ssh -vvvv " \
        GIT_PAGER= \
        git '$opts' \
      ' 2>&1 \
  )"
  echo "$tgit_log" | grep -vaE '^(([0-9]{2}:){2}[0-9]{2}\.[0-9]{6}|debug[0-9]:|\[pid [0-9]+\]|strace: Process [0-9]+|.* = -1 ENOENT) '
}

tglog () {
  echo \"$tgit_log\" | grep -aE '^(([0-9]{2}:){2}[0-9]{2}\.[0-9]{6}|debug[0-9]:|\[pid [0-9]+\]|strace: Process [0-9]+|.* = -1 ENOENT) '
}
```

Basically, tgit runs the command with all the tracing options I could find set inside another bash shell with strace attached. It stores all of that output in a variable and displays enough to show that the command ran successfully (a bit more than basic Git output). And tglog displays everything else in that variable that may be useful (literally the reverse grep of the other command). The tgit function also calls ssh in very verbose mode for more network data. The tgit function also disables the Git pager because the pager complicates working with the output data.

With just Git’s internal tracing, we can see how Git resolves aliases and we can start to see what Git does to clone (or push) a repo. With strace, we can see where Git looks for files. With verbose ssh, we can see how connections are made. There are other strace options we can use to see more data but then we need to handle that data and write longer grep regexes. I believe these functions give a good starting point to analyze Git with - just expand or reduce their functionality to serve your needs.

Note: `GIT_PAGER` and `GIT_SSH_COMMAND` was mainly used for style. Doing
```console
git —no-pager -c core.sshCommand="ssh -vvvv"
```
may give similar results.

## Subcommands

Now that we have our analysis functions, let’s see how subcommands work. We’ll first look at some aliases I use to see logs and tags:

```
lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset %C(bold green)%GS [%G?]%Creset' --abbrev-commit --date=relative
treelog = log --oneline --graph --decorate --all
tg = for-each-ref --format='%(refname:short) %(taggerdate) %(subject) %(body)' refs/tags
```

If I run any of my aliases through the tgit function and then look for the subcommands git runs, I’ll see the git-<command> resolution like so:
```console
$ echo "$tgit_log" | grep -a run_command
```

Aliases work fine for simple operations. However, when you start embedding functions into them, it might be time to look into making them actual subcommands. About a decade ago, Git introduced the ability to call external commands with aliases by using a ‘!’ at the beginning which also also allows for parameterized anonymous functions. I feel that when you start reaching for this feature, you should consider creating an external subcommand instead of an alias. To this end, Git will search all directories in your path and the paths in `GIT_EXEC_PATH` for git-<command> executables.

Let’s look at an example alias called t1 that defines a function that then calls itself:

```shell
t1 = "!f() { \
    set -x; \
    echo \"$@\"; \
  }; \
f"
```

The alias itself looks like:
```console
$ git config alias.t1
!f() {       set -x;       echo "$@";     };   f
```

None of this looks very nice. However we can easily rewrite it as a simple shell script and stick it in a bin directory like this:

```console
$ cat ~/bin/git-t1 
#!/bin/bash

set -x
echo "$@"
$ git t1 foo      
+ echo foo
foo
```

And I’d argue that, even for something that simple, the subcommand looks much nicer. This writeup provides some more explanation: [https://memcpy.io/git-alias-function-syntax.html](https://memcpy.io/git-alias-function-syntax.html)

If you write a subcommand and you forget to make it executable, you’ll get a cryptic error message like:
```console
fatal: bad numeric config value 'true' for 'help.autocorrect' in file /home/azureuser/.gitconfig: invalid unit
```

There is also a github project called hub: [https://github.com/github/hub](https://github.com/github/hub) that has some interesting features. However, a shell script (or python script or any other script really) with business logic to allow interfacing to your repo and ticketing system etc., seems like a much better option. I was considering using GitHub’s graphql implementation to show this, but this post is already too long  andvthe ideas to do that should already be expressed. I’d assume the reader has their own specific ideas and business logic for such a tool anyway.

## Network Protocol

A repo’s git config describes the two protocol layers of Git. The remote(s) describe how to handle network interactions and the branch(es) describe how to move data between the filesystem and work tree. 

When you clone a repo, Git will create a `.git/config` file (as a part of its filesystem), which has lines like:

```
[remote "origin"]
        url = https://github.com/git/git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
```

Let’s consider how a workflow refers to this in order to work. We need to be in a branch to do useful work. We could be in a detached head, but then we’re not tracked and we want to be tracked, so we’re in master. Since we’re in master, assuming your files are staged/indexed, your commit will update the `.git/refs/heads/master` - i.e., the merge line. A merge into this branch will also update this ref, but that’s not as normal of an operation as a commit from a user perspective. And since we don’t normally want to specify the remote when we push, we list the remote that our branch tracks here, too (as “origin” in this case).

So the “branch” section basically tells Git how the work tree should be updated from the object store part of our Git directory. But then Git also needs to know how to interact with a remote source. This is done with the first section.

We see remote section listed with the “origin” because origin is the default remote repo name (`man git-remote`) and we see the url of the repo we just cloned listed under that along with a fetch line. This post does a fair job of describing this part of the process: [https://towardsdatascience.com/demystifying-git-references-aka-refs-bdd09029d072](https://towardsdatascience.com/demystifying-git-references-aka-refs-bdd09029d072)

A nice command to use to look at these refs is:

```shell
for i in .git/refs/**/*; do
  [[ -f "$i" ]] && echo -n "$i: " && cat "$i";
done
```

Two other good references in this area are:  
[https://git-scm.com/docs/protocol-v2](https://git-scm.com/docs/protocol-v2)  
[https://git-scm.com/book/en/v2/Git-Internals-The-Refspec](https://git-scm.com/book/en/v2/Git-Internals-The-Refspec)  

## Getting history

For me, the biggest selling poing of Git is being able to see what I (or someone else) was thinking when they wrote something. The obvious command to demonstrate this is git log. However, by default the log command isn’t as useful as it could be and I showed aliases to log above (`lg` and `treelog`) what I use instead of the normal `git log` command.

This just scrapes the surface though. The `git rev-parse` defines syntax (linked below) that allows us to see the differences in any point in time between any local or remote repo branches we have defined. Lets give a few examples: 

Lets assume I’m looking at a pull request of a repo I have cloned. The pull request is against the remote I call “origin”. The pull request is probably against a development branch, but I want to see how it compares against the current tagged release version. The obvious command is: `git diff`, but how do I compare those two versions?

I picked a target to show this at random: [https://github.com/ansible/ansible/pull/79072](https://github.com/ansible/ansible/pull/79072) and this links to [https://github.com/bartdorlandt/ansible/tree/patch-1](https://github.com/bartdorlandt/ansible/tree/patch-1) (which is a “patch-1” branch of bartdorlandt’s ansible repo). So let’s add a remote for it:

```console
$ git remote add blog-target https://github.com/bartdorlandt/ansible
```

Then grab all the branches:
```console
$ git fetch —all
```

Find the latest tags (using my ‘tg’ alias):
```console
$ git tg
```

Write the diff:
```console
$ git diff 'remotes/blog-target/patch-1...tags/v2.9.9'
```

Now, compare the latest tagged release with what was in devel yesterday:
```console
$ git diff 'remotes/origin/devel@{yesterday}...tags/v2.9.9'
```

Note that some of the range syntax Git has needs quoting so that a shell doesn’t try to do other things with it. This syntax is documented here: [https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection) the reference for this behavior is here: [https://git-scm.com/docs/git-rev-parse#_specifying_revisions](https://git-scm.com/docs/git-rev-parse#_specifying_revisions)

If you’d like to search commit messages for all commits not made by someone at Redhat:
```console
$ git log --author='@(?!redhat.com)' --perl-regexp
```

Lastly, let’s say I’m looking at this setup.py sdist target that the makefile is creating:
```console
$ git blame Makefile | grep 'setup.py sdist'
5f227fe260a (Toshio Kuratomi        2019-08-20 23:53:35 -0700 129)      _ANSIBLE_SDIST_FROM_MAKEFILE=1 $(PYTHON) setup.py sdist --dist-dir=$(SDIST_DIR)
5f227fe260a (Toshio Kuratomi        2019-08-20 23:53:35 -0700 136)      _ANSIBLE_SDIST_FROM_MAKEFILE=1 $(PYTHON) setup.py sdist --dist-dir=$(SDIST_DIR)
a196c7d737d (Brian Coca             2016-01-13 10:17:43 -0500 140)      $(PYTHON) setup.py sdist upload 2>&1 |tee upload.log
```

But I don’t believe that’s how that code started out and I want to see when someone first started building it. I have a git-seen command that I pass a file and a string to that tells me when the string was first ‘seen’:
```console
$ git seen Makefile 'setup.py sdist'        
001937976f408cfb8290d044be1571bc78628560:Makefile:      python setup.py sdist
```

Well, that’s different than what `git blame` is telling me - it was introduced before those commits. I can then use that hash to look into it further. My subcommand is:
```console
$ cat ~/bin/git-seen 
#!/bin/bash

# Find the first commit file(1) contained string(2)
if [[ -e "$1" ]]; then
  git grep "$2" $( \
    git log --reverse --pretty=format:%H -- "$1" \
  ) -- \
  "$1" \
  | head -1
else
  echo "No file $1"
fi
```

We may also use the format option of many git commands that provide output along with eval to preform other operations as well:
```console
$ eval $(git log --format='git cat-file -p %h')
```

## Changing history

There’s been much said on this subject. I just have one thing to add. I’ve seen many monolithic repos. We often start a project in a larger codebase because it’s a small script that has functionality that is part of the code that surrounds it. That’s fine. But then that functionality expands and we’d like to give it a home of its own. Maybe we’d even like others to contribute to it, but our larger codebase has business logic or, for some other reason, is none of their business.

Often people just copy the directory into a new directory and do a `git init` for the new project and call it a day. I don’t like this because that loses history. What were you thinking when you created this line of code or made this fix? Maybe I even had issue ticket information along with my commit messages and could track down why something was done further. If I just create a new repo, I loose all of this metadata. Lets not do that. Instead lets extract that code and the commits into a new branch.

> WARNING: I make a copy of my repo before using this script - you probably should too :) You may have also mentioned other parts of your business in some commits that you don’t want to release. You might considerauditing and squashing/rebasing/ammend’ing those.
{: .prompt-warning }

```console
$ git branchdir test new
$ cat ~/bin/git-branchdir 
#!/bin/bash

# From commits into a directory(1) create and apply them to a branch(2)
git branch "$2"
git filter-branch --subdirectory-filter "$1" -- "$2"
git filter-branch --tree-filter "mkdir -p \"$1\" && \
  find -maxdepth 1 \
    | grep -vE '^(\.|\.git|./\"${1%%/*}\"' \
    | xargs -i{} mv {} \"$1\" \
"
```

This post gives more description of what’s going on than I am:
[https://manishearth.github.io/blog/2017/03/05/understanding-git-filter-branch/](https://manishearth.github.io/blog/2017/03/05/understanding-git-filter-branch/)

I wrote this script years ago. I now get warning messages and am told about `git-filter-repo` which I haven’t used, but it has lots of interesting information in the wiki and a python library, etc. I highly recommend reading through their documentation and probably using their script if doing this type of work: [https://github.com/newren/git-filter-repo/tree/main/Documentation](https://github.com/newren/git-filter-repo/tree/main/Documentation) along with that, there’s also a “Discussions” section.

## Security

Let’s assume that we wanted to do evil things with a Git repo. Can we?

Well, Git is fairly secure in the implementation. The caveats to that statement are the recent collision attacks, which means any government can probably forge any `git commit` they want: [https://sha-mbles.github.io/](https://sha-mbles.github.io/).

There’s not too much that can be done about that if using a public Git repo service like GitHub or BitBucket or GitLab. However, there is currently a feature you may enable in your Git repo to move to SHA2. This assumes everyone using the repo is using a fairly new version of Git in which you run your own hosting solution, and the feature has some backward compatability issues you are ok with. It is a move in the right direction though, and given Google’s prior SHA1 collision work, it’s pretty disheartening that this feature came as late as it did: [https://git-scm.com/docs/hash-function-transition/](https://git-scm.com/docs/hash-function-transition/).

There are two much more interesting security issues around Git though. The first (and more simple of the two) is the ability to impersonate anyone you want. This is a feature of Git. However, the repo services help this hack by backing it with a nice web interface with user metadata.

```console
$ echo “” >> README.md
$ git -c user.email="torvalds@linux-foundation.org" -c user.name="Linus Torvalds" commit -m "Oh fuck. If I kill this guy, I'll have millions of nerds on my case." README.md
```

Which created this: [https://github.com/sandboxcom/test/commit/9fe450b5c5b0d42066cf9419add33300d1b4c928](https://github.com/sandboxcom/test/commit/9fe450b5c5b0d42066cf9419add33300d1b4c928)

Before I get sued for libel, obviously the creater of Linux and Git didn’t just update a readme file with a blank line in a test repo I have with an old quote of his as a commit message. However, when I look at it on GitHub it sure looks like he did.

There is the ability to add pgp or s/mime signatures to commits, which could prevent this from happening. I’ve yet to see this done in a large project though. If I make a pull request (PR), web authentication is used to login as the account that makes that PR, and as we’ll see next, all PRs should probably be squashed. So not too big of a deal as long as git is being used with a central repo mechanism.

The second workflow issue is: if I create a PR, the person reviewing my code is probably only going to look at the diff from the head of my PR and not the individual commits. So lets setup the scenario:

Lets put some malicous code into a file and commit it with an innocuous commit message:
```console
$ echo “malicous code” >> file
$ git commit -m “I had a good idea.” file
```

Then lets create a reverse commit:
```console
$ sed -i ‘/malicous code/d’ file
$ git commit -m “Saving my good idea for later.” file
```

And add some useful code for my PR:
```console
$ echo “Awesome code” >> file
$ git commit -m “Please merge my great work.” file
```

This goes into a PR that gets merged and doesn’t get squished. If I have the ability to update a tag that people or systems point to or create a new minor version tag that systems will think is a code update:
```console
$ git tag -f v999 deadbeef
$ git push -f origin HEAD:refs/tags/v999
```

You will run my malicous code even though everyone checking out and looking at the repo is looking at sane, non-malicous code. I only require a malicous hash anywhere in your repo and the ability to point to it - that’s it.

With open source projects, you normally need to be a member of the project in order to create or update tags. Within organizations, more people often have this ability, but there’s also financial incentive not to do malicous things in your workplace. If those external guard rails aren’t in place, there’s not much else to stop someone here.

If you implement a git server on a Unix box you control, you may use standard Unix permissions to lock down refs. The repo services have also started allowing the ability to lock down refs further, but this is a recent feature and not enabled by default.

GitHub: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/configuring-tag-protection-rules
https://github.blog/changelog/2022-03-09-tag-protection-rules/

GitLab: [https://docs.gitlab.com/ee/api/tags.html](https://docs.gitlab.com/ee/api/tags.html)

BitBucket: [https://confluence.atlassian.com/bitbucketserverkb/how-do-i-block-all-tags-from-being-pushed-to-a-repository-822021700.html](https://confluence.atlassian.com/bitbucketserverkb/how-do-i-block-all-tags-from-being-pushed-to-a-repository-822021700.html)

## End

As you can see, the system is pretty expansive and well thought out. I did not intend for this post to be as long as it is. There’s code in this post, but this is going to be slightly under 4500 words which is quite a long post for me. Being this long even though I was trying to stay brief and not cover topics others had should give some indication to the scope of the Git program and it’s use.

If you have taken the time to read this far, thank you and I hope you’ve learned something. If there are missing parts (that aren’t covered by posts I’ve linked) or erronious comments, please let me know.

## References

[https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)

[https://kamalmarhubi.com/blog/2016/10/07/git-core/](https://kamalmarhubi.com/blog/2016/10/07/git-core/)

[https://git-annex.branchable.com/tips/fully_encrypted_git_repositories_with_gcrypt/](https://git-annex.branchable.com/tips/fully_encrypted_git_repositories_with_gcrypt/)

[https://medium.com/mindorks/what-is-git-object-model-6009c271ca66](https://medium.com/mindorks/what-is-git-object-model-6009c271ca66)

[https://codewords.recurse.com/issues/three/unpacking-git-packfiles](https://codewords.recurse.com/issues/three/unpacking-git-packfiles)

[https://githooks.com/](https://githooks.com/)

[https://marklodato.github.io/visual-git-guide/index-en.html](https://marklodato.github.io/visual-git-guide/index-en.html)

[https://yunwuxin1.gitbooks.io/git/content/en/f38db0734dd1d1fca52030d15f93a77c/92abc307328bd414f4cd589a4400994b.html](https://yunwuxin1.gitbooks.io/git/content/en/f38db0734dd1d1fca52030d15f93a77c/92abc307328bd414f4cd589a4400994b.html)

[https://blog.plover.com/prog/git-rev-parse.html](https://blog.plover.com/prog/git-rev-parse.html)

[https://www.linuxfromscratch.org/blfs/view/svn/general/gitserver.html](https://www.linuxfromscratch.org/blfs/view/svn/general/gitserver.html)

