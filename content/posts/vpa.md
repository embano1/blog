---
title: "Kubernetes and the fine art of scaling - UP!"
date: 2017-07-31T22:17:38+02:00
draft: false
description: "Distributed runtime (i.e. Kubernetes) optimization. The idea was mainly about day two operations for distributed systems written for and running on Kubernetes. We were mainly interested in how the platform can optimize the applications (services) it hosts."
tags:
- Resources
- Scheduling
- Autoscaling
---

<!--more-->

Do you know these days, when a certain topic you have thought about a while hits you again and again? 

It so happened to me last week, when I first discussed the topic of distributed runtime (read Kubernetes) optimization with Emad Benjamin, Chief Technologist/ Application Platforms at VMware (make sure to follow him on [Twitter](https://twitter.com/vmjavabook)). The idea was mainly about day two operations for distributed systems written for and running on Kubernetes. What do I mean by day two operations? Well, it´s a broad topic, but we were mainly interested in how the platform can optimize the applications (services) it hosts. Something you might know from the Distributed Resource Scheduler (DRS) in vSphere, but on a more granular process (container) level instead of virtual machines (VMs). 

DRS continuously monitors the distributed state of the cluster and virtual machines, making recommendations or taking action (whatever the vSphere admin decides) on which VMs to migrate between hosts. This could be due to resource contention, restoring (anti-) affinity relationships, policy drift, etc. This is done transparently without downtime for the guest operating system and services in the VMs. You can read more about DRS in this fantastic whitepaper ["VMware Distributed Resource Management: Design, Implementation, and Lessons Learned"](https://labs.vmware.com/vmtj/vmware-distributed-resource-management-design-implementation-and-lessons-learned)

Now lets get back to our initial topic, runtime optimizations for distributed services running on Kubernetes. The Kubernetes scheduler is fairly limited (today) when it comes to day two operations. It was made for day one, meaning finding resources (nodes) to assign pods to. It has improved significantly in the latest releases, especially around (anti-) affinity, priorities and multi-scheduler capabilities. But it´s still focused on the initial placement and does not (yet) care about runtime optimizations. Perhaps it won´t ever, and this will be a task of another controller (see below). It´s this separation of concerns I like in the design of Kubernetes. 

When I was discussing a design for re-architecting a monolithic business service into smaller services at a customer last week, the following discussion came up. Your service consists of several autonomous units (i.e. microservices). You follow best practices for letting Kubernetes know their resource consumption, e.g. reservations and limits. You also think about inter and intra service relationships (bandwidth & latency) and model them in form of pods as well as affinity settings. These get honored when the pods initially get scheduled.

All is good, until you have a lot of long-running services and resource usage as well as communication pattern between your services changes. This could be due to a new feature being implemented, sudden traffic spikes ("Black Friday" syndrome), imbalanced cluster resources, etc. It reminded me to an article Emad has written years (2014!) ago when [troubleshooting a microservices architecture](https://vmjava.wordpress.com/2014/08/15/practical-microservice-architecture-and-implementation-considerations/). After ripping off the monolith into dozens of microservices, his customer had seen increased inter-service latency and thus poor user experience. The solution was to somewhat revert back the design into a more balanced scale up/ out design with larger services. However, it meant a lot of manual (re)work with downtime to the service.

This is where the [tweet](https://twitter.com/mhausenblas/status/889851241702055936) from my fellow [Michael Hausenblas](https://twitter.com/mhausenblas) struck me. I had the video from Tim Hockin on my ever growing youtube watch list, but the tweet got me interested and made me finally watching it. Michael referred to the "Vertical Pod Autoscaler" (VPA) Tim mentions in the last quarter of the talk. Something Google has built internally with "Autopilot" and Borg. Immediately I started research in the Kubernetes repos and issues for the latest update on various projects, e.g. rescheduler, VPA and add-on resizer (see the links on the bottom of this post).

If you´re really interested in this topic, make sure to read the resources in the links section below. Long story short: if at all, analyzing and reacting to resource usage (cpu and mem) is in alpha/ beta (["Addon Resizer"](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer)). This doesn´t mean that you should move away from this important topic. It´s a matter of resources, requests and community feedback to change the priorties. 

Currently the way most approaches in this area are heading is to monitor the cluster state (mainly cpu and mem consumption or out of memory restarts), e.g. with Prometheus, over a period of time and changing reservations/ limits with a custom controller. This is disruptive to pods (not necessarily to your service, depending on the design), because in-place resource updates are not in yet. For the changes to take effect, pods get restarted. Before a Docker expert jumps in with the correct statement, that non-disruptive resource updates to containers are possible with `docker update`. The challenge is less to make this (in-place) work per pod. The big challenge in getting this right in a highly distributed system like Kubernetes. Various sub-components, like the kubelet, scheduler, and controller manager, need to be aligned in order to not run into inconsistencies with the cluster state.

I want to close this post with something I found missing from looking through the docs, issues, code, etc. It goes back to the discussion with Emad I mentioned in the beginning of this post. What if you´re not constrained by cpu or memory resources but latency is killing you (your service)? What if you could observe the communication pattern between services (intra-pod) and make recommendations/ take action? Something vSphere DRS does for VMs, but at a pod (deployment) level. If Kubernetes would have this capability (gather metrics and run control loop), the troubleshooting Emad did for the customer mentioned in his blog perhaps could have been done fully automatic, without manual intervention and possibly avoiding full downtime. The controller flow could be something like:

- Continuously observe latency (tail latency) between services (Istio anyone ;) ?)
- If the latency or traffic between certain services (deployments) increase:
  - Co-schedule their pods (in a rolling update fashion) and/ or
  - Increase resources (cpu/ mem) in the pods (in case latency happens to be caused by memory pressure)
  - May be at some point Kubernetes will automatically reconfigure the deployment manifest(s) to make inter-pod traffic intra-pod traffic :D ? 

I hope, especially for those new to this area, you found this reasoning and summary on the matter useful. Feel free to jump in and discuss this post with me on [Twitter](https://twitter.com/embano1). 

Links and resources:

- [Rescheduler design space](https://github.com/kubernetes/community/blob/795d951bba8bad8bcb67feacc18d741eac8c3597/contributors/design-proposals/rescheduler.md)
- [Vertical Scaling of Pods #21](https://github.com/kubernetes/features/issues/21)
- [Vertical pod auto-sizer #10782](https://github.com/kubernetes/kubernetes/issues/10782) 
- [Support pod resource updates #5774](https://github.com/kubernetes/kubernetes/issues/5774)
- [Vertical Pod Autoscaler](https://github.com/kgrygiel/community/blob/581de636e38399b1d15342b46402c68f8690e91d/contributors/design-proposals/vertical-pod-autoscaler.md)
- [resorcerer—resource sorcerer](https://github.com/mhausenblas/resorcerer)
- [External QoS controller and implement resizing a Pod/Container resource](https://github.com/kubernetes/kubernetes/issues/28316)
- [Kubernetes Autoscaler](https://github.com/kubernetes/autoscaler)
- [In-place rolling updates #9043](https://github.com/kubernetes/kubernetes/issues/9043)
- [Does kubernetes support docker update command when pod is running? #39060](https://github.com/kubernetes/kubernetes/issues/39060)
- ["docker update" options](https://docs.docker.com/engine/reference/commandline/update/#options)
- [Determine if we should support cpuset-cpus and cpuset-mem #10570 / NUMA Topology Awareness](https://github.com/kubernetes/kubernetes/issues/10570)