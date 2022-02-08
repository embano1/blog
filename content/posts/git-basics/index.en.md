---
title: "Git rebase, squash...oh my!"
subtitle: ""
date: 2021-05-31T21:43:57+02:00
lastmod: 2021-05-31T21:43:57+02:00
draft: false
description: "Ever wanted to contribute to an open source project but quickly got overwhelmed with git terminology? I'm glad you asked..."

tags: ["git","open source"]

resources:
- name: "featured-image"
  src: "featured-image.jpg"

toc:
  enable: true
lightgallery: true
---

<!--more-->

## Happy little Trees and bumpy Roads

Your first contribution to an open source project can be a very rewarding
experience. Once your feature, fix or enhancement as part of a Github pull
request[^gitlab] (PR) is merged, your brain is likely going to start looking to
solve the next problem in open source land with your newly acquired `git` foo
🙅‍♀️

If you've already went through this exercise though, you also know that
mastering the `git` Swiss Army knife, and related code collaboration platforms
(Github, etc.), might feel like black art and overwhelming. This is especially
true, when things go south, e.g. the reviewer requests changes to your code.

But even in the case where your code contribution is fine, the reviewer(s) might
ask you to perform additional steps on your commits before accepting ("merging")
them into the upstream ("target") repository.

Here's a short list of typical reviewer (or bot) comments you might see during
a review:

- "You must sign all of your commits!"
- "Please squash your commits before merging."
- "Can you please rebase your commits onto the latest changes?"
- "Please follow our contribution guidelines and add `XYZ` to the message
  title/body"

