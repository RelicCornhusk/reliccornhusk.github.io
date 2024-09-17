---
layout: post
title: Migrating from Docker to Podman
subtitle: "How I migrated from Docker containers to Kubernetes-ready Podman pods"
tags: Podman, Docker, Kubernetes
giscus_comments: true
thumbnail: assets/img/podman-post.png
og_image: "https://www.vitormuzzi.dev/assets/img/podman-post-800.webp"
---

{% include figure.liquid loading="eager" path="assets/img/podman-post.png" class="img-fluid rounded z-depth-1" %}

## Background

For the past few years, I have used Docker and Docker Compose extensively to self-host applications on my "homelab server" -- a nice way of calling an old laptop running Debian 24/7 mounted to the underside of my desk. I have been doing this mainly to self host a stack of open-source applications centered around media management. Self-hosting services taught me a lot about Linux systems administration, networking and all things containers. I cannot recommend this enough as a personal project for any DevOps engineer.

For my learning purposes, the stack itself is not so interesting as much as _how_ it's deployed, and I'm constantly tweaking it to test out new tools and automations. I've already created Ansible playbooks that deploy it from scratch, Docker Compose files to version the containers and their configuration, connected all the services behind a reverse proxy using Caddy, and in the future I might migrate it to Kubenertes using a single-node cluster with K3S. But before that, I wanted to give Podman a try.

## Why migrate to Podman?

For me, it boils down to its greater compatibility with Kubernetes and its daemonless architecture. To elaborate on the former, [Kubernetes has deprecated their Docker runtime support](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) in favour of runtimes compliant with the Container Runtime Interface (CRI), like Podman, containerd or CRI-O. This is understandable, since supoprting Docker meant maintaining Dockershim, an appendage needed to make it work with Kubernetes. To add to that, the Docker project has taken a turn away from its original free open source philosophy in recent years as it started implementing a subscription model and restricting the usability of the tool for free users.

For this reason, open-source alternatives like RedHat's Podman has started gaining a lot of traction. Additionally, if like me your end-goal is working with Kubernetes, Podman allows you to convert your Podman pods to Kubernetes definitions files and to build pods from Kubernetes files with neat commands like `podman kube generate` and `podman kube play`.

Secondly, Podman uses a daemonless architecture to run containers, which means it does not require a service running in the background to spin up and manage the containers. This makes it more secure, since it does not require your containers to interact with a root-owned daemon, but also reduces the overhead required for running containers. Another edge it has over Docker is the ability to run containers as a rootless Linux user, which greatly improves the security of your deployments. In the scenario of a container exploit gaining access to the host, it would access the host from the non-privileged user's perspective. Running rootless containers greatly improves the security of your deployments, though it should be noted that it also requires some amount of comfort around Linux permissions and has a few limitations, [as described in their documentation](https://github.com/containers/podman/blob/main/rootless.md), when compared to running with root permissions.

Finally, if you know how to use Docker, you already know how to use Podman. For a regular user, you can `alias docker=podman` in your terminal and the same Docker commands you are used to will also work with Podman due to the effort of the project in keeping the commands compatible across the two. Also, there is Podman Compose for those used to Docker Compose, although power users should probably just stick to Quadlet + kube definition files for versioning their containers, since not all of the features available in a Docker Compose file will work with Podman.

## The migration itself: the good, the bad and the ugly

The process of migrating everything was unfortunately not as simple as putting down the Docker containers and bringing them back up with Podman. Maybe it could have been if I had opted to use Podman with a root account, but then it would be too easy! So here is an overview of the steps I took to get the all the same functionalities out of Podman using rootless containers, along with what I learned along the way.

### Setting up Podman and creating a separate user

I began by setting up a new user on the host without sudo access. Using a privileged user, I performed the following steps:

