---
layout: post
title: My adventures migrating from Docker to Podman
---

## Background 

For the past few years, I have used Docker and Docker Compose extensively to self-host applications on my "homelab server" -- a nice way of calling old laptop running Debian mounted to the underside of my desk. It taught me a lot of Linux sysadmin skills and containerized applications and I cannot recommend enough as a personal project for any DevOps engineer. I have been using mine to self host mainly a stack of open-source applications centered around media management. It gets informed of new TV show episodes and movies I'm interested on through RSS feeds (Radarr and Sonarr), downloads them through different providers (in my  case, through Prowlarr and QBitTorrent) and serves it for streaming through Jellyfin (an open-source Plex). I am able to manage my server and watch my media remotely (connecting through [Tailscale](https://tailscale.com/)) and at home.

## Why migrate to Podman?

There are very good arguments as to why developers and engineers using Docker should migrate to Podman. For me, it boils down to its greater compatibility with Kubernetes and its daemonless architecture. To elaborate on the former, [Kubernetes has deprecated their Docker support](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) in favour of container runtimes which leverage the Container Runtime Interface (CRI), which Podman does. Docker still has its place in testing, building and running images locally, but I figured making the switch to Podman would come in handy for putting to practice my Kubernetes studies. Not only that, you can convert your pods to Kubernetes pod and service definitions files and build pods from Kubernetes files with neat commands like `podman kube generate` and `podman kube play`. 

Secondly, unlike Docker, Podman does not require a daemon running on the host with root privileges to run containers. This means you can run containers using a non-root Linux user, which greatly improves the security of your deployments. In the unlikely scenario of getting hacked (I don't expose my services on the internet), and were the hacker able to exploit the container to gain access to the host, they would only be able to access a non-privileged user without permissions over my personal home folder or system files. Though be aware that following this route involves some amount of comfort around playing with Linux permissions and has [a few shortcomings](https://github.com/containers/podman/blob/main/rootless.md) as compared to running with a root user.

Also it is not a big learning curve if you are already comfortable with Docker. For a regular user, you can literally `alias podman=docker` and the same commands will work just fine with Podman due to the amazing work keeping the tool compatible with Docker. Also, if you're used to using Docker Compose to manage your containers as YAML files, there is the third-party tool Podman Compose which has support for almost all of its features, and uses essentially all the same commands.

## The migration itself: the good, the bad and the ugly
The process of migrating everything was unfortunately not as simple as running as putting down the docker containers and bringing up the Podman ones. Maybe it would have been if I had opted to use Podman with a root account, but then I wouldn't be reaping the security benefits of running Podman on a rootless user. So here is an overview of the steps I took to get the all the same functionalities out of Podman.


### Setting up Podman and creating a separate user
I began by setting up a new user on the host without root access. Using a privileged user, I performed the following steps:
1. [Install Podman](https://podman.io/)
1. `sudo useradd -m -s /usr/bin/zsh podman`: -m creates a home folder for the new user and -s sets that user's shell. If you don't use ZSH, replace it for /bin/bash.
2. `sudo passwd podman`: set up a new password for that user.
4. Install [podman-compose](https://github.com/containers/podman-compose) if migrating from docker-compose
3. `su -- podman`: login as the 'podman' user.

## Linux permissions hell
So here is where it started getting tricky. I thought I would be able to simply spin up all my containers with the new user and call it a day. However, after a few frustrating config sessions where I could not realize why my containers were not reachable, I started reading their logs and realizing they could no longer access their configuration files. So I tried recursively `chown`ing' (changing the ownership of files) all of the required files to the podman user and it still would not work. I took a step back and really got down to the nitty-gritty of how Podman manages Linux permissions (a topic which has always been a bit daunting for me). Fortunately there are some amazing blog posts by Red Hat on the topic which taugh me a lot (see links at the end). Let's hope I don't lose you, dear reader, with the explanation ahead.

Containers (not just Podman) make use of an ingenious UNIX feature called user namespacing to ensure your container can do essentially the same things as your user without needing to use the same User ID (UID) and Group ID (GID) as your user. In the case of Podman, this means that when you run a 'rootless' container (i.e. created by a regular user), that container will by default use a root user on the inside. However, try the following exercise: `touch` a file on a shared volume from inside the container and look up its permissions from the host's perspective (by running an `ls -l` for example). It will display as if that the file created by the container's root user is owned by the host user itself! Why does it do that? Mainly so that an escaped process from the container will not have the same permissions outside the container as it had inside. 

To add to the complexity, the rootless Podman was running a non-root user on the inside ([which is actually the best practice](https://www.redhat.com/sysadmin/rootless-podman-makes-sense)) So if the root user in the container (UID 0) gets mapped to the host user runnning Podman (let's assume UID 10000), the user performing the processes in ther container (in my case it was UID 1000) was actually mapped to UID 11000 on the host (10000 + 1000).

To make this simpler, they created the command `podman unshare` to allow you to issue any command from inside Podman's user namespace without needing to spin a container and `exec` into it. This is especially useful for this type of situation, where it allows you to abstract the mapping of UIDs and GIDs and simply issue the command from the container's perspective. Thus, I was able to run `podman unshare chown -R 1000:1000 /data` to give the user with ID 1000 inside the container ownership over the files required by the containers. 

### Running