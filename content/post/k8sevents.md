---
title: "Events, the DNA of Kubernetes"
date: 2018-12-05T10:07:55+01:00
draft: false
excerpt: "I see a lot of people having problems to understand how the Kubernetes platform works at the fundamental level, e.g. resiliency and behavior. If you start thinking about Kubernetes as a fully event-driven system, there's answers to so many \"Why\"'s"
tags:
- Kubernetes
- Mechanics
- Event-Driven
---

Understanding the various components and overall system design of Kubernetes is not an easy task. This is partially due to the designers of Kubernetes fully embracing a microservices based architecture. I.e. every piece of the *control ("masters") and data plane ("nodes/workers")* is implemented as distinct services (controllers). Major advantages of this architecture are flexibility and independence in development/deployment, horizontal scalability and failure tolerance, i.e. running two or more (typically n+1) instances of critical processes like etcd, scheduler, API server, etc. 

The following diagram from the Kubernetes docs shows the various components. You can read more about that [here](https://kubernetes.io/docs/concepts/overview/components/).

<center>![Kubernetes Architecture - Source: Kubernetes.io](/images/k8s-arch.png)</center>

The downside is that such a microservices based architecture always comes with a steep learning curve. But this knowledge is critical for both, developers and operators, maintaining Kubernetes or developing workloads on top of it. It's a distributed system par excellence. In my engagements with customers, colleagues and conferences I dedicate a lot of time going into the details of how Kubernetes works. Because across the board, there is often confusion or misunderstanding of the architecture, opening the door for unnecessary discussions or, even worse, failure. 

> *"If you cannot explain something in simple terms, you don't understand it."*
>
> Richard Feynman

I realized that just by describing the individual components and adding some buzzword-magic like "choreography", "controllers", "stateless" and "leader election" does not always make it easier to understand what's going on and how Kubernetes delivers on the aforementioned promises of a microservices based design. 

If I would have to describe Kubernetes in one sentence it would be: 

> "Autonomous processes reacting to events from the API server". 

These *processes* are modeled as *control loops* (observe->analyze->act) and **only communicate** with the API server, i.e. never directly with each other. An *event* is simply an immutable fact that happened, e.g. "pod created". This is also referred to as *event-driven architecture*.

While working on another (long overdue) blog post I had to create an event-workflow diagram, using the [Kubernetes Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). In order to not let wait my followers, I did a short tweet on how I think of and explain Kubernetes. Dear lord, did that thing go viral! It certainly hit a nerve.

<center>{{< tweet 1067537816324845569 >}}</center>

Many of you have asked for a blog post (and even Kubecon talk!) on this. So here's what I wrote in a more readable and Google Search friendly way.

## How I understand Kubernetes

Think of the API server as an *immutable (replicated) log* (or queue), each representing a stream of events. Events are facts that can be causally related (*happened-before*) or not related at all (then we say they happened *concurrently*). *etcd* is important for the durability of events, but an implementation detail.

All processes (controllers), e.g. the scheduler, deployment controller, endpoint controller, Kubelet, etc. can be understood as *producers and/or consumers* of events (consumers can be producers as well, and vice versa).

Consumers specify the objects (and optionally namespace) they want to receive events from the API server. This is called a *watch* in Kubernetes. Think of the combination of object+namespace as a dedicated (virtual) event queue the API server handles.

Consumers and producers don't know about each other as they're fully decoupled (by the queue) and autonomous. This makes the whole system extremely scalable, robust and extensible (adaptable to change).

Thus, by design, it's a fully *asynchronous and eventually consistent* platform. Information takes time to propagate from producers(s) to consumers(s). The diagram below shows this where the Horizontal Pod Autoscaler hasn't caught up with the events coming from the metrics server, which also affects the downstream chain of producers and consumers in our example.

<center>![Kubernetes API Server - Source: Kubernetes.io](/images/k8s-api-server-queues.png)</center>

There's NO guarantee that the system will converge to the desired state (even if you got an "OK"/ACK from the control plane). For example: 

```bash
$ kubectl scale <deploy> --replicas <n> # where sum(cpu_requests) > cluster_capacity
Scaled deployment <deploy>
```

If the control plane is running fine, you'll get a HTTP 200 return code (i.e. "OK"), even though there is no remaining capacity in the cluster. Marko Luksa put together [another great example](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/) of some unwanted *race conditions* between controllers when scaling down services.

Controllers are *stateless*. For efficiency and speed, received events are placed in an in-memory cache (typically also modeled as a queue) in each controller. What if a controller crashes as it does not persist state?

Persistency in the controller is not needed! The event-driven design will replay all (appropriate) events when the controller (re)starts, also known as *event-sourcing*. This property is also very useful as events from the API server are delivered *at most once*, i.e. could be lost during transmission. Another intrinsic property for increased robustness: Kubernetes is said to be [*level-triggered*.](https://speakerdeck.com/thockin/edge-vs-level-triggered-logic) If an event gets lost during transmission (e.g. network issue), the next time there's a (re)sync of your controller you're going to receive the latest state from the API server. I described how reconciliation works in the scheduler [here]({{<ref "sched-reconcile.md">}}).

Since information can get delivered more than once (e.g. after failure, resync, etc.) and controllers don't talk to each other directly, there's a potential for race conditions when state is to be changed (e.g. a write to same object from different controllers).

This is called *optimistic concurrency* and needs to be handled in the application layer (i.e. in each controller logic). *Idempotency and compare and set with retries* (based on monotonically increasing resource versions) are patterns used to solve that.

Event-driven design has many more advantages and can address a lot of problems in distributed systems (back-pressure, queuing, retries, scale-out, etc.). However, it's not a new idea. Not at all! For example, relational databases work the same way (transaction log). The database is said to be the (stale) cache of the immutable log 😄. If you want to learn more about event-driven design, make sure to check out [this video](https://www.youtube.com/watch?v=iDey1GoAJy0) from [Jonas Boner](@boner).

Building systems using events as a first class citizen is not specific to computer science. Events are the reason for all being. Without events, there would be no change! Think about it, it's an applied and battle-tested design in nature (and can easily become a very long philosophical discussion)...

Btw: [Stefan Schimanski](@the_sttts) and [Michael Hausenblas](@mhausenblas) have written a [great series](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/) of how the API server actually is implemented. Remember: I was describing a thought model for a better understanding.

I was so happy to see all the great responses from the community and never thought of how this tweet could reach almost 100k people (according to Twitter stats) in less than a week. Thank you!

<center>{{< tweet 1068517869422489600 >}}</center>
