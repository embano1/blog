---
title: "Kubernetes is for Systems Engineers" 
# title: "Kubernetes is not for Developers (and what you should learn instead)"
date: 2018-12-05T11:20:55+01:00
draft: false
<!-- TODO: FIX excerpt -->
excerpt: "Kubernetes is hard. Too hard for developers who just want to run their code instead of dealing with a low-level syscall-ish interface that Kubernetes provides today. How should we think about Kubernetes in order to deliver value to end-users? Which abstractions are needed? To answer this, we also have to rethink the way we design our distributed systems."
tags:
- Kubernetes
- Cloud
- Developer
- Architecture
- Linux
---

## tl;dr

The Kubernetes founders got many things right from the beginning. It's tremendous capabilities, inspired by Google Borg, and *event-driven microservices architecture* had a big impact on Kubernetes' growth and success thus far. It's hard to find a business today, which does not take a bet on containers and Kubernetes as an important technical building block on its digital transformation journey. But, as with everything in life, nothing comes for free. 

Let's be honest: it might not be the right tool for your job. For application developers, Kubernetes today feels very low-level and puts a lot of unnecessary cognitive load on them. In my opinion, Kubernetes is a powerful platform for *software developers* ([ISVs](https://en.wikipedia.org/wiki/Independent_software_vendor) or in-house) to build scalable and robust distributed platforms and frameworks. Over the next few years, a significantly larger group of (application) developers, i.e. those writing business logic, will mostly leverage higher-level and event-oriented abstractions. 

Kubernetes itself will become the *cloud-agnostic engine for composable building blocks* (platforms) forming the backbone for your business. Ultimately, it will become an implementation detail we take for granted to be available everywhere.

## Where I am coming from

