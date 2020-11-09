---
layout: post
title:  "Reach Your Next Level in Git Foo"
date:   2020-11-09 09:12:00 +0200
categories: programming
---

## Introduction

[Git] has become the de-facto standard [version control] system in the
software industry. Sure, there's an ecosystem of contemporary competitors,
both open-source and commercial, and quite a number of legacy systems still
in production use, but Git is certainly the most popular. Long past are the
days where it was so quirky only a select few were able to use it efficiently.
Now it is an extremely powerful tool with a dauntingly large feature set.
Pushing past the basics and developing higher skills that make you a better,
more productive, developer can be a lot of effort.

So, here goes: with this article I want to introduce a few concepts that go
beyond the basic `git add`, `git commit`, `git pull` and `git push` everybody
should know. And yes, pure command line. While some graphical tools offer an
advanced feature set, it is often best to understand what happens under the
hood first. If you then decide to use a graphical interface, that's perfectly
fine; but you know what's going on and you know that there are thing the
GUI doesn't allow you to do. After all, that's what it means to be 

First we'll look at some advanced staging/unstaging techniques. Next up
is _rebasing_, a number of commands and techniques that help creating a
coherent change history. Lastly, taking the rebasing chapter a step further,
we take a look at [Stacked Git] &ndash; a tool that provides an extremely
interactive rebase workflow.

> ### A Note On Text Editors
> The commands introduced in this article often prompt Git to open a text
> editor where the user is expected to make some changes. Owing to its
> Linux origins, Git by default uses the one and only text editor, [Vim]. 
> However, many users without a Linux background find Vim to be an extremely
> obtuse text editor whose mastery is on a level with black sorcery.
>
> If you are not familiar with Vim or otherwise prefer to use a different
> text editor, Git lets you make your choice, of course. As any good Linux
> citizen, it honors the `EDITOR` environment variable. However, most Windows
> users are not comfortable modifying their environment variables (for
> whatever reason...) and sometimes you might want Git to use a different
> editor than what you have configured in `EDITOR`. The `core.editor` setting
> allows you to provide any command that Git will invoke. E.g.
>
> ```
> $ git config --global core.editor emacs
> ```
>
> would configure the [Emacs] text editor. Not that you should. To configure
> [Notepad++] on Windows you could use
>
> ```
> $ git config --global core.editor "'C:/Program Files (x86)/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
> ```
>
> Of course, you would need to adjust the installation path to `notepad++.exe`.
> Users of [VS Code] have the following option:
> ```
> $ git config --global core.editor "code --wait"
> ```
>
> This assumes that the VS Code installer added the `code` command to your
> `PATH` environment variable. Make sure to include the `--wait` option.
> Without it, the `code` command will start a new instance of VS Code (or
> open the file in an existing one) and immediately return, letting Git think
> that you've already finished editing the file.

## Advanced Staging and Unstaging

Every Git tutorial show how to use the [`git add`] command to _stage_ one
or more files before actually performing the [`git commit`]. So I won't
bother with introducing these basics for the umpteenth time.

However, Git allows for much more fine grained control when adding changes to
the staging area. Commits should be very narrow in purpose. They should
introduce one change, and one change only. Git supports this by allowing you
to add individual diff hunks in place of all the changes at once to the
staging area. After all, that's the very reason the staging area exists.
Otherwise the approach of combining the staging and committing into a single
step, as e.g. taken by Subversion, would be perfectly sufficient.

### Hunk Selection And Editing

Git gives you three different, but related, ways of achieving this. The
first, and simplest, is the _patch mode_. It is invoked by
[`git add --patch`][git add], or `git add -p` for short. This starts and
interactive question-answer loop. Git groups all changes in all files
into so-called hunks and then prompts for each of those hunks whether it
should be staged or not. Large hunks can be further split into smaller hunks
for more fine grained control, and the hunk can even be edited by the user,
giving the user full control over what gets staged. For each hunk, Git
displays the unified diff and following prompt:

```
(2/3) Stage this hunk [y,n,q,a,d,k,K,j,J,g,/,s,e,?]?
```

