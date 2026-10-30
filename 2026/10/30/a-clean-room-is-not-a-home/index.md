# A Clean Room Is Not a Home

> A coding sandbox protects a task. A sandbox for a long-running agent has to protect a tenant. They look alike from the outside, but they need different guarantees.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-10-30
- Tags: agents, security, sandbox, ai-gateway, koala, building
- Canonical: https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/

When I hear "sandbox" today, the picture is usually a coding agent: a container, a repo, run the tests, throw it away. That works. But an agent that answers email or runs a workflow for months is not a coding agent with a longer timeout. It lives somewhere.

> A coding sandbox protects a task. An agent sandbox has to protect a tenant.

## Same box, different life

![A coding sandbox is created, runs one task, and is thrown away after a human reviews the diff. A long-running agent stays, reads untrusted input all day, keeps state, and reaches the outside world through a gateway.](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/diagram-two-sandboxes.svg)

The box looks the same. What changes is time, who is watching, and what the agent gets to keep. Disposability and review do a lot of security work for free. A long-running agent loses both.

| | Coding sandbox | Long-running agent |
|---|---|---|
| Lifetime | Minutes | Weeks, months, or years |
| Oversight | A human reviews the diff | Unattended by design |
| Inputs | A repo and a prompt | Untrusted content, all day |
| Credentials | Short-lived | Durable access to SaaS, APIs, money |
| State | Thrown away | Persists |
| Failure | Loud | Quiet |
| Recovery | Start again | Survive crashes and upgrades mid-task |

## Persistence is the new attack surface

![In a coding sandbox a poisoned input dies with the task. In a long-running agent it is written to memory, read back days later, and acted on after the input itself is gone.](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/diagram-bad-input.svg)

A bad input in a coding sandbox dies with the container. In a long-running agent, it can be written to memory and acted on days later.

## The gateway is where an agent is governed

A coding agent can live with a token that expires before lunch. A long-running agent needs durable access, so the keys cannot live inside the box. Last week, in [The Clock Has Changed](https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/), I argued that every agent needs a gateway with guardrails and policy. For a long-running agent, that gateway is where it is governed.

## Quiet failures need loud instruments

![A drifting agent raises no errors while spend and requests creep up. A dashboard only reports it; a budget enforced outside the box stops it.](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/diagram-quiet-failure.svg)

A control the agent can reach is not a control. The box also has to outlive the process: crashes, restarts, and upgrades mid-task. For Koala, the crash and upgrade tests matter as much as the escape tests. More on how I test it in the next post.

## Where Koala fits

![Koala covers strict test runs in throwaway VMs, long-running dev boxes for developers on Linux servers, and autonomous agents, all on the same boundary: one VM per workload, keys on the host, every connection checked on the host, and recovery from crashes and upgrades.](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/diagram-koala-fit.svg)

[Koala](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/) was built for the long-running column, without giving up the short one.

- **Strict tests:** `koala run` boots a VM, runs the command, and deletes it, pass or fail. About five seconds.
- **Dev boxes:** one long-running VM per developer on a shared Linux server. Volumes outlive the VM.
- **Agents:** keys never enter the VM. The broker allows only what a policy lists, limits rate per VM, and keeps an audit record.
- **Off switch:** `koala policy disable` cuts an agent's access at once.

Underneath all of it, Koala uses two walls, not one.

![A typical coding sandbox puts one wall, a process sandbox or container, around the agent on the shared host kernel. Koala puts the agent in a VM with its own kernel, and the host processes that run the VM are sandboxed too.](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/diagram-two-walls.svg)

**Not yet:** spend caps, request-content logging, snapshots.

Disposability is the right default for a coding task. Koala starts from distrust instead: the code inside may not be friendly, whether it stays for one test run or for years.

It is a free, open-source preview:

```
# Apple silicon Mac (macOS 26+)
brew install kunalkushwaha/koala/koala

# Ubuntu 24.04 (x86-64, KVM): .deb from the releases page
```

[Releases](https://github.com/kunalkushwaha/homebrew-koala/releases)

## Ask any sandbox

1. Is it thrown away, or does it keep state?
2. Who reviews what it does?
3. Where do its credentials live, and who can revoke them?
4. What stops it when it fails quietly?
5. What happens when the host restarts?

A clean room answers the first two. A home has to answer all five.

Build the room for who is going to live in it.
