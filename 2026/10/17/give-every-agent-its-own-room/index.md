# Give Every Agent Its Own Room

> Running a model locally is the easy part. The agent still needs a shell, a network, and tools it can reach. Koala gives every workload its own Linux VM on Apple Silicon, with access controls that stay on the Mac.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-10-17
- Tags: ai-native, agents, security, sandbox, apple-silicon, building
- Canonical: https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/

![Your Mac Can Run AI Agents. Can It Keep Them in Check? Three robots work in their own glass-walled rooms, connected to a Mac Mini on the desk below.](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/hero.png)

Running a model locally was the easy part. The agent connected to it is a different problem.

Mac Mini and Mac Studio quietly became the favourite hardware for home and small-team LLM labs. The unified memory architecture handles surprisingly large models, the machines run silent all day, and the price is reasonable compared to a GPU workstation. People have Ollama and MLX running on the shelf next to the router, and they are starting to point agents at them. That is where the interesting question appears.

An agent needs a shell, a filesystem, tools it can reach, somewhere to write its work. Give it those without a boundary and you have moved the risk rather than removed it. The model lives on your Mac, but the agent is browsing, writing code, calling APIs, and reading files. All of it through your network, your credentials, and your kernel.

I explored this problem a few months ago on a Linux host with [Kagebox](https://github.com/kunalkushwaha/Kagebox): a spare laptop running one Linux VM, an egress latch, and a credential broker that kept API keys on the host side. The lesson from that experiment was simple: a control the agent can reach is not a control. Koala is the same idea, native on Apple Silicon.

## The boundary is the design

macOS already ships a native virtualization framework. Most lightweight container workflows on Mac still put workloads behind one shared Linux VM. Colima and Docker Desktop both do this. That means one kernel, one network stack, one compromise radius. Koala makes each job, service, or agent its own VM instead. It is written mostly in Swift, so the host-side controls use macOS APIs directly rather than wrapping Linux tooling.

![The Koala manager runs on the Mac. The agent runs in its own guest. Network checks and credential grants stay on the host side, where guest root cannot reach them.](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/diagram-boundary.svg)

Every workload gets its own guest. The access controls stay outside it, on the Mac host. That asymmetry is the whole design.

## Container or VM: where you draw it decides who can erase it

| Tool | What it gives you | Where workloads are separated |
|---|---|---|
| Lima | General Linux VMs; supports Apple's vz backend | VM boundary; often used to host container tools |
| Colima | A convenient Docker/containerd environment built on Lima | Containers share the Linux VM's kernel |
| Docker Desktop | Managed Docker Engine and containers on macOS | Containers share the managed Linux VM's kernel |
| **Koala** | One Linux VM per job, service, or agent | Each workload gets its own kernel, disk, and network policy |

The difference is where the policy enforcement runs. In a container model, the agent runs inside the shared VM, and with root in that VM it can adjust its own firewall. In Koala's model, the network check runs on the Mac host, in a Go process the guest has no access to. A workload can be compromised down to root and still cannot rewrite the rule that limits it.

AWS shipped Strands Box around the same time: an open-source sandbox in developer preview that uses macOS Seatbelt for containment, with policies covering shell, Python, MCP, and network actions. It keeps the agent in the developer's existing environment. Koala takes the VM route. Both are right that the boundary matters. They disagree on where to draw it.

## Lean when small is enough. Full when it isn't.

![Two profiles: Lean runs one OCI image with minimal init. Full boots a Linux distribution with systemd and Docker Engine inside the guest. Both use a verified kernel from the Koala runtime.](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/diagram-profiles.svg)

**Lean** starts one OCI image with a minimal init: a microVM for a focused task. One image, one job, as small as it can be.

**Full** boots a Linux distribution with systemd and Docker Engine inside the guest, for agents that need a familiar server environment, or need to run Docker inside their own VM.

In both cases the image provides the filesystem and Koala supplies the kernel. The guest cannot swap the kernel out.

## The guest sends packets. Koala checks the destination.

Each VM has a dedicated `koala-net` helper, written in Go and built on [gVisor's userspace network stack](https://github.com/containers/gvisor-tap-vsock). Before traffic leaves through a Mac host socket, Koala checks where it is going against that VM's policy.

![Guest packets pass through koala-net and hit host policy. Permitted traffic reaches its destination. Traffic not on the allowlist is dropped on the host, outside the guest.](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/diagram-network.svg)

The check runs on the host. Guest root cannot turn it off. VMs have no shared bridge or peer access by default, so a compromised workload cannot reach its neighbors through the network either.

Koala also brings **policies and agent templates** together. A template can prepare an agent's VM, its setup, its volume mounts, and its access grants, then show you the full plan before anything runs. With the credential broker, an agent can use an approved API capability while the key stays in the Mac's Keychain. The agent sees a proxied endpoint, never the secret.

## Why this matters now

The scarce thing in agent work is not capability. Models are capable. The scarce thing is a safe place to run them. [I argued a few months ago](https://kunalkushwaha.github.io/2026/06/09/safe-deployment-is-the-new-scarcity/) that safe deployment is the new scarcity; agents are where that argument goes next. A sandbox, a credential broker, a network policy enforced from outside the guest: these are the controls we already built for people and for services, pointed at a new kind of worker.

Koala is in preview. It requires Apple Silicon and macOS 26 or newer.

Explore [how it works](https://kunalkushwaha.github.io/homebrew-koala/how-it-works/) or the [project on GitHub](https://github.com/kunalkushwaha/homebrew-koala).

---

The model lives on your Mac. The agent lives in its own room. The controls never cross the line between them.

That asymmetry is the whole design.
