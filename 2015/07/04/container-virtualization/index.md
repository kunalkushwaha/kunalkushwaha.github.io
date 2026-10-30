# Container Virtualization and its building blocks

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2015-07-04
- Tags: Containers, Virtualization, Docker
- Canonical: https://kunalkushwaha.github.io/2015/07/04/container-virtualization/

Since 2014, Linux containers have become buzz word in Cloud Infrastructure. Almost all, from Big corporations to startups, all have started using it. Huge credit goes to [Docker](https://kunalkushwaha.github.io/2015/07/04/container-virtualization/www.docker.com) for making using containers so easy to use.

Linux Containers are there in Linux systems for alomst decade old, But making them work, was not so easy, and generally required linux admin experts for doing same. Few Solution as linux containers like FreeBSD Jails, LXC, openVZ, Solaris Zones etc exists for quite some time.

These are also known as OS level Virtualization.
To understand other type of virtualization please read [Layman guide to Platform virtualization](https://kunalkushwaha.github.io/2015/07/04/container-virtualization/../../../../post/Layman-guide-to-Platform-Virtualization)

## Operating System level Virtualization

Quoting below from Wikipedia, I it explains beautifully in technical and yet not too complex.

> OS level Virtualization is a server virtualization method where the kernel of an operating system allows for multiple isolated user space instances, instead of just one. Such instances (often called containers, virtualization engines (VE), virtual private servers (VPS), or jails) may look and feel like a real server from the point of view of its owners and users.

In simple words, it allows to run multiple rootfs (user-space) simultaneously and all running rootfs have their own view of filesystem and devices. So they are not aware of each others and resource usage can be configured.

Sounds similar to virtual Machines? Yes it is!

#### How this isolation is achieved?

This isolation is achieved using linux features like namespaces, cgroups and chroot. To understand details we need to first understand each of them.

#### Namespaces

Namespace wraps a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.

Currently, Linux implements six different types of namespaces

1. ##### Mount namespaces (CLONE\_NEWNS)

This isolate the set of filesystem mount points seen by a group of processes. Thus, processes in different mount namespaces can have different views of the filesystem hierarchy. With the addition of mount namespaces, the mount() and umount() system calls ceased operating on a global set of mount points visible to all processes on the system and instead performed operations that affected just the mount namespace associated with the calling process.

- This is also an alternative to chroot system call.
- This is supported since Linux 2.4.19

2. ##### UTS namespaces (CLONE\_NEWUTS)

This isolate two system identifiers—nodename and domainname—returned by the uname() system call; the names are set using the sethostname() and setdomainname() system calls.

- In the context of containers, the UTS namespaces feature allows each container to have its own hostname and NIS domain name. This can be useful for initialization and configuration scripts that tailor their actions based on these names.
- This is supported in Linux kernel since Linux 2.6.19.

3. ##### IPC namespaces (CLONE\_NEWIPC)

This isolate certain interprocess communication (IPC) resources, namely, System V IPC objects and (since Linux 2.6.30) POSIX message queues.

- The common characteristic of these IPC mechanisms is that IPC objects are identified by mechanisms other than filesystem pathnames. Each IPC namespace has its own set of System V IPC identifiers and its own POSIX message queue filesystem.
- This is supported in Linux kernel since Linux 2.6.19

4. ##### PID namespaces (CLONE\_NEWPID)

   This isolate the process ID number space. In other words, processes in different PID namespaces can have the same PID.

- This helps migrating containers between hosts while keeping the same process IDs for the processes inside the container.
- PID namespaces also allow each container to have its own init (PID 1), the “ancestor of all processes” that manages various system initialization tasks and reaps orphaned child processes when they terminate.
- This is supported since Linux 2.6.24

5. ##### Network namespaces (CLONE\_NEWNET)

This provide isolation of the system resources associated with networking. Thus, each network namespace has its own network devices, IP addresses, IP routing tables, /proc/net directory, port numbers, and so on.

- Network namespaces make containers useful from a networking perspective: each container can have its own (virtual) network device and its own applications that bind to the per-namespace port number space.
- started in Linux 2.4.19 2.6.24 and largely completed by about Linux 2.6.29

6. ##### User namespaces (CLONE\_NEWUSER)

This isolate the user and group ID number spaces. In other words, a process’s user and group IDs can be different inside and outside a user namespace.

- The most interesting case here is that a process can have a normal unprivileged user ID outside a user namespace while at the same time having a user ID of 0 inside the namespace. This means that the process has full root privileges for operations inside the user namespace, but is unprivileged for operations outside the namespace.
- This was partially supported since Linux 2.6.23 and completed in Linux 3.8.

#### Control groups a.k.a. cgroups

Cgroups allow you to allocate resources—such as CPU time, system memory, network bandwidth, or combinations of these resources—among user-defined groups of tasks (processes) running on a system.

- One can configure cgroups, deny cgroups access to certain resources, and even reconfigure cgroups dynamically on a running system.
- The cgconfig (control group config) service can be configured to start up at boot time and reestablish your predefined cgroups, thus making them persistent across reboots.
- By using cgroups, we gain fine-grained control over allocating, prioritizing, denying, managing, and monitoring system resources. Hardware resources can be appropriately divided up among tasks and users, increasing overall efficiency.
- These are like process, hierarchical in nature i.e. child cgroups inherit certain attributes from their parent cgroup.

Follwing resources are supported currently in cgroups.

- blkio — this subsystem sets limits on input/output access to and from block devices such as physical drives (disk, solid state, USB, etc.).
- cpu — this subsystem uses the scheduler to provide cgroup tasks access to the CPU.
- cpuacct — this subsystem generates automatic reports on CPU resources used by tasks in a cgroup.
- cpuset — this subsystem assigns individual CPUs (on a multicore system) and memory nodes to tasks in a cgroup.
- devices — this subsystem allows or denies access to devices by tasks in a cgroup.
- freezer — this subsystem suspends or resumes tasks in a cgroup.
- memory — this subsystem sets limits on memory use by tasks in a cgroup, and generates automatic reports on memory resources used by those tasks.
- net\_cls — this subsystem tags network packets with a class identifier (classid) that allows the Linux traffic controller (tc) to identify packets originating from a particular cgroup task.
- net\_prio — this subsystem provides a way to dynamically set the priority of network traffic per network interface.
- ns — the namespace subsystem.

For details you may refer : <https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt>

## So how these helps in containers?

By now, you must have understood, namespaces and cgroups help to create isolated environment.

- Namespaces provides isolation of filesystem view, devices, network and processes.
- Cgroups helps to allocate, devices accessibility, and allocate quota to use the devices.

## Is this all sufficient for Virtualization?

No, still security is left. To create secure containers features like Capablities, secomp , SELinux and Apparmor are used for that. These all are integrated with new container solutions like Docker, CoreOS rocket etc.

Few Container projects worth wating are

- LXC - Linux containers : This is default container hypervisior on all linux based systems. <https://linuxcontainers.org/>
- Docker libcontainer:
  This is now donated by Docker to Linux foundation and new name will is OpenContainers. <http://www.opencontainers.org/>
- CoreOS - Rocket - <https://github.com/coreos/rkt>
- Redhat’s systemd-nspawn. <http://www.freedesktop.org/software/systemd/man/systemd-nspawn.html>

I hope this blog would have help you to understand Linux Containers.

I will be writing more on current status of linux containers projects in my next blog so stay tuned :)