The first part is a progress meter (hunk 1 of 3 in this case). The
possible answers you can give are:

* `y`: stage this hunk
* `n`: do not stage this hunk
* `q`: do not stage this hunk and quit
* `a`: stage this and all remaining hunks in this file
* `d`: do not stage this and all remaining hunks in this file
* `k`: leave this hunk undecided, go to previous undecided hunk
* `K`: leave this hunk undecided, go to previous hunk (whether decided or not)
* `j`: leave this hunk undecided, go to next undecided hunk
* `J`: leave this hunk undecided, go to next hunk (whether decided or not)
* `g`: select hunk number to go to
* `/`: search for a hunk matching the regex following `/`
* `e`: manually edit the current hunk
* `?`: display a help information explaining all the options.

Depending on the context, not all options are available. E.g. the options
`k` and `K` don't make sense for the first hunk, so they are not displayed
in the prompt. Also, the `k` and `j` options will not be available if there
are is no previous or next hunk that is undecided.

After typing in the character matching your choice confirm your selection
by pressing the <kbd>Enter</kbd> key. The `g` option can be followed by
the hunk index. If not given, Git will prompt for it. Analogously, the `/`
option can be followed by a regular expression to search for before hitting
<kbd>Enter</kbd>. Again, Git will prompt you for the regex if not provided.

The `e` option warrants some more explanation. When given, Git will open up
a text editor containing the diff hunk and some additional instructions, e.g:

```diff
# Manual hunk edit mode -- see bottom for a quick guide.
@@ -1,9 +1,9 @@
 namespace hello
 {
  
-class Program
+private class Program
 {
-    void Main()
+    internal void Main()
     {
         System.Diagnostics.Trace.WriteLine("Hello, World!");
     }
# ---
# To remove '-' lines, make them ' ' lines (context).
# To remove '+' lines, delete them.
# Lines starting with # will be removed.
#
# If the patch applies cleanly, the edited hunk will immediately be
# marked for staging.
# If it does not apply cleanly, you will be given an opportunity to
# edit again.  If all lines of the hunk are removed, then the edit is
# aborted and the hunk is left unchanged.
```

It is important to understand the file format. The first column of each
line indicates what is to be done with the remainder of the line:

* Lines starting with a `@@` are so-called _hunk headers_. They have the
  format `@@ -S1,N1 +S2,N2 @@` where `S1` is the hunk start line number of
  the original file, `N1` is the number of lines in the hunk in the original
  file, and `S2` and `N2` are the start line number and number of lines in the
  modified file. There should be no need to change these numbers when working
  with `git add --patch`. Git takes care of fiddling with these numbers
  (particularly if `N1` or `N2` would change due to edits).
* Lines with a `#` in the first column are comments and will be removed
* Lines with a <c>&blank;</c> (space) in the first column will be left as they
  are
* Lines with a `-` in the first column will be deleted
* Lines with a `+` in the first column will be added

To help the user (and make its own algorithm more robust) Git adds three lines
of unmodified context before and after the hunk.

Git diffs are, if not overridden, always line based. Hence, changing a single
character in a line will result in a full removal of the original line and a
full addition of the changed line.

You can now go ahead and go crazy:

* To skip removing a line, simply replace the leading `-` by a space character.
* To skip adding a line, simply delete it.
* To only partially apply a line change, keep the line deletion but modify
  the corresponding addition to match what you want to stage.

It is important that you do not simply delete the `+` or `-`. Doing so would
result in an invalid diff format and Git will reject your edits, giving you
the option to edit again or discard the edits.

The modifications you make to the diff will only affect the Git index (a.k.a
the staging area). The working directory copy remains untouched.

You might ask what all of this is good for.

![but-why]

As stated above, commits should have a well-defined, narrow purpose. However,
as happens quite often when working on a complex problem,  unrelate changes
are close together in a file, or even on the same line. Without using
`git add --patch` there would be no way of splitting these changes into
separate commits. Or rather, you'd have to monkey around with backing up
the modified file, retrieving the original copy and then manually re-apply
the changes again, committing them individually.