If you've been following me for a while, you can definitely tell that I am a Kubernetes fanboy. Since more than three years, I've the luxury to work with this powerful technology and its community on a daily basis, while primarily focussing on [resource management](https://www.youtube.com/watch?v=8-apJyr2gi0), architectural guidance and best practices for developers and operators. 

As with all new software, Kubernetes follows a [technology hype cycle](https://en.wikipedia.org/wiki/Hype_cycle). From a raising star with the potential to solve all problems we have in computer science, through a phase of disillusion, to the final stage of maturity (or alternatively, becoming niche/obsolete which I heavily doubt in the case of Kubernetes).

![Gartner Hype Cycle](https://upload.wikimedia.org/wikipedia/commons/9/94/Gartner_Hype_Cycle.svg "Source: Wikipedia")


Depending on who you ask (the Kubernetes developer community, end users/customers of different sizes and geographies, vendors, the broader ecosystem, etc.), you might get different perspectives on which phase they currently see Kubernetes. In my opinion, there's currently two camps. Those who just discovered/heard of Kubernetes and those who have been developing/using Kubernetes for more than a year (perhaps since its inception). 

The first group is typically very excited about this technology ("raising star" phase) and can't wait to migrate all workloads onto Kubernetes, which is mainly based on success stories from cloud pioneers and all the good stuff you [read about Kubernetes](https://kubernetes.io/). I was no different when I started exploring this technology in late 2015, coming from an [Apache Mesos](http://mesos.apache.org/) background. 

The other camp, I think, would mostly fall into the phase of disillusion (or call it "realization"). As usual, your mileage might vary since this is not only very subjective but also depending on the size of your environment (i.e. the company you work for, whether you're coming from a vendor's perspective, or end user, etc.), the number of applications you have in scope, and so on. I do acknowledge that there are many (your?) companies successfully running their business on Kubernetes.

Disillusioned he feels. Why?

## 1st Rule of <s>Distributed Systems</s> Kubernetes: Don't.

It's amazing how much you have to do and know about Kubernetes before you can deploy your first application to production. You don't necessarily have to take the Kelsey Hightower exam [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way). It's probably sufficient to listen to talks from other thought leaders in the community, for example [Kubernetes the very hard way at Datadog](https://www.youtube.com/watch?v=2dsCwp_j0yQ) or [Kris Nova's great Keynote at Velocity Conference](https://www.oreilly.com/ideas/the-freedom-of-kubernetes). 

Titles like [101 Ways to “Break and Recover” Kubernetes Cluster](https://www.youtube.com/watch?v=likHm-KHGWQ) or [A Million Ways to Crash Your Cluster](https://de.slideshare.net/try_except_/running-kubernetes-in-production-a-million-ways-to-crash-your-cluster-container-camp-uk) from black-belt Kubernetes users like Yahoo and Zalando speak for themselves. Always ask yourself: Is the effort really worth what you're trying to achieve? 

<center>{{< tweet 1064530941639458817 >}}</center>

Kubernetes is hard. Period. And even though you might think that you can outsource all complexity by using a managed Kubernetes service from the many cloud provider offerings out there, don't be a fool. Just because someone takes care of the control plane and quarterly updates for you, doesn't free the developer from understanding this technology and all components (dependencies) involved to run applications in the most resilient, secure, scalable and efficient way. Choose your tools wisely as [complexity is creepy: It’s never just “one more thing"](https://medium.com/@kadavy/complexity-is-creepy-its-never-just-one-more-thing-79a6a89192db). 

No better way to wrap this up with an on point quote from fabulous Jessie Frazelle:

> "Anyways, the point I am trying to make is **you should use whatever is the easiest thing for your use case and not just what is popular on the internet**. With complexity comes a steep learning curve and with a massive number of pluggable layers comes yak shaves until the end of time."
> 
> [You might not need Kubernetes - Jessie Frazelle (Github)](https://blog.jessfraz.com/post/you-might-not-need-k8s/)

## Kubernetes is for Systems Engineers

Kubernetes offers powerful primitives, like pods, health checking, service discovery, autoscaling, etc. to build distributed systems. Bilgin Ibryam (Red Hat) has a good [blog](http://www.ofbizian.com/2017/04/new-distributed-primitives-for.html) on the matter. Not only you have to know and keep up with them as Kubernetes evolves, but also understand how they work together and what the platforms expects from you. And we haven't even spoken about platform-independent design best practices for scaling, resiliency, how to containerize and secure your code and so on...

I keep a growing list of best practices (like [this](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes) and [that one](https://www.digitalocean.com/community/tutorials/modernizing-applications-for-kubernetes)). Many [books](https://leanpub.com/k8spatterns) are written on the topic. But how can an application developer, who's focus is on creating business value for the company he works for, learn and keep up with all that? Does Kubernetes offer the right abstractions for developers at all?

> "And yet, I think we're writing our distributed systems in assembly language."
> 
> [Compiling to Containers - Brendan Burns (Microsoft)](https://www.youtube.com/watch?v=VQ7kpxPXTm4) 

I couldn't agree more with Brendan, co-founder of the Kubernetes open source project (for details see the following section). He shared his vision of a ["cloud native standard library"](https://github.com/metaparticle-io/) in his [KubeCon US keynote](https://www.youtube.com/watch?v=gCQfFXSHSxw) in 2017. And he's definitely not alone with the sentiment. Michelle Noorali (Microsoft) pointed this out [earlier](https://www.youtube.com/watch?v=aOQwyN0bTk4) in her KubeCon EU keynote. 

Let me use a well-known technology as an analogy to illustrate this: The Linux kernel. Yes, I am not the first one to make this comparison ([Kubernetes Distributions and 'Kernels' - Tim Hockin & Michael Rubin, Google](https://www.youtube.com/watch?v=fXBjA2hH-CQ)). And while the jury is still out on whether Kubernetes is a kernel or distribution, it certainly helps to better understand the abstractions provided by Kubernetes. But definitely take it with a grain of salt.

The beauty of the Linux kernel, from a (user space) developer's perspective, is that it offers clean abstractions (API) between your code in *user space* and the kernel (space). This separation of concerns is achieved via the [syscall interface](https://linux.die.net/man/2/intro) including a [very bold promise](https://lkml.org/lkml/2012/12/23/75) from the creator himself, Linus Torvalds, to *never break user space*. The latter is noteworthy as it means that a user space program adhering to the syscall API should always compile even in the case of changes to the internal kernel structures, e.g. a new release or bug fix.

> A system call is a request for service that a program makes of the kernel. **The service is generally something that only the kernel has the privilege to do**, such as doing I/O.
>
> [The GNU C Library](https://www.gnu.org/software/libc/manual/html_node/System-Calls.html#System-Calls)

<center>![Linux Syscalls](/images/linux-syscalls.png "The Linux Kernel")</center>

Unless you're a kernel or hardware systems engineer, chances are good that you use an even higher level of abstraction: a programming language like Java, Python or Javascript which provide even further simplifications for the developer to focus on the business logic instead of system internals. This is also true when you're working with a programming language geared towards systems and software engineering like Go, Rust or C. 

Standard libraries (stdlib) in each runtime for I/O, thread management, sockets, etc. have you covered. Be it via the runtime itself or linking against a C library, e.g. the [GNU C Library (libc.so)](https://www.gnu.org/software/libc/). Not only do these APIs abstract away the internals of the kernel itself but also most of the differences in the underlying hardware (drivers and system architecture). Ultimately, they reduce (but don't eliminate) the risk of software bugs. No need to dive into the [fascinating world of syscalls](https://sysdig.com/blog/fascinating-world-linux-system-calls/) to do reads and writes, create operating system threads, open a (secure) socket, etc. Granted, [mechanical sympathy](https://github.com/ardanlabs/gotraining/blob/cef810ce5836c41e4dfc1255611b23e872c83362/reading/README.md#mechanical-sympathy) never hurts.

Here's a very basic comparison between directly using a syscall vs. deferring the heavy lifting to a runtime, e.g. [Go](https://golang.org/). I let you decide which one you prefer...

```
# This is Go - the runtime provides a simple print() function wrapping the write() syscall
package main
func main() {
	print("hello, world!\n")
}
```

```c
// This is C - syscall() is a helper in GlibC to directly execute a syscall, e.g. (SYS_)write
#include <unistd.h>
#include <sys/syscall.h>
int main(void) {
  syscall(SYS_write, 1, "hello, world!\n", 14);
  return 0;
}
```

https://sysdig.com/blog/fascinating-world-linux-system-calls/



So how does Kubernetes feel to application developers? Are we really writing in "assembly language" like Brendan said? Let's look at the following picture.

<center>![Kubernetes Syscalls](/images/k8s-syscalls.png)</center>

Leaving aside all the (important) details on how to containerize and securely deploy your code, Kubernetes seems to offer similar abstractions for the underlying hardware (cloud infrastructure) and distributed systems build on top of it. Like the Linux syscall interface, the API server hides the internals of the machinery (cluster). 

But an important part is missing, at least from an application developer's perspective. There's no such thing as a \(C\) standard library equivalent (not talking about the libraries included in a container image here). Thus we're basically programming against a *low-level* syscall interface when we think in terms of "pods", "persistent volume claims" and other [Kubernetes API objects](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/). How would the world look like if we would not have a standard library in Linux? We would be back in the 1970's and not be talking about some of the great tech we have today, including Kubernetes of course 😉

<center>{{< tweet 1059953894317539329 >}}</center>

## Cloud-native Programming Languages

Perhaps we can make Kubernetes more developer-friendly by creating 
https://hackernoon.com/the-rise-of-cloud-native-programming-languages-211a5081f1b2

Besides the aforementioned open source project ["Metaparticle"](https://github.com/metaparticle-io/package) from Brendan, there are cloud-native programming languages like [Ballerina](https://ballerina.io/) and [Pulumi](https://www.pulumi.com/) emerging trying to fill the blanks. While not strictly bound to Kubernetes, essentially they build a layer on top of the Kubernetes API, reducing the plumbing work with type-safety to deploy your code. But even there you might still be [exposed](https://ballerina.io/learn/how-to-deploy-and-run-ballerina-programs/#deploying-ballerina-programs-and-services) to the Kubernetes "syscall" interface, as these primitives are not completely abstracted away (Metaparticle comes close though, but is in a [very early prototype](https://twitter.com/brendandburns/status/1060030925369561089) stage). 

**Note:** Kubernetes [Custom Resources (CR/CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) could be seen as higher-level libraries, but again there is no standardized "generic" *stdlib:CR*. (CRDs will play a critical role in this post, stay tuned).

But may be I am wrong again? Perhaps there will never be something like a C Standard Library for Kubernetes? My always sharp colleague [Ryan](https://twitter.com/ryandotclair) wrote a thorough [analysis](https://medium.com/@ryandotclair/an-interrogation-of-metaparticle-and-abstraction-layers-44ba2fe07b62) (based on Metaparticle) on why he thinks so. While I don't fully agree with argumentation (from a developer's point of view), the conclusion he draws makes sense to me.

We're back at the beginning: how can we abstract away the complexity of Kubernetes? How can we ultimately hide Kubernetes from the developer? Or to say it in Onsi Fakhouri's words:

<center>{{< tweet 599255828772716544 >}}</center>

## CI/CD and PaaS to the Rescue (?)

If you as an application developer are lucky, your platform team has abstracted most of the underlying details into a *"commit & push"* CI/CD workflow, perhaps using a platform-as-a-service (PaaS). But from a developers' perspective, can you really check most (if not all) of the following points below?

[ ] Flexibility to choose the right tool for the job
[ ] Low barriers of entry 
[ ] The platform does not expose you to primitives like containers, pods, scaling, volume management, etc.   
[ ] The opinionated style of writing business logic with this platform does not get in your way.  
[ ] The platforms automatically scales your application for increased responsiveness and efficiency.  
[ ] The amount of YAML you have to write is significantly less than your actual code (*cough*).

In my opinion, CI/CD pipelines or PaaS (as of today) are critical enablers for an improved developer experience, yet alone they're not sufficient to keep Kubernetes relevant in the future. In fact you could argue that Cloud Foundry, a leading PaaS in the enterprise, does not (yet?) leverage Kubernetes. The same is true for many PaaS-ish platforms in the public cloud. Why would you need Kubernetes in the first place then?

<center>{{< tweet-single 935251954191679488 >}}</center>

Kubernetes shines 



In order to achieve that, we have to completely change [how we think about architecting distributed systems](https://www.youtube.com/watch?v=W6GAhzfBFik&index=2&t=0s&list=WL). With "we" I mean those of us (incl. me) coming from a procedural/imperative programming mindset. 

## Events as the Reason for all Being

Think about what *time* is for you. At the fundamental level, time is change. Without change, there is no time as nothing progresses. (Let's not digress here whether time emerges from something deeper ;) So how can we characterize change? Something happened, exactly! And what happened? Generally speaking, an event. Events are fundamental to everything in nature. Without change (events) we would not exist. So your life could be described as a complex (immutable) stream of "*happened before*" events. 

> *A business is a series of events and the reactions to those events.*
> 
> [Kafka and Event-Oriented Architecture - Jay Kreps (Confluent)](https://www.youtube.com/watch?v=HeNegOzjnJY)

What if we apply the same principle to computer science? Welcome to event-oriented architecture. In fact, this idea is not new and goes back to the very early days of computers (Input/Process/Output) and programming languages. Perhaps you have heard of functional (reactive) programming, publish/subscribe queues, asynchronous event loops (JavaScript), event sourcing (CQRS, Saga pattern), stream processing (Kafka) or most prominent these days [(serverless) FaaS](https://martinfowler.com/articles/serverless.html#WhatIsServerless)?

<center>{{< tweet 985917244029878272 >}}</center>

Let's talk about AWS Lambda. Lambda quickly gained popularity as this service freed the developer from thinking in terms of virtual machines, IP addresses or containers. Instead, the developer just provides her code which is basically a handler that gets triggered every time it is called:

```javascript
exports.handler = function(event, context) {
 context.succeed(“Hello, World!”);
};
```

Event triggers could come from anywhere, e.g. outside (users/devices) or other internal AWS services like S3 or DynamoDB. The "serverless" part in AWS Lambda makes it attractive from an economical perspective as well. But in essence, you deal with composable autonomous services (functions), that don't share state, solely acting based on emitting/receiving events. This approach of *single-responsibility* has several benefits, e.g. scalability, resiliency, decoupling, auditability and efficiency for the overall system. 

<center>![Linux Syscalls](/images/lambda-history.png)</center>

For this blog we don't need to go into the details of approach of designing distributed systems as the basics are sufficient for what comes next. If you want to learn more, this great [presentation](https://www.youtube.com/watch?v=1hwuWmMNT4c) on "Designing Events-first Microservices" from Jonas Boner (Lightbend) provides some great content and ideas.

Ok, so what does this have to do with our challenge of not having developer-oriented abstractions in Kubernetes?

## Kubernetes has the Answer

We can find the answer to our problem at the core of Kubernetes. First of all, the Kubernetes platform is fully event-driven. All components (kubelet, scheduler, controller-manager, etc.) are control loops [acting on events](https://blog.heptio.com/core-kubernetes-jazz-improv-over-orchestration-a7903ea92ca) from the API server. That's why the founders *don't* speak of an orchestration system. Second, the API server hides the implementation details. The scheduler doesn't know about the controller-manager or kubelet, and vice versa. Etcd is an implementation detail.

In the following diagram you can see a *logical* representation of the internal event flow between the actors, where the API server represents append-only (i.e. immutable) queues. These queues are just plain and (strongly consistent) replicated event stores. Various controllers *produce* messages into their respective queues. Consumers *watch* queues they're interested in and when new messages are appended to a queue, the API server *emits* an event and its consumers will process the message. 

<center>![Kubernetes event-driven](/images/k8s-api-server-queues.png)</center>

**Note:** The whole system is *asynchronous and eventually consistent by design*. I.e. its distributed components (and their local states) (thrive to) converge to the desired state over time, yet not immediately. Using a queue (immutable log) is one but not the only way to implement communication between autonomous (decoupled) systems without sharing state. [Reactive systems](https://reactivemanifesto.org/), leveraging  message-passing between actors, share the same philosophy.

**Note:** For the actual implementation of the API server, see [here](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/).

It seems logical to extend this approach to the applications we deploy on Kubernetes. From a developer's perspective all we need now is:

1. An API where we can simply push our code to.
2. Ready to use building blocks with support for event-driven communication.


## Meanwhile at AWS

Text

<center>![Werner Vogels on Business Logic](/images/aws-werner1.png)</center>

Text

<center>![Werner Vogels on Architectural Principles](/images/aws-werner2.png)</center>

Video https://youtu.be/nFKVzEAm-ts




<!-- 

"I'm seeing talks about istio, kubernetes, etc all related to microservices and thinking...I really don't want to have to understand this stuff to write software. Like really. Just get out of my way and let me build products." https://twitter.com/chuhnk/status/1116364990414503938?s=12

about std libs: https://www.internalpointers.com/post/c-c-standard-library

serverless best practices https://medium.com/@PaulDJohnston/serverless-best-practices-b3c97d551535

Gardener (https://github.com/rfranzke/gardener/blob/1bc82b2e5f58be1f9d80ad7003d6d6ae4760b093/docs/proposals/01-extensibility.md#summary) and knative as examples

mercari example: https://twitter.com/deeeet/status/1069769105039749120?s=11

https://twitter.com/chuhnk/status/1062087518265466880
https://twitter.com/kelseyhightower/status/935251954191679488

WEBHOOKS!

"If we do our job right, people will stop talking about k8s in the next 5 years. Not because it goes away, but because it becomes a normalized and boring substrate supporting waves of new innovation above it." - C. McLuckie
https://twitter.com/cmcluck/status/1102684093743951872?s=12

Conclude with Hausenblas on every developer (workflow) is different: https://twitter.com/mhausenblas/status/1065640870773489665?s=12

Alex Richardson Keynote https://www.youtube.com/watch?v=qUK-F40oLVQ

Close with

> When you start modeling events, it forces you to think about the behavior of the system, as opposed to thinking about structure inside the system.
> 
> [A Decade of DDD, CQRS, Event Sourcing - Greg Young](https://www.youtube.com/watch?v=LDW0QWie21s)  

operators and a list of all https://kubedex.com/operators/

event-driven: promises, trust, forgiveness/apologies

CRD examples: istio, knative, docker stack, cnab (?)

> The highest level of abstraction in use is the best one. People will continue to build up and up until Kubernetes is just a thing that is an implementation detail. Most people assume you got Linux available somewhere, you just assume you've got Kubernetes available somewhere.
> Tim Hockin https://www.youtube.com/watch?v=4wN8lo-8_GQ


> Functions Are Glue, Not Calling Each Other
> https://www.infoq.com/articles/serverless-sea-change


https://www.oreilly.com/ideas/reactive-programming-vs-reactive-systems

> In a distributed system there's no such thing as a guarantee.

May be the operation you performed was very expensive (data size, time taken, etc.). Why not store this for future use instead of throwing it away and potentially having to do it again?

event (data) flow

integration patterns from AWS reinvent: API, Orchestration, Eventing
https://www.youtube.com/watch?v=IPOvrK3S3gQ

"Async design needs to move into infrastructure"
https://youtu.be/AqhDFbL6Nb4?t=197

knative in SAP context
https://blogs.sap.com/2018/12/10/kubernetes-sap-cloud-platform-extension-factory-extensibility-on-a-cloud-native-open-source-stack/?source=social-Global-sapcp-TWITTER-MarketingCampaign-Developers-SAPCloudPlatform&campaigncode=CRM-YD18-SOC-DEFLT

Github actions (workflows and async coordination)
https://twitter.com/sarah_edo/status/1079418097218420738?s=12

Giant Swarm predictions 2019 on CRDs
https://blog.giantswarm.io/the-state-of-kubernetes-2019/

future of app development by marco
https://twitter.com/pracucci/status/1078337450752192512?s=12

crossplane IO
https://t.co/8B0kIqgVCX

> -->