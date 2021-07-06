---
title: "Kubernetes Resource Management for vSphere Admins" 
date: 2019-11-30T16:52:27+02:00 
draft: false 
description: "VMworld 2019 session about Kubernetes Resource Management for
vSphere Admins [KUB1208BU]" 
tags:
- Kubernetes
- Mechanics
- vSphere
- DRS
- HA
- NUMA

resources:
- name: "featured-image" 
  src: "featured-image.jpg"

lightgallery: true

toc: 
  auto: false 
  enable: false
---

<!--more-->

This year at VMworld I had the pleasure to co-present with my fantastic
colleague [Pranshu Jain](https://twitter.com/pranshujain28) a session I always
wanted to deliver to our VMware communities: Kubernetes Resource Management for
vSphere Admins. Initially we did not want to make it a deep dive to also attract
newcomers to the world of Kubernetes. But after the first internal reviews and
stats on the session signups for VMworld US we decided that it actually is deep
dive content. 

That was the right choice, because VMworld participants love deep dives and the
knowledge on Kubernetes compared to the previous years has grown a lot. So we
could assume some basic knowledge on Kubernetes for the session. And since the
presentation was using a lot of vSphere constructs participants were able to
relate the concepts discussed.

After a short VMworld 2018 recap and a quiz to get everyone excited and
interested in what's to come in the session, we covered what resource management
is, why it's important and why you need to think beyond the layer you
own/manage/operate. Turns out it's a pretty complex topic where a lot of things
can (and will) go wrong. And just because you follow a best practice for your
platform, that *does not automatically translate* into a best practice for the
upper/lower layers involved in delivering a full software and application stack.

If you are a vSphere and/or Kubernetes platform administrator (architect), you
definitely want to understand how the different platform layers interact to
build a *scalable, robust and efficient application platform*. To simplify the
understanding of all the moving pieces, we covered the lifecycle of a pod
(application) in Kubernetes from resource management perspectives. That includes
*pod definition, admission control, scheduling and enforcement* (execution). Since
this was a deep dive, we could also cover *container internals* such as Linux
Kernel cgroups used by Kubernetes to model resource requirements and best
practices for aligning them with vSphere constructs (*resource
requests/limits*). 

{{< image src="vmworld-kub1208bu-1.png" caption="Resource Management Layers" >}}

The feedback (scores) we got for both sessions (US/EU) were outstanding. And as
discussed during the presentation, we do take them seriously. 

So a **BIG THX** for everyone who attended, especially the one in Barcelona
where we got the last slot on the last day - with some incredible presenters
like [Ben Corrie](https://twitter.com/bensdoings) presenting deep dives on
[Project Pacific](https://www.vmware.com/products/vsphere/projectpacific.html)
in parallel.  

You can watch the session recording
[here](https://www.vmware.com/vmworld/en/video-library/video-landing.html?sessionid=15614127484680019FXJ&region=EU).

{{< admonition type=note title="Note" open=true >}} 
You might need to log in with your (free to register) VMware account until the
videos are made available for everyone. If you want to browse the slides, find
the PDF
[here](https://cms.vmworldonline.com/event_data/12/session_notes/KUB1208BE.pdf)
(no login required). 
{{< /admonition >}}

If you have any feedback on the session and material/concepts discussed, feel
free to reach out to me on [Twitter](https://twitter.com/embano1). I'm not sure
if I'll try to submit a similar session again for 2020. But depending on the
feedback and if this content is valuable I might reevaluate my decision :)
