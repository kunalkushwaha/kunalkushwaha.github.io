# Tutorial setting up docker-swarm with multihost networking using docker-machine.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2016-01-14
- Tags: Docker, Swarm, Networking, Tutorial
- Canonical: https://kunalkushwaha.github.io/2016/01/14/tutorial-docker-swarm-with-multihost-networking/

[Docker-machine](https://github.com/docker/machine) is tool to create Docker hosts on computer, on cloud providers, and inside data center. It creates [Linux based](https://docs.docker.com/machine/drivers/os-base/) server, and installs and configures docker. It is also capable of configuring docker-swarm nodes.

This blog, will explain stepwise walkthrough for docker host creation using docker-machine.

## Installation.

If you use Windows or Mac, Docker has already made awesome packaged installer for you [Docker-toolbox](https://www.docker.com/products/docker-toolbox). Download and Install it.
For Linux users, you can download docker-machine binaries from [here](https://github.com/docker/machine/releases).

NOTE:  Linux users, do remember, you need to install docker client also on your machine.

## Creation of docker host with machine.

I will be using VirtualBox in this demo, but you can explore cloud options too.

Docker host can be created with command `docker-machine create`. Here `-d` flags specifies driver-name. So command looks like as follows.

`$ docker-machine create -d <driver-name> <name-of-machine>`

This command does couple of things at backend such as

- It downloads latest boot2docker image, if not locally available.
- Create a machine with `name-of-machine`
- Creates ssh keys and copies to machine.
- Installs `docker`
- Configures docker daemon at port 2376, so that docker daemon accessible at `tcp://<HostIP>:2376`

Lets create our first host `testhost`.

```
$ docker-machine create -d virtualbox testhost
Running pre-create checks...
Creating machine...
(testhost) Copying C:\Users\kunal\.docker\machine\cache\boot2docker.iso to C:\Users\kunal\.docker\machine\machines\testhost\boot2docker.iso...
(testhost) Creating VirtualBox VM...
(testhost) Creating SSH key...
(testhost) Starting VM...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect Docker to this machine, run: C:\Program Files\Docker Toolbox\docker-machine.exe env testhost
```

Here our docker host is ready for use. But wait! host is not same machine as your machine right? It is running inside Virtual Machine.
So how docker daemon is accessed remotely?

Docker works on client-server model. Docker daemon is server and it can communicates though [REST API’s](https://docs.docker.com/engine/reference/api/docker_remote_api/)

NOTE: `docker-machine create` is most important command of docker-machine. Understanding various flags help you to get best of docker-machine. You must spend some time in understanding all [flags](https://docs.docker.com/machine/reference/create/)

- To access docker host with docker client binary remotely, we need to export some environment variables.
  Docker-machine provides a handy command for printing these variables and their values for us as shown below.

  ```
$ docker-machine env testhost
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.110:2376"
export DOCKER_CERT_PATH="C:\Users\kunal\.docker\machine\machines\testhost"
export DOCKER_MACHINE_NAME="testhost"
# Run this command to configure your shell:
# eval $("C:\Program Files\Docker Toolbox\docker-machine.exe" env testhost)
```
- To export all environment variable at once, simply run

  ```
$ eval $(docker-machine env testhost)
```
- Now all docker commands will communicates with docker daemon of `testhost`

  ```
$ docker info
Containers: 0
Images: 0
Server Version: 1.9.1
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Execution Driver: native-0.2
Logging Driver: json-file
Kernel Version: 4.1.13-boot2docker
Operating System: Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015
CPUs: 1
Total Memory: 996.2 MiB
Name: testhost
ID: 26MF:TVKB:JI7Q:TL4S:7RXM:U5CD:VHIR:W2NH:CUNZ:RGB3:C6GC:POBS
Debug mode (server): true
 File Descriptors: 14
 Goroutines: 21
 System Time: 2016-01-07T07:09:51.081394779Z
 EventsListeners: 0
 Init SHA1:
 Init Path: /usr/local/bin/docker
 Docker Root Dir: /mnt/sda1/var/lib/docker
Labels:
 provider=virtualbox
```

Lets create our first docker container on this host.

```
$ docker run busybox sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
c00ef186408b: Pulling fs layer
ac6a7980c6c2: Pulling fs layer
ac6a7980c6c2: Verifying Checksum
ac6a7980c6c2: Download complete
c00ef186408b: Verifying Checksum
c00ef186408b: Download complete
c00ef186408b: Pull complete
ac6a7980c6c2: Pull complete
Digest: sha256:e4f93f6ed15a0cdd342f5aae387886fba0ab98af0a102da6276eaf24d6e6ade0
Status: Downloaded newer image for busybox:latest
```

NOTE: If you are one of user, who works behind proxy, then you may face problem like, your newly created docker host may not communicate with [docker-hub](https://hub.docker.com/).

To Fix that, you need to pass few environment flags at creation time as below.

```
$ docker-machine create -d virtualbox \
--engine-env HTTP_PROXY="http://example.com:8080" \
--engine-env HTTPS_PROXY="http://example.com:8080" \
testhost
```

So now we learnt how to create a docker host and run docker containers using docker-machine.
But Docker-machine not only lets you to create independent hosts, but also can put these host in one cluster!

Isn’t it interesting?

Let’s see how we can build whole cluster of docker-hosts using docker-machine.

Docker [Swarm](https://www.docker.com/products/docker-swarm) is native clustering solution for docker. Here with native means, Swarm understands and exports [all(almost)](https://docs.docker.com/swarm/swarm-api/) docker API’s. i.e. API for docker and docker swarm are same.
If one product/scripts works with docker, it will automatically work with swarm :)

## Docker Swarm

Docker swarm has three components.

- Discovery Backend : Swarm requires a discovery backend, which is used by each node(agent) to discover the master.
  - Default discovery is provided by Docker Hub. (Not for production usage)
- Master : Swarm master takes care of all scheduling logic and HA.
- Agent : These runs on each node and communicates with Swarm master.
  - All agent node must listen to the same network interface (TCP port).
  - Each node runs a node agent that registers the referenced Docker daemon, monitors it, and updates the discovery backend with the node’s status

In this demo, We will create a docker swarm consisting one Master and three agents including master.
Also, Consul will be used for discovery backend.

### Discovery Backend using consul.

We will create a dedicated machine for consul, but running [consul](https://www.consul.io/) using docker is just one command task.

```
$ docker-machine create -d virtualbox \
   --engine-env HTTP_PROXY="http://example.com:8080" \
   --engine-env HTTPS_PROXY="http://example.com:8080" \
   swl-consul

$ docker $(docker-machine config swl-consul) run \
   -e "http_proxy=http://example.com:8080" \
   -e "https_proxy=http://example.com:8080" \
   --restart="always" \
   -d -p "8500:8500" \
   -h "consul" \
   progrium/consul -server -bootstrap
```

Check if consul container is created as expected.

```
$ docker-machine ls
NAME         ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER   ERRORS
swl-consul   -        virtualbox   Running   tcp://192.168.99.100:2376           v1.9.1

$ eval $(docker-machine env swl-consul)

$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
5357e85abcb5        progrium/consul     "/bin/start -server -"   16 minutes ago      Up 16 minutes       53/tcp, 53/udp, 8300-8302/tcp, 8400/tcp, 8301-8302/udp, 0.0.0.0:8500->8500/tcp   goofy_aryabhata
```

### Creation of Swarm Master.

In cluster, we will have one master and 2 agents. But master will also have agent, so practically we will have 3 agents.

Docker machine helps to setup docker host with swarm in single command.

Few highlights of docker machine provisioned swarm enabled host.

- Swarm master and agent runs as docker containers.
- Swarm manager bind on “3376” port, So to communicate with master `tcp://<IP:3376>>` should be used.

  ```
$ docker-machine create \
>    -d virtualbox \
>    --swarm \
>    --swarm-master \
>    --swarm-discovery=consul://$(docker-machine ip swl-consul):8500 \
>    --engine-env HTTP_PROXY=http://example.com:8080 \
>    --engine-env HTTPS_PROXY=http://example.com:8080 \
>    --engine-env NO_PROXY=192.168.99.100 \
>    --engine-opt="cluster-store=consul://$(docker-machine ip swl-consul):8500" \
>    --engine-opt="cluster-advertise=eth1:3376" \
>    node1
Running pre-create checks...
 .
 .
(node1) Starting VM...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Configuring swarm...
Checking connection to Docker...
Docker is up and running!
To see how to connect Docker to this machine, run: C:\Program Files\Docker Toolbox\docker-machine.exe env node1

$ eval $(docker-machine env node1)

$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
7e644b71f606        swarm:latest        "/swarm join --advert"   19 seconds ago      Up 16 seconds                           swarm-agent
58f2a4da438b        swarm:latest        "/swarm manage --tlsv"   22 seconds ago      Up 19 seconds                           swarm-agent-master
```

NOTE: You may have seen `NO_PROXY` is also set as environment variable.

This is to skip proxy for communicating with consul server

Here,

- `--swarm` indicates, swarm agent will be configured.
- `--swarm-master` indicates, swarm master will be configured.
  `--swarm-discovery` sets the backend discovery for swarm.

Here you go, we have our swarm master node up and running. Before communicating to swarm, let me remind you again.

- Swarm and docker talks in same API signature. But on our host, we have both docker daemon and swarm running.
- Our docker client can communicate with both, but we need to choose whom it has to talk.
  - To talk with docker daemon use `eval $(docker-machine env <host-name>)`
  - To talk with swarm use `eval $(docker-machine env --swarm <host-name>`

So, since we want our docker client should communicate with swarm, we will add `--swarm`

```
$ eval $(docker-machine env --swarm node1)

$ docker info
Containers: 2
Images: 1
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 1
 node1: 192.168.99.102:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
CPUs: 1
Total Memory: 1.021 GiB
Name: node1
```

Now add two nodes with only agent.

```
$ docker-machine create \
>    -d virtualbox \
>    --swarm \
>    --swarm-discovery="consul://$(docker-machine ip swl-consul):8500" \
>    --engine-env HTTP_PROXY="http://example.com:8080" \
>    --engine-env HTTPS_PROXY="http://example.com:8080" \
>    --engine-env NO_PROXY="192.168.99.100" \
>    --engine-opt="cluster-store=consul://$(docker-machine ip swl-consul):8500" \
>    --engine-opt="cluster-advertise=eth1:3376" \  
>    node3
```

- Command for adding agent doesn’t requires `--swarm-master`.
- `cluster-advertise` and `cluster-store` are required for overlay networking also, Will be explained later.

So after adding two more machines with swarm agent, we have 4 machines running as below.

```
$ docker-machine ls
NAME         ACTIVE   DRIVER       STATE     URL                         SWARM            DOCKER   ERRORS
node1        -        virtualbox   Running   tcp://192.168.99.102:2376   node1 (master)   v1.9.1
node2        -        virtualbox   Running   tcp://192.168.99.103:2376   node1            v1.9.1
node3        -        virtualbox   Running   tcp://192.168.99.104:2376   node1            v1.9.1
swl-consul   -        virtualbox   Running   tcp://192.168.99.100:2376                    v1.9.1
```

Lets see information of our cluster.

```
$ eval $(docker-machine env --swarm node1)

$ docker info
Containers: 4
Images: 3
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 3
 node1: 192.168.99.102:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
 node2: 192.168.99.103:2376
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
 node3: 192.168.99.104:2376
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
CPUs: 3
Total Memory: 3.064 GiB
Name: node1
```

Superb! we have three nodes running in our cluster.
This completes the swarm setup!!

## Understanding few more Swarm functionality.

1. Strategy

Currently “Spread” strategy is set. So the containers will be spread over all the hosts in cluster.

e.g.

```
$ docker run -d -P -m 1G -e MYSQL_ROOT_PASSWORD=test123 --name db mysql
57a1af46d72117306b833a28a219d3523e1531c98a19ef25cba00a7bc94c9645

$ docker run -d -P -m 1G --name frontend nginx
f40776d7c9f7583965263340c1542a09eb7cb881c53bcf18a290df0d6ce2fd51

$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                         NAMES
f40776d7c9f7        nginx               "nginx -g 'daemon off"   4 seconds ago       Up 4 seconds        192.168.99.103:32770->80/tcp, 192.168.99.103:32769->443/tcp   node2/frontend
57a1af46d721        mysql               "/entrypoint.sh mysql"   3 minutes ago       Up 3 minutes        192.168.99.104:32769->3306/tcp                                node3/db
```

- Here node3 have `db` and node2 have `frontend` container.
- If two nodes have the same amount of available RAM and CPUs, the spread strategy prefers the node with least containers.

Other available strategies are BinPack and Random.

- BinPack tries to pack all possible containers on single node first and so on.
- If two nodes have the same amount of available RAM and CPUs, the binpack strategy prefers the node with most containers.

Things to Try:

- Create swarm with BinPack and create multiple containers to see the behavior.

### Filters

Filters tell Docker Swarm scheduler which nodes to use when creating and running a container.

Filters are divided into two categories, node filters and container configuration filters.

- Node filters operate on characteristics of the Docker host or on the configuration of the Docker daemon.
- Container configuration filters operate on characteristics of containers, or on the availability of images on a host.
- Each filter has a name that identifies it.

  The node filters are:

  - constraint
  - health

  The container configuration filters are:

  - affinity
  - dependency
  - port

When you start a Swarm manager with the swarm manage command, all the filters are enabled.
If you want to limit the filters available to your Swarm, specify a subset of filters by passing the `--filter` flag and the name:

```
$ swarm manage --filter=health --filter=dependency
```

In case of docker machine, while provisioning `--swarm-opt` can be used to set filters.

#### Use a constraint filter

Node constraints can refer to Docker’s default tags or to custom labels. Default tags are sourced from docker info. Often, they relate to properties of the Docker host. Currently, the dafult tags include:

- node to refer to the node by ID or name
- storagedriver
- executiondriver
- kernelversion
- operatingsystem
- Custom node labels can be applied while provisioning docker machine.
  - `--engine-label` can be used for setting custom label.
  - custom label like `environment`, `storage` etc are helpful.

Since we have all labels same except node name, we will try creating new container using node-name constraint.

```
$ docker run -itd -e constraint:node==node3 --name test1 ubuntu
ae4013d014931e43cd5342a625f6b549a8c920cb146b1371a7226359a4bcf014

$ docker run -itd -e constraint:node==node3 --name test2 ubuntu
a6b371773e1a2609122ca68df00d1e03ee274ba3db504d3942c301f8e2c45504

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a6b371773e1a        ubuntu              "/bin/bash"         3 seconds ago       Up 2 seconds                            node3/test2
ae4013d01493        ubuntu              "/bin/bash"         19 seconds ago      Up 18 seconds                           node3/test1
934501aed30f        ubuntu              "/bin/bash"         19 hours ago        Up 2 hours                              node3/n4
30358292be89        ubuntu              "/bin/bash"         19 hours ago        Up 19 hours                             node1/n3
6a13aa3eca8b        ubuntu              "/bin/bash"         21 hours ago        Up 21 hours                             node2/n1
```

Similarly other filters can be used. While using filters the syntax to use in `docker run` is as below.

```
- ``-e <filter-name>:<key><operator><value>``
- Here operator used are  ``==`` , ``!=`` & ``~``
- For ``==`` & ``!=``, exact match is found.
  - If nothing satisfies the condition, it does schedule container.
- ``~`` is for soft condition. i.e. if nothing matches, it ignores the condition and use default scheduler.
- Here value can contain any valid regex (https://github.com/google/re2/wiki/Syntax)
```

Hope this article will be helpful in getting started with docker-machine and swarm :)
