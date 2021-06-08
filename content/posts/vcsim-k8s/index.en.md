---
title: "Using govc with vcsim in Kubernetes"
date: 2021-05-23T14:58:32+02:00
lastmod: 2021-05-23T14:58:32+02:00
draft: false
description: "This post explains how to perform operations against the vCenter Simulator (vcsim) deployed in a Kubernetes environment with the vSphere CLI govc"

tags: ["kubernetes","vcsim","vsphere"]

resources:
- name: "featured-image"
  src: "featured-image.jpg"

toc:
  enable: true
lightgallery: true
---

<!--more-->

The [vCenter Simulator](https://github.com/vmware/govmomi/tree/master/vcsim)
`vcsim` is a powerful Go application to simulate VMware `vCenter` Server
environments. I personally use it a lot together with Github Actions as part of
continuous integration pipelines, e.g. in the [Tanzu Sources for
Knative](https://github.com/vmware-tanzu/sources-for-knative) and [VMware Event
Broker
Appliance](https://github.com/vmware-samples/vcenter-event-broker-appliance).

Recently a colleague reached out to me for some help to deploy and use `vcsim`
to verify an upstream contribution. As part of her testing, she wanted to
trigger a `vCenter` event (`VmPoweredOffEvent`) so her code could react
accordingly. 

Her code and `vcsim` were supposed to run inside a Kubernetes environment. This
isn't well documented so she wasn't quiet sure how to call the `vcsim` APIs
manually from a local command line outside a Kubernetes cluster to trigger the
desired event.

So here's a quick step-by-step tutorial how to use
[`govc`](https://github.com/vmware/govmomi/tree/master/govc), a vSphere CLI
built on top of [`govmomi`](https://github.com/vmware/govmomi/) to perform API
calls against `vcsim` inside a Kubernetes cluster.

{{< admonition type=info title="Requirements" open=true >}}

To follow along you need `Docker`
([Download](https://www.docker.com/products/docker-desktop)), `kind`
([Download](https://github.com/kubernetes-sigs/kind#installation-and-usage)),
`kubectl` ([Download](https://kubernetes.io/docs/tasks/tools/)) and `govc`
([Download](https://github.com/vmware/govmomi/tree/master/govc#binaries)) installed
on your machine. 

{{< /admonition >}}

### Create a Kubernetes Cluster

For this demo, we're using [kind](https://github.com/kubernetes-sigs/kind)
(Kubernetes in Docker) which quickly has become one of the most important assets
in my toolbox to spin up local Kubernetes environments.

Let's go ahead and create a cluster.

```bash
$ kind create cluster --wait 3m --name "vcsim"
Creating cluster "vcsim" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Waiting ≤ 3m0s for control-plane = Ready ⏳
 • Ready after 28s 💚
Set kubectl context to "kind-vcsim"
You can now use your cluster with:

kubectl cluster-info --context kind-vcsim

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

`kind` will correctly set the Kubernetes context (i.e. target cluster), but
let's quickly verify that.

```bash
$ kubectl get nodes
NAME                  STATUS   ROLES                  AGE    VERSION
vcsim-control-plane   Ready    control-plane,master   4m9s   v1.20.2
```

### Deploy `vcsim`

We can use the official `vcsim` Docker container
[image](https://hub.docker.com/r/vmware/vcsim), which makes it dead simple to
deploy `vcsim` in a Kubernetes environment. The following command deploys
`vcsim` in the `default` namespace.

```bash
$ cat << EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vcsim
spec:
  selector:
    matchLabels:
      app: vcsim
  template:
    metadata:
      labels:
        app: vcsim
    spec:
      containers:
        - name: vcsim
          image: vmware/vcsim:latest
          args: ["/vcsim", "-l", ":8989"]
          ports:
            - name: https
              containerPort: 8989
EOF
```

We can verify that the deployment succeeded with `kubectl wait`.

```bash
$ kubectl wait --timeout=3m --for=condition=Available deploy/vcsim
deployment.apps/vcsim condition met
```

{{< admonition type=tip title="Tip" open=true >}}

Alternatively (or in case of trouble) check whether the associated Kubernetes
resources were created correctly with `kubectl get deploy,rs,pod`. You should
see a Deployment, ReplicaSet and Pod successfully deployed.

{{< /admonition >}}

### Create Port-Forwarding

To keep this post simple and to not assume any load-balancing infrastructure, we
do not create a Kubernetes
[Service](https://kubernetes.io/docs/concepts/services-networking/service/) to
expose `vcsim` to the outside world.

Instead, we can create a
[port-forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
to directly talk to the deployment from the local machine where we are going to
run the `govc` commands.

```bash
$ kubectl port-forward deploy/vcsim 8989:8989
Forwarding from 127.0.0.1:8989 -> 8989
Forwarding from [::1]:8989 -> 8989
```

The above command will forward traffic against the host-local address
`127.0.0.1:8989` to the pod backing the `vcsim` deployment inside Kubernetes.

### Set up `govc` Environment Variables

As a last step, we need to give `govc` information about the vCenter (Simulator)
API endpoint.

Since `kubectl port-forward` is a blocking call and needs to remain running [^job], the
following steps are performed in a separate terminal.

```bash
# ignore self-signed certificate warnings
$ export GOVC_INSECURE=true

# use default credentials and local port-forwarding address
$ export GOVC_URL=https://user:pass@127.0.0.1:8989/sdk
```

Now verify that we can establish a connection.

```bash
# list vcsim inventory
$ govc ls
/DC0/vm
/DC0/host
/DC0/datastore
/DC0/network
```

### Trigger an Event

As a final exercise, let's trigger an event by powering off a virtual machine.

To see the events, open another terminal and set the environment variables as
described [above](#set-up-govc-environment-variables) again.

```bash
# tail and follow the vcsim event stream
$ govc events -f

# omitting events generated by the system
[...]
```

Now go ahead and power off a VM.

```bash
$ govc vm.power -off /DC0/vm/DC0_C0_RP0_VM1
Powering off VirtualMachine:vm-66... OK
```

Once you powered off the VM via the command below you should see the following
events in the other console.

```bash
[Sun May 23 21:42:08 2021] [info] DC0_C0_RP0_VM1 on host DC0_C0_H0 in DC0 is stopping
[Sun May 23 21:42:08 2021] [info] DC0_C0_RP0_VM1 on DC0_C0_H0 in DC0 is powered off
```

When you're done you can throw away the environment in one command.

```bash
$ kind delete cluster --name vcsim
Deleting cluster "vcsim" ...
```

### Wrap Up

I really like how easy it is to spin up even complex infrastructure environments
on your local machine with the great open source tooling around us these days.

The vCenter Simulator `vcsim` is a great tool to simplify access to vCenter APIs
and can be easily integrated into CI environments, e.g. with Github Actions. To
dig deeper, check out the earlier mentioned repositories [Tanzu Sources for
Knative](https://github.com/vmware-tanzu/sources-for-knative) and [VMware Event
Broker
Appliance](https://github.com/vmware-samples/vcenter-event-broker-appliance) for
some inspiration how we use `vcsim` with Kubernetes in Github Actions.

### Credits 

Photo by <a href="https://unsplash.com/@thisisengineering?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">ThisisEngineering RAEng</a> on <a href="https://unsplash.com/s/photos/simulator?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

[^job]: You could also send and resume the command to the background, but I am
assuming you know how to perform this in your `$SHELL`, since you're asking 😉