### Editing the Full Patch

Taking it one step further is the next method. `git add --edit` (or
`git add -e`) opens a text editor containing the diffs for all hunks
in all modified files. You can modify it exactly in the same way as
described above. In addition, if you want to skip a full hunk, you
delete it start to end (including the hunk header). Further, the
diff now contains file name information. You should leave it alone,
unless you want to prevent (or amend) a file-renaming. Only notice
that the original file path is prefixed with `a/` and the modified
file path with `b/`. The patch file could look like this:

```diff
diff --git a/foo.cs b/foo.cs
index a9cebe5..75ab5eb 100644
--- a/foo.cs
+++ b/foo.cs
@@ -1,6 +1,7 @@
 namespace hello
 {

-class Foo
+class Foo : ICloneable
 {
+    public override object Clone() => return new Foo();
 }

 }
diff --git a/hello.cs b/hello.cs
index f1f7a9a..90de285 100644
--- a/hello.cs
+++ b/hello.cs
@@ -1,7 +1,7 @@
-class Program
+private class Program
 {
-    void Main()
+    internal void Main()
     {
         System.Diagnostics.Trace.WriteLine("Hello, World!");
     }
 }
```

The line starting with `index` is for Git's internal use. It should
definitely left unmolested.

The advantage of `git add --edit` over `git add --patch` is that it can be
substantially quicker to work with. However, editing a patch file manually
takes some practice and it is quite easy to get completely confused and get
lost.

The last method of advanced staging is the _interactive mode_ invoked with
[`git add --interactive`] (or `git add -i`). It is a menu-based automation
tool for existing Git commands. For the above displayed differences it would
show the following:

```
  1:        +7/-0      nothing foo.cs
  2:        +7/-0        +2/-2 hello.cs

*** Commands ***
  1: status       2: update       3: revert       4: add untracked
  5: patch        6: diff         7: quit         8: help
What now>
```

You can either use the numbers or first letters of the command names.
A number of commands will prompt for more inputs. E.g. the `patch` command
starts a process similar to `git add --patch`, but first prompts the user
to select which files it should run on. The `update` command, on the other
hand simply stages all changes, similar to `git add --update`. The `diff`
command shows the staged changes, exactly what `git diff --cached` would
do. As you can see, the interactive mode provides the only advantage that
you don't have to remember so many git commands and their option flags.

#### Unstaging

The old-school way of unstaging was to use the [`git revert`] command.
However, this command is problematic as it is used in a number of
different contexts, doing very different things.

Recent versions of Git got the [`git restore`] command with a better
defined purpose that should be less ambiguous and confusing. Passing
it a file name would simply unstage the full file. To have more control,
the `git restore --patch` option can be used. It offers the exact same
workflow as `git add --patch`. However, notice that when editing the hunk,
the diff is in _reverse_ direction. I.e. it describes what will be unstaged.
Lines starting with `+` will be _removed_ from the index, lines starting
with `-` will be _added_ to the index. This can become very confusing real
quick. Personally I often find it easier to simply unstage the whole file
and start over, except for very simple scenarios.

![confusing]

## Cleaning Up with Rebasing

### Basics

Many developers flinch when they hear the word _rebasing_. And then their
eyes go blank. Somehow rebasing has gained the reputation of being very hard
to use and difficult to understand. Let me reassure you, this is not the
case.

So, what is it about? Rebasing describes the process of temporarily removing
a series of commits, re-applying them on top of another starting point in the
Git history. Say we have the following Git history:

```
    A     B     C     D
--- o --- o --- o --- o  develop
     \ 
      o --- o --- o --- o feature/super-duper
```

Here, a branch called `feature/super-duper` has been created, branching off
from the main development branch `develop` at the commit `A`. Unfortunately,
it turns out that some functionality has been added to `develop` in commit
`C` that is required for the further development of `feature/super-duper`.
In traditional workflows there would have been only two options:

