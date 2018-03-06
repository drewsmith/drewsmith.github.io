---
title:  "Docker Notes"
categories: DevOps Docker
read_time: 15
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
# container with new, not host volume
-V /mount/point
# service
--mount source=myvolumename,target=/mount/point
# for host file (type=bind)
--mount source=X,target=Y,type=bind
```

* Important component of making containers portable

# Networking

* The ability for any node in a cluster to answer for an exposed service port even if there is no replica for that service running on it, is handled by **routing mesh**.
* default network interface is `docker0`

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
* Use `-P` to publish the container a host port above 32xxx
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

* Docker uses PID & Network namespaces to maintain isolation

### Signing
* Signed through push process
* Use `export DOCKER_CONTENT_TRUST=[1|0]` to enable/disable

### Identity Roles (UCP)

* NONE - no access to swarm
* VIEW ONLY - VIEW but cannot C, U, D
* RESTRICTED - ability to edit resources , but not run containers/services (cannot mount or exec)
* SCHEDULER - view nodes and schedule workloads. Needs additional permissions to perform other tasks
* FULL - full access to user's resources. cannot see other user's resources

# Swarm
1 or more managers, 1 or more workers
Maintain quorum (majority), min. HA quorum = 3

## Init
```
docker swarm init --advertise-addr [IP Address] > manager.out
```

Join another manager to the swarm

```
docker swarm join-token manager
```

## Add Worker Nodes

Get command from manager to join as worker

```
docker swarm join-token worker
```

List swarm nodes

```
docker node ls
```

## Remove worker

from worker

```
docker swarm leave
```

from manager

```
docker node rm [node id]
```

## Backup/Restore

```
sudo systemctl stop docker
sudo su -
cp -rf /var/lib/docker/swarm /backupdir/swarm
tar cvf swarm.tar /backupdir/swarm
scp swarm.tar user@node
ssh user@node
tar xvf swarm.tar
mv swarm /var/lib/docker
```

# Namespaces

Provides isolation so that other pieces of the system are unaffected by whatever is in the namespace

* PID
* Mount
* IPC (interprocess communication)
* User namespaces
* Network

# cgroups

provides means for allocation and granular control of resources

* CPU, Memory, Network Bandwidth, Disk, Priority

# Copy on Write (CoW)

* `fork()` to create process
* write without permission = segfault
* Docker uses UnionMount for copy on write

## Storage Driver

### AUFS
* legacy
* Copy up to top level for write
* mount() is fast so containers are quick

### Devicemapper
* complex
* copy on write at block instead of file
* each container gets a block device
* each container gets a virtual disk, easier to port or limit
* uses data and metadata sparse files which are large, which makes CoW slow

### BTRFS
* snapshat at subvolume level

### Overlayfs
* ufs but in kernel

### VFS
* not copy on write, it's copy on copy
* space ineffecient and slow
* use for legacy

## TL;DR
* PaaS = use AUFS or overlfs
* Big CoW = BTRFS or DeviceMapper

