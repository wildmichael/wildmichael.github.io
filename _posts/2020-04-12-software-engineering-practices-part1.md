---
layout: post
title:  "Software Engineering Practices - Part 1"
date:   2020-04-12 12:48:00 +0200
categories: programming
---

This is the first part of a small series on software engineering practices.
While topics such as requirements engineering, portfolio management,
architecture and design are of immense importance I will mostly focus on the
"craftmanship" surrounding programming. Consequently, I hope that this
content will also prove relevant for wider range of programming environments
and even applications, such as machine learning and data analysis.

In this first part I will talk about version control and how to use it to
implement a process that helps delivering high quality software while
remaining agile and nimble.

## Introduction

When creating a product that gets shipped to production it is of immense
importance that the creators of the product keep track of _what_ was
delivered _when_ to _whom_. Also, it is vital to have processes in place
that allow the creators to understand and reason about the product after
it has been delivered. If the clients report problems, guarantees are not
met, bugs appear, all this information is quintessential for the
troubleshooting. If the creators of the product no longer understand _why_
the product is as it is, or dont' even know _what exactly_ was delivered
to the client, they run the risk of not being able to fix the problem in
a timely manner because they first have to laboriously determine the
state of the product the client has a problem with and then reverse
engineer it in order to understand it.

Consequently, if you are creating a product, no matter whether it is
machinery, software or a service, you must have processes and tools
in place that allow you to find out _why_ the product is as it is and
_when_ you delivered _what_ to your client. Ideally, those processes
are integrated with your product development workflow. If they are not,
the project members will not apply them because they will be perceived
as overhead and an unnecessary drag that slows them down.

What does all of this mean for software engineering?

First off, it mandates the use of a [_version control system_ (VCS)][VCS].
This is the one tool that allows you to keep track of

* what was changed when and by whom.
* when was what shipped.

Second, you need solid processes around using the VCS that ensure
that high quality information is stored in it. As so often the
proverb "garbage in, garbage out" applies. If not properly used
the mere presence of a VCS will not help you. Many people use
the VCS as if it was an overly sophisticated backup system. If used
in such a way the information contained in the VCS will be of very
little value.

So, this article will first introduce the processes that I find to
be useful and then in the follow-up article I will proceed to explain the
mechanics from the perspective of the programmer

## Process

As mentioned in the introduction, processes need to be embedded in the
workflow. The assumption is, as mentioned, that the software is delivered
in releases with considerable time between deliveries. The advantage of
such a process is that it is much easier to maintain a stable production
system. It follows that it is useful to define two main classes of releases:
Feature and hotfix releases. The former deliver new features and low-priority
bug fixes to the user. These releases often require significant efforts in
testing to ensure that the new features work as intended and that their
introduction did not accidentally break existing functionality.

Hotfix releases, on the other hand, redress faults and bugs in the system
that slipped through QA and that need to be repaired ASAP. To reduce the
amount of testing necessary before shipping the fix, no new features must
be included in this type of release. Also, it is very important that if
the software communicates with other systems, that the changes do not modify
these interfaces in an incompatible way that would break the production
environment for the users.

Here I present a development management process for software that is largely
inspired by an article describing a workflow that became to be known as [Git
Flow][git-flow]. The process presented here is not the "latest fad" in that
it does not attempt to achieve the holy grail of [continous delivery
(CD)][CD]. It is a more traditional approach where developments are bundled
into so-called releases that are shipped to the clients. Whether these
releases occur at periodic intervals (such as every month, quarter, half
year) or "whenever it is ready" does not change the below described process.

It leverages the VCS to implement a process that ensures that the creators
of the product always know what state of the code was used when in production,
that allows for the controlled introduction of so-called hot fixes that are
not mingled with new, and potentially untested, new features and that promotes
the good practice of code review (a.k.a. the four or six eyes principle).