1. Cherry-pick the required code from commit `C` into `feature/super-duper`,
   creating the new (modified) commit `C'`. Git supports this with the [`git
   cherry-pick`] command. Unfortunately that means the same changes are now
   present in two (or more) branches:

   ```
       A     B     C     D
   --- o --- o --- o --- o  develop
       \ 
         o --- o --- o --- o --- o feature/super-duper
         E     F     G     H     C'
   ```

2. Merge the `develop` branch into the `feature/super-duper` branch. This is
   probably the option most developers would have gone for. The drawback is
   that the commit graph gets messy and the "railway tracks" are hard to
   follow and that the commits `A`, `B` and `D` that might not have been
   strictly required are now also merged.

   ```
       A     B     C     D
   --- o --- o --- o --- o ---  develop
       \                       \
         o --- o --- o --- o --- o feature/super-duper
         E     F     G     H     M
   ```

There is, however, another option if discard the idea of Git history being
unmodifiable. You could move all the commits from the `feature/super-duper`
branch starting after commit `A` and reattach them to commit `C`:

```
    A     B     C     D
--- o --- o --- o --- o develop
                 \
                   o --- o --- o --- o feature/super-duper
                   E'    F'    G'    H'
```

What Git does when you apply this process can be described as follows:

* For each of the commits `E` through `H` create the _diff_ (or _patch_) file.
* Reset the `feaure/super-duper` branch to the state of the commit `D`.
* Re-apply the previously generated _patches_ in order, creating now the new
  commits `E'` through `H'`.

Just as with traditional merging, reapplying the patches can result in
conflicts that need to be resolved by the developer. There's one advantage,
however: the conflicts are always in the context of a single commit. When
performing a branch merge, all the changes generate potential conflicts
at once. It can become really difficult to understand why a certain change
was introduced and how it should be resolved. When rebasing, however, each
conflict is clearly associated with the commit that introduced it and it
becomes easier to understand what the resolution should be.

Let's look at the [`git rebase`] command that would perform above action:

```sh
$ git rebase --onto <new_base> <upstream> <branch>
```

Here, `<new_base>` is the commit we want to attach the new series to.
`<upstream>` is the reference given to Git so it knows where to start
creating the patch series. As is usual with Git, `<upstream>` will be
the last commit **not** included in the patch series. Hence, Git will
_cut_ the commits after `<upstream>` until `<branch>` and then reapply
them starting from `<new_base>`.

Our artificial example from above would be produced by the following
command:

```sh
$ git rebase --onto C A feature/super-duper
```

There's a few shortcuts we can take:

* `<upstream>` is not necessarily a direct parent commit of `<branch>`.
  It could as well be `<develop>`. Git would then automatically figure out
  the newest common commit and use that as the point after which to apply
  the scissors.
* `--onto <new_base>` can also be omitted if it coincides with `<upstream>`.
* Lastly, `<branch>` defaults to the currently checked out branch if not given.

Hence, if we wanted to rebase `feature/super-duper` onto `develop` instead of
the intermediate commit `C` and `feature/super-duper` is the currently
checked-out branch, we could have used the much simpler command:

```sh
$ git rebase develop
```

> ### Avoiding Merges When Pulling
> When multiple programmers work on the same branch it often happens that
> a direct `git push` doesn't work because another colleague already has
> pushed new commits to the common branch. The traditional approach is to
> first `git pull` the new changes and the push the merge result back to
> the remote. However, this workflow creates a lot of unnecessary and
> difficult to follow merge commits.
>
> For this reason Git introduced an alternative workflow. Instead of the
> traditional `git fetch`+`git merge` model for `git pull`, there is now
> a `git fetch`+`git rebase` type of workflow which can be enabled by using
> the `git pull --rebase` command. With this option, Git performs the
> `git fetch` operation as usual, but instead of performing a merge with
> the tracking branch, it rebases the local branch on top of the remote
> branch.
>
> It is highly recommendable to use this workflow instead. Git even allows
> it to be the default by setting the following configuration option:
>
> ```sh
> git config pull.rebase true
> ```
>
> If you want to apply this setting globally, use the following instead:
>
> ```sh
> git config --global pull.rebase true
> ```