1. [Install Podman](https://podman.io/)
1. `sudo useradd -m -s /usr/bin/zsh podman`: -m creates a home folder for the new user and -s sets that user's shell. If you don't use ZSH, replace it for /bin/bash.
1. `sudo passwd podman`: set up a new password for that user.
1. Install [podman-compose](https://github.com/containers/podman-compose) if migrating from docker-compose
1. `su -- podman`: login as the 'podman' user.

Following that I thought I would be able to simply spin up all my containers in my Docker Compose files with the new user and call it a day. Nope. After spending a while trying to understand why my containers were not reachable, I started reading their logs before they exited and realized they could no longer access their configuration files. So I tried recursively changing the ownership of all of the required files to the podman user, but that still did not work. I took a step back and really got down to the nitty-gritty of how Podman manages Linux permissions, a topic which had always been a bit daunting for me. Fortunately there are some amazing blog posts by Red Hat on the topic which taugh me a lot (see [Further reading](##further-reading)). Let's hope I don't lose you, dear reader, with the explanation ahead.

### Linux permissions around containers

Containers (not just Podman) make use of an ingenious UNIX feature called user namespacing to ensure your container can do essentially the same things as your user would but without needing to use the same identity (User ID and Group ID) as your user. In the case of Podman, when you run any container, rootful or rootless, that container will be running by default under that user's namespace as the root user of that namespace. Just like a VM that "thinks" it is its own machine but resides in another host, the container's root user "thinks" it is the root user (UID 0 and GID 0), but in reality it is inside of another user's namespace. Here is a quick demo to help wrap your head around that:

1. In a running container, attach a shell and `touch` a file on a volume shared with the host (e.g.: `podman exec -it example-container touch /data/foo` if /data is a shared volume).
2. Look up that file's permissions from the container's perspective (`ls -l`) and take note of the owner of the file (UID and GID 0, i.e. root).
3. Now look up that same file's permissions from the host's perspective. It will display as if that the file was created by the host user itself.

Why does it do that? Mainly so that an escaped process from the container will not have the same permissions outside the container as it had inside.

To add to this complexity, in my case the rootless Podman container was running with a non-root user inside of its namespace. To be more specific, the container's main process was running under UID:GID 1000:1000. It turns out this [is actually best practice](https://www.redhat.com/sysadmin/rootless-podman-makes-sense) if your containers' main process does not require elevation. So if the root user in the namespace (UID 0) gets mapped to the host user runnning Podman (let's assume UID/GID 15000), the user performing the processes in ther container was actually getting mapped to UID 16000 on the host (15000 from the host user + 1000 from the namespace UID/GID). So if we were to perform the same demo above, the file from the host's perspective would be owned not by itself, but by a user with UID/GID 16000.

To try to abstract all of this, Podman's developers created the weirdly-named command `podman unshare` to allow you to issue any command from inside Podman's user namespace without needing to spin a container and `exec` into it. This is especially useful for this type of situation, where it allows you to abstract the mapping of UIDs and GIDs and simply issue the command from the container's perspective. Thus, I was able to run `podman unshare chown -R 1000:1000 /data` to give the user with ID 1000 inside the container ownership over the files required by the containers.

### Deploying and versioning

After recursively changing the ownership of the files managed by the container, I was able to deploy everything and join the containers into pods as desired. I opted mostly for turning each container into its own pod and joined them all under the same bridge network to facillitate the integration with the reverse proxy. The next step was to generate Kubernetes definition files for these pods to version the deployment itself and make changes declaratively to my services. I did so by running `podman kube generate` to each of the pods and later redeployed them with `podman kube play` to ensure everything worked as before, and it did.

To fully replicate all of the functionalities I had before, I still had to come up with a way for bringing up the containers automatically upon server reboot. With Docker, upon each reboot its daemon would bring up any container with a restart policy of 'always' or 'unless-stopped'. With Podman's daemonless architecture, this means you are in charge of defining systemd services for your containers/pods. They made this a lot easier with version 4.4 through a tool called Quadlet that allows you to create systemd services through reusable, simplified configuration files. Think of it like a `k8s` kubelet that got squashed and became a *quad*let. It accepts using Kubernetes definition files in their recipe files, which made everything easier.

So now, all of my containers became _rootless, kubernetes-ready, version-controlled_ pods and services managed by a kubelet-like tool to manage them as a systemd service. I think I've reached container runtime nirvana.

## Further reading

- [Make systemd better for Podman with Quadlet](https://www.redhat.com/sysadmin/quadlet-podman)
- [Running rootless Podman as a non-root user](https://www.redhat.com/sysadmin/rootless-podman-makes-sense)