All VCS track their content as a series of snapshots, so-called _commits_
that capture a certain state and a given point in time. Each of these
snapshots is associated with an author and a description that should briefly
explain what was changed and why. It is these commits that allow your future
self to reason about why things are as they are now. Hence it is very
important to craft good commits that are easy to understand. Each commit
should be motivated and have a well defined purpose and scope.

If unrelated changes are intermixed it is difficult to understand why a
certain modification was made. Also, if non-functional changes (such as
code-formatting) to unrelated parts of the source code are mixed in with
functional modifications the signal-to-noise-ratio is reduced unnecessarily.
All of these bad things happen if the VCS is used like a backup system. I.e.
it is very bad practice to just commit everything "every once in a while".

Also, it is important that the commits are telling a coherent story. After a
few months no one will be interested in a detailed history of how a certain
feature came to be with all the trials, failures, tears and sweat. Make sure
that the commit history paints a clear picture of what modifications were
necessary to introduce a new feature. Later I will show the techniques
related to creating a good commit series.

Any VCS worth its salt will have the notion of _branches_. Branches allow the
team to apply changes to the source code on "different tracks" that do not
interfere with each other. This very useful feature is used here to keep
what is in _production_ (i.e. was shipped) separate from what is being worked
on for the next release. The production branch is traditionally called
`master` or `trunk`. The branch that tracks the changes for the next
release is often called `develop` or `next`. While you can choose any name
you want, sticking with conventions makes things easier for new project
members and project members that frequently have to switch between projects.
Hence I will call these branches `master` and `develop` since that is what
is most commonly used when working with [Git][git]. Visualizing
the commit history as a graph, the situation could look like this:

```lang-none
(root) o--o--o--o--o develop
        \
         o---------- master

       ------------> time
```

Here both branches start at a common root-commit and from then on have
independent lives. The circles indicate commits and time advances from
left to right.

### Release Process

As development progresses the product eventually reaches a state where it is
considered to be feature-complete and it should soon be released. Such a
release usually involves extensive testing by selected key users and
(hopefully not too much) bug-fixing, often referred to as the _beta-phase_.
Also, the release process often involves a number of steps, such as
increasing version numbers. The testing, bugfixing and release preparation
phase naturally takes some time during which no new features should be
introduced. However, it often is inconvenient or inappropriate to stop all
development during the release phase. Hence, the release process should
be split out into a separate _release branch_ that only lives as long as
it takes to finish the release. Usually it is named after the version number
with `release/` prefixed; i.e. for version _1.2.0_ the branch name would be
`release/1.2.0`. The branch-visualization is hence augmented by a new branch:

```lang-none
(root) o--o--o--o--o develop
        \     \
         \     o-o-o release/1.2.0
          \
           o-------- master
```

The question is now the following: How does the it all end? How does the
released state end up in `master` and how does it find its way back into
`develop`? Because for sure, all the bug-fixes that were applied to the
release branch should end up there too.

That is where _merging_ comes in. A _merge_ is the joining of two (or more)
branches into one. Technically, it means that in the commit graph there
is a commit with more than one parent. For our release-process this means
that the _release branch_ is merged first into `master`, recording the state
with which our product was shipped. Second, the `master` branch is merged
back into `develop`, thus bringing back all the changes made during the
release process to the main development branch. The commit graph hence looks
like this:

```lang-none
(root) o--o--o--o--o--o--o develop
        \     \         /
         \     o-o-o   /   release/1.2.0
          \   v1.2.0\ /
           o---------o     master
```

In order to uniquely identify the released version number (or release name)
on the `master` branch a so-called _tag_ is applied. Tags are markers that
give individual commits a special name that can be referenced. The name
of the tag is the version number, prefixed with `v` (to distinguish it from
other tags you might want to have). In our case it is `v1.2.0`, labelled
top-left of the tagged commit.

### Feature Development