Let me tell you that I've been in the same situation several times, both as a
contributor and reviewer, e.g. on the [VMware Event Broker
Appliance](https://github.com/vmware-samples/vcenter-event-broker-appliance)
(VEBA) project.

I have witnessed many times how quickly an enthusiastic (first-time)
contributor, often with little to no knowledge about `git` and software
development, can *desperately fail* due to tooling, terminology or cryptic error
messages like the one below:

```bash
git push upstream HEAD:issue-1
To <some_remote_URL>
 ! [rejected]        HEAD -> issue-1 (non-fast-forward)
error: failed to push some refs to '<some_remote_URL>'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

Over the time I established a set of patterns for my daily work with `git`. They
help me to stay organized and protect me from common mistakes, e.g. pushing to
the wrong branch/repository. This is even more important when I work on multiple
PRs in parallel. 

Don‘t ask how I know 😎

Of course, I adjust these patterns here and there, depending on the contribution
guidelines/workflows of a project or team I‘m working with. I hope you find them
useful, especially when you are blocked or frustrated 😄

{{< admonition type=info title="Note" open=true >}}

This post aims to address some common challenges a `git` novice might face
during a contribution, e.g. via Github pull request. The advice and best
practices given here are definitely opinionated based on my own observations and
the way I `git` things done, so take them with the typical grain of salt. 

Basics or internals of git and related tooling and platforms, such as Github,
won't be covered. See the [end](#references) of this post for some useful links
with details on `git`(hub) concepts, workflows and internals.

{{< /admonition >}}

## Before you're `git`-ting started

Always make sure you *read and understand the project‘s contribution guidelines*
and follow the provided issue/pull request templates before you start coding and
open a pull request. 

If the project does not provide any of these, first search for related issues.
If your idea/fix has not already been discussed, open an issue to *avoid lengthy
discussions* during a review on why you filed a PR. Upfront, clear and friendly
communication is key in an open source project.

## How I `git` Things done

The nice thing with `git` when dealing with repositories, aka `remotes`, is that
it does not differ between a remote URL or local folder. We can use this to our
advantage here to show some of concepts in action without creating a repository
on Github.

{{< admonition type=info open=true >}}

Github is not just a fancy UI for the `git` CLI. Thus the latter lacks Github
related primitives to work with issues, releases, pull requests, etc. I highly
recommend [installing](https://cli.github.com/) the Github CLI `gh` as a
productivity booster.

{{< /admonition >}}

Let's create a local demo repository `awesome-project` to understand the
patterns explained in the subsequent sections. 

```bash
mkdir awesome-project && cd awesome-project
git init .
Initialized empty Git repository in /Users/mgasch/awesome-project/.git/

# Seed repository with the first commit
touch README.md && git add README.md && git commit -s -m "Add README"
```

We can verify that a commit was created with the powerful `git log` command.

```bash
git log --oneline --decorate --graph --all
* fdb642a (HEAD -> main) Add README
```

To avoid having to remember all the different command line flags, I heavily rely
on `$SHELL` aliases, e.g. `gloga` which is a shortcut to the command above. 

{{< admonition type=tip open=true >}}

Many `$SHELL` and plugin managers, e.g.
[ohmyzsh](https://github.com/ohmyzsh/ohmyzsh) provide useful pre-defined `git`
aliases. For clarity, I will use the full commands in this post though.

{{< /admonition >}}

Let's create a couple more commits to make this post a bit more realistic.

```bash
for i in {1..5}; do touch file_${i}; git add file_${i}; git commit -s -m "feat: add feature ${i}" -m "Closes: #${i}"; done
[output omitted]

git log --oneline --decorate --graph --all
* aa08aaf (HEAD -> main) feat: add feature 5
* 3f00922 feat: add feature 4
* 525a0eb feat: add feature 3
* d800962 feat: add feature 2
* a78afae feat: add feature 1
* fdb642a Add README
```

Ignore the text that is added to each commit with the `-m` flags for now. I'll
come back to them in the [Commits](#commits) section.

### Forks in the Road

By default, you can't make direct changes to a repository (unless you are a
owner or maintainer). That's why Github established a fork/pull request
[workflow](https://docs.github.com/en/github/collaborating-with-pull-requests/getting-started/about-collaborative-development-models)
for contributions.

A fork creates a point-in-time *clone* of the original (`"origin"` or
`"upstream"`) repository in your own account. Your changes always are made in
that forked repository. Then you can create (open) a pull request on the
`origin`.

The `git` CLI does not have a concept of forks. But we can mimic the workflow
locally with a plain ol' *clone*. Even though we won't be able to cover the full
`origin/fork/local_fork_clone` lifecycle with the following examples, they help
to keep the complexity at a minimum, while focusing on useful patterns.

```bash
# leave the origin repository folder
cd ..

# simulate a fork by creating a named clone
git clone awesome-project fork-awesome-project
Cloning into 'fork-awesome-project'...
done.

cd fork-awesome-project
```

A nice thing is that `git` will remember where the clone was created from and
automatically configures the clone's `main` branch to track `origin/main`.

```bash
# show remote details
git remote show origin
* remote origin
  Fetch URL: /Users/mgasch/awesome-project
  Push  URL: /Users/mgasch/awesome-project
  HEAD branch: main
  Remote branch:
    main tracked
  Local branch configured for 'git pull':
    main merges with remote main
  Local ref configured for 'git push':
    main pushes to main (up to date)

# branch tracking details
git branch -avv
* main                aa08aaf [origin/main] feat: add feature 5
  remotes/origin/HEAD -> origin/main
  remotes/origin/main aa08aaf feat: add feature 5
```

We can also quickly verify that the clone is identical to the source, i.e.
`origin`.

```bash
git log --oneline --decorate --graph --all
* aa08aaf (HEAD -> main, origin/main, origin/HEAD) feat: add feature 5
* 3f00922 feat: add feature 4
* 525a0eb feat: add feature 3
* d800962 feat: add feature 2
* a78afae feat: add feature 1
* fdb642a Add README
```

Here, `HEAD -> main` tells you your current position (commit) in the
current repository (well, folder). But also note the additional `origin/main`
and `origin/HEAD` *references*. They show you the position of the `HEAD` and
the `main`[^master] branch in the `origin` repository. 

And there's the first catch: in a real-world scenario, more commits might have
been already added to the `origin` repository. Thus, it's important to keep up
with the remote's changes to avoid issues at a later stage, e.g. merge conflicts
due to concurrent changes on the same code path. 

{{< admonition type=tip open=true >}}

`HEAD`, branches or tags like `main` respectively `v1.0.2` are nothing but
human-readable
[references](https://git-scm.com/book/en/v2/Git-Internals-Git-References), aka
`refs`, in a `git` repository pointing to a specific commit SHA.

{{< /admonition >}}

The topic of synchronizing with a remote will be covered in [keeping your fork
in sync](#keeping-your-fork-in-sync). But since one can easily get lost with all
the different repositories, I'll cover some important naming patterns first.

### Naming Conventions
#### Remotes

The first step I perform after creating a clone is *renaming* the `origin`
remote to something more meaningful, e.g. "upstream".

```bash
git remote rename origin upstream

# show last commit only
git log --oneline -1
aa08aaf (HEAD -> main, upstream/main, upstream/HEAD) feat: add feature 5
```

To me, `upstream/main` reads more naturally. But feel free to pick whatever name
you prefer to not get lost.

{{< admonition type=tip open=true >}}

`git clone` provides a `-o (--origin)` flag to provide a custom remote name during
a clone operation.

{{< /admonition >}}

In addition, and to prevent us from *accidentally pushing* to the wrong `remote`,
we can configure a dummy URL for `git push`. With this little trick, `git` will
prevent us from pushing to the `upstream` repository. 

```bash
# no_push is an invalid URL
git remote set-url --push upstream no_push

# oops, I did not meant to push to upstream
git push upstream
fatal: 'no_push' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

You might be wondering why this is needed since typically you would not have
`push` permissions to the upstream anyways. Well, over time you might actually
become a collaborator or even maintainer of a project. That's great, but if
you're not careful your commit(s) end up in the wrong repository...especially
when you don't follow an intuitive `remote` naming strategy.

Here's another tip: If we'd used a real remote repository, e.g. Github, as an
example here, you would have cloned a fork to your local machine. Depending on
the platform and tooling, the fork's `remote` might show up as `origin`, whereas
the "real" source repository ("upstream") might not show up at all when you
perform a `git remote show`. As usual, there's an easy fix.

```bash
# assumes origin is actually the fork
git remote rename origin fork

# add the real upstream, e.g. the VEBA project
git remote add upstream git@github.com:vmware-samples/vcenter-event-broker-appliance

# fetch information for all repositories and prune stale branches
git fetch -avp
[output omitted]
```

#### Branches

For branch names I settled on *using Github issue numbers*, which has a couple
of benefits:

1) I'm forced to create an issue upfront which is a good thing anyways
2) I can directly tell which branch belongs to which issue(s)
3) I can easily clean up merged branches by pattern matching

