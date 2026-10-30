# docker 1.12RC demo

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2016-07-25
- Tags: Docker, Swarm
- Canonical: https://kunalkushwaha.github.io/2016/07/25/docker-1.12rc-demo/

Last friday (22nd July), I gave DockerCon16 Recap Talk and demo at [Docker Meetup Tokyo](https://www.meetup.com/Docker-Tokyo/events/231737938/)

This blog post gives a walk through my Docker Meetup demonstration, and you can follow these steps to try at your machines.

In DockerCon16, docker 1.12 RC was announced. This release has an important & interesting feature called docker *swarm mode*

### Get Docker 1.12 RC.

If you are using [Docker for Mac](https://docs.docker.com/engine/installation/mac/#/docker-for-mac) or [Docker for Windows](https://docs.docker.com/engine/installation/windows/#/docker-for-windows), You already have docker 1.12RC installed on your machine.

You can check version

```
$ docker version
Client:
 Version:      1.12.0-rc4
 API version:  1.24
 Go version:   go1.6.2
 Git commit:   e4a0dbc
 Built:        Wed Jul 13 03:28:51 2016
 OS/Arch:      darwin/amd64
 Experimental: true

Server:
 Version:      1.12.0-rc4
 API version:  1.24
 Go version:   go1.6.2
 Git commit:   e4a0dbc
 Built:        Wed Jul 13 03:28:51 2016
 OS/Arch:      linux/amd64
 Experimental: true
```

In case you are using linux or not using above, you can get latest releases from <https://github.com/docker/docker/releases>

### Create demo docker hosts.

To create a swarm of docker hosts, you require more then one docker-host. I use [docker-machine](https://docs.docker.com/machine/) to create my docker hosts.

With following commands, my three docker hosts will be ready with docker 1.12 RC installed with default [boot2docker](http://boot2docker.io/) image.

```
$docker-machine -d virtualbox manager

$docker-machine -d virtualbox worker1

$docker-machine -d virtualbox worker2

$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
manager   *        virtualbox   Running   tcp://192.168.99.100:2376           v1.12.0-rc4
worker1   -        virtualbox   Running   tcp://192.168.99.101:2376           v1.12.0-rc4
worker2   -        virtualbox   Running   tcp://192.168.99.102:2376           v1.12.0-rc4
```

*Note: If you work behind proxy, use command as `docker-machine -d virtualbox -engine-env http_proxy="example.com:port" manager`*

Once, all machines are up and running, we can create swarm.

### Initialize swarm.

To run command on my *manager* node, I need to set my environment variables, so all docker commands gets executed at manager.
This can be achieved by single command as follows.

```
$ eval $(docker-machine env manager)
```

Now initialize docker swarm mode.

```
$ docker swarm init --listen-addr $(docker-machine ip manager):2377
No --secret provided. Generated random secret:
	aj92ivakk282slwal0ujwaloj

Swarm initialized: current node (2ksrqgk2x3vbjh3bw0aly5dwr) is now a manager.

To add a worker to this swarm, run the following command:
	docker swarm join --secret aj92ivakk282slwal0ujwaloj \
	--ca-hash sha256:1f7176d2474cf8dd3fa7a29e46ce42250c5a0aecaf07e40e014f039a7bf1e5ba \
	192.168.99.100:2377
```

The above command initializes and generates relevant secrets and CA Key for swarm.

Currently swarm is created with only one node “master”

```
$ docker node ls
ID                           HOSTNAME  MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
2ksrqgk2x3vbjh3bw0aly5dwr *  manager   Accepted    Ready   Active        Leader
```

### Adding Workers node

To add swarm workers `docker swarm join` command will be used.

```
$ eval $(docker-machine env worker1)

$ docker swarm join --secret aj92ivakk282slwal0ujwaloj \
>     --ca-hash sha256:1f7176d2474cf8dd3fa7a29e46ce42250c5a0aecaf07e40e014f039a7bf1e5ba \
>     192.168.99.100:2377
This node joined a Swarm as a worker.

$ eval $(docker-machine env worker2)

$ docker swarm join --secret aj92ivakk282slwal0ujwaloj \
    --ca-hash sha256:1f7176d2474cf8dd3fa7a29e46ce42250c5a0aecaf07e40e014f039a7bf1e5ba \
    192.168.99.100:2377
This node joined a Swarm as a worker.
```

To view complete list of nodes in swarm, we need to query swarm master.

*NOTE: In case of multiple masters, query can be made to any master*

```
$ eval $(docker-machine env manager)

$ docker node ls
ID                           HOSTNAME  MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
0cs0e5phve5onp41pxfe9c1kj    worker1   Accepted    Ready   Active
2ksrqgk2x3vbjh3bw0aly5dwr *  manager   Accepted    Ready   Active        Leader
c6nvb1ljj9mntt1v8qlmxl2my    worker2   Accepted    Ready   Active
```
> ***Thats it! two commands create whole docker swarm. No external KV Store, No tricky CA Key generation, It all just two commands***

### Monitoring events of swarm

In swarm mode all managers have consistent view that enables, the monitoring of whole swarm through any manager node.
All you need to listen at /var/run/docker.sock of any manager node.

### Creating Visualizer for Swarm.

Visualizer is a simple nodejs app, which listen on `/var/run/docker.sock` and show nodes present in swarm and containers within as boxes.

```
$ docker run -it -d -p 8080:8080 -e HOST=$(docker-machine ip manager) -v /var/run/docker.sock:/var/run/docker.sock manomarks/visualizer
```

Open the visualizer in browser with http://192.168.99.100:8080, here “192.168.99.100” is manager node IP.

### Lets deploy some Application in swarm

Till now applications can be deployed using `docker run` command. To support orchestration, docker 1.12 have introduced `docker service` command. Using `service` commands, we can define properties of an application, which docker swarm mode tries to reconcile in case of any application errors or node failure.

For demo, I have a simple application [demoapp](https://github.com/kunalkushwaha/slides/tree/master/examples/webserver), which is a simple web-server listen on port `5000`, prints the some message. Lets try to deploy this application.

Since, I want my application to be spread all over swarm, and all instance should be discoverable, at first, I will create an [overlay network](https://en.wikipedia.org/wiki/Overlay_network).

Lets create overlay network using `docker network create` command.

```
$ docker network create -d overlay mynet
2xjpxwugr0glog7sqywdsnbn6

$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
01e7e286803b        bridge              bridge              local
7a512b485351        docker_gwbridge     bridge              local
6712d943241e        host                host                local
dyk8o6w5cddo        ingress             overlay             swarm
2xjpxwugr0gl        mynet               overlay             swarm
ae968fa64609        none                null                local
```

In above list of docker networks, there are two overlay networks.

- ingress is default overlay network which uses [IPVS](http://www.linuxvirtualserver.org/software/ipvs.html) and blazing fast due to kernel only data path. This is used for exposing services to external [load balancers](https://docs.docker.com/engine/swarm/key-concepts/#/load-balancing).

### Create service

Lets create a service with 2 replicas(instance) attached to `mynet` from image [kunalkushwaha/demoapp\_image](https://hub.docker.com/r/kunalkushwaha/demoapp_image/)

```
$ docker service create \
> --name demoapp \
>     --replicas 2 \
>     --network mynet \
> -p 5000:5000 \
> kunalkushwaha/demoapp_image:v1
8fwyujxnc1rm0jwewx5tfeai1
```

`Service create` commands defines that service and docker swarm manager, picks up the definition to schedule it.
You can monitor, service state by following commands.

`docker service ls` just lists the service and shows how many instance running.
`docker service tasks` command give details like on which node service instance is running.

```
$ docker service ls
ID            NAME     REPLICAS  IMAGE                           COMMAND
8fwyujxnc1rm  demoapp  0/2       kunalkushwaha/demoapp_image:v1

$ docker service tasks demoapp
ID                         NAME       SERVICE  IMAGE                           LAST STATE              DESIRED STATE  NODE
a2yczjfyhyryg4pkg8m27ztd6  demoapp.1  demoapp  kunalkushwaha/demoapp_image:v1  Running 21 seconds ago  Running        manager
aj080rydyghazbhco6h9dhaaw  demoapp.2  demoapp  kunalkushwaha/demoapp_image:v1  Running 21 seconds ago  Running        worker2
```

To read configuration of service, you can use `docker service inspect` command.

```
$ docker service inspect  demoapp --pretty
ID:		8fwyujxnc1rm0jwewx5tfeai1
Name:		demoapp
Mode:		Replicated
 Replicas:	2
Placement:
 Strategy:	Spread
UpdateConfig:
 Parallelism:	0
ContainerSpec:
 Image:		kunalkushwaha/demoapp_image:v1
Resources:
Reservations:
Limits:
Networks: 2xjpxwugr0glog7sqywdsnbn6Ports:
 Name =
 Protocol = tcp
 TargetPort = 5000
 PublishedPort = 5000
```

Now our service is deployed successfully, lets try to access and see if, it works properly.

```
$ curl 192.168.99.100:5000
This is DemoApp v1
```

So far all good! Now lets try to scale this service by 6 instances.

```
$ docker service scale demoapp=6
demoapp scaled to 6

$ docker service ls
ID            NAME     REPLICAS  IMAGE                           COMMAND
8fwyujxnc1rm  demoapp  4/6       kunalkushwaha/demoapp_image:v1

$ docker service ps demoapp
ID                         NAME       SERVICE  IMAGE                           LAST STATE                DESIRED STATE  NODE
a2yczjfyhyryg4pkg8m27ztd6  demoapp.1  demoapp  kunalkushwaha/demoapp_image:v1  Running 13 minutes ago    Running        manager
aj080rydyghazbhco6h9dhaaw  demoapp.2  demoapp  kunalkushwaha/demoapp_image:v1  Running 13 minutes ago    Running        worker2
7c0dbomrbljlqdq9gitthjg1f  demoapp.3  demoapp  kunalkushwaha/demoapp_image:v1  Running 10 seconds ago    Running        manager
dzooellh3a2vsbsr2soesfhir  demoapp.4  demoapp  kunalkushwaha/demoapp_image:v1  Running 10 seconds ago    Running        worker2
5r114sf8i0n5o53s0hwv911g4  demoapp.5  demoapp  kunalkushwaha/demoapp_image:v1  Preparing 10 seconds ago  Running        worker1
85fbs7z0ntazachrxw98xnjfo  demoapp.6  demoapp  kunalkushwaha/demoapp_image:v1  Preparing 10 seconds ago  Running        worker1
```

If you observe carefully, column of `LAST STATE` & `DESIRED STATE` shows `demoapp.5` and `demoapp.6` as “Preparing” and “Running”
This is same as explained above i.e. `docker scale` command changes the configuration of service. Now docker swarm mode is trying to achieve desired state.

### Node failure.

Let try to delete one of worker node and see how swarm it tries to reconcile the configuration of service.

```
$ docker-machine rm worker2
About to remove worker2
Are you sure? (y/n): y
Successfully removed worker2

$ docker service ps demoapp
ID                         NAME       SERVICE  IMAGE                           LAST STATE              DESIRED STATE  NODE
a2yczjfyhyryg4pkg8m27ztd6  demoapp.1  demoapp  kunalkushwaha/demoapp_image:v1  Running 27 minutes ago  Running        manager
5afbnsfl5p3xyygbk1b6aawc6  demoapp.2  demoapp  kunalkushwaha/demoapp_image:v1  Accepted 3 seconds ago  Accepted       worker1
7c0dbomrbljlqdq9gitthjg1f  demoapp.3  demoapp  kunalkushwaha/demoapp_image:v1  Running 13 minutes ago  Running        manager
8lgqtsewc0vbgh318fx27sm0m  demoapp.4  demoapp  kunalkushwaha/demoapp_image:v1  Accepted 3 seconds ago  Accepted       manager
5r114sf8i0n5o53s0hwv911g4  demoapp.5  demoapp  kunalkushwaha/demoapp_image:v1  Running 13 minutes ago  Running        worker1
85fbs7z0ntazachrxw98xnjfo  demoapp.6  demoapp  kunalkushwaha/demoapp_image:v1  Running 13 minutes ago  Running        worker1
```

You can see now, worker2’s instances (`demoapp.2` & `demoapp.4`) got rescheduled on manager and worker1. All by its own :).

### Rolling updates.

I have v2 of demoapp, which appends IP address of container to message. Lets try to upgrade the application.
Here we can define how many instances should get upgraded at one point `--update-parallelism` and also delay between two updates `--update-delay`.

These both are important feature, which helps to upgrade the application without any downtime. Also, while upgrading if some error occurs, you can rollback the service.

In this example, I will try to upgrade `2` instances at a time and with delay of `30s`.

```
docker service update demoapp --update-parallelism=2 --update-delay 10s --image kunalkushwaha/demoapp_image:v2
```

Now try to get the output of app.

```
$ curl  192.168.99.100:5000
This is DemoApp v2 @  IP: 10.255.0.13

$ curl 192.168.99.100:5000
This is DemoApp v2 @  IP: 10.255.0.9

$ curl 192.168.99.100:5000
This is DemoApp v1

$ curl 192.168.99.100:5000
This is DemoApp v1

$ curl 192.168.99.100:5000
This is DemoApp v1
```

You can see output is mixed from v1 and v2 application instance. This is due to docker’s internal load-balancer. Docker has embedded DNS, which is used for service discovery and load balancing. By default its round-robin.

### Routing Mesh.

With routing mesh, service discovery of services in swarm can be done through any node of swarm. i.e. even if service instance not running on any particular node, still if request comes of application/service port, it will be redirected to one of running instance of service. This is achieved by `ingress` network.

To demonstrate, lets add one more node into swarm.

```
$ docker-machine create -d virtualbox worker3

$ eval $(docker-machine env worker3)

$ docker swarm join --secret aj92ivakk282slwal0ujwaloj \
    --ca-hash sha256:1f7176d2474cf8dd3fa7a29e46ce42250c5a0aecaf07e40e014f039a7bf1e5ba \
    192.168.99.100:2377
This node joined a Swarm as a worker.

$ eval $(docker-machine env manager)

$ docker node ls
ID                           HOSTNAME  MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
0cs0e5phve5onp41pxfe9c1kj    worker1   Accepted    Ready   Active
2ksrqgk2x3vbjh3bw0aly5dwr *  manager   Accepted    Ready   Active        Leader
5gfj06su4l885zg88qyze7imu    worker3   Accepted    Ready   Active
c6nvb1ljj9mntt1v8qlmxl2my    worker2   Accepted    Down    Active
```

Now, demoapp is running only on worker1 and manager nodes.

```
$ docker service ps demoapp
ID                         NAME       SERVICE  IMAGE                           LAST STATE              DESIRED STATE  NODE
9ucohwuw91e8uuquhxidhplsu  demoapp.1  demoapp  kunalkushwaha/demoapp_image:v2  Running 15 minutes ago  Running        manager
a1odej6m4pz1k4lowzt2l48rz  demoapp.2  demoapp  kunalkushwaha/demoapp_image:v2  Running 16 minutes ago  Running        worker1
crm4zjjb1hjxdr59zraef4yj9  demoapp.3  demoapp  kunalkushwaha/demoapp_image:v2  Running 15 minutes ago  Running        worker1
9q84itlob65mbzzvpb1dyjtij  demoapp.4  demoapp  kunalkushwaha/demoapp_image:v2  Running 15 minutes ago  Running        worker1
7bg8r15bq617ghupei8yegqdi  demoapp.5  demoapp  kunalkushwaha/demoapp_image:v2  Running 16 minutes ago  Running        worker1
5uu9qppsh7ohopc7td1tjs9lg  demoapp.6  demoapp  kunalkushwaha/demoapp_image:v2  Running 15 minutes ago  Running        manager
```

If I try to send request on worker3, still I will be able to get demoapp output and that too with load balancing.

```
$ docker-machine ip worker3
192.168.99.103

$ curl --noproxy 192.168.99.103 192.168.99.103:5000
This is DemoApp v2 @  IP: 10.255.0.13

$ curl --noproxy 192.168.99.103 192.168.99.103:5000
This is DemoApp v2 @  IP: 10.255.0.14

$ curl --noproxy 192.168.99.103 192.168.99.103:5000
This is DemoApp v2 @  IP: 10.0.0.7
```

Isnt’t this cool!

> Docker swarm mode is about making orchestration simple, so anyone without deep understanding of distributed computing, clustering, security should be able to create a robust, scalable and super secure cluster and still focus on his main work.

> Also Docker swarm mode do no expect you to change your workflow and application deployment. It just adapts to you.

If you find docker swarm mode interesting, You should look at [SwarmKit](https://github.com/docker/swarmkit). This project does all magic for docker swarm mode and you can use to build your own distributed application.

Hope this blog will help you to explore docker 1.12 RC.
