# Understanding Infrakit

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2016-10-14
- Tags: Docker, Infrastructure
- Canonical: https://kunalkushwaha.github.io/2016/10/14/understanding-infrakit/

Docker recently open sourced new project [Infrakit](https://github.com/docker/infrakit) at ContainerCon 2016, Berlin.

As project describes itself as

> “A toolkit for creating and managing declarative, self-healing infrastructure”

Lets try to understand what it does and how it adds value to existing tools.

### Infrakit

##### Problem

Managing docker on different infrastructure is difficult and not portable.

#### History

- Initially Docker machine project was focused for building docker environment on various platforms, but there were lot of issues mainly due to unreliable third party drivers.

  - User experience was not great on developer platforms like Mac (VirtualBox issues)
- Idea of declarative infrastructure was thought in Docker Machine Project also.
- Few [Proposals](https://github.com/docker/machine/issues/773) and [PR were made for declarative infrastructure](https://github.com/docker/machine/pull/2422) but never got merged due to user experience issues and Docker Inc’s long term Roadmap vision.

  *NOTE:Docker Machine is still very much active project with great userbase.*
- Later during [Dockercon16](http://2016.dockercon.com/), while interacting with docker developers & maintainer of Docker Machine, I understood, long term plan is to merge docker machine capabilities into docker engine.
- Meanwhile Docker had already released [Docker For Mac](https://docs.docker.com/engine/installation/mac/). It is amazing product, as it just works and upgrade itself for new releases too.
- Docker for AWS/Azure were announced in [Dockercon16](http://2016.dockercon.com/). Infrakit is born out of these projects.

#### Overview

Unlike most of docker projects, Infrakit is not about containers, it is about preparing & managing Infrastructure for distributed computing.
Infrakit by design is build of active process(a.k.a. plugins) which collaborate with each other to analyze and take action to bring infrastructure in desired state. As other plugins in Docker Projects, Infrakit plugins also communicate over unix sockets and use HTTP protocol. This makes plugin implementation language agnostic and can be deployed separately as containers.

These Plugins are also building block of Infrakit to manage infrastructure.

Currently there are three kind of plugins supported by Infrakit which represents different layer of abstraction representing whole stack of infrastructure.
Lets go through each and try to understand their roles.

#### Groups

Groups plugins are for managing cluster of machine. These clusters can be represented by two set of machines.

- Cattle or Replica : Machines with identical configuration.
- Pets or Quorum : Machine with slightly different properties like Ordering and Identity.

The size of group can be increased and decreased. While changing configuration of machines, rolling updates by swarmkit ensures, no downtime and smooth up-gradation.
Following actions can be done on group plugin.

- watch/ unwatch a group (start / stop managing a group)
- inspect a group
- trigger an update the configuration of a group - like changing its size or underlying properties of instances.
- stop an update
- destroy a group

#### Instance

Instance plugin represent, one instance of machine can be Cloud instance or just a VM. These Cloud instance or VMs have same specs.
Instances define spec of VM or Cloud instance and also we can associate some tags to identify etc.

#### Flavors

Flavor plugin give flavor to each instance. i.e. What exactly should one instance should contain and running. e.g. setting up software packages etc.

With these plugins all kind of infrastructure from machine instance point of view can be configured.

I won’t go into implementation details in this blog, but if you want to play with Infrakit, they have very good introductory [tutorial](https://github.com/docker/infrakit/blob/master/docs/tutorial.md). You must try.

### How Infrakit is different from existing products and solutions available?

##### Configuration Management softwares(Puppet/Chef etc).

- All configuration management software works, once machine are up and running. They do not have capability to create and destroy the computing instances.
- Infrakit’s itself don’t have any defined syntax for configuring such as Puppet/Chef. So Puppet/Chef could to be used in Infakit’s flavor plugins, where computing node is configured.

##### Terraform/CloudFormation/Heat.

- Infrakit is similar to these, in terms of it also works on configuration plus state reconciliation feature i.e. Infrakit not only builds the infrastructure but also monitors them for changes and restore to expected state. i.e. If some instances in cluster stops, it will create new instances.
- Terraform/CloudFormation/Heat can be used as Instance Plugins. Example of [Terraform plugin](https://github.com/docker/infrakit/tree/master/example/instance/terraform) is provided by Infrakit
- Also, Infrakit is not polished product like Terraform, it is a framework which can be used by any existing/new product, to manage different infrastructure is similar fashion.
  It abstracts the details of different infrastructure’s to end user.

#### Final Notes

Infrakit is amazing piece of code & concept for build self healing infrastructure, like Swarmkit to building distributed softwares. Though it is very early in development, so community contribution in ideas and code may bring more changes(in good ways) in Project.

Issues like preparing docker platform with third party services/framework enabled for Storage/Networking/Security plugins should be far more smoother and I am looking forward to work on same.
