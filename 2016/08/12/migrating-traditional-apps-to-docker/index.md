# Migrating traditional apps to docker

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2016-08-12
- Tags: Docker, Migration
- Canonical: https://kunalkushwaha.github.io/2016/08/12/migrating-traditional-apps-to-docker/

If title of this blog, attracted you towards this blog, Most likely, you will be one of us, who want to migrate old traditional application’s in container environment. This blog covers my experience of Migrating old traditional application into Docker.

Before beginning with migration procedure, Let me explain requirements I had to met with migrated system.

### Problems

- Old apps running on servers/VMs, resource utilization not optimal.
- High maintenance cost.
  - OS running is no longer supported by vendors, so in-house support required.
  - Maintenance downtime high.
- Platform to adopt DevOps practice for fast refactoring.

### Challenges

- Migrate Application with least changes in existing platform and infrastructure use.
  - Use same OS distribution.
  - No rewriting application for new OS.
  - No changes in network infrastructure.
- Behavioral changes for accessing the application should be none.
  - Use hardware load balancer.
  - Use Fixed-IP with pre-approved MAC address.

### Solution

Migrating old applications into containers can be divided into two parts.

1. Application code/binary migration into image.
2. Application deployment, as per requirements.

The first part is specific to each application, as every application have different challenges for migration.
I will not discuss details of that. Though few of things need be considered before migrating.

#### Docker Host

Which OS/distribution to be chosen for docker host.
Ideally any distribution which have LTS should be fine.

Though, sometime you may need to consider alternate from latest due compatibility with application.
To check application compatibility, follow the below Checklist.

#### Application compatibility Checklist

- Check if any changes in ABI for linux kernel functionality and libraries used by application.
  - If ABI breaks, either we need to choose old kernel, where you may need to work little bit to make docker work properly.
  - OR, recompile application with new libraries.
- Any special Kernel settings required by application.
  - Host kernel parameters may need to be tweaked and/or cgroups/namespaces need to be adjusted accordingly.

After detailed analysis, we can decide and configure Docker Host.

#### Application migration

Application migration in my case is simple. Just containerize the whole application.
i.e. Build an image/rootfs with old libraries and application code. Typically old application requires more then one process to run, using [Supervisord](https://docs.docker.com/engine/admin/using_supervisord/), it can be achieved.

Now once, you successfully able build & test image for application, Lets focus on how it can be deployed with least changes to current setup.

### Deployment

As started in challenges section, we are using hardware load-balancer and want to continue the same.
Also, this application is bind to fixed-IP and specific MAC address.

Since docker supports [macvlan](https://github.com/docker/libnetwork/blob/master/docs/macvlan.md) network driver, the above two goals can be easily achieved.
Using macvlan driver, we can put a container on same network as old servers. Also macvlan user less CPU and slightly better [throughput](http://events.linuxfoundation.org/sites/events/files/slides/LinuxConJapan2014_makita_0.pdf) then bridge network.

Lets go through its details.

On docker host, lets create a macvlan network.

```
$ docker network create -d macvlan --subnet=172.20.20.0/18 --gateway=172.20.0.1 -o parent=enp1s0 -o macvlan_mode=bridge maclan
798b8a2ee7988dc0388c9985eaa7f0bc9373f11cf1be4a3b3a44abee162442fd

$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9a9ff4851e71        bridge              bridge              local               
ddbc211bcf8f        docker_gwbridge     bridge              local               
a7211f1ae74d        host                host                local               
8aetfb2dtwmu        ingress             overlay             swarm               
798b8a2ee798        maclan              macvlan             local               
d43c2f84509b        none                null                local               
```

*NOTE: macvlan network is local to host, so this cannot be used over swarm.*

Now, I need to create an network interfaces for my containers. I need to deploy two containers as replacement of two Application servers.
For security reason, I need to assign predefined MAC address to this interface. But with macvlan driver this is also possible.

Now my network is ready, I can deploy my applications on same.
Lets create containers on these network.

```
$ docker run --net=maclan --ip=172.20.20.51 --mac-address 08:00:27:0B:1C:FE -itd alpine sh                                                                                                     
1f0307c12ea7a666bffc4224665f304aae67056a7cbc6f037ec19d73b5a8a64d

$ docker run --net=maclan --ip=172.20.20.52 --mac-address 08:00:27:3F:FE:8E -itd alpine sh                                                                                                     
72849de4f1e23b2bd2394ef579c48045260b4f8f0026a601c5ac93514bc90e0d

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
72849de4f1e2        alpine              "sh"                5 seconds ago       Up 2 seconds                            grave_pike
1f0307c12ea7        alpine              "sh"                49 seconds ago      Up 46 seconds                           elegant_yalow
```

Test it and containers are reachable from other machine on network.

```
$ ping 172.20.20.51
PING 172.20.20.51 (172.20.20.51): 56 data bytes
64 bytes from 172.20.20.51: seq=0 ttl=64 time=0.090 ms
64 bytes from 172.20.20.51: seq=1 ttl=64 time=0.057 ms
64 bytes from 172.20.20.51: seq=2 ttl=64 time=0.058 ms
^C
--- 172.20.20.51 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.057/0.068/0.090 ms

$ ping 172.20.20.52
PING 172.20.20.52 (172.20.20.52): 56 data bytes
64 bytes from 172.20.20.52: seq=0 ttl=64 time=0.075 ms
64 bytes from 172.20.20.52: seq=1 ttl=64 time=0.058 ms
64 bytes from 172.20.20.52: seq=2 ttl=64 time=0.066 ms
^C
--- 172.20.20.52 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.058/0.066/0.075 ms
```

Now these containers have successfully replaced the two app servers, and for external load balancer, there is no changes.

*NOTE: Alternative to macvlan is IPVlan, but since it is not yet stable in docker, I choose macvlan*

So what are the Benefits, I finally achieved through this migration.

### Benefits

- Portability: Application,can be migrated to different machines easily.
- Flexibility: Image can be shared and experiment within team.
- Modularity: Provides a base to break application into further smaller services.
- Reuse: Reuse current infrastructure like network, load balancer. Least disturbance.

Hope this blog will help you to find some answers, if you are looking for migrating legacy/old/monolithic applications to docker.