Once you work on multiple issues/branches in parallel, this approach has saved
me several times from accidentally pushing the wrong code. Whatever naming
pattern you pick, as long as it's consistent (can be automated) you're fine.

Here is an example using the `gh` CLI to create an issue and respective branch.

```bash
# can also be done through Github UI
gh issue create --title "ABC is broken" --assignee "@me" --body "ABC is broken because of XYZ"
Creating issue in <some_remote>

https://<some_remote>/issues/20

# use issue number above and create and switch to new branch
git checkout -b issue-20
Switched to a new branch 'issue-20'

# create commits...
```

{{< admonition type=tip open=true >}}

`git checkout -` brings you back to your previous branch.

{{< /admonition >}}

Once my branches are merged, I can easily clean them up too.

```bash
# fetch the latest repository information
git fetch -avp
[output omitted]

git branch --merged | egrep "issue-" | xargs git branch -d
[output omitted]
```

The `-d` flag tells `git` to only delete branches which have been fully merged.
Note that this command might not work for you if the repository does not create
merge commits for PRs. In that case (or if you really want to get rid of your
branch for whatever reason), use the brute-force way `git branch -D` instead.

### Keeping your Fork in Sync

Over time, especially in very active projects, the `upstream` repository will be
ahead of your `fork` in terms of commits. Thus, you need to keep up with these
`upstream` changes and synchronize them into your `fork`.

Let's create a couple more commits in our example `awesome-project` to make this
concrete.

```bash
# assumes we're in the fork directory
cd ../awesome-project

# create commits
for i in {6..10}; do touch file_${i}; git add file_${i}; git commit -s -m "feat: add feature ${i}" -m "Closes: #${i}"; done
[output omitted]

git log --oneline --decorate --graph --all
* 3c86329 (HEAD -> main) feat: add feature 10
* f2a1e6a feat: add feature 9
* 17b3210 feat: add feature 8
* f4896b0 feat: add feature 7
* 68d3931 feat: add feature 6
* aa08aaf feat: add feature 5
* 3f00922 feat: add feature 4
* 525a0eb feat: add feature 3
* d800962 feat: add feature 2
* a78afae feat: add feature 1
* fdb642a Add README
```

Now we need to switch back to the fork and bring it in sync with `upstream`.

