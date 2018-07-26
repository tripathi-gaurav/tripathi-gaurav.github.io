---
layout: post
title:  "Security analysis of Docker containers"
date:   2018-06-25 23:27:07 -0400
categories: technology security
---


# What is Docker? How is it different from Virtual Machine?

`Docker` is a container technology that creates a virtualized environment using:
1. Isolation
2. Host Hardening

Unlike the OS-based virtualization offered by Virtual Machines, (Docker) container's do not require a [Hypervisor or VMM](https://en.wikipedia.org/wiki/Hypervisor). Rather, containers rely on Linux Kernel features to provide a secure, isolated environment. This results in an increase in performance when compared to other OS-based Virtual Machines as it reduces the turn around time of calls between VMM and host OS. Docker is also considered `lightweight`, as it builds a tree of changes on top of a base image thereby reducing the size of snapshots.

`Isolation` in (Docker) containers is attained by making use of `Linux` kernel technologies like `namespaces` and `cgroups`, which have been merged since `Kernel 2.6.24`. Namespaces are used to restrict visibility of process over the system by creating different namespaces for:
* PID : Process ID
* IPC : Inter process communication
* NET : Networking functionality
* MNT : Filesystem mounting capabilities
* UTS : Hostname and Domain resolution
* USER : File and group permissions
* CGROUP :  Cgroup process virtualization

As an example you can see the /proc/`pid`/maps of a running process on the Docker guest instance and then find the same process running on the host OS with a different PID whose maps will correspond to a different /proc/`pid`/maps on the host machine.

Docker containers rely on [SELinux](https://selinuxproject.org/page/Main_Page) and [AppArmor](https://en.wikipedia.org/wiki/AppArmor) for hardening of the host machine in order to prevent attacks on other Docker instances that are running ( or also attacks on the host itself).

Now the reason that the Docker technology has security implications are two fold:
1. Violation of Docker isolation
2. Security breach in the Docker tooling ecosystem

## 1. Violation of Docker isolation

Docker containers provide options to the host that offer elevated access to the host machine (ex. running a container with `--priviledged` flag or `--cap-add=SYS_ADMIN`). NOTE: Usage of some of these options can break Docker's isolation feature.
Running the container on a host machine also requires resource, and therefore lack of resources on the host machine can result in a DoS attack.

Solution: Containers must be run with recommended configurations and follow best practices. Best practices include having one container to serve just one purpose by running one service. The default configurations of Docker have a reasonable sense of security, although it can be tightened further. For example, the `docker-default` AppArmor policy has some policies to protect against a malicious container escape.

## 2. Security issues with Docker tooling ecosystem

Docker tooling ecosystem can contain a Docker repository (like the public [Dokcer Hub](https://hub.docker.com/)) or an orgnization's private repository. Docker should make use `TLS` connection with repository and use `signed` images to verify the downloaded image.
Docker `images` could be outdated and therefore, contain vulnerabilities. Users must ensure they update and commit the `images` downloaded from the Docker repository.