Often new features are quite complex and take some time until they are
finished. When working in a team it is not desireable that the histories of
unrelated features are interleaved with each other on the `develop` branch.
This would make it very hard to reason about a certain feature in the future.
Hence, it is good practice to split out the work on these features into
so-called _feature-branches_. It is conventional to give these branches a
short, descriptive name that is prefixed with `feature/`. E.g. a feature with
the description _"Add shopping cart"_ would be called
`feature/add-shopping-cart`. When working with a issue tracking system (which is
highly recommended) then it is a good idea to include the ticket number in
the branch name. Say the ticket for the shopping cart feature had the number
`SHOP-102`, the branch name should be `feature/SHOP-102-add-shopping-cart`.
Once the feature is done, again merging is used to bring it back into
the main development branch. Returning to the graph visualization, an
additional family of feature-branches is added:

```lang-none
           o--o----o-- feature/SHOP-102-add-shopping-cart (ongoing)
          /
         /     o---o   feature/SHOP-130-add-payment-integration (merged and finished)
        /     /     \
(root) o--o--o--o----o develop
        \
         o------------ master
```

Sometimes it can be useful to also have non-feature branches for other types
of development, such as non-critical, low-priority bug-fixes that do not need
to be delivered ASAP to the users. For such developments it might be a good
idea to add other side branch prefixes, such as `bugfix/`.

### Hotfix Releases

As described at the beginning of the section, hotfix releases must not
introduce new features that would extend the testing period. Hence their
development should not branch off `develop` as other feature and bugfix
branches do. Instead, they should start from `master` to ensure that only the
problem fix is introduced. Otherwise the process is the same as for release
branches:

```lang-none
(root) o--o--o-o--o--o--o--o develop
        \                 /
         \   o---o-o-o   /   hotfix/1.2.1
          \ /   v1.2.1\ /
           o-----------o     master
```

### Maintenance Branches

Quite often software is sold to include bug fixes but no new features during
a certain maintenane period. My advice: Try to avoid this if you can. It
introduces a huge amount of complexity for your software development processes
that is hard to justify by the potential business benefits. However, probably
you are not in a position to make this decision, so here we go.

For each version that was sold with bugfix maintenance to one or more
clients, a so-called maintenance branch is added. Initially these branches
start at their respective version-tag on `master`. Whenever a new bug is
discovered, the oldest (alive) maintenance branch that exhibits the bug
must be identified. A hotfix branch is started from there and when finished,
merged back into its maintenance branch. That maintenance branch is then again
merged into the next younger maintenance branch, and so on, until the youngest
is merged into `master` which finally is then merged into `develop`. This chain
of merges can be stopped early if the problem does not appear in younger
maintenance branches, the current `master` or `develop`. Also, it might be
necessary to develop the same bugfix multiple times using multiple hotfix
branches if the underlying implementation has been changed too much by
feature developments in between releases. The commit-graph could look
as shown below:

```lang-none
----o-o--o--o--o--o--o-o---o--o--o---o--o--o-o--o-o--o-o--o--o--o-o---o-- develop
       \            /   \           /       /       /            /
        o--o-o     /     \         /       /       /            /         release/1.2.0
              \   /       o-o-o   /       /       /            /          release/1.3.0
v1.1.0   v1.2.0\ /       v1.3.0\ / v1.3.1/ v1.3.2/      v1.3.3/
------o---------o---------------o-------o-------o------------o----------- master
       \         \           o-o-o-o   /       /            /             hotfix/1.2.1
        \         \         / v1.2.1\ / v1.2.2/      v1.2.3/
         \         \-----------------o-------o------------o-------------- maint/1.2.x
          \                       o-o-o     /            /                hotfix/1.1.1
           \                     /     \   /  o--o--o   /                 hotfix/1.1.2
            \                   / v1.1.1\ /  / v1.1.2\ /
             \ --------------------------o------------o------------------ maint/1.1.x
```

As can be seen, dealing with maintenance branches is non-trivial and with
every feature-release that needs to be maintained the complexity increases
considerably.

### Pull Requests

So far the overall process has been layed out. One big question remains,
however: How can the process be used to ensure early quality checks and
implement a four or six-eyes principle?

