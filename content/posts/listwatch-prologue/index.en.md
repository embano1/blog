---
title: "The Anatomy of Kubernetes ListWatch(): Prologue"
subtitle: ""
date: 2021-01-15T17:28:15+01:00
lastmod: 2021-01-15T17:28:15+01:00
draft: false
description: "A multi-part deep dive series into the internals of the Kubernetes ListerWatcher interface"
page:
  theme: "classic"
upd: ""
authorComment: ""
tags: ["Kubernetes","Events","boltdb","etcd","watch"]
hiddenFromHomePage: false
hiddenFromSearch: false
resources:
- name: "featured-image"
  src: "featured-image.jpg"
toc:
  enable: false
math:
  enable: false
lightgallery: true
license: ""

---

<!--more-->

When I was in high school, I took languages (German, English) as my major
classes. Reading, interpreting and writing about literature, i.e. books, became
a daunting task. And to be honest with you, I don't recall much from the
Shakespeare's and other legends of past centuries. But there was one book which
I still remember. 

Perhaps you have heard about Johann Wolfgang von Goethe and one of the most
famous books in German literature: Faust: A Tragedy. In this book, Goethe
describes *the restless pursuit of knowledge* and the never-satisfied desires of
a person who is dissatisfied with his life. In order to satisfy his need for
knowledge and pleasure, Faust dedicated himself to the devil and to the life
(and love) of a person.

{{< image src="faust-1.jpg" caption="Dr. Faust (Source: Britannica)" width=300 >}}

With the ever increasing rate of innovation in information technology, many of
us have a desire to continuously learn and grow, seeking for answers to small
and big questions, like Dr. Faust. Reflecting on myself, this part of a
monologue by Faust became a theme throughout my whole career (and still is). 

> *„To enlighten me more, What holds the world together at its innermost core“*
>
> (Faust, J. W. Goethe)

Whenever I am confronted with a complex problem space or piece of technology, I
try to *distill it to its essence*, its core. What are the building blocks
(atomic units)? How are they assembled to form a more complex structure? What
are the decisions and tradeoffs the architects had to make? How did these
influence the overall design?

Not only does this help me to better understand whether and how to optimally use
a certain technology. I also became *a better software engineer*, standing on
the many shoulders of the giants in our industry. 

Since my early days with [Kubernetes](https://kubernetes.io/), my inner Dr.
Faust pushed me to learn more about the core concepts, e.g. how the (many)
*autonomous and stateless* controllers (*control loops*) react to state changes
(*events*) without a central orchestrator. I wrote a couple of blog posts[^1]
[^2] and gave presentations[^3], to answer the same questions many of you also
had, doing my best to educate and grow our great community.

Even though over time I developed a good understanding and mental model on the
Kubernetes architecture, I still had the feeling to not fully grok certain
details how these events are generated, propagated and reliably consumed
throughout the various Kubernetes components and actors. The `ListerWatcher`
interface in the `client-go` SDK, used in all Go-based Kubernetes controllers,
plays a critical role here:

```go
// ListerWatcher is any object that knows how to perform an 
// initial list and start a watch on a resource.
type ListerWatcher interface {
  Lister
  Watcher
}
```

It all started with a simple question: how does `ListerWatcher` work? As you see from
this series, it turned out to be another journey into a fascinating and deep
rabbit hole.

For example, is there a canonical definition of a state change *event*[^4] in
Kubernetes? How is the *level-triggered* state change notification implemented?
Why do I always get `ADDED` events from an initial `kubectl` or controller
`WATCH`, when the object's last state is actually `MODIFIED`? How is the
`resourceVersion`, which is critical for *optimistic concurrency control*[^5] in
such an asynchronous system, generated? And if it's so important, why is
`resourceVersion` not explicitly persisted in the persistence layer `etcd` then?

Speaking of `etcd`, how is the Kubernetes object registry physically
represented? What is a ["flat binary key
space"](https://etcd.io/docs/v3.4.0/learning/data_model/) (to quote from the
docs) anyways? And what's the difference between a compaction and
defragmentation operation related to `resourceVersions`? 

Lastly, how does an end-to-end event notification `WATCH` stream work under the
covers, e.g. when using `client-go` or `controller-runtime` (kubebuilder)? How
can a *collection*[^6] ("LIST") have a `resourceVersion`, if the latter is based
on individual object-level changes? What are all these `Indexers` and `Queues`
used for in the controller SDKs? And why don't we see `ADD`/`UPDATE`/`DELETE`
events anymore in the `Reconcile()` handler of controller-runtime?

The list goes on...in fact, it looks like I am not the only one with these
questions 😄

<center>{{< tweet 1308598213067157504>}}</center>

To be fair, some of these questions are answered in various blogs, talks and the
official Kubernetes documentation[^7]. To keep the documentation concise,
implementation details or parts which are not relevant for all personas, are
left out though. Sometimes, the information is partially repeated or spread over
different pages and Git repositories, making it hard to get a complete picture
of how all the pieces fit together. 

So Dr. Faust, err I, decided to describe the *end-to-end flow* of the event
notification mechanism in Kubernetes, also known as the `ListerWatcher` interface
in the [controller
SDK](https://pkg.go.dev/k8s.io/client-go@v0.20.1/tools/cache#ListWatch). 

{{< image src="listwatch-overview.png" caption="End-to-End Communication Flow of ListWatch" >}}

Initially I wanted to start at the client (controller) layer. But I quickly
realized, that in order to understand certain concepts, like `resourceVersion`,
a fundamental understanding of the Kubernetes data model is required. Or more
specifically, how resource objects are persisted in and projected from the database (`etcd`).

Thus, we will work us backwards through this rabbit hole, starting with `etcd` in the
first part of this series. Our journey starts here...

**Part 1:** [Onwards to the Core: etcd]({{< relref "/posts/listwatch-part-1" >}})  
**Part 2:** Source of Truth: The API Server  
**Part 3:** Informers, Controllers, Reconciliation...oh my  

> *"Whatever you can do or dream you can, begin it. Boldness has genius, power and magic in it. Begin it now."*
>
> (John Anster, inspired by Goethe's Faust)

As a final note, I cannot thank the creators and community enough for how much I
learned from the Kubernetes system architecture. 🙏 🙏 🙏 

[^1]: [Reconciliation in the Kubernetes scheduler]({{< relref "/posts/sched-reconcile" >}}).
[^2]: [Events, the DNA of Kubernetes]({{< relref "/posts/k8sevents" >}})
[^3]: [Inside Kubernetes Resource Management](https://www.youtube.com/watch?v=8-apJyr2gi0)
[^4]: Note that I am not talking about the event log in Kubernetes. For a better understanding of the differences see this [talk](https://www.youtube.com/watch?v=rQBoK-QHxFA)
[^5]: [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency)
[^6]: A LIST is returned from GET operations on collections, e.g. `kubectl get pods`
[^7]: [The Life of a Kubernetes Watch Event](https://www.youtube.com/watch?v=PLSDvFjR9HY) is a great talk describing the end-to-end WATCH flow. But since Kubecon talks are time-constrained, the presenters had to omit certain details which I wanted to explore deeper.