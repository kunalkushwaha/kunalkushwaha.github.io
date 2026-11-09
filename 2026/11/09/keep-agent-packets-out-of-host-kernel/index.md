# Keep the Agent's Packets Out of the Host Kernel

> Autonomous agents with shell access have superpowers, and a big attack surface. What this week's Firecracker escape shows about VM boundaries, and how to set one that holds for tomorrow's bug.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-11-09
- Tags: agents, security, sandbox, virtualization, koala, building
- Canonical: https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/

![An AI agent works inside a glass box, its own VM. Its packet travels along a blue path through Koala, a koala at a laptop beside a stack of layers, and out to the globe. A red dashed path from the agent's box toward a large host server with a glowing core is blocked by a red X at the server's door.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/hero.png)

An autonomous agent with a shell has superpowers. It installs packages, runs scripts, opens connections and fixes its own mistakes, without waiting for anyone. Nobody reviews each command. That's the point.

It's also a big attack surface: every command it chooses, every package it pulls, every script a web page talks it into.

> Give an agent a shell, and the boundary is the only reviewer left.

So I run every agent in its own VM. This week, a guest VM got root on its host anyway. The first step was telling the host kernel **where to write a checksum**. The [write-up](https://pwn.ai/blog/firecracker-escape), published on 7 November, targets Firecracker. I read it with one question: could this happen in Koala?

It can't, and the reason is a design choice worth checking in any VM setup.

## First, what a checksum offload is

Every network packet carries a checksum. Computing it costs CPU, so virtio lets the guest hand that job to the host: "the checksum starts here, write it there." That shortcut is an **offload**, and it's on by default in most fast VM networking.

It also means the host kernel reads two numbers the guest wrote.

> Anything the guest writes is input. What matters is where it gets parsed.

## How the escape worked

![Step one, the way in: a guest VM sends packets with checksum offload hints, csum_start and csum_offset. The host kernel's IPv4 code takes those offsets on trust, which lets the guest read host memory 32 bytes at a time and, in the authors' lab, make small writes. Step two, the escape: they rewrite a Firecracker vCPU thread's saved registers, and because the VMM ran as root with no jailer and no seccomp, that means root on the host. It only works when guest packets enter the host kernel's network stack with the hints attached, through TAP, vhost-net or AF_PACKET.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/diagram-escape-chain.svg)

- **The way in.** A bug in the host kernel's IPv4 code trusted those two numbers. With crafted values, the guest could read host memory 32 bytes at a time. The authors also report small writes in their lab.
- **The escape.** They used that access to rewrite a Firecracker vCPU thread's saved registers. That Firecracker ran as root with no jailer and no seccomp, so it meant root on the host.

The kernel fix landed upstream on 14 October 2026. The bug gets patched. The pattern stays.

## Where your guest's packets get parsed

![In the usual fast path, the guest VM sends a frame with offload hints through a TAP device or vhost-net, and the host kernel network stack parses the guest's frame and trusts its hints. In Koala, the guest sends complete, checksummed frames over vhost-user queues handed over by Cloud Hypervisor to koala-net, a process that runs as you with gVisor's netstack in userspace. When policy allows a connection, koala-net opens an ordinary socket and the host kernel builds its own packets. The kernel only sees koala-net's own sockets.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/diagram-packet-path.svg)

The attack needs guest packets to reach the host kernel's network stack with the hints attached. TAP and vhost-net, the usual fast paths, do exactly that. Koala uses neither.

- **Frames stay in userspace.** Each VM's network card hands its frames to `koala-net`, a small process that runs as you, not root. It handles them with [netstack](https://github.com/google/gvisor/tree/master/pkg/tcpip), the network stack from Google's [gVisor](https://gvisor.dev/), used as a library. Only the network stack: the isolation boundary is still the VM.
- **Traffic leaves through ordinary sockets.** When policy allows a connection, `koala-net` opens a normal socket. The host kernel builds those packets itself.
- **The Mac works the same way.** Apple's Virtualization framework also hands frames to `koala-net` in userspace.

The host kernel never parses a frame the guest built.

## Two bits, and a test that keeps it that way

![A strip of virtio feature bits 0 to 39. Only bit 5, the MAC address, and bit 32, VERSION_1, are agreed with the guest. Bit 30 is a vhost-user bookkeeping bit that never reaches the guest. The checksum bits 0 and 1 and the TSO, UFO and ECN offload bits 7 to 14 are refused. Asking for any of 13 offload features, or anything else not offered, makes the device refuse the connection.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/diagram-two-bits.svg)

A virtio device and the guest agree on features at boot. Koala's network device offers only two that reach the guest: a MAC address and modern virtio. No offloads, so the guest can't even send the hints.

You can see it from inside a running VM. Bit 0 comes first:

```
$ koala exec dev -- cat /sys/bus/virtio/devices/virtio5/features
00000100000000000000000000000000100000000000000000000000…
```

![Two tests keep offloads off. A unit test of the device requires the offered features to be exactly MAC, VERSION_1 and the vhost-user bit, written as literal bits, and requires 13 named offload requests to be refused; adding GUEST_TSO4 as a trial failed it with offered 0x1400000a0, want 0x140000020. A check in a real VM on KVM reads the guest's negotiated bits and fails on anything but 5 and 32; it passes 5 of 5 today and fails, naming it, for a guest that reports GUEST_TSO4. Both run on every Linux build, which also found a test that had never run on Linux.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/diagram-pinned.svg)

That was true from day one. But a property nobody tests can disappear the day someone turns offloads on for speed. So now:

- **A unit test** fails if the device offers anything else, or accepts any of 13 offload features. Adding one as a trial broke it straight away: `offered 0x1400000a0, want 0x140000020`.
- **A check in a real VM** reads the bits from inside the guest and fails on anything but 5 and 32. It passes 5/5 today.
- **Both run on every Linux build.** Wiring that up also found a test that had never run on Linux. That's its own post.

## If something else gets in

![A hijacked hypervisor in Koala sits inside a second wall: each VM's Cloud Hypervisor runs as your user, not root, under Landlock, a seccomp allow-list and no_new_privs, in its own namespaces where allowed, and can open only Unix sockets to its own helpers. It can do only what this profile allows. The honest limit: code that can already write host kernel memory does not need the hypervisor, and no userspace sandbox stops it. What protects Koala here is the first wall, the guest has no path into that kernel code. Patch the host kernel anyway, because Docker or any VM tool that uses TAP can still reach the bug.](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/diagram-second-wall.svg)

The escape's second step, hijacking the hypervisor, meets the [second wall](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/). Each VM's Cloud Hypervisor runs as your user, under Landlock, seccomp and no_new_privs, and can't open network sockets.

To be honest about the limit: code that can already write host kernel memory doesn't need the hypervisor, and no userspace sandbox stops it. What protects Koala here is the first wall. The sandbox is for the next bug. A boundary has to hold against tomorrow's bug, not just today's.

And patch your host kernel anyway. Docker, or any VM tool that uses TAP, can still reach this one.

## The cost, honestly

| | TAP with offloads | Koala |
|---|---|---|
| Who parses guest frames | Host kernel | `koala-net`, in userspace |
| Who does checksums | Host kernel, on the guest's word | The guest |
| CPU per byte | Lower | Higher |
| Built for | Moving huge amounts of data | Agents |

For bulk data, that's a real cost. Agents aren't that workload. They call model APIs, fetch pages and clone repositories. For an agent, how far its traffic can reach matters more than how fast it goes.

## What I'd take from this, beyond Koala

- **A shell is a superpower.** An agent that can run anything will eventually try anything. Build the boundary for that.
- **Know where the host reads the agent's data.** Find every place where data from the agent is read by the host. That's where the wall is weakest.
- **Agree to as little as you can.** Every feature the host agrees to is one more thing that can break.
- **Turn properties into tests.** "Offloads are off" used to be just true. Now the build breaks if anyone turns them on.
- **Say what you gave up.** Koala trades some speed for a smaller attack surface. Say that openly, so readers can decide if the trade fits them.

## Check your own setup

1. Where are guest packets parsed? If it's the host kernel through TAP or vhost-net, offload hints are attack surface.
2. Run `cat /sys/bus/virtio/devices/*/features` in the guest. Count the set bits.
3. Is the hypervisor running as a normal user, sandboxed? This escape needed root and no seccomp.
4. Which of these properties does a test enforce?

Koala runs each AI agent, or any Linux workload, in its own VM on your Mac or Linux machine. Everything here is the default. The latest release is 0.1.0-rc.2:

```
# Apple silicon Mac (macOS 26+)
brew install kunalkushwaha/koala/koala

# Ubuntu 24.04 (x86-64, KVM): .deb from the releases page
```

[Releases](https://github.com/kunalkushwaha/homebrew-koala/releases)

Agents get more capable every month. Set the boundary now, for tomorrow's agent and tomorrow's bug.
