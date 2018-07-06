---
title: "Inside Docker's \"FROM scratch\""
date: 2017-11-21T21:18:04+01:00
draft: false
excerpt: "Statically compiled languages, like Go, are quiet popular these days. In combination with another blockbuster technology, Docker containers, a big advantage is the minimal size of the resulting container image. This can be achieved by using the special Docker \"scratch\" image. But how does it look like inside \"FROM scratch\"? [continue to read]"
tags:
- Docker
- Container
- Scratch
---

The idea to this post was triggered by Julia Evans "So you want to be a wizard"...

Thanks to [Ajeet](https://twitter.com/ajeetsraina), [Bjoern](https://twitter.com/bbrundert) and [Timo](https://twitter.com/timoreimann) for your input and review.

## "Wizard skill: Understand your abstractions (sometimes)" - Julia Evans

Statically compiled languages, like [Go](https://golang.org/), are quiet popular these days. In combination with another blockbuster technology, Docker containers, a big advantage is the minimal size of the resulting container image. This can be achieved by using the special (and reserved) Docker `scratch` image.  

> This image is most useful in the context of building base images (such as debian and busybox) or super minimal images (that contain only a single binary and whatever it requires, such as hello-world). (Source: Docker)

There are other reasons, besides the resulting image size, why you would want to use this *special* image as your base layer:

- Reduced dependencies (layers) on other images
- Reduced attack surface because only what´s really needed to execute your application is in the image
- Pull and startup times are minimal

## "In Linux, everything is a file (descriptor)"

OK, but what´s actually in this base layer? Really nothing? But there must be something inside. Like a basic runtime a Linux process might depend on. E.g. `/etc/host*` for name resolution, `dev` for file descriptors (`STDIN/OUT/ERR`), or pseudo-filesystems to query information (`/proc` and `/sys`)?!? Don't get me wrong. Not every process needs them. But a typical user process might assume those exist and thus `scratch` at least should incorporate them, no? 

Are these files and filesystems injected during the build phase or mounted during creation of the corresponding container instance? Put simply, how does it look like inside a container built `"FROM scratch"`? Let´s try to pull and extract the image:

```bash
~ docker pull scratch
Using default tag: latest
Error response from daemon: 'scratch' is a reserved name
``` 

Hm, that didn´t work. Let´s cheat:

```bash
# We need an empty dummy file and of course a Dockerfile for "docker build" to work
~ touch test
~ cat <<EOF > Dockerfile
FROM scratch
COPY test .
EOF

# Build
~ docker build -t scratchtest .
~ docker images scratchtest
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
scratchtest         latest              9860662f4cfc        12 seconds ago      0B <-- "Zero Bytes"
```

Indeed, the container image is of zero bytes size. Good. Let's be 100% sure and extract the image:

```bash
# Save image to a tar file
~ docker save -o scratchtest.tar scratchtest

# Untar and change into image folder
~ tar xvf scratchtest.tar
~ cd 7f576d166b7d57e1db38033c08aba53190dc94aea1f04761e1fb41b4ed75b592 # your ID will be different

# Check what's in layer.tar
~ tar tvf layer.tar
-rw-r--r--  0 0      0           0 20 Nov 14:46 test
```     

OK. The promise of `scratch` is real. There's nothing inside a scratch container, besides the file(s) we add. But we won't stop here, because we're curious explorers, right? Let´s build the most simple program, put it into a `scratch` container, run it and have a look inside. 

## Here comes `sleep:latest`

```bash
~ cat <<EOF >main.go
package main

import "time"

func main() {
        time.Sleep(time.Hour)
}
EOF

# Build binary for Linux
~ GOOS=linux go build -o sleep main.go
```

```bash
# Create our Dockerfile again
~ cat <<EOF > Dockerfile
FROM scratch
COPY sleep /
ENTRYPOINT ["/sleep"]
EOF

# Build the container
~ docker build -t sleep .
```

```bash
# Run container
~ docker run --rm --name sleep sleep

# Get PID of the "sleep" process 
~ PID=$(docker inspect -f '{{.State.Pid}}' sleep)
```

Now, let's look inside our running container. These commands only work if you run them **on** the Docker engine host. Beware, Docker for OSX uses a [xhyve](https://github.com/mist64/xhyve) virtual machine. See the end of this post for a workaround.

```bash
~ cd /proc/$PID/root
~ ls -al
total 1064
drwxr-xr-x   10 root     root          4096 Nov 19 21:34 .
drwxr-xr-x   10 root     root          4096 Nov 19 21:34 ..
-rwxr-xr-x    1 root     root             0 Nov 19 21:34 .dockerenv
drwxr-xr-x    5 root     root           340 Nov 19 21:34 dev
drwxr-xr-x    2 root     root          4096 Nov 19 21:34 etc
dr-xr-xr-x  118 root     root             0 Nov 19 21:34 proc
-rwxr-xr-x    1 root     root       1074119 Nov 19 21:27 sleep <-- our binary from the Dockerfile
dr-xr-xr-x   13 root     root             0 Nov 19 21:34 sys
```

Interesting! There is indeed more inside our scratch container than a `docker save` would have shown us. Where do these entries come from? The answer lies in your container runtime implementation. Since this example is based on building containers `"FROM scratch"` with Docker, consequently I used the Docker engine (Community Edition 17.09). Internally, this uses runC (libcontainer). This can be confusing and [this](https://youtu.be/sK5i-N34im8?t=38m0s) part in Jérôme Petazzoni's talk explains it pretty well. The whole talk is worth watching, especially if you're not familiar with containers.

Back to our situation. The folders and files you see above are actually the root filesystem (`rootfs`) of our Docker container *instance*, when created with the default configuration. This is specified in the [runC libcontainer spec v1](https://github.com/opencontainers/runc/blob/master/libcontainer/SPEC.md), which follows the OCI runtime [specification](https://github.com/opencontainers/runtime-spec). To quote from the libcontainer spec:

> A root filesystem must be provided to a container for execution.  The container
will use this root filesystem (rootfs) to jail and spawn processes inside where
the binaries and system libraries are local to that directory.  Any binaries
to be executed must be contained within this rootfs.
>
> Mounts that happen inside the container are automatically cleaned up when the
container exits as the mount namespace is destroyed and the kernel will 
unmount all the mounts that were setup within that namespace.  
> For a container to execute properly there are certain filesystems that 
are required to be mounted within the rootfs that the runtime will setup.
>
> Path        |  Type  
> ----------- | ------ 
> /proc       | proc   
> /dev        | tmpfs  
> /dev/shm    | tmpfs
> /dev/mqueue | mqueue 
> /dev/pts    | devpts
> /sys        | sysfs

There are extra runtime files, like those in `/etc`, which are also taken care of. If you want to look at the code for how this is done in libcontainer, [here's](https://github.com/opencontainers/runc/blob/v1.0.0-rc4/libcontainer/rootfs_linux.go#L162) the initial call to mount the `/proc` filesystem into the container's root filesystem (`/`):

```
func mountToRootfs(m *configs.Mount, rootfs, mountLabel string) error {
	...

	switch m.Device {
	case "proc", "sysfs":
		if err := os.MkdirAll(dest, 0755); err != nil {
			return err
		}
		// Selinux kernels do not support labeling of /proc or /sys
		return mountPropagate(m, rootfs, "")
```

## How to access the scratch container on OSX or if your Docker engine host runs on a remote machine

If you try the above steps on OSX, or you don't have direct CLI access to the engine's host OS, use the following workaround:

```bash
# Since we don't have a shell in our "sleep" container, use a debugging container which runs in the "sleep" namespaces
~ docker run --rm -it --pid container:sleep alpine

# Here, "sleep" is Pid1 (namespaces!)
~ cd /proc/1/root
~ ls -al
total 1064
drwxr-xr-x   10 root     root          4096 Nov 19 21:34 .
drwxr-xr-x   10 root     root          4096 Nov 19 21:34 ..
-rwxr-xr-x    1 root     root             0 Nov 19 21:34 .dockerenv
drwxr-xr-x    5 root     root           340 Nov 19 21:34 dev
drwxr-xr-x    2 root     root          4096 Nov 19 21:34 etc
dr-xr-xr-x  118 root     root             0 Nov 19 21:34 proc
-rwxr-xr-x    1 root     root       1074119 Nov 19 21:27 sleep
dr-xr-xr-x   13 root     root             0 Nov 19 21:34 sys

~ ls -al etc
total 20
drwxr-xr-x    2 root     root          4096 Nov 19 21:34 .
drwxr-xr-x   10 root     root          4096 Nov 19 21:34 ..
-rw-r--r--    1 root     root            13 Nov 19 21:34 hostname
-rw-r--r--    1 root     root           174 Nov 19 21:34 hosts
lrwxrwxrwx    1 root     root            12 Nov 19 21:34 mtab -> /proc/mounts
-rw-r--r--    1 root     root           153 Nov 19 21:34 resolv.conf
```  

## Wrapping up

First of all it's important to make the distinction between a container *image* and container *instance*. As you have seen in this example, the latter is more than the sum of the Dockerfile pieces. 

I think it can also be quite interesting to take a look inside the *live* environment of a container instance which has been built `"FROM scratch"`. This helps to understand potential pitfalls and what to take into account for troubleshooting containers. Something we're going to look at in the next blog post, which will focus on Linux Kernel Control Groups, aka `cgroups`. Stay tuned!

As always, feel free to share or reach out, e.g. on [Twitter](https://twitter.com/embano1).

## Further recommended reading and viewing

Manual Page: user_namespaces - overview of Linux user namespaces  
http://man7.org/linux/man-pages/man7/user_namespaces.7.html

Containers from Scratch  
https://ericchiang.github.io/post/containers-from-scratch/

What Have Namespaces Done for You Lately?  
https://www.youtube.com/watch?v=MHv6cWjvQjM
