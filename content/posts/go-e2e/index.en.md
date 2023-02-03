---
title: "Go 1.20 Coverage Profiling Support for Kubernetes Apps"
subtitle: ""
date: 2023-02-02T10:28:15+01:00
lastmod: 2023-02-02T10:28:15+01:00
draft: false
author: ""
description: ""
page:
  theme: "classic"
authorComment: ""
tags: ["Kubernetes","Go","Golang"]
hiddenFromHomePage: false
hiddenFromSearch: false
resources:
- name: "featured-image"
  src: "featured-image.jpg"
math:
  enable: false
lightgallery: true
license: ""

---

<!--more-->

During the pre-release phase of the new Go [release](https://tip.golang.org/doc/go1.20) (1.20), one particular feature
caught my attention which I immediately tried out. Now, coverage profiles can be generated for programs (binaries), as
opposed to just unit tests. This can be useful for debugging and, more importantly, for integration and end-to-end tests
during local development and in CI pipelines.

The Go team also created a [landing page](https://go.dev/testing/coverage/) with FAQs, explaining how to configure and
enable this new thingy. In a nutshell, a Go binary compiled with `GOFLAGS=-cover` and executed with the environment
variable `GOCOVERDIR=<output_dir>` will create a coverage metafile during execution (helpful with restarts) and
individual coverage profiles when the application terminates, irrespective of the exit code. 

With the documentation provided by the Go team, I was quickly able to create coverage profiles on my local machine.
Success 🥳

However, most of my work these days still happens in (on) Kubernetes, such as writing controllers for the AWS ACK
[project](https://aws-controllers-k8s.github.io/community/docs/community/overview/). And this is where using the new Go
feature wasn't as straightforward. 

In a Kubernetes application (`Pod`), state is ephemeral unless it is written to an external location. I thought about
using persistent volumes, but it would have complicated the matter as I would have to know when the application has
successfully terminated before extracting the data from the persistent volume with a helper application. Alternatively,
I could have used a sidecar container running in the application `Pod`, but again I would have to write coordination
logic to know when the application (test) was successful to read and write the coverage data to an external location,
e.g. NFS.

There must be an easier solution... 🧐

## My Development Flow

My typical setup to develop on Kubernetes involves `kind` ([Kubernetes in Docker](https://kind.sigs.k8s.io/)) to create
local Kubernetes clusters and `ko` to [build and deploy](https://ko.build/get-started/) container images without having
to create and worry about a `Dockerfile` 🤷

You might be familiar with `kind` as it's also often used in CI, such as Github Actions. `ko`'s adoption is growing
slowly but steadily and I can only encourage you to take a look at it as I can't live without it anymore.

For integration and end-to-end tests I use the excellent and minimalistic [E2E
Framework](https://github.com/kubernetes-sigs/e2e-framework) from Kubernetes SIG Testing (Special Interest Group),
created by my friend [Vladimir Vivien](https://twitter.com/vladimirvivien). Vladimir wrote a nice article how easy it is
to get started
[here](https://medium.com/programming-kubernetes/end-to-end-testing-of-kubernetes-resources-with-the-e2e-framework-ac52e7e58db8).

{{< admonition type=tip title="Tip" open=true >}} The E2E framework can be used for all sorts of integration and
end-to-end tests in Kubernetes, but not just for building controllers and operators. You might find it useful to verify
that your containerized application behaves as expected, write tests for complex Kubernetes microservice deployments,
assert that a new library you wrote can be imported and deployed in a container, etc. {{< /admonition >}}

But independent from the framework you use for your integration and end-to-end tests in Kubernetes, so far it wasn't
possible[^filippo] to generate coverage reports for compiled Go applications. With the new 1.20 release of Go and a
little bit of Docker and Kubernetes trickery, we now have all the pieces in place to create coverage reports for
integration and end-to-end tests 🤩

## Creating Coverage Reports in Kubernetes

The following steps describe how to create coverage profiles for Go binaries packaged as containers and deployed to a
Kubernetes cluster using `kind`. To keep things simple and tangible, I'll use a small Go package I created to interact
with the VMware vSphere APIs as an example. You don't have to be an expert in VMware technology or the package itself as
the focus is on creating coverage reports. The full code, including running end-to-end tests with coverage in Github
Actions is available in [github.com/embano1/vsphere](https://github.com/embano1/vsphere).

{{< admonition type=tip title="Tip" open=true >}} If you want to execute the steps outlined below, you need the following
tools installed: `git`, [`Go`](https://go.dev/dl/) (1.20+), [Docker](https://www.docker.com/),
[`kind`](https://kind.sigs.k8s.io/docs/user/quick-start) and [`ko`](https://ko.build/install/) {{< /admonition >}}


You might be wondering why I wrote E2E tests for this package since I have good unit test coverage already. Well, unit
tests with client/server API mocks can only get us so far. Deploying and verifying an end-to-end setup in a container
environment, such as Kubernetes, makes me more confident that users of my package won't face the typical "works on my
machine" issues, such as network permissions, authentication (secrets), keep-alives, etc. - while also avoiding to write
and maintain brittle mocks.

The simple E2E test used for our coverage example creates a vSphere server (simulator) and a client application deployed
as a Kubernetes `Job` to perform a login using the `vsphere` package. Once the client has successfully connected, it
will exit with code `0` and the `Job` moves to the completed state (which the test suite asserts). The full code
is in the [test](https://github.com/embano1/vsphere/tree/main/test) folder.

As described earlier, the tricky part is getting the coverage data out of the Kubernetes test application. Luckily, with
`kind` we can use Docker [Volumes](https://docs.docker.com/storage/volumes/) to mount a folder from the host, such as my
MacBook or a Github Actions runner, to a Kubernetes worker, a container created by `kind`. Then we can mount that volume
into a Kubernetes `Job` i.e., our test application, using the Kubernetes
[`HostPathVolumeSource`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath). If you feel like being right
inside the movie "Inception", you are not alone...

{{< admonition type=note title="Note" open=true >}} Yes, this only works in environments where you can use local
volumes, such as `kind`. For remote Kubernetes clusters, you would have to use a persistent volume and some coordination
logic. I'll leave this one up for you, smart reader 😜 {{< /admonition >}}

### Step by Step

First we need to check out the example source code and create a folder where the final coverage data will be stored:

{{< highlight bash >}}
git clone https://github.com/embano1/vsphere.git && cd vsphere
mkdir coverdata
{{< /highlight >}}

Next, we'll create a Kubernetes cluster with a `kind` configuration file mapping our local coverage folder to the worker
node:

{{< highlight bash >}}
cat > kind.yaml <<EOF
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
- role: worker
  extraMounts:
  - hostPath: $PWD/coverdata # path on local machine
    containerPath: /coverdata # worker node folder
    readOnly: false
EOF

export KIND_CLUSTER_NAME=e2e
kind create cluster --config kind.yaml --wait 3m --name ${KIND_CLUSTER_NAME}
{{< /highlight >}}

{{< admonition type=tip title="Tip" open=true >}} Make sure your Docker engine has access and privileges to mount the
`hostPath` folder {{< /admonition >}}

With a Kubernetes cluster running, we can compile the test application binary with active coverage, create a container
image and upload it into the Kubernetes cluster. Sounds complicated? Meet `ko`!

{{< highlight bash >}}
# tell ko to upload to the active kind Kubernetes context
export KO_DOCKER_REPO=kind.local

# sanity check ;)
go version
go version go1.20.0 darwin/arm64

# compile binary for ARM (M1 Mac, use amd64 alternatively), create image and upload to kind
GOFLAGS=-cover ko build -B --platform=linux/arm64 ./test/images/client
2023/02/03 21:46:19 Using base distroless.dev/static:latest@sha256:a218b8525e4db35a0ce8fb5b13e2a980cc3ceef78b6bf88aabbb700373c1c2e2 for github.com/embano1/vsphere/test/images/client
2023/02/03 21:46:20 Building github.com/embano1/vsphere/test/images/client for linux/arm64
2023/02/03 21:46:22 Loading kind.local/client:3c4f716fcbdcf71aaf7f1e25dfe0101b860eccb3d51a1f910fa57e955840799e
2023/02/03 21:46:23 Loaded kind.local/client:3c4f716fcbdcf71aaf7f1e25dfe0101b860eccb3d51a1f910fa57e955840799e
2023/02/03 21:46:23 Adding tag latest
2023/02/03 21:46:24 Added tag latest
kind.local/client:3c4f716fcbdcf71aaf7f1e25dfe0101b860eccb3d51a1f910fa57e955840799e
{{< /highlight >}}

Next, we need to instruct the Kubernetes E2E test suite to create a `Job` for the test client which uses the container
image created above, the mounted coverage volume and set `GOCOVERDIR` accordingly. 

Expand the code block below to see the specific lines in the E2E function which creates the test client `Job`.

{{< highlight go "hl_lines=17 35 49-53 65-72" >}}
func newClient(namespace, secret string) *batchv1.Job {
	var e envConfig
	if err := envconfig.Process("", &e); err != nil {
		panic("process environment variables: " + err.Error())
	}

	l := map[string]string{
		"app":  job,
		"test": "e2e",
	}

	const coverDirPath = "/coverdata"

	k8senv := []v1.EnvVar{
		{Name: "VCENTER_URL", Value: fmt.Sprintf("https://%s.%s", vcsim, namespace)},
		{Name: "VCENTER_INSECURE", Value: "true"},
		{Name: "GOCOVERDIR", Value: coverDirPath},
	}

	client := batchv1.Job{
		ObjectMeta: metav1.ObjectMeta{
			Name:      job,
			Namespace: namespace,
			Labels:    l,
		},
		Spec: batchv1.JobSpec{
			Parallelism: pointer.Int32(1),
			Template: v1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: l,
				},
				Spec: v1.PodSpec{
					Containers: []v1.Container{{
						Name:            job,
						Image:           fmt.Sprintf("%s/client", e.DockerRepo),
						Env:             k8senv,
						ImagePullPolicy: v1.PullIfNotPresent,
						// TODO (@embano1): investigate why this is required in Github Actions to solve "permission
						// denied" error writing to volume (w/ Docker on OSX this is not needed)
						SecurityContext: &v1.SecurityContext{
							RunAsUser: pointer.Int64(0),
						},
						VolumeMounts: []v1.VolumeMount{
							{
								Name:      "credentials",
								ReadOnly:  true,
								MountPath: mountPath,
							},
							{
								Name:      "coverdir",
								ReadOnly:  false,
								MountPath: coverDirPath,
							},
						},
					}},
					Volumes: []v1.Volume{
						{
							Name: "credentials",
							VolumeSource: v1.VolumeSource{
								Secret: &v1.SecretVolumeSource{
									SecretName: secret,
								},
							},
						},
						{
							Name: "coverdir",
							VolumeSource: v1.VolumeSource{
								HostPath: &v1.HostPathVolumeSource{
									Path: coverDirPath,
								},
							},
						},
					},
					RestartPolicy:                 v1.RestartPolicyOnFailure,
					TerminationGracePeriodSeconds: pointer.Int64Ptr(5),
				},
			},
		},
	}

	return &client
}
{{< /highlight >}}

Now we can run our E2E tests as usual:

{{< highlight bash >}}
go test -race -count=1 -v -tags=e2e ./test
=== RUN   TestWaitForClientJob
=== RUN   TestWaitForClientJob/appsv1/deployment
    client_test.go:66: vcsim ready replicas 1
=== RUN   TestWaitForClientJob/appsv1/deployment/client_job_completes
    client_test.go:86: client job complete
--- PASS: TestWaitForClientJob (15.05s)
    --- PASS: TestWaitForClientJob/appsv1/deployment (15.05s)
        --- PASS: TestWaitForClientJob/appsv1/deployment/client_job_completes (5.02s)
PASS
ok      github.com/embano1/vsphere/test 26.840s
{{< /highlight >}}

Let's check if we got some coverage data...

{{< highlight bash >}}
cd coverdata
ls
covcounters.1846795f27c6071dca49109ab71c15e6.1.1675457440336943752
covmeta.1846795f27c6071dca49109ab71c15e6
{{< /highlight >}}

Heureka! But wait, we're not done yet. We need to create a human-readable coverage report as these are binary files.

{{< highlight bash >}}
# print a summary
go tool covdata percent -i=.
        github.com/embano1/vsphere/client       coverage: 61.2% of statements
        github.com/embano1/vsphere/logger       coverage: 57.1% of statements
        github.com/embano1/vsphere/test/images/client   coverage: 86.7% of statements

# generate HTML report
go tool covdata textfmt -i=. -o profile.txt
go tool cover -html=profile.txt -o coverage.html

# open the file
open coverage.html

# cleanup and delete the cluster
kind delete cluster
Deleting cluster "e2e" ...
{{< /highlight >}}

This is a damn cool new feature if you ask me! In fact, it spotted an error I had in one of the AWS ACK controllers I'm
currently writing where the E2E tests showed green but a critical code path was never executed. I wouldn't have caught
this without being able to inspect the HTML coverage report for the controller binary now available with Go 1.20.

Bonus: since we're using tools available in many CI environments, such as Github Actions, it's super easy to create
these coverage reports on pull requests and upload the coverage files to the CI check summary page.
[Here's](https://github.com/embano1/vsphere/actions/runs/4082292404) an example from the `vsphere` repository.

If you enjoyed this post, share it with your friends, and hit me up on [Twitter](https://twitter.com/embano1).

## Credits 

Photo by <a
href="https://unsplash.com/@sharegrid?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">ShareGrid</a>
on <a
href="https://unsplash.com/photos/DG6umKd8LEQ?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

[^filippo]: Well, of course Filippo Valsorda [got it
    working](https://blog.cloudflare.com/go-coverage-with-external-tests/) even before 1.20...