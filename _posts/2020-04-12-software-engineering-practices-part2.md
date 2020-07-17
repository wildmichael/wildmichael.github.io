---
layout: post
title:  "Software Engineering Practices - Part 2"
date:   2020-04-12 12:48:00 +0200
categories: programming
---

This is the second part of a small series on software engineering practices.
In the [first part][part 1] a general workflow that implements a solid
development process has been introduced. Now the focus will be on the
mechanics, the craftmanship, that a software engineer needs to know
in order to implement the workflow. As mentioned at the end of
[Part 1], [Git] will be used as the [version control system (VCS)][VCS]
of choice.

## Introduction

While there are a number of excelent graphical user interfaces for working
with [Git], its roots lie in the Linux development community, and hence
it is not surprising that it fully shines when used from the command line
via its command line interface (CLI). However, this is not a unique trait
of Git, many other VCS also have the CLI as their primary user interface
(e.g. Subversion or Mercurial, just to name two). Many power users find it
even to be more intuitive and productive to use the CLI. And there is no
doubt, that if the concepts of the CLI commands are understood, that
the programmer will be able to use a graphical user interface with much
more confidence.

Hence, I urge the reader to bite the bullet, push through the learning
curve, and learn the Git CLI before going on to explore the graphical
user interfaces or IDE integrations. Consequently, all techniques
and instructions will be given in the form of CLI commands in
this article.

## Installation

Installing [Git] is beyond the scope of this article. The process is very
simple and well documented on the [project webpage][git].

## Basic Git Concepts

[Git] has a few characteristics that might appear confusing at first for
new users. Hence a short introduction to some of the most basic concepts
is presented here.

[Git] is a truly distributed [VCS]. In pricinciple it is possible to work in
a completely decentralized fashion where every project member maintains a
full copy of the history and commits are propagated from peer to peer. Most
companies, even most open-source projects, will however want to maintain a
central _"blessed"_ repository for various reasons. First, it is easier to
"find the code", something that can be otherwise absolutely non-trivial,
especially in larger enterprises. And secondly it is easier to exchange
information in a star-like fashion instead of doing the
all-to-all-communications dance.

But what does the word "distributed" mean in the context of a VCS? For
[Git] it means that every developer has a full copy of the repository,
including the full history. That enables the developers to work off-line
and to make new commits without any server communication. Of course, that
this implies that the code base diverges while the developers work
locally and that there needs to be a way to join these branches
back together.

### Tracking Branches

In order to enable the distributed nature, [Git] uses so called _remotes_ and
_tracking branches_. The former gives a remote repository a name.
Traditionally the central main repository is called `origin`. The _tracking
branches_ are there for Git to remember the state of each branch of each
remote locally. The _local_ branches, on the other hand are initially only
copies of the _tracking_ branches but then will be modified by the developer
when new commits are added. So, the situation could look like this:

```lang-none
+--------------------------+     +----------------+     +--------------------------+
|     Developer: Harry     |     | Central Server |     |     Developer: Sally     |
+---------+----------------+     +----------------+     +----------------+---------+
| Local   | Tracking       |     | Local          |     | Tracking       | Local   |
+---------+----------------+     +----------------+     +----------------+---------+
| master  | origin/master  |<--->| master         |<--->| origin/master  | master  |
| develop | origin/develop |<--->| develop        |<--->| origin/develop | develop |
+---------+----------------+     +----------------+     +----------------+---------+
```

All server-communication goes through the tracking branches first. Of course,
the server does not need tracking branches.

### Staging Area / Index / Cache

A special feature of Git is the so called _index_, sometimes also called the
_staging area_ or the _cache_. It is temporary place where the developer can
gradually add changes before finally creating the commit. This is a very
flexible mechanism that allows the user to first make a number of changes and
only later decide which of them should be included in the commit by adding
them individually to the _index_.

### The `.git` Folder

Every local copy of a Git repository features a special folder at its root
directory that is called `.git`. This is where Git keeps its data, including
all the commits. While the directory contents is surprisingly simple to
understand (and for advanced users to modify directly), for now consider it
to be _off limits_. Do not delete it, do not modify it.

## Basic Git Commands

It is time to try the first commands. Fire up a command line (for Windows
users it is called `Git Bash`) and use [cd] and [mkdir] to navigate to a
folder where you want to start the journey. I will use `$HOME/projects/` as a
my starting place:

```sh
$ mkdir $HOME/projects
$ cd $HOME/projects
```

### Configuring your Identity

We should introduce ourselves to Git so that the commits cary our name:

```sh
$ git config --global user.name "John Doe"
$ git config --global user.email "john.doe@notfound.com"
```

Above commands set the configuration variables `user.name` and `user.email`
globally.

### Configuring a Text Editor

For a number of operations Git requires the user to edit some text, e.g. for
providing extended commit messages. Git tries quite hard to find a suitable
text editor, but this search is geared heavily towards advanced Linux users.
For most users the [`vi`][vi] text editor is rather arcane and very unitutive
at first. Hence, let's switch out to more sensible alternatives. One such is
[_Visual Studio Code_][vscode].

### Creating a Repository

Next up we need a repository. There are two options:

1. Creating a new repository from scratch.
2. Copying an already existing repository from a remote server.

The former is very easy:

```sh
$ git init hello
Initialized empty Git repository in /home/wildmichael/projects/hello/.git/

$ cd hello
```

When starting a fresh repository I consider it to be good practice to
have an empty, pristine root commit from which everything starts:

```sh
$ git commit --allow-empty --message "Empty initial commit"
[master (root-commit) 7c5c13e] Emtpy initial commit
```

Copying a repository from a remote server is also easy once you know the
remote URL. The process is referred to as _cloning_, because quite literally
the full repository is being copied, i.e. cloned. There are, however, a
number of pitfalls and nitty-gritty-details, so it is best to follow the
instructions given by the code hosting platform. When using my very simple
_hello world_ repository the steps would look as follows:

```sh
$ git clone https://github.com/wildmichael/hello.git
Cloning into 'hello'...
remote: Enumerating objects: 503, done.
remote: Counting objects: 100% (503/503), done.
remote: Compressing objects: 100% (444/444), done.
remote: Total 503 (delta 79), reused 446 (delta 49), pack-reused 0
Receiving objects: 100% (503/503), 722.83 KiB | 1.82 MiB/s, done.
Resolving deltas: 100% (79/79), done.

$ cd hello
```

> ### Getting Help
>
> Git comes with excelent documentation and there is even a [free
> book][git-book]. The documentation is either available [online][git-docs],
> as so-called man-pages (named `git-<subcommand>`, i.e. `git-clone` or
> `git-init`) or directly from the commands themselves via the `-h` and
> `--help` options:
>
> ```sh
> $ git init -h
> usage: git init [-q | --quiet] [--bare] [--template=<template-directory>] [--shared[=<permissions>]] [<directory>]
>
>     --template <template-directory>
>                           directory from which templates will be used
>     --bare                create a bare repository
>     --shared[=<permissions>]
>                           specify that the git repository is to be shared amongst several users
>     -q, --quiet           be quiet
>     --separate-git-dir <gitdir>
>                           separate git dir from working tree
> ```
>
> The `--help` flag will start the man-page viewer on Linux-like operating
> systems. On Windows however, this will not just dump out raw text to the
> command line, but open a web browser with a nicely rendered HTML help page.

### Creating and Switching Branches

As outlined in the [previous article][part 1] it is recommended to not
directly work on the `master` branch. Hence, we will create the `develop`
branch from the reference `master` and switch to it:

```sh
$ git branch develop master
$ git checkout develop
Switched to branch 'develop'
```

### Adding Files

It is time to create a new text file named `hello.cxx` with below content
(skip if you `clone`ed the example repository):

```cpp
#include <iostream>

int main()
{
  std::cout << "Hello, World!\n";

  return 0;
}
```

Next we ask Git what is up:

```sh
$ git status
On branch develop

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        hello.cxx

nothing added to commit but untracked files present (use "git add" to track)
```

Here Git quite verbosely tells us which files it doesn't keep track yet of
and what we could do next. Let's follow the advice and _add_ the file and
then see how things changed:

```sh
$ git add hello.cxx
$ git status
On branch develop

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   hello.cxx
```

### Making a Commit

Finally, we are ready to make our first real commit:

```sh
$ git commit --message="Add hello world program"
[develop 8a54d8e] Add hello world program
 1 file changed, 8 insertions(+)
 create mode 100644 hello.cxx
```

Looking at the current state we see the following:

```sh
$ git status
On branch develop
nothing to commit, working tree clean
```

### Reading the Commit History

Using `git log` we can display the history of commits that lead up
to the latest commit in the currently checked out branch:

```sh
$ git log
commit 8a54d8ec65988a09d96c9755b7a54319cc910e4f (HEAD -> develop)
Author: John Doe <john.doe@notfound.com>
Date:   Sun Apr 12 18:05:35 2020 +0200

    Add hello world program

commit 7c5c13ef31832703730f39cf4d2bd9a691702c7c (master)
Author: John Doe <john.doe@notfound.com>
Date:   Sun Apr 12 17:57:55 2020 +0200

    Emtpy initial commit
```

Next we want to restrinct the displayed history to the commits that have been
added since `develop` branched off from `master`:

```sh
$ git log master..
commit 8a54d8ec65988a09d96c9755b7a54319cc910e4f (HEAD -> develop)
Author: John Doe <john.doe@notfound.com>
Date:   Sun Apr 12 18:05:35 2020 +0200

    Add hello world program
```

This is a short-hand notation for `git log master..HEAD` where `HEAD` again
is a convenience feature of Git that allows us to refer to the latest commit in the
currently checked out branch.

When the history gets longer it is impractical to just dump it to the
terminal and Git will instead _pipe_ it to a pager &ndash; an application
allows the user to scroll up and down through the content and to search it.
Most commonly `less` is used as the paging application. Please refer to its
[documentation][less] for usage instructions. Just as a small life-safer: The
`q` key exits the application.


[part 1]: ./2020-04-12-software-engineering-practices-part1.html
[VCS]: https://en.wikipedia.org/wiki/Version_control
[CD]: https://en.wikipedia.org/wiki/Continuous_delivery
[git-flow]: https://nvie.com/posts/a-successful-git-branching-model/
[git]: https://git-scm.org
[cd]: http://manpages.ubuntu.com/manpages/eoan/man1/cd.1posix.html
[mkdir]: http://manpages.ubuntu.com/manpages/eoan/man1/mkdir.1posix.html
[git-book]: https://git-scm.com/book
[git-docs]: https://git-scm.com/docs
[less]: https://manpages.ubuntu.com/manpages/eoan/en/man1/less.1.html
[vi]: https://en.wikipedia.org/wiki/Vi
[vscode]: https://code.visualstudio.com/