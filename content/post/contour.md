---
title: "Project Contour in Virtual Appliances"
date: 2019-10-15T20:49:50+02:00
draft: false
excerpt: Why we chose Project Contour as a flexible ingress controller for the vSphere Event-Broker Appliance (VEBA).
tags:
- Kubernetes
- Event-Driven
- Appliance
- vSphere
---

Chances are good that you have heard about VEBA, the *vSphere Event Broker Appliance*, my exceptional colleague [William Lam](https://twitter.com/lamw) and I are working on at VMware. Soon to be released as a VMware Fling, VEBA empowers our customers and partners to build modern event-driven workflows with minimal effort. Write your *business logic* in any programming language, subscribe to vSphere events, deploy the function, done. 

If you heard about AWS Lambda, that's exactly the capability we want bring to our vSphere communities and partner ecosystem with VEBA. You can read more about VEBA in this recently published [VMware OCTO blog post](https://octo.vmware.com/vsphere-power-event-driven-automation/), which also links to a well-received session William and I delivered on the matter at VMworld (US).

The initial prototype we built and showed at VMworld had some rough edges, though. The appliance was not straight forward to deploy and operate, HTTP endpoints did not enforce authentication, and sensitive data (passwords) was not encrypted. When we decided to open source our work by releasing the appliance as a [VMware Fling](https://flings.vmware.com), we definitely had to address these shortcomings. 

### Kubernetes, of course!

Early on we decided to use [Kubernetes](https://kubernetes.io) as a core building block in the appliance. Kubernetes gives us high-availability, scalability and extensibility out of the box. [OpenFaaS](https://www.openfaas.com), the function-as-a-service backend used in the appliance, natively supports Kubernetes and was deployed with a few clicks. [vCenter-connector](https://github.com/openfaas-incubator/vcenter-connector), the "glue" between the worlds of vSphere and OpenFaaS, also benefited from Kubernetes distributed systems capabilities, e.g. resource management, automatic restarts, configuration management, etc.

To keep things simple, initially we used Kubernetes' [integrated](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) `NodePort` mechanism to expose the various HTTP endpoints to the external (i.e. routable) network. Unfortunately, this has some drawbacks. First, you can't use standard HTTP ports (80 or 443), as Kubernetes will pick a port within a pre-defined range (default: 30000-32767). Second, traffic encryption (TLS) is beyond the scope of type `NodePort`. This is where Project Contour comes into play.

{{< figure src="/images/veba-contour.png" alt="The VEBA Stack">}}
<center>*Figure 1 - The VEBA Stack*</center>

### Project Contour to the Rescue

[Project Contour](https://projectcontour.io) is a high performance ingress controller for Kubernetes, initially developed by Heptio (now part of VMware). In a nutshell, an ingress controller in Kubernetes terms is a layer 7 HTTP(S) proxy for the applications you want to expose outside of Kubernetes. 

Contour provided all we needed to manage and secure the HTTP traffic, e.g. using standard HTTP(S) ports, TLS support, dynamic reconfiguration (no restarts) and flexible (pattern-based) HTTP path routing. The team behind Contour is also working hard to deliver the v1 release, which signals API stability and maturity. Migrating VEBA to use the Contour v1 API is on our roadmap, and will only require minimal changes to the deployment manifests (outlined [below](#deploying-contour-in-veba)). The Contour team also provides a detailed [migration guide](https://projectcontour.io/from-ingressroute-to-httpproxy/) to help during the transition.

Using Contour in a virtual appliance is a straightforward process. The only requirement is Kubernetes since Contour can only be used in Kubernetes environments as of today. Basically, the steps to deploy Contour and expose an application via HTTP/HTTPS are:

* Download a Contour [release](https://github.com/projectcontour/contour/releases) from Github
* Configure `envoy`, the data plane in Contour, to bind to the host network so we can use ports 80/ 443
* Deploy Contour and its dependencies
* Create a certificate for your application (HTTPS/TLS), stored as Kubernetes secret
* Create a Contour `HTTPProxy` definition, which defines the routing and TLS settings for your application

In VEBA we currently use *self-signed certificates*. This has some implications, i.e. users get a "identity cannot be verified" warning and CLI tools, such as `faas-cli` used to deploy functions, need to pass an [additional flag](https://github.com/openfaas/faas-cli/issues/700) `--tls-no-verify` to ignore the message. 

An option would have been to use a certificate manager with trusted certificates as described in a recent Contour blog [post](https://projectcontour.io/guides/cert-manager/). The downside, from the perspective of an appliance maintainer, is that it adds more dependencies which need to be maintained and validated during every release. Furthermore, not every user environment permits these mechanisms, be it due to firewalls blocking the outgoing traffic or strict security policies. To address these issues, especially in larger enterprise environments, we will allow the user to provide his/her own certificates in an upcoming VEBA release.

### Deploying Contour in VEBA

William created a fully automated installation routine to build VEBA appliance. The following description highlights the steps how we exposed and secured the various HTTP(S) endpoints in VEBA. 

> **Note:** You can find the source code for VEBA [here](TODO:FIXME).

First, we download and check out a specific release of Contour:

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "git-clone.sh" >}}

Because we want to expose our applications on the `hostNetwork`, which is handled by the Contour control plane component `envoy`, we have to make a small change in the `envoy` configuration located in `examples/contour/03-envoy.yaml` (see "add these lines" comment):

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "03-envoy.yaml" >}}

Now it's super easy to deploy Contour with a single command:

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "deploy.sh" >}}

With Contour up and running, we now generate the (self-signed) certificate used to encrypt traffic to our applications:

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "create-certs.sh" >}}

> **Note:** Make sure the fully qualified domain name (FQDN) of your appliance, in the example returned by `$(hostname)`, matches the `/CN=` name.

The certificate then is stored as a Kubernetes [secret](https://kubernetes.io/docs/concepts/configuration/secret/) in the namespace where we deploy our HTTP services, e.g. OpenFaaS UI and some helpers for inspection/debugging ("tinywww"). Ingress definitions can reference this secret to configure TLS, which we will see in a bit.

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "deploy-secret.sh" >}}

The last step is to configure the routing rules, i.e. 

{{< gist embano1 2b0c7f99af839b2097598446e61ececc "ingressroute-gateway.yaml" >}}