```bash
cd ../fork-awesome-project

# fetch information about our recent changes
git fetch -avp
[output omitted]

git log --oneline --decorate --graph --all
* 3c86329 (upstream/main, upstream/HEAD) feat: add feature 10
* f2a1e6a feat: add feature 9
* 17b3210 feat: add feature 8
* f4896b0 feat: add feature 7
* 68d3931 feat: add feature 6
* aa08aaf (HEAD -> main) feat: add feature 5
* 3f00922 feat: add feature 4
[snip]
```

In the above commands, I only fetched the information about changes, but not the
changes (commits) themselves. You can see this because our current position,
i.e. `HEAD`, is still pointing to `main` at commit SHA `aa08aaf`.

My intention here is, that I want to train your muscle memory with an
alternative way instead of the usual `git pull` to get the job done.

{{< admonition type=tip open=true >}}

If you're still confused about that `HEAD` thing, think of it as the "you are
here" marker on a map to indicate the current position. As you traverse through
the map (commits and `refs`), `HEAD` will move accordingly.

{{< /admonition >}}

#### Enter Rebase

[Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) is an amazing,
but often misunderstood, concept in `git`. It allows you to move `refs` and even
commits onto another commit, branch, or any other `ref`.

Concerning our example above, we want to move the current position of our forked
`main` branch (`HEAD`) to match its `upstream/main` counterpart. Since we have a
linear history without any conflicting changes between the two repositories,
bringing `main` in sync with `upstream/main` is as simple as:

```bash
# executed within the fork's main branch
git rebase upstream/main
Successfully rebased and updated refs/heads/main.

git log --oneline --decorate --graph --all
* 3c86329 (HEAD -> main, upstream/main, upstream/HEAD) feat: add feature 10
* f2a1e6a feat: add feature 9
* 17b3210 feat: add feature 8
* f4896b0 feat: add feature 7
* 68d3931 feat: add feature 6
[snip]
```

Because you will perform code changes in dedicated branches, i.e. not `main` (or
the corresponding "primary" branch in a project), merge or rebasing conflicts
should never arise. Due to the linear commit history, `git rebase` can simply
move the `HEAD` to the desired `ref`. 