### Recovering from Conflicts

I mentioned above that rebasing can result in conflicts. If this happens,
Git stops the rebase process and leaves your working directory in a conflicted
state, just as `git merge` would do. It is now your task to resolve all
conflicts and stage the fixes with `git add`. However, instead of committing
yourself, you can use the command `git rebase --continue` to get things going
again.

Sometimes it happens that a conflict occurs and when you inspect the changes
you realize that the commit is no longer necessary at all. In such a case
Git allows you to jump over this commit by first cleaning up the working
directory (`git reset --hard` &ndash; careful with that one!) and then using
`git rebase --skip`.

If you bungled things and you want to abort the rebasing process (to maybe
start over), you can use `git rebase --abort`.

### Words of Caution

Rebasing is disruptive. You should think twice, and then again, before you
rebase commits that you have already pushed to a remote server. This is
particularly true for _stable_ branches, such as `main` (previously called
`master`) and `develop`. Your team members will be very confused when they
can't simply `git pull` anymore. It is preferable that within the team you
set up a few ground rules, such as:

* `main` and `develop` should not be rebased. Most Git servers have some
  policy setting to prevent _force-pushes_ to certain branches. It would be a
  good idea to apply such a setting to these branches.
* `feature` and `bugfix` branches are volatile. They can and should be rebased
  frequently by their owners. Everybody on the team knows this and hence should
  not be taken by surprise if a `git pull` fails.
* `feature` and `bugfix` branches should not be based on another `feature` or
  `bugfix` branch. The volatility of the base branch would require the dependent
  branches to be also rebased frequently; something that becomes quickly quite
  confusing and difficult to get right consistently.

## Advanced Rebasing

Now we know what rebasing is and how it works. But there's much more to it than
just moving a patch series from one place to another. Git rebasing allows you
to modify the series by reordering it, dropping patches, rewriting commit
messages, amending commits, squashing one or more commits into one, etc.
These possibilities make it the perfect tool to clean up a messy _"commit
early, commit often"_ feature branch before it is merged into the main
development branch.. No one is interested in seeing all the dead ends, all
the attempts that didn't work out, all the mistakes in the line of _"Ooops I
forgot to add this file"_ and all the typos and their fixes.

As mentioned in the introduction, Git commit series should represent a
logical sequence of changes taken to drive in a certain feature or bugfix.
Git is not about maintaining an absolutely accurate historical record of
all the mistakes ever made. It is about answering questions like:

* What did the source code contain when we released version X.Y.Z?
* Who can I ask about this method or class that I don't understand?
* What was the problem being solved with this implementation?

When trying to find answers to such questions, a messy commit history is just
the worst. Sifting out the trivial commits, dissecting commits that contain
unrelated changes, connecting separate commits that fix earlier commits
because there was a typo or some file missing, all of this is very tedious
and a huge cognitive load for anyone, even if it is yourself in a few months
time trying to figure out what the heck you were thinking when you wrote this
stinking pile of garbage.

Let's get started with `git rebase --interactive`. Assume we have below
commit graph and commit log of the `feature/refactor-string-handling` branch:

```
  5a3fc
--- o --- o --- o --- o  develop
     \ 
      o --- o --- o --- o feature/refactor-string-handling
    1a2b3 2c3e4 3e4f5 4f5a6

2c3e4 Refactor to use StringExtensions
1a2b3 Forgot to add class StringExtensions for common string handling
3e4f5 Fix typos in StringExtensions
4f5a6 Fix formatting errors when using StringExtensions
```

As you can see, the first two commits look reasonable if you ignore the fact
that they are in the wrong order. You could still argue whether they should
be separate commits or a single commit; however, the next two (`3e4f5` and
`4f5a6`) are basically fixes for previous mistakes.

Say you want to clean this mess up, you would do the following (provided
`feature/refactor-string-handling` is your current branch):

```sh
$ git rebase --interactive 5a3fc
```

