---
title: "Git rebase, squash...oh my!"
subtitle: ""
date: 2021-05-31T21:43:57+02:00
lastmod: 2021-05-31T21:43:57+02:00
draft: false
description: "Ever wanted to contribute to an open source project but quickly got overwhelmed with git terminology? Glad you came here..."

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
request[^gitlab] is merged, your brain is likely going to start looking to solve
the next problem in open source land with your just acquired `git` foo.

If you've already went through this exercise though, you also know that
mastering the `git` Swiss army knife, and related code collaboration platforms
(Github, etc.), feels like black art and overwhelming. This is especially true,
when things go south, e.g. the reviewer requests changes to your code.

But even in the case where you code contribution is fine, the reviewer(s) might
ask you to perform additional steps on your commits before accepting ("merging")
them into the upstream ("target") repository.

Here is a short list of typical reviewer (or bot) comments you might see during
a review:

- "You must sign all of your commits!"
- "Please squash your commits before merging."
- "Can you please rebase your commits onto the latest changes?"
- "Please follow our contribution guidelines and add `XYZ` to the message
  title/body"

Let me tell you that I've been in the same situation several times, both as a
contributor and reviewer, e.g. on the
[VEBA](https://github.com/vmware-samples/vcenter-event-broker-appliance)
project.

I have witnessed many times how quickly an enthusiastic (first-time)
contributor, often with little to no knowledge about `git` and software
development, can desperately fail due to tooling, terminology or weird `git`
error messages like the one below:

```terminal
git push upstream HEAD:issue-1
To <some_remote_URL>
 ! [rejected]        HEAD -> issue-1 (non-fast-forward)
error: failed to push some refs to '<some_remote_URL>'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

{{< admonition type=info title="Note" open=true >}}

This article intentionally does not cover the basics or internals of git and
related tooling and platforms, such as Github. See the end of this post for some
useful links.

{{< /admonition >}}

## Further Reading

https://chris.beams.io/posts/git-commit/
http://ndpsoftware.com/git-cheatsheet.html
https://guides.github.com/

https://git-scm.com/book/en/v2

https://ohshitgit.com/
https://sethrobertson.github.io/GitFixUm/fixup.html

## Credits

Title photo by <a
href="https://unsplash.com/@synkevych?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Roman
Synkevych</a> on <a
href="https://unsplash.com/s/photos/git?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
[^gitlab]: Or merge request in Gitlab