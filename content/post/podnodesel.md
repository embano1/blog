---
title: "Understanding and using the Kubernetes PodNodeSelector Admission Controller"
date: 2018-01-24T21:41:06+01:00
draft: false
excerpt: "During a recent customer engagement, a discussion about Kubernetes `NodeSelectors` came up. There was some confusion about whether and how to use them for a multi-tenant cluster deployment. In the end, we decided to leverage the Kubernetes `PodNodeSelector` admission controller."
tags:
- Kubernetes
- Labels
- Admission
- Scheduling
---

*Thanks to [Timo Reimann](https://twitter.com/timoreimann), [Bjoern Brundert](https://twitter.com/bbrundert) and [Tom Schwaller](https://twitter.com/tom_schwaller) for their feedback and improving this post.*

During a recent customer engagement, a discussion about Kubernetes `NodeSelectors` came up. There was some confusion about whether and how to use them for a multi-tenant cluster deployment. In the end, we decided to leverage the Kubernetes `PodNodeSelector` admission controller. Since not all implementation details were clear to me, I did some experiments and wanted to share my findings with you. 

# About NodeSelectors

Using `NodeSelectors` in Kubernetes is a common practice to influence scheduling decisions, which determine on which node (or group of nodes) a pod should be run. `NodeSelectors` are based on key-value pairs as labels. Common use cases include:

- Dedicate nodes to certain teams or customers (multi-tenancy)
- Distinguish between different types of nodes ("expensive" nodes with specialized hardware, e.g. GPUs and FPGAs, or resources, ephemeral "spot" instances)
- Define topologies for rack/zone/region awareness and high-availability

You can read more about `NodeSelectors` and other options in [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector).

Given these use cases are often requirements in enterprises, this topic comes up in almost every architecture workshop or design review I conduct with customers deploying Kubernetes on VMware vSphere. VMware vSphere VMs can be of any size and may have different quality of service (QoS) policies, for example resource guarantees or limits (cpu, memory, disk, network) applied. 

Just like with Kubernetes resource management, it is a [service level agreement](https://en.wikipedia.org/wiki/Service-level_agreement) (SLA) and contract the infrastructure provider agrees to with the consumer. Thus `NodeSelectors` play a critical role to ensure that, for example, production workloads (Kubernetes deployments, or pods to be precise) run on VMs with QoS == "production". Otherwise we would be at risk breaking the contract. As a best practice, `NodeSelectors` are typically set during CI/CD pipeline stages without human intervention.  

---
**NOTE**

You might be wondering why I am not discussing related concepts, like `node (anti-) affinity` or `taints and tolerations`. Please see the section "Roadmap and alternative Approaches" at the end of this post.

---

# Admission Controllers

Quoting from the Kubernetes [documentation](https://kubernetes.io/docs/admin/admission-controllers/):

> An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. The controllers [...] are compiled into the kube-apiserver binary, and may only be configured by the cluster administrator. [...] If any of the controllers in either phase reject the request, the entire request is rejected immediately and an error is returned to the end-user.

There are many admission controllers in the Kubernetes core, for example `LimitRanger` or `PodPreset`. A complete list is provided [here](https://kubernetes.io/docs/admin/admission-controllers/#what-does-each-admission-controller-do).  

## The Use Case for PodNodeSelector

The `PodNodeSelector` admission controller has been in the source for a while, based on an upstream contribution from Red Hat (Openshift's project node selectors). Quoting again from the Kubernetes documentation:

> This admission controller defaults and limits what node selectors may be used within a namespace by reading a namespace **annotation** and a **global configuration**.

Ok, so why would I use that one instead of simply specifying `NodeSelectors` in the pod manifest? Well, using this admission controller has some advantages:

- As the name implies, it enforces node selectors at admission time before creating the pod object
- This puts less pressure on the master components, for example the scheduler; if there is no matching namespace or a conflict, the pod object will not be created
- You can define a default selector for namespaces that have no label selector specified
- Ultimately, it gives the cluster administrator more control over the node selection process (hard enforcement)

The last one is very important if you have large distributed teams or different departments within the organization. I have seen customer environments, where there was no communication between infrastructure operations, the Kubernetes platform engineers and the development teams. The lack of alignment led to performance issues and, sometimes even worse, production outages. Simply speaking, every team had a different understanding and definition of "production". 

Translated into an enterprise deployment, the following (highly abstracted) picture tries to make it clear. Here, a `kubectl` user is only allowed to deploy nginx pods into the namespace "development", enforced by Kubernetes role-based access controls ([RBAC](https://kubernetes.io/docs/admin/authorization/)). However, if she doesn't specify a `NodeSelector`, this pod could end up being scheduled on the production VMs. The `PodNodeSelector` automatically injects a `NodeSelector`, based on a default policy. 

On the other side, the CI/CD pipeline has a workflow defined to deploy mission-critical nginx proxies into the namespace "production". Those should always land on a production VM, because these have a higher QoS policy applied by the VMware vSphere cluster administrator (remember the "contract" we spoke about earlier?).

<img src="/images/podnodeselector01.png" width="800"></img>

---
**NOTE**

I am working on a follow-up article describing distributed resource management and QoS in the enterprise. A fascinating topic if you ask me :)

---

But using admission controllers, in their current implementation, can also have downsides, most notably: 

- They need to be compiled into the Kubernetes API server
- A change in configuration requires a restart of the API server
- If you use a managed Kubernetes environment, like Google Kubernetes Engine (GKE), configuration options might be limited 

This is why [Dynamic Admission Control](https://kubernetes.io/docs/admin/extensible-admission-controllers/) has been added to Kubernetes recently.

Back to our admission controller. Now that we understand the pros and cons, let's deploy it and see it in action.

# Setting Things up

My demo environment is based on a three node `kubeadm` deployment, running in [VMware Fusion](https://www.vmware.com/products/fusion.html):

- Kubernetes version: 1.9.2
  - 1x "master" VM: Just system pods allowed, i.e. core Kubernetes services
  - 1x "production" VM: Only run pods from namespace "production"
  - 1x "development" VM: Run any pod from any namespaces, except for "production"
- Kubernetes `PodNodeSelector` admission controller enabled in the API server

## Terms used in the Examples

To better understand the examples, let me first explain some terminology used in the `PodNodeSelector`: 

- `clusterDefaultNodeSelector` (global) - Unless a `whitelist` has been specified, add this `NodeSelector` to all pods (applies to all namespaces **without** a `whitelist` specified)
- `Whitelist` (per namespace) - Only allow this `NodeSelector` (requires a matching namespace `annotation`)
- Namespace `annotation` - Key/value Metadata attached to namespaces; the differences between `annotations` and `labels` are described [here](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

## Preparing the API Server

Assuming your cluster has been deployed with `kubeadm`, the `kube-apiserver.yaml` file is the configuration manifest for the Kubernetes API server. It is located in `/etc/kubernetes/manifests`.  Add the `PodNodeSelector` admission controller in the `--admission-control=` flag. 

---

**NOTE**

Strictly speaking, the plugin order matters. We will ignore this in our demo example. You can read more about it in [Using Admission Controllers](https://kubernetes.io/docs/admin/admission-controllers/#what-are-they).

---

Also add `--admission-control-config-file`, `volumeMounts` and `volumes` sections as depicted below. They relate to the default selectors and whitelist, which we will create in a second.

```
apiVersion: v1
kind: Pod
[...]
spec:
  containers:
  - command:
    - kube-apiserver
    [...]
    - --admission-control=[...],PodNodeSelector,[...],ResourceQuota
    - --admission-control-config-file=/etc/kubernetes/adm-control/admission-control.yaml
    [...]
    volumeMounts:
    - mountPath: /etc/kubernetes/adm-control
      name: adm-control
      readOnly: true
    [...]
  volumes:
  - hostPath:
      path: /etc/kubernetes/adm-control
      type: DirectoryOrCreate
    name: adm-control
  [...]
```

Create the file `admission-control.yaml` in `/etc/kubernetes/adm-control/`. 

```
podNodeSelectorPluginConfig:
  clusterDefaultNodeSelector: "env=development"
  production: "env=production"
  development:
  noannotation: "env=notpresent"
```

This file defines the `default NodeSelector` for the cluster, as well as whitelist for each namespace. Note the following:

- Every pod created in the "production" namespace will be injected (i.e. get assigned) the `NodeSelector` "env=production"
- Every pod in the "development" namespace will inherit the `clusterDefaultNodeSelector`, in this case "env=development"
  - Effectively, you do not have to mention namespaces with an empty whitelist in this file
- The "noannotation" namespace is to demonstrate what happens when there is no matching `annotation` in the namespace properties but a whitelist has been specified

---

**NOTE** 

For whitelists to work, every namespace has to have a matching `annotation` (this is **not** a label!). "noannotation" shows what happens when there is no (matching) annotation.

---

## Prepare the Nodes (kubelets)

Apply labels to the nodes, matching the whitelist `NodeSelectors` above.

```bash
$ kubectl label node kubedev01 env=development
$ kubectl label node kubeprod01 env=production
```

Now, the cluster looks like this:

```bash
$ kubectl get node -o wide --show-labels

NAME           STATUS     ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME     LABELS
kubemaster01   Ready      master    1d        v1.9.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-87-generic   docker://17.12.0-ce   [...]
kubeprod01     Ready      <none>    1d        v1.9.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-87-generic   docker://17.12.0-ce   [...],env=production,kubernetes.io/hostname=kubeprod01
kubedev01      Ready      <none>    1d        v1.9.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-87-generic   docker://17.12.0-ce   [...],env=development,kubernetes.io/hostname=kubedev01
```

## Prepare the Namespaces

```bash
# Create the namespaces
$ for i in production development noannotation; do kubectl create namespace ${i}; done

# Add an annotation only to namespace "production"
$ kubectl patch namespace production -p '{"metadata":{"annotations":{"scheduler.alpha.kubernetes.io/node-selector":"env=production"}}}'
```

This is how namespace "production" looks like now:

```
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: env=production
[...]
```

If you also wonder about the `alpha` in the `annotations` spec, there is an [issue](https://github.com/kubernetes/kubernetes/issues/57424) discussing it.

# Ready to go!
## Deploy some Pods

```bash
# We are not specifying any NodeSelectors
$ kubectl run -n production nginx --image=nginx --port=80
$ kubectl run -n development nginx --image=nginx --port=80
$ kubectl run -n noannotation nginx --image=nginx --port=80
$ kubectl run -n default nginx --image=nginx --port=80
```

What Kubernetes created for us:

```bash
$ kubectl get deployment --all-namespaces|grep -v system

NAMESPACE      NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
default        nginx      1         1         1            1           19s
development    nginx      1         1         1            1           19s
noannotation   nginx      1         0         0            0           19s <-- Note: No pods created
production     nginx      1         1         1            1           19s

$ kubectl get pods --all-namespaces -o wide|grep -v system

NAMESPACE     NAME                                   READY     STATUS     RESTARTS   AGE       IP               NODE
development   nginx-7587c6fdb6-cvs4j                 1/1       Running    0          4m        10.47.0.2        kubedev01  <-- looks good
default       nginx-7587c6fdb6-2ffhk                 1/1       Running    0          4m        10.47.0.1        kubedev01  <-- looks good
production    nginx-7587c6fdb6-n54zv                 1/1       Running    0          4m        10.44.0.1        kubeprod01 <-- looks good
```

First, let's examine each pod in "production" and "development" (the pod in "default" is equal to the "development" case):

```
# pod in namespace "production"
apiVersion: v1
kind: Pod
[...]
spec:
  containers:
  - image: nginx
  [...]
  nodeName: kubeprod01 <-- picked the right node
  nodeSelector:
    env: production <-- auto-injected

# pod in namespace "development"
apiVersion: v1
kind: Pod
[...]
spec:
  containers:
  - image: nginx
  [...]
  nodeName: kubedev01 <-- picked the right node
  nodeSelector:
    env: development <-- auto-injected
```

But where's the nginx pod from the "noannotation" deployment? Let's ask Kubernetes:

```bash
# The deployment won't tell us much, so instead query the ReplicaSet
$ kubectl -n noannotation describe rs nginx-7587c6fdb6
Name:           nginx-7587c6fdb6
Namespace:      noannotation
[...]
Controlled By:  Deployment/nginx
Replicas:       0 current / 1 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
[...]
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                From                   Message
  ----     ------        ----               ----                   -------
  Warning  FailedCreate  9s (x12 over 19s)  replicaset-controller  Error creating: pods is forbidden: pod node label selector labels conflict with its namespace whitelist
```

As you can see, no pod is being created due to the admission controller. We intentionally triggered this since we did not specify a correct `namespace annotation` matching the `whitelist` (as we did with the "production" namespace).

## Deploy a Pod with a conflicting NodeSelector
Now let's try to create a pod in the "production" namespace, but specify a conflicting `NodeSelector` in the pod manifest. This could happen for various reasons: A typo, a greedy user, not cleaning up pod manifests after changing labels, etc.

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx2
    ports:
    - containerPort: 80
    resources: {}
  nodeSelector:
    os: windows
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
$ kubectl -n production create -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "nginx2" is forbidden: pod node label selector conflicts with its namespace node label selector
```

This is nice. The admission controller checks for conflicts. And since the "nginx2" pod is supposed to be deployed in the "production" namespace, we only allow label selectors listed in the whitespace section of `admission-control.yaml`. 

## Some suggestions
To keep things simple and understandable, I usually follow these best practices when using this admission controller:

- Come up with a good node labelling scheme (sounds easy, but can be challenging at times)
  - Kubelet's `--node-labels` argument can be very useful (**warning:** this is an alpha feature)
- Use a `clusterDefaultNodeSelector` defaulting to nodes != "production", which has the following effect: 
    - Namespaces with no `annotations` **and** no whitelists specified will inherit this setting
    - Additional `NodeSelectors` **are allowed** in the pod manifest
- For mission-critical namespaces specify `annotations` **and** whitelists
  - Pods deployed therein will have `NodeSelectors` strictly checked and enforced, that is **no pod manifest conflicts/drift allowed**
- Use role-based access controls to secure your namespaces


## Roadmap and alternative Approaches
You might be wondering why I am not touching on `node (anti-) affinity`, `external admission webhooks` or `taints and tolerations`. The latter would be appropriate as well, but one would have to [write](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) a custom admission controller. 

`node (anti-) affinity` (currently in beta) is supposed to become the successor of `NodeSelectors`, with richer query logic. When they graduate from `beta` to `stable`, `NodeSelectors` eventually will [be deprecated](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity). 

For `external admission webhooks`, as part of `dynamic admission control`, it's still early days and for some simple use cases they might be overkill. But this is just my opinion at the time of writing this article. Kubernetes is moving fast and the amazing community quickly resolving gaps and issues. 

So why would you use `PodNodeSelector` admission controller today? Here's my point of view:

- So far, `NodeSelectors` and as such the corresponding admission controller have not been marked deprecated
- The implementation and configuration is straight forward, robust and easy to understand
- At the time of writing, there is no admission controller for `node (anti-) affinity`, so you lose the advantages of an admission controller described in this article
  - I am not the only one interested in fixing this: https://github.com/kubernetes/kubernetes/issues/58198
  - Of course, you could write your own admission controller/webhook

Thanks for taking the time to read this post. I hope you found it interesting and it at least answered some questions. As always, feel free to share it on social media and reach out to me on [Twitter](https://twitter.com/embano1) for any feedback, comments or corrections.