This command will pop up a text editor with the following _rebase script_ as
its contents:

```
pick 2c3e4 Refactor to use StringExtensions
pick 1a2b3 Forgot to add class StringExtensions for common string handling
pick 3e4f5 Fix typos in StringExtensions
pick 4f5a6 Fix formatting errors when using StringExtensions

# Rebase 5a3fc..4f5a6  onto 5a3fc (7 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

As you can see, Git is being helpful here by including the instructions at
the bottom of the script. As soon as you save and close the text editor,
Git will launch the rebasing process.

Only the first two columns matter. The commit descriptions are only there
to help you identify the commits. The first column describes the operation,
the second the commit the operation applies to. Lines starting with a
pound character (`#`) are comments and will be ignored.

Also, it is worth noting that the order of commits is top to bottom (older
commits first, latest commit last).

To bring commits into a more logical order, you can simply reorder the lines.
Notice that doing so is often likely to result in conflicts if the commit
being moved affects files that are also modified by commits it is moved past.

If a commit is completely useless, simply delete the respective line.
Alternatively, replace `pick` with `delete` or `d` in the first column.

To change only the commit message of a given commit, replace the word `pick`
in the first column by `reword`, or `r` for short. Once the rebase starts and
Git gets to this commit, it will display a text editor allowing you to modify
the commit message.

If you want to stop at a certain commit you can use `edit` or `e`. Git will
then stop at this commit and e.g. allow you to modify some files and amend
the commit with `git commit --amend`. Or you could also make some
modifications and create a new commit that will then be included in the Git
history. As with conflicts, you can continue the rebase process with
`git rebase --continue`.

To combine one or multiple commits into one we can use the `squash` (`s`)
or `fixup` (`f`) actions. Both of them will fold the changes of the commit
into the preceding commit. The difference is that `squash` will give you
the opportunity to modify the commit message, while `fixup` is really
intended for fix-ups where the original commit message is to be kept and
the message of the fix will be discarded. If multiple commits should be
folded, just move their lines to be in sequence.

For now, ignore the other options explained in the comments. They are
mostly used when the rebase spans a merge, something you should try
to avoid if possible. We'll look at this scenario in a minute, but for
now let's finish our cleanup.

We'll want to do two things:

1. Fix the ordering of the first two commits.
2. Fold the fixes into their respective commits.

With the previous explanations in mind, we come up with the following
commit script:

```
pick 1a2b3 Forgot to add class StringExtensions for common string handling
squash 3e4f5 Fix typos in StringExtensions
pick 2c3e4 Refactor to use StringExtensions
fixup 4f5a6 Fix formatting errors when using StringExtensions
```

Was you can see, I decided that I want to keep the commit adding the
`StringExtensions` class, but that also means I should change the commit
message. I could do this by using the `reword` action. However, it is
followed by a fix-up commit. Using the `squash` action instead of `fixup` we
can kill two birds with one stone and handle the rewording of the commit
message also there. Then follows the commit where the code is refactored to
_use_ the `StringExtensions` class. Other than the reordering, no action is
required and the `pick` action is fine. At the end follows the last fix, for
which I have chosen the `fixup` action.

After saving the file and closing the text editor, Git applies the rebase
script. Provided no conflicts occurred, Git should now prompt you to
edit the commit message for the first commit. When closing the text editor,
Git will finish the rebase and the commit graph and log should look like
this:

```
  5a3fc
--- o --- o --- o --- o  develop
     \ 
      o --- o  feature/refactor-string-handling
    1e7a2 92af5

1e7a2 Add class StringExtensions for common string handling
92af5 Refactor to use StringExtensions
```

So much better. Clear steps &ndash; no mess.

![hackerman]

### Rebasing Across Merges

For a very long time Git didn't really support rebasing when one of the
spanned commits was a merge commit. Or rather, it did, but in an unexpected way:
Instead of preserving the merge commit, Git replaced it with the individual
commits from the merged branch(es) that were not previously present in the
target branch. For illustration, assume a history that looked like this:

```
    A     B     M     C
--- o --- o --- o --- o  some-branch
     \         /
      o ----- o  other-branch
      D       E
```

Now you find a mistake has been made in commit `B` an you want to fix it up
by squashing `C` into it. Naively doing

```sh
$ git rebase --interactive A
```

would result in the following rebase script:

```
pick B ...
pick D ...
pick E ...
pick C ...
```

That's right, Git dropped the merge commit `M` and inlined the commits `D`
and `E` in its place. Probably not what we had in mind.

![disappointed]

Recent versions of Git, however, have an option to preserve the merges:

```sh
$ git rebase --rebase-merges --interactive A
```

This would greet you with the following rebase script:

```
label onto

# branch other-branch
reset onto
pick D ...
pick E ...
label other-branch

reset onto
pick B ...
merge -C M other-branch # Merge branch 'other-branch' into 'some-branch'
pick C ...
```

Wow, **that** is something else... Let's try to understand this:

* First, a named label `onto` is created that can be referred to later.
* Then the whole state is reset to this label. Unnecessary now, but it helps
  when you start reordering the lines.
* Next, the commits from the branch `other-branch` are being picked.
* A new label `other-branch` is created.
* The state is reset to the `onto` label.
* The commits from the `some-branch` branch are being picked.
* The label `other-branch` is merged. The `-C M` option is there to copy the
  message from commit `M`. You could also use `-c M` to get a prompt allowing
  you to modify the message. Lastly, you could leave the option altogether away
  and simply edit the comment following `#` to provide a new merge commit
  message.
* Lastly, the commit `C` that followed the original merge commit `M` is picked.

Having dissected this, it becomes clear that this isn't a single rebase. It is
at least two rebases in one script as the `other-branch` is being separately
rebased and then merged into `some-branch`.

So, if `C` is the fix-up commit for `B`, we would modify the script as follows:

```
label onto

# branch other-branch
reset onto
pick D ...
pick E ...
label other-branch

reset onto
pick B ...
fixup C ... # <-- APPLY THE FIXUP HERE
merge -C M other-branch # Merge branch 'other-branch' into 'some-branch'
```

All of this is quite involved and trying to explain it makes one look a bit
like Charlie:

![pepe silvia]

Take my advice on this: only ever rebase across merges if you really can't
avoid it. E.g. if you need to modify a commit that disclosed some kind of
secret, like an encryption key or a password. Cleaning up your feature or
bugfix branch should not require this.

## Conclusions

This article has introduced the most important tools for a clean Git
history: Selective staging/unstaging and interactive rebasing. As it is,
this post has become quite a bit lengthy. Instead of making it even longer
I will take it a bit further and introduce the fantastic tool [Stacked Git]
in my next post.

[Git]: https://git-scm.org
[version control]: https://en.wikipedia.org/wiki/Version_control
[Vim]: https://www.vim.org
[Emacs]: https://www.gnu.org/software/emacs/
[VS Code]: https://code.visualstudio.com/
[`git add`]: https://git-scm.com/docs/git-add
[`git commit`]: https://git-scm.com/docs/git-commit
[git add]: https://git-scm.com/docs/git-add
[but-why]: https://media.giphy.com/media/s239QJIh56sRW/giphy.gif
[`git add --interactive`]: https://git-scm.com/docs/git-add#_interactive_mode
[`git revert`]: https://git-scm.com/docs/git-revert
[`git restore`]: https://git-scm.com/docs/git-restore
[confusing]: https://media.giphy.com/media/3o7btPCcdNniyf0ArS/giphy.gif
[`git cherry-pick`]: https://git-scm.com/docs/git-cherry-pick
[`git rebase`]: https://git-scm.com/docs/git-rebase
[hackerman]: https://media.giphy.com/media/QbumCX9HFFDQA/giphy.gif
[disappointed]: https://media.giphy.com/media/3eNx5SV39lH6o/giphy.gif
[pepe silvia]: https://media.giphy.com/media/l0IylOPCNkiqOgMyA/giphy.gif
[Stacked Git]: https://stacked-git.github.io/