When looking at the workflow layed out in the previous section, the merging
points stand out as the natural place where peer-checks are to be applied.
That means that before merging any of the branches, a request must be filed
with one or two colleagues who verify the work, provide feedback and
sometimes require modifications before agreeing for the modifications to be
merged. This is what is commonly called a _pull request_ (PR). Many VCS
hosting platforms, such as Microsoft Azure DevOps, Atlassian Bitbucket,
GitHub or Gitlab allow policies to be defined how many positive reviews
are necessary for merging and assist the reviewer by providing tools for
commenting on the proposed changes.

However, pull requests are not only useful for the gatekeeping of feature
and bugfix branch merges, but can also be used when implementing QA
procedures before merging a release or hotfix branch into `master`.

More importantly, PR's not only fulfill a pure QA function, as vital as that
already is. No, they also help a team to spread know-how. Firstly, the
reviewer gets to see what has changed and where, in a sometimes vast code
base, functionality is located that was previously unknown. Also, reviewers
and submitters get to exchange opinions about design, techniques and style.
Quite often neat tricks can be learned, either by the requester or the
reviewers. In general the practice of code reviewing will make the team more
resilient against individuals dropping out, be it due to illnes, vacation or
job change.

### Summary

The product development process is supported by a low-overhead workflow in the VCS.
There are two main branches that track the product:

1. The `master` branch only ever represents the current state of the released
   product.
2. The `develop` branch contains all the developments that will lead up to
   the next release.

To isolate the testing and stabilization phase before a formal release, a
so-called _release branch_ is opened that is named `release/<version-name>`.
When finished, the release branch is merged into `master` where the release
is tagged. Finally, `master` is merged back into `develop` to finish the
cycle.

High-priority fixes that must be applied to the production release as
soon as possible without introducing new, potentially incomplete or
untested features, are branched off from the release-tag on `master`.
After that they are treated just like release branches. Instead of
using the `release/` prefix the `hotfix/` prefix is used to name
the branch.

Features and bugfixes that require more than a single commit should be
also developed in _feature_ or _bugfix_ branches. Their names should be
of the form `feature/<issue-id>-<feature-name>` and
`bugfix/<issue-id>-<bug-fix-description>`.

So-called pull-requests are used to implement the four- or six-eye principle.
They give the requester and the reviewers the opportunity to learn from each
other and inicidentally improve the overall quality of the software. Whether
a sophisticated tool is used for the submission and reviewing or a simple
E-Mail conversation is used is not that important.

If the product needs to maintain older versions, so-called _maintenance
branches_ come into the picture. They branch off from their respective
release tag on the `master` branch and are named `maint/<version-name>`. For
each feature, bugfix or hotfix that needs to be developed, the oldest
supported maintenance branch for which the modification is necessary is
identified and it is used to branch off a feature or bugfix branch and then a
release branch or, hopefully more common, a hotfix branch. When finished, the
release or hotfix branch is merged into the maintenance branch and the
release is tagged. This release is then merged to the next maintenance branch,
where it again is tagged, and so on until the last branch is reached where
the modification needs to be applied. If this is `master`, it is finally
merged back into `develop`. If at some stage the merging does not simply work
because a feature between the versions make modifications or even a
re-development of the fix or feature necessary, the merging chain can be
interrupted and the process is restarted from the maintenance branch where
the merging failed. The complexity of dealing with maintenance branches is
considerable and the business decision whether do so, and for how long a
version is maintained, should be carefully weighed against the costs.

## Next Up

Now that the overall development process has been introduced, the next part
will explore the concrete tools and steps that a programmer needs in order to
follow the workflow using [Git][git]. While most VCS will be capabable enough
to support the described workflow, Git is probably the most widely adopted
VCS and offers some unique features that make it particularly well suited.
Hence the next article will focus on this tool.

[VCS]: https://en.wikipedia.org/wiki/Version_control
[CD]: https://en.wikipedia.org/wiki/Continuous_delivery
[git-flow]: https://nvie.com/posts/a-successful-git-branching-model/
[git]: https://git-scm.org