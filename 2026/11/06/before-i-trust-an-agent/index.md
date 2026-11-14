# Before I Trust an Agent, I Try to Break Its Box

> Koala gives every agent its own room. A room is only a claim until something hostile tries the door. This is how I test the boundary from the inside, what broke, and what I have not tested yet.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-11-06
- Updated: 2026-11-15
- Tags: agents, security, sandbox, testing, koala, building
- Canonical: https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/

![Untrusted code runs inside the VM. The host stays outside the guest boundary.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/hero.jpg)

An AI agent reads web pages, messages, and files, and it may act on instructions hidden in any of them. Sooner or later, something untrusted will run wherever the agent runs.

Koala is my answer to where that should be: [one Linux VM per agent](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/), on a Mac or a Linux machine, with the network policy and the API keys kept on the host. That is a design claim. Claims are cheap. So I attack it from the inside.

## An attacker, a bystander, and witnesses

Every test runs on real VMs: Apple's Virtualization framework on Macs, KVM and Cloud Hypervisor on Linux. Each hostile run has three parts.

The attacker is a VM running as root: an agent fully taken over. The bystander is a healthy VM beside it, answering a check every half second. If the attack breaks the neighbor, the test fails, even if nothing escaped. The witnesses live on the host: listeners that record every packet, file hashes taken before and after, and fake secrets ("canaries") planted so any leak has a name.

> Never believe the attacker. Judge every result by what the host saw.

Code inside the VM can print "access denied" while it gets through. A test that reads the guest's own report is testing the guest's honesty, not the boundary.

![Root in the guest attacks the boundary, fuzzers hit every parser of guest data, and crash checks kill the manager, VM processes, and helpers at chosen moments. A canary key never shows up inside the VM.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/diagram-tested-by-attacking.svg)

## What attacking it looks like

| What the attacker tried | What the host saw |
|---|---|
| Connect to 10 host ports, including the ones Koala opens for a command's input and output | 0 of 10 connected |
| Race Koala to those command ports, 80,000 to 86,000 attempts per port | Won 0 times |
| Read a victim VM's disk, or attach its volume | Only its own disks visible; attach refused |
| Plant a symlink in a shared folder pointing at a host credential, then wait for `koala cp` | Symlink refused, credential untouched. A mid-copy swap was never hit, so that case is untested |
| Spoofed, fragmented, and IPv6 packets; posing as the router; DNS rebinding | Only the two allowed listeners received anything |
| 2,000-query DNS flood | 486 answered, the rest refused at once (16 lookups at a time); the helper survived |

The symlink row is the one I keep thinking about. It does not attack Koala's code. It attacks Koala's helpfulness: wait for the host to reach into the guest, and hand it something poisoned. Boundaries rarely fail where the guest reaches out. They fail where the host reaches in.

Around the attacks, more than 40 fuzz targets hit every parser of guest data, and about 1,400 crash checks per platform kill Koala's processes at chosen moments.

## Break it, and leave it running

During the flood, the test also kills Koala's network helper, and kills its manager outright with SIGKILL. Memory peaked at 325 MiB and open files did not leak. The bystander kept answering: 97% of its checks succeeded. Not 100%: in the Linux run, every miss fell within two seconds of the manager kill. It also found two real bugs: after a manager kill, VMs lost their network, and one process was orphaned. Both failed safe, and both are fixed.

Then Koala runs for eight hours with faults injected along the way. On a Mac, all 9,884 operations succeeded. On Linux, one VM stopped taking commands after a restart; that bug is now fixed. Long runs also found that one command took 6.1 seconds once Koala had 67,000 operations of history. It now takes 0.35.

## Assume the box already failed

What if the attacker gets out anyway, through a hypervisor bug? On Linux, every Koala process that touches a VM runs in its own locked-down sandbox. So an escape probe runs inside each of them, as if the attacker were already there. It tries to read an SSH key and other VMs' disks, open network connections, start a shell, and freeze Koala's manager.

Every attempt was refused, in every sandbox. The same probe, run without the sandbox, got through 17 times. That control is what gives the refusals meaning.

This is not hypothetical: the recent [Firecracker escape](https://pwn.ai/blog/firecracker-escape) went through the host kernel, then took over a hypervisor running as root with no seccomp ([how Koala avoids the first step](https://kunalkushwaha.github.io/2026/11/09/keep-agent-packets-out-of-host-kernel/)).

## The bug was four bytes

That probe also caught a real bug: after one refused connection, the sandbox refused every later legitimate one too.

Koala's Linux sandbox intercepts connections with seccomp, and reads the target address with `process_vm_readv`, called through C's variadic `syscall()`. Its last argument, a 64-bit flags value, was passed as the `int` 0. Only 4 of its 8 bytes were written; the rest was leftover stack. When those bytes were not zero, the read failed, and the connection was refused. The unit test passed even before the fix, because the leftover bytes depend on the caller. A small C program, run under the real sandbox, now catches it.

It failed closed, which is the right direction. It was still a bug, and now it has a regression test.

## What I have not tested yet

A VM can always lie about its own commands. It cannot lie about anything else, which is why nothing on the host trusts a guest's report about itself.

The detailed attack runs were recorded in October 2026 on a Mac; the same suites run unchanged on Linux, and pass. [Domain grants](https://kunalkushwaha.github.io/homebrew-koala/guide/networking/), new in the latest release, have not been through the root-in-the-guest suite yet. There has been no independent security audit, and no testing beyond macOS 26 and Ubuntu 24.04. Koala is a release candidate; its source goes public with 0.1.0.

## Questions for any sandbox

1. How was the boundary attacked?
2. Who judged the result: the guest, or the host?
3. Did the tests run against the real system, or a mock of it?
4. What happened when something crashed halfway through?
5. What has the team not tested yet?

Ask them of any vendor that says its agent sandbox is isolated.

Read [how Koala is tested](https://kunalkushwaha.github.io/homebrew-koala/#hostile-guests-crashes-and-fuzzing-on-real-vms), or get the [releases and user guide](https://github.com/kunalkushwaha/homebrew-koala).

I trust the box after it has been attacked, not before.