My recommendation: rebasing instead of `git merge` should become your *default
way* of synchronizing (integrating) changes from `remote` into local branches.
You will see the real strength of `git rebase` in the [Oh my `git`!](#oh-my-git)
section below...

### Commits

#### What goes in a Message?

A commit represents an *atomic change* in the *append-only history* in `git`,
like in a log. Thus, it's important that a well crafted commit includes a
meaningful but brief description about the included changes.

{{< admonition type=info title="Note" open=true >}}

Details on what actually goes into a commit (code, documentation, tests, etc.)
are out of scope of this article.

{{< /admonition >}}

Even though the tooling might not enforce a standard, e.g. character limit,
etc., there are certain commit best practices you should know and follow. I
usually point newcomers to this great post: [How to write a commit
message](https://chris.beams.io/posts/git-commit/).

Read it? Great, let's move on...

You might have also wondered about the `-m` flags and text like `"feat:"` used
in the earlier examples when executing `git commit`. 

Let's take this example: `git commit -s -m "feat: add feature X" -m "Closes:
#34"`

The `-m` flag can be used multiple times and replaces the interactive editor
which otherwise comes up to create a commit title and message. Even multi-line
strings are supported (just keep typing after the first `"`).

Depending on the project you're contributing to, it might use certain patterns
in a commit title or body to craft a
[CHANGELOG](https://keepachangelog.com/en/1.1.0/). Here `"feat:"` is recognized
and the commit will be highlighted in a "Feature" section if the project uses
this. Take a look at this example from the
[VEBA](https://github.com/vmware-samples/vcenter-event-broker-appliance/releases/tag/v0.6.1)
project.

{{< admonition type=tip open=true >}}

Read the project's CONTRIBUTION guide or ask the maintainers how (if) commit
messages should be crafted.

{{< /admonition >}}

If you include a
[keyword](https://docs.github.com/en/issues/tracking-your-work-with-issues/creating-issues/linking-a-pull-request-to-an-issue#linking-a-pull-request-to-an-issue-using-a-keyword)
like `"Closes: #34"` in the commit message body and open a pull request, Github
will try to automatically link the specified issue (here `#34`) to the PR so it
gets automatically closed after the PR is merged. Just like with the
aforementioned prefixes, these keywords might be used in a `CHANGELOG` or release
note.

Lastly, the `-s` flag will add a `Signed-off-by` footer which is a [good
practice](http://web.archive.org/web/20160507011446/http://gerrit.googlecode.com/svn/documentation/2.0/user-signedoffby.html)
to traceback the author of a patch and is often required in projects. Don't
confuse this with [digitally
signing](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work) your
commits, though.

#### When to commit?

Perhaps the biggest question is when to create a commit and how to break them up
into individual chunks. 

The latter really depends on the type of work and project guidelines (if any).
In my opinion, everything that belongs together ("atomic") goes into the same
commit, including documentation updates. This makes it simpler to revert or
cherrypick changes.

If you work on a larger pull request, likely this can be broken down into
sub-tasks which nicely map to individual commits. Alternatively, the PR itself
might have to be broken up into multiple PRs with smaller (single) commits.

Creating commits sounds easy, right? Well, often you don't know upfront the
scope of your work and how (when) to break them up. It might also be that the
maintainers ask you to do so after the fact (see the following section). 

If you're paranoid like me, you might actually want to commit often and push to
a remote (fork) as a cheap backup in case you badly messed up on your local
machine...or your disk dies suddenly and the backup (you do backups, don't you?)
is from two days ago 😱

Long story short, in most cases you simply don't know in advance what the final
commit history will be. Your (temporary) commit history might actually look like
mine, ehhh this one:

```bash
# in descending commit time order
* 7010aa5 I need another job
* 3b3361e So close...
* 58327d5 Damn it!
* 0a4652e Another fix
* b1cc213 WIP
```

Let me tell you that this *is the norm* and not bad at all! As usual with `git`,
there's a couple of ways out of this mess. My preferred one is using `git reset`
once my work is ready for a PR.

Here's how to fix it based on the commit history above in our fork and the
branch `issue-31`:

```bash
# we are ahead of main by couple of commits
git log --oneline --decorate --graph --all main^..HEAD
* 7010aa5 (HEAD -> issue-31) I need another job
* 3b3361e So close...
* 5832755 Damn it!
* 0a4652e Another fix
* b1cc213 WIP
* 3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10

# soft-reset our HEAD to main to unstage all our commited changes (nothing is lost!)
git reset main

# rinse and repeat the following two steps if you want multiple commits
# add our changes again
git add <stuff>

# create a nice commit
git commit -s -m "fix: Fix bug in ABC" -m "Closes: #31"

# HEUREKA!
git log --oneline --decorate --graph --all main^..HEAD
* 96c6889 (HEAD -> issue-31) fix: Fix bug in ABC
* 3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

The `reset` command is super convenient to revert committed changes as if they'd
never been committed - but without losing the actual changes. They get simply
marked as `unstaged` in your working directory again. Think of `reset` as the
reverse action of `commit`.

{{< admonition type=tip open=true >}}

There might be other (temporary) changes in your branch which you did not want
to include in a commit. To get rid of these changes use `git checkout .`. This
will discard all remaining changes in your local git repository. `checkout` also
accepts a path/file name if you want to be more specific in what gets discarded.

{{< /admonition >}}

The `reset` command also comes in handy if you want to rearrange what goes into
a commit. Say you want `awesome_script.sh` not in the first but second commit.
Simply (soft) reset your commits to a previous state (commit) as shown above and
create individual commits again with the desired contents.

{{< admonition type=warning open=true >}}

Be carful though: `git reset` has a `--hard` option which will discard all
changes. If this is not what you wanted `git reflog` is there to help...

{{< /admonition >}}

## Oh my `git`!

Finally, I am going to cover some of the typical challenges you might run into
during your first pull request...or in case you keep forgetting this stuff 😄

The following sections are broken up into typical conversations you might
experience during a PR review.

### "How do I commit during reviews?"

Depending on the source code management (SCM) system you're using, e.g.
[Github.com](https://github.com/), addressing review comments can quickly become
an issue - at least for the reviewer. 

Most contributors would make the change and then `force-push` to the pull
request instead of adding and pushing review commit(s). This way of addressing
review comments means more work for reviewers, especially on large pull (merge)
requests.

Depending on the SCM you use, commits from `force-pushes` might not be rendered
nicely so that the reviewer often sees all instead of the incremental changes.

{{< admonition type=tip open=true >}}

In a Github pull request, you can click on the "\<user\> force-pushed ..." text
in the bottom of the PR to see the actual changes. That (diff) link is not
immediately obvious and falls short when a rebase was performed as well.

{{< /admonition >}}

Ideally the contributor creates additional commits during a review because many
SCMs have support for incremental changes. Even if your SCM does not support
such a functionality, the reviewer can easily inspect the individual commits if
needed.

Once the review is done, either all commits are automatically squashed and
merged into one commit (depending on the SCM/repository setup) or one manually
performs a `rebase` operation (see further below for ways to do this).

Of course, in that case you would still need to eventually `force-push` your
changes. So are we back to the beginning? 

Well, yes and no. On Github the reviewer can inspect `force-push` changes and
would then see an empty diff, signaling that nothing has changed since the last
review commit[^diff] (see tip above). Furthermore, I expect the SCM vendors and
platform providers to improve the user experience for this common scenario.

#### Scenario with multiple Review Commits

**Problem:** You are going through a lengthly review and want to easily merge
your review commits eventually.

```bash
# current commit history
git log main^..HEAD --oneline
96c6889 (HEAD -> issue-31) fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

**Solution:**

During the review you are additional commits using the `--fixup` flag
to easily squash them once the review is done.

```bash
# we had to make a change to the README
git add README.md

# simply commit, referencing the commit ID this one should be squashed into later
git commit --fixup 96c6889
[issue-31 05ff5a2] fixup! fix: Fix bug in ABC
 1 file changed, 1 insertion(+)

# after another round we were asked to make a change in main.go
git add main.go

# keep fixup-ing
git commit --fixup 96c6889
[issue-31 b1936b6] fixup! fix: Fix bug in ABC
 1 file changed, 1 insertion(+)
 create mode 100644 main.go
```

Here's our new history.

```bash
b1936b6 (HEAD -> issue-31) fixup! fix: Fix bug in ABC
05ff5a2 fixup! fix: Fix bug in ABC
96c6889 fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

Notice how the `git` CLI automatically added the `fixup!` prefix to the review
commits to indicate that these commits shall be squashed into the corresponding
reference commit (`96c6889`).

Once the review is done and you are asked to squash your commits `git` will do
most of the work for you.

```bash
# perform an interactive rebase with the --autosquash option
git rebase main -i --autosquash

# the editor opens, marking the fixup commits for squashing
1 pick 6fad9e1 fix: Fix bug in ABC
2 fixup 05ff5a2 fixup! fix: Fix bug in ABC
3 fixup b1936b6 fixup! fix: Fix bug in ABC

# then save and leave the editor, e.g. :wq (vi)
Successfully rebased and updated refs/heads/issue-31.

git log main^..HEAD --oneline
817078c (HEAD -> issue-31) fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10

# push the final commit
git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

### "You forgot to sign your Commit(s)"

Often you will be greeted by a friendly 🤖 stating that you *forgot to sign* one
or more commits. This could also happen after a rebase/squash (see below), where
you did not sign the resulting commit.

#### Scenario with one Commit

**Problem:** Commit `96c6889` in branch `issue-31` has not been signed-off and
is part of a PR in the `upstream` repository.

```bash
# current commit history
git log main^..HEAD --oneline
96c6889 (HEAD -> issue-31) fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

**Solution:** *Amend the commit* and `force-push` to the fork (so it gets
reflected in the associated PR).

```bash
# keep commit contents/message as is but sign it off (-s)
git commit --amend --no-edit -s
[issue-31 a2817f0] fix: Fix bug in ABC
 Date: Wed Jun 9 14:48:26 2021 +0200
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 file_fixed.sh

git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

Remember: you *cannot change existing commits* in `git`. Thus under the hood,
amending creates a new commit (`a2817f0`) with the exact same content/message
and replaces the previous one. This rewrites the history and thus would be
rejected during a `git push` to a remote. The flag `--force-with-lease`
instructs `git` to ignore such errors and *forcefully* overwrite the `remote`'s
history too.

{{< admonition type=warning open=true >}}

As you might have guessed, options with `--force` in the name can be dangerous
as `git` won't get in your way to protect you. Here we use a preferred option
`--force-with-lease` which I highly recommend using instead for [various
reasons](https://stackoverflow.com/a/52823955).

{{< /admonition >}}

#### Scenario with multiple Commits

**Problem:** Multiple (or all) commits in a branch are not signed off.

```bash
# oops, couple commits here are not signed
git log main^..HEAD --oneline
4f5c00c (HEAD -> issue-31) fix: Fix file 4
d73b7f6 fix: Fix file 3
890d79a fix: Fix file 2
a2817f0 fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

**Solution:**

Perform a `rebase` against the parent branch (`main`) to change all commits in
the current branch (`issue-31`). Use `--exec` to execute arbitrary commands,
such as signing off by amending.

```bash
git rebase --exec 'git commit --amend --no-edit -s' main
Executing: git commit --amend --no-edit -s
[detached HEAD 1bfdba6] fix: Fix bug in ABC
[snip]

# push changes
git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

{{< admonition type=tip open=true >}}

Use the `-i` option in the `rebase` command above and an interactive dialog will
open where you can skip certain commits (and do other fancy rebase stuff).

{{< /admonition >}}

### "Please squash your Commits"

**Problem:** Some repositories might have been set up in a way that they can't
(or won't) *squash* ("collapse") multiple commits of a PR into a single one
during merging. The contributor must then squash these commits and (force) push
again before merging.

Continuing with our signed-off commits from the example above, there're two ways
you can achieve this.

#### Option 1: `git rebase`

```bash
# current commit history
git log main^..HEAD --oneline
6c3fa63 (HEAD -> issue-31) fix: Fix file 4
870c066 fix: Fix file 3
2d6777e fix: Fix file 2
1bfdba6 fix: Fix bug in ABC
3c86329 (upstream/main, upstream/HEAD, main) feat: add feature 10
```

With `git rebase` you can interactively perform a squash operation:

```bash
# executed from issue-31 branch
git rebase -i main

# an interactive editor opens with the following options
# note commit order is ascending from old to new
pick 1bfdba6 fix: Fix bug in ABC
pick 2d6777e fix: Fix file 2
pick 870c066 fix: Fix file 3
pick 6c3fa63 fix: Fix file 4

# Rebase 3c86329..6c3fa63 onto 3c86329 (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
[snip]
```

Replace `pick` with `s` (short for squash) for all but the first commit and exit
the editor (e.g. `:wq` in `vim`):

```bash
pick 1bfdba6 fix: Fix bug in ABC
s 2d6777e fix: Fix file 2
s 870c066 fix: Fix file 3
s 6c3fa63 fix: Fix file 4
```

{{< admonition type=tip open=true >}}

You can also reorder commits during an interactive rebase.

{{< /admonition >}}

Upon exit, a new editor window will pop up allowing you to specify the title and
message for the resulting (single) commit. Squashing preserves commit details
which you can use to craft a proper message. If you don't want to reuse the
previous commit details, use `f` (short for `fixup`) instead of `s` in the step
above.

Once you're done don't forget to push to your fork/PR:

```bash
git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

{{< admonition type=tip open=true >}}

Did you know that `vim` supports [multi-line
editing](https://stackoverflow.com/questions/253380/how-to-insert-text-at-beginning-of-a-multi-line-selection-in-vi-vim/253391#253391)
with `CTRL-V` so you don't have to replace every single `pick` line?

{{< /admonition >}}

#### Option 2: `git reset`

This is my preferred option in this case, as it's usually the faster approach
but your mileage may vary 😉

```bash
# executed from issue-31 branch
# soft-reset back to main
git reset main

# all changes are unstaged
git status
On branch issue-31
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        file_fixed.sh
        fix2.sh
        fix3.sh
        fix4.sh

# add all changes and commit
git add .
git commit -s -m "fix: Fix bug in ABC" -m "Closes: #31"

# push to remote
git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

The only drawback I can see here is that you have to retype your commit
title/message, unlike with `squash` which preserves these details.

### "Your PR needs a Rebase"

**Problem:** The repository owners configured the target ("base") branch to only
allow merges from up-to-date branches. In most cases though, this message will
come up when conflicting changes were introduced to the base branch (e.g.
`main`) while you were working on your PR.

**Solution:** The actual fix to this problem is similar to what we did in the
squash scenario [above](#option-1-git-rebase). We continue our example with
branch `issue-31` and execute the following steps within that branch.

```bash
# fetch index about recent changes on upstream
git fetch --all --prune

# the current history
git log --oneline --decorate --graph --all
* fdb244e (upstream/main, upstream/HEAD) Change ABC to include Y
* 75a0bb2 Add feature 11
| * 673a5b0 (HEAD -> issue-31) fix: Fix bug in ABC
|/
* 3c86329 (main) feat: add feature 10
* f2a1e6a feat: add feature 9
* 17b3210 feat: add feature 8
[snip]
```

The above history tells you that your `HEAD` is still originating from `main`
(which is stale, i.e. behind `upstream/main`). We could first sync the recent
changes from `upstream/main` into `main`, but this is not required for this
exercise.

Instead, we move (`rebase`) our `HEAD` (commit) onto `upstream/main` so our
branch is up to date again.

```bash
git rebase upstream/main
Successfully rebased and updated refs/heads/issue-31.
```

{{< admonition type=tip open=true >}}

This is a trivial example without conflicts. If a conflict arises, rebase will
tell you what to do. Many editors, IDEs and tools like
[GitKraken](https://www.gitkraken.com/) provide visual assistance to resolve
merge/rebase conflicts.  At any time you can abort a rebase with `git rebase
--abort`.

{{< /admonition >}}

After we verified that everything worked, we can push again. As we are not
changing history, i.e. `HEAD` moves forward, a `force` option is not required.

```bash
git log --oneline --decorate --graph --all
* 5beac9c (HEAD -> issue-31) fix: Fix bug in ABC
* fdb244e (upstream/main, upstream/HEAD) Change ABC to include Y
* 75a0bb2 Add feature 11
* 3c86329 (main) feat: add feature 10
 
git push fork HEAD:issue-31
[output omitted]
```

### "Please adjust your Commit Message"

**Problem:** You might have forgotten to include some details, like a prefix or
issue reference in one or more commit messages.

#### Scenario with one Commit

**Solution:** Amend your commit.

```bash
# this will open an editor where you can make your changes
git commit --amend -s

# push to remote
git push fork HEAD:issue-31 --force-with-lease
[output omitted]
```

#### Scenario with multiple Commits

**Solution:** Perform an interactive rebase. Let's continue with our previous
`issue-31` example branch which for this exercise contains two commits that need
to be changed.

```bash
# current status
git log upstream/main^..HEAD --oneline
a2f3897 (HEAD -> issue-31) fix: Fix bug in XYZ
33e3dcf fix: Fix bug in ABC
fdb244e (upstream/main, upstream/HEAD) Change ABC to include Y

# perform interactive rebase
git rebase -i upstream/main
```

An editor shows up again where you replace `pick` with `r` (short for `reword`)
on the commits you want to adjust the message.

```bash
r 33e3dcf fix: Fix bug in ABC
r a2f3897 fix: Fix bug in XYZ
```

After you exit this window, for each selected commit a new editor window opens
where you can perform your changes. Then perform a `force-push` as in the other
examples.

## Conclusion

I have seen so many enthusiastic newcomers getting stuck or frustrated in an
open source project due to tooling or technical terms. Not too seldom ending up
dropping the ball on a contribution. This has nothing to do with how smart you
are or that these tools and platforms are only for "real developers".

I hope the real-world examples, my very opinionated best practices and
references below help you to easily navigate `git` and it's huge ecosystem, e.g.
the Github platform. 

I want **YOU** to shine as a leading example and inspire even more people to
contribute their ideas, documentation fixes or other forms of improvements to
the world of open source. No matter how "big" your contribution is, every useful
commit counts!

And now, `git` your feet wet 😉

## References

### Further Reading

- [How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)
- [Interactive Git Cheatsheet](http://ndpsoftware.com/git-cheatsheet.html)
- [Official Github Guides](https://guides.github.com/)
- [The Pro Git Book (free)](https://git-scm.com/book/en/v2)
- [Mastering Git Tutorials](https://thoughtbot.com/upcase/mastering-git)
- The legendary [Oh Shit, Git!?!](https://ohshitgit.com/)
- [On undoing, fixing, or removing commits in git](https://sethrobertson.github.io/GitFixUm/fixup.html)
- Step by step Tutorial [Contributing to the VEBA
  Project](http://www.patrickkremer.com/2019/12/vcenter-event-broker-appliance-part-v-contributing-to-the-veba-project/)

### Credits

Thanks to [Robert Guske](https://twitter.com/vmw_rguske) for your thorough review! Title photo by <a
href="https://unsplash.com/@synkevych?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Roman
Synkevych</a> on <a
href="https://unsplash.com/s/photos/git?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
[^gitlab]: Or merge request in Gitlab

[^master]: Formerly the default branch was usually "master" but has been changed
to reduce offensive language.

[^diff]: Unless a rebase against the `main` trunk was involved which would show
    all the commits from these operations.