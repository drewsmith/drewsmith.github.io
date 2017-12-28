---
title:  "Docker Notes"
categories: Tech, DevOps
read_time: 15m
---
# Dockerfile
* Each line (command) roughly equates to a layer
* Combine commands to reduce layers
* More commands in fewer instructions = optimization

## FROM
* Declares the base image
* Should be first command

## ARG
* The only command that can come before FROM
* Passes ARG to FROM:

```
ARG TAGVERSION=6
FROM centos:${TAGVERSION}
```

* MAINTAINER deprecated, use LABEL instead

```
LABEL maintainer="you@number1bestprogrammer.com"
```

## COPY
* Only works with files

## ADD
* Supports URLs

## VOLUME
* Creates mount within image
* No way within Dockerfile itself that ties to host storage
* No guarantee that the mount point would be available within underlying host
* Images have to remain portable

## ENTRYPOINT
* Can't be overwritten

## CMD
* Can override CMD

## WORKDIR
* Current context of the container

# Storage

## Drivers
* When changing drivers, images will not be available so export/import
* Storage is 1 to N (container to layers)
* Layer writes are copy on write (only change if modified)
* Deleted files stay in proceeding layers

### aufs, overlay, overlay2

* Operate at file level
* More efficient memory utilization but container layer grows quickly

### devicemapper, btrfs, zfs

* Operate at block level
* Better perf in write heavy but worse for memory

### overlay

* Workloads with many small writes
* Containers with many layers or deep filesystems
* Performs better than overlay2

## Persistent Volumes

* No inherit file sharing by default
* Create docker volume

```
docker create volume myvolumename
```

* Link volume via mount

```
# container
-v myvolumename:/mount/point
# service
--mount source=myvolumename,target=/mount/point
# for host file (type=bind)
--mount source=X,target=Y,type=bind
```

* Important component of making containers portable

# Networking

# Bridge
* Create a network with a bridge type and subnet `docker network create --driver=bridge --subnet=192.168.1.0/24 --opt "com.docker.network.driver.mtu"="1501" devel0`
* Inspect with `docker container inspect --format="{{.NetworkSettings.Networks.bridge.IPAddress}}"`

## External DNS
* DNS passes through host `/etc/resolv.conf`
* Use dns flags to force write to `resolv.conf`

```
--dns=8.8.8.8 --dns=8.8.4.4
```

* Default DNS by overwriting dns in daemon.json

```
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

## External Ports
* Use `-P` to publish the container on a random port
* Force port `-p LOCALPORT:CONTAINERPORT` or `--publish`

## Overlay Network
* `docker network create --driver=overlay --subnet=PRIVATEIP1/24 --gateway=PRIVATEIP2 overlay0`
* Populate in swarm by using in service `--network=overlay0`

## Network Drivers
* Determine behavior, accessibility, routing of container networking

### Bridge
* default for standalone hosts
* consists of private network internal to host, all containers on host can communicate
* external access granted by publishing port or static routes added to host as gateway to network

### None
* no networking
* only accessed via host
* attached directly `docker exec -it`

### Host
* Host only networking
* only accessed via host
* external access granted by publishing ports

### Overlay
* allows communication by all docker daemons in swarm
* is a swarm scope driver that extends all swarm daemons
* default swarm communication mode

### Ingress
* special overlay network that load balances networking traffic across service worker nodes
* maintains list of ip address from nodes that host service
* provides routing mesh to expose externally without running replica in swarm

### Gateway Bridge
* special bridge that allows networks to access daemon's physical network
* automatically created when swarm is initialized

## Publishing

### Host
* containers on host are not available externally
* use in single host
* You are responsible for knowing where instances are

### Ingress
* All published ports available to all hosts/workers in swarm regardless if a replica is running

## Container Network Model (CNM)

### Sandbox
Encompasses network stack including interfaces, routing, DNS of 1 to N endpoints on 1 to N networks

### Endpoint
interfaces, switches, ports, etc & belongs to 1 network at a time

### Network
collection of endpoints that can communicated directly (bridges, VLANS) consists of 1 to N endpoints

### IPAM Problem
**Internet Protocol Address Management**

Managing addresses across multiple hosts on separate physical networks while providing routing to the underling swarm networks. This is less of an issue on a single host.

Network drivers enable IPAM through DHCP drivers or plugin drivers

## Security

### Signing
* Signed through push process
* Use `export DOCKER_CONTENT_TRUST=[1|0]` to enable/disable

### Identity Roles (UCP)

* NONE - no access to swarm
* VIEW ONLY - VIEW but cannot C, U, D
* RESTRICTED - ability to edit resources , but not run containers/services (cannot mount or exec)
* SCHEDULER - view nodes and schedule workloads. Needs additional permissions to perform other tasks
* FULL - full access to user's resources. cannot see other user's resources
