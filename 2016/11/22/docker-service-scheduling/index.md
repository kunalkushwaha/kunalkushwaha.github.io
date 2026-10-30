# Docker Services Scheduling

> Understanding how docker services scheduling works in swarm mode

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2016-11-22
- Tags: Docker, Swarm
- Canonical: https://kunalkushwaha.github.io/2016/11/22/docker-service-scheduling/

#### Overview

Docker service scheduling is one of important aspect in planing and deployment of service over docker cluster.
Your Docker swarm mode default’s scheduling strategy is “spread”. With spread strategy, swarm scheduler ranks node based on available CPU,RAM and number of containers.

*NOTE: Docker containers eat resources during its life-cycle irrespective of their state, even containers with `Exited` state. Hence scheduler consider all containers including container with `Exited` state*

# TODO: Why needs to be controlled?

But it can be controlled by adding various constraints.

Constraints: Operators can control or limit the set of nodes on which tasks can be scheduled.

Constraints: Operators can limit the set of nodes where a task can be scheduled by defining constraint expressions. Multiple constraints find nodes that satisfy every expression, i.e., an AND match. Constraints can match node attributes in the following table. Note that engine.labels are collected from Docker Engine with information like operating system, drivers, etc. node.labels are added by cluster administrators for operational purpose. For example, some nodes have security compliant labels to run tasks with compliant requirements.

| node attribute | matches | example |
| --- | --- | --- |
| node.id | node’s ID | `node.id == 2ivku8v2gvtg4` |
| node.hostname | node’s hostname | `node.hostname != node-2` |
| node.role | node’s manager or worker role | `node.role == manager` |
| node.labels | node’s labels added by cluster admins | `node.labels.security == high` |
| engine.labels | Docker Engine’s labels | `engine.labels.operatingsystem == ubuntu 14.04` |

Why node lebels when docker engines had engine labels?

- Engine labels can’t be trusted for security sensitive scheduling decisions, since any worker can report any label up to a manager. They are, however, useful for things like “Does this host have an SSD?", or “how much ram/disk space does this host have?".

These swarm labels, on the other hand, are always trustworthy, since they are set by an administrator explicitly. This means that we can label a host as pci-dss, and create a constraint to ensure that credit-card-app only gets scheduled on worker nodes that are labeled as pci-dss, and that no malicious worker can self-label itself as pci-dss.

```
$ docker-machine create -d virtualbox --engine-env HTTP_PROXY="http://proxy.rdh.ecl.ntt.co.jp:20066" --engine-env HTTPS_PROXY="http://proxy.rdh.ecl.ntt.co.jp:20066" --engine-label node=node2 --engine-label storage=ssd node2
Running pre-create checks...
Creating machine...
(node2) Unable to get the latest Boot2Docker ISO release version:  Get https://api.github.com/repos/boot2docker/boot2docker/releases/latest: x509: certificate is valid for inetnf.auth, not api.github.com
(node2) Copying /Users/kunalkushwaha/.docker/machine/cache/boot2docker.iso to /Users/kunalkushwaha/.docker/machine/machines/node2/boot2docker.iso...
(node2) Creating VirtualBox VM...
(node2) Creating SSH key...
(node2) Starting the VM...
(node2) Check network to re-create if needed...
(node2) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env node2

$ eval $(docker-machine env node2)

$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.3
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: overlay null host bridge
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Security Options: seccomp
Kernel Version: 4.4.27-boot2docker
Operating System: Boot2Docker 1.12.3 (TCL 7.2); HEAD : 7fc7575 - Thu Oct 27 17:23:17 UTC 2016
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 995.8 MiB
Name: node2
ID: 5HAZ:LAGO:KVF4:WLRP:P356:O6IA:W6GD:3LYO:JGDJ:5QN6:U6L6:J3BZ
Docker Root Dir: /mnt/sda1/var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 13
 Goroutines: 23
 System Time: 2016-11-22T02:20:13.630275393Z
 EventsListeners: 0
Http Proxy: http://proxy.rdh.ecl.ntt.co.jp:20066
Https Proxy: http://proxy.rdh.ecl.ntt.co.jp:20066
Username: kunalkushwaha
Registry: https://index.docker.io/v1/
Labels:
 node=node2
 storage=ssd
 provider=virtualbox
Insecure Registries:
 127.0.0.0/8

 $ eval $(docker-machine env default)

~/Downloads ⌚ 11:21:03
$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (4r4qnxd8ztxuiusldvrf9gub6) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1b7a6a6ophwlo85ihe3ss2lkg678d0re52p5dqezk5m3xtwur5-2vsgk9in2cm2b97f3gg3xmii4 \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

~/Downloads ⌚ 11:21:22
$ eval $(docker-machine env node2)

~/Downloads ⌚ 11:21:37
$ docker swarm join \
    --token SWMTKN-1-1b7a6a6ophwlo85ihe3ss2lkg678d0re52p5dqezk5m3xtwur5-2vsgk9in2cm2b97f3gg3xmii4 \
    192.168.99.100:2377
This node joined a swarm as a worker.

$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.3
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge null overlay host
Swarm: active
 NodeID: 5z607lfu3sigaliuzcpo0wsrf
 Is Manager: false
 Node Address: 192.168.99.101
Runtimes: runc
Default Runtime: runc
Security Options: seccomp
Kernel Version: 4.4.27-boot2docker
Operating System: Boot2Docker 1.12.3 (TCL 7.2); HEAD : 7fc7575 - Thu Oct 27 17:23:17 UTC 2016
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 995.8 MiB
Name: node2
ID: 5HAZ:LAGO:KVF4:WLRP:P356:O6IA:W6GD:3LYO:JGDJ:5QN6:U6L6:J3BZ
Docker Root Dir: /mnt/sda1/var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 22
 Goroutines: 68
 System Time: 2016-11-22T02:21:46.868274301Z
 EventsListeners: 0
Http Proxy: http://proxy.rdh.ecl.ntt.co.jp:20066
Https Proxy: http://proxy.rdh.ecl.ntt.co.jp:20066
Username: kunalkushwaha
Registry: https://index.docker.io/v1/
Labels:
 node=node2
 storage=ssd
 provider=virtualbox
Insecure Registries:
 127.0.0.0/8

 docker service create \
  --replicas 2 \
  --name helloworld1 \
  --constraint 'engine.labels.storage == ssd' \
  alpine ping docker.com

  ~/work/GlobalMentorWeek on  master! ⌚ 13:53:25
  $ docker node update --label-add key1=master default
  default

  ~/work/GlobalMentorWeek on  master! ⌚ 13:54:05
  $ docker node inspect 4r4qnxd8ztxuiusldvrf9gub6 --pretty
  ID:			4r4qnxd8ztxuiusldvrf9gub6
  Labels:
   - key = primary
   - key1 = master
  Hostname:		default
  Joined at:		2016-11-22 02:21:21.596149476 +0000 utc
  Status:
   State:			Ready
   Availability:		Active
  Manager Status:
   Address:		192.168.99.100:2377
   Raft Status:		Reachable
   Leader:		Yes
  Platform:
   Operating System:	linux
   Architecture:		x86_64
  Resources:
   CPUs:			1
   Memory:		995.8 MiB
  Plugins:
    Network:		bridge, host, null, overlay
    Volume:		local
  Engine Version:		1.12.3
  Engine Labels:
   - node = node1
   - provider = virtualbox
   - storage = hdd

$ docker service create \
  --replicas 2 \
  --name helloworld2 \
  --constraint 'node.labels.key == primary' \
  alpine ping docker.com
b2g40mrmok4ii9wcmqdlgv7s0
```
