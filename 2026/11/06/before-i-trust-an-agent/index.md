# Before I Trust an Agent, I Try to Break Its Box

> Koala gives every agent its own room. A room is only a claim until something hostile tries the door. This is how I test the boundary from the inside, what broke, and what I have not tested yet.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-11-06
- Updated: 2026-11-15
- Tags: agents, security, sandbox, testing, koala, building
- Canonical: https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/

The attack results below were recorded in October 2026 on Koala's preview builds. This update adds [the eight-hour runs](#leave-it-running-for-eight-hours) and domain grants from rc.3, and marks the bugs fixed since.
![Untrusted code runs inside the VM. The host stays outside the guest boundary.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/hero.jpg)

An AI agent is not a predictable app. It reads web pages, messages, and files, and it may act on instructions hidden in any of them. The safer assumption is that, sooner or later, something untrusted will run wherever the agent runs.

Koala is my answer to where that should be: [one Linux VM per agent](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/), on a Mac or a Linux machine, with the network check and the credentials kept on the host. That is a design claim. Claims are cheap.

So I attack it from the inside. If the code inside the VM is hostile, what can it reach, and what breaks when something goes wrong? This is how I test that, what broke, and what I have not tested yet.

## Tested by attacking it

Every test runs on real virtual machines, on both platforms Koala supports: Apple silicon Macs, through Apple's Virtualization framework, and x86-64 Linux, through KVM and Cloud Hypervisor. After each run, the host and a neighboring control VM are checked, not just the VM under attack.

A few parts of Koala come up below. On the host, a manager runs every VM, each through its own worker process. A network helper sits between each VM and the network, and a broker holds API keys so the VM never sees them. A lean VM runs a single container image; a full VM runs a Linux distribution with Docker.

![Root in the guest attacks the boundary, fuzzers hit every parser of guest data, and crash checks kill the manager, VM processes, and helpers at chosen moments. A canary key never shows up inside the VM.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/diagram-tested-by-attacking.svg)

Three numbers set the scale: more than 40 fuzz targets, at least 100,000 generated inputs for each, and about 1,400 crash checks on each platform. The numbers matter less than where they point. Every one of them aims at the boundary.

## An attacker, a bystander, and witnesses

Each hostile-guest run puts three things together.

The attacker is a VM running as root, the most powerful user inside it. That is the worst case for an agent: fully taken over. The question is whether it can get out.

The bystander is a healthy VM running beside it, answering a check every half second for the whole run. If the attack slows or breaks the neighbor, the test fails, even if nothing escaped.

The witnesses live on the host, where the attacker cannot reach them. Listeners record every packet that arrives. Files are hashed before and after. Fake secrets ("canaries") are planted on the host, so any leak has a name.

> Never believe the attacker. Judge every result by what the host saw.

This is where sandbox tests most often go wrong. Code inside the VM can print "access denied" while it gets through. A test that reads the guest's own report is testing the guest's honesty, not the boundary.

The attacks are small programs copied in with `koala cp`, run against the real, installed, signed Koala. Not a special test build.

## What attacking it looks like

| What the attacker tried | What the host saw |
|---|---|
| Connect to 10 host ports, including the ones Koala opens for a command's input and output | 0 of 10 connected |
| Race to connect to those command ports before Koala does, 80,000 to 86,000 attempts per port | Won 0 times |
| Flood the console to fill the disk | The log stopped at exactly 4 MiB, as designed |
| Rewrite Koala's shared, read-only system files | Refused; hashes identical afterwards |
| Read a victim VM's disk, or attach its volume | Only its own four disks visible, and no trace of a marker planted in the victim's data; attach refused at creation |
| Plant a symlink in a shared folder pointing at a host credential, then wait for `koala cp` | Symlink refused, credential untouched. Copies made while the file was being swapped returned only safe content, though none happened to land mid-swap |
| Spoofed, fragmented, oversized, and IPv6 packets; claiming to be the router; DNS rebinding | Only the two allowed listeners received anything, and only expected test messages |
| 2,000-query DNS flood | Capped at 486 answers; the network helper survived |

The symlink row is the one I keep thinking about. It does not attack Koala's code. It attacks Koala's helpfulness: wait for the host to reach into the guest, and hand it something poisoned.

Boundaries rarely fail where the guest reaches out. They fail where the host reaches in.

## Break things while it is under attack

Next, Koala itself is stressed while the attacker floods every channel at once. During the flood the test cancels a command, stops a VM, kills the network helper, and kills the manager outright with SIGKILL before restarting it. A sampler records memory, open files, and disk use every second.

Resources stayed bounded. Memory peaked at 325 MiB, open files went from 30 to 18, and logs stayed within quota. When its network helper died, the attacker VM was marked degraded instead of quietly carrying on. The bystander kept answering: 97% of its checks succeeded across the run. Not 100%, and that gap is worth knowing too.

The round also found two real problems. Killing the manager took down every VM's network until the VM was restarted, and one VM's process was left orphaned. Neither was a security hole, and both failed safe. Both are fixed: a restarted manager now rebuilds each running VM's network while the VM keeps running, and it reaps a worker it can no longer use.

## Leave it running for eight hours

An attack run lasts minutes. An agent's VM runs for days. So Koala also runs for eight hours at a time, with faults injected along the way.

On a Mac: 32 lean and 8 full VMs, then eight hours of services, jobs, and environments. All 9,884 operations succeeded. The Mac slept and woke on a schedule, and the run survived a manager kill, the loss of a network helper and of a VM's worker, and a runtime update and rollback.

On Linux, with KVM, the run passed every check but four, and all four were one VM that stopped taking commands after a restart. That one took longer to pin down: a race in how Koala reads a VM's event journal, which a later check reproduced in about 300 fast commands. It is fixed in rc.3, and the fixed build ran 2,000 commands without it. Neither eight-hour run has been repeated since the last fixes.

Long runs also found bugs a short test never runs long enough to hit. Commands slowed down as Koala's history grew: with 32 VMs and 67,000 recorded operations, one command took 6.1 seconds to run. It now takes 0.35. After about 3,400 commands in one VM, every later command hung, because the VM's event journal had filled up. And a Mac that slept with its lid closed could miss the wake, leaving VMs that no longer took commands. All of these are fixed in rc.3, and none of them is a security bug.

## Assume the box already failed

The Linux suite asks a harder question. What if the attacker gets out anyway, through a hypervisor bug for example?

On Linux, every Koala process that touches a VM runs in its own locked-down sandbox: the hypervisor, the shared-folder server, the network helper, and the VM's worker. So an escape probe runs inside each of those four, as if the attacker were already there. It tries to read the home folder and a planted SSH key, read Koala's secret store or another VM's disk, talk to or freeze the manager, open network connections, become root or start a shell, and create devices or setuid-root files in the shared folder.

Every attempt was refused, in all four sandboxes: 19, 19, 22, and 19 attempts. The same probe, run without the sandbox, got through on 17 of those attempts. That control is what gives the refusals meaning. They came from Koala's sandbox, not from a machine that simply lacked those things.

Defense in depth is easy to draw on a slide. This is what it looks like as a test.

## The awkward paths

Normal success paths are the easy part. The bugs live where nobody looks on a good day.

| Check | What it does | What passing means |
|---|---|---|
| Fuzzing | 40+ targets in Swift, Go, and C hit every parser of guest data, each with at least 100,000 generated inputs and at least five minutes of run time. Those are minimums per target, not long campaigns. Swift and C targets run under AddressSanitizer. | No crashes, hangs, or memory errors |
| Crash recovery | About 1,400 checks per platform kill the manager, each VM's process, and every helper (image extractor, file transfer, network helper, broker) at chosen moments | No lost or duplicated VMs, no orphaned processes, no damaged state, and the VM starts again |
| Secret handling | A canary key stays on the host and is used through the broker. The guest disk, volumes, and image are searched for it. | The key appears only in the approved upstream request, outside the VM |
| Release testing | Clean machines, upgrades from the published release, and removal | What users download works, not just what passed on a developer's machine |

## A test suite that never fails is testing the wrong things

These checks have found real bugs. Fuzzing found deeply nested input that could crash the manager. Crash testing exposed recovery cases where a VM could be left running after its manager stopped. The Linux escape probe caught something subtler: after one refused connection, the sandbox wrongly refused every later legitimate one.

The cause was four bytes. Koala's Linux sandbox intercepts a confined process's connections with seccomp, and reads the address it is connecting to with `process_vm_readv`, called through C's variadic `syscall()`. Its last argument, a flags value the kernel reads as 64 bits, was passed as the `int` 0. Only 4 of its 8 bytes were written; the other 4 were whatever was left on the stack. When those bytes were not zero, the read failed, the fallback was blocked by the supervisor's own Landlock rules, and the connection was refused. The unit test passed before the fix too, because the leftover bytes depend on the caller. The regression check that catches it is a small C program run under the real sandbox.

That bug failed closed: it blocked more, not less. That is the right direction, but it would have broken real features, and it is still a bug.

> Finding a bug is only useful when the failure becomes a regression test.

Each fix now has a test that fails if the problem comes back.

## What the numbers do not say

These numbers describe specific checks. They are not a promise that every failure is impossible.

One limit is by design. A VM can fake the result of its own command, and claim a failed command succeeded. The test confirmed it. A VM can always lie about itself; it cannot lie about anything else. That is recorded as a known limit, and it is why nothing on the host should trust a guest's report about itself.

The detailed hostile-guest runs above were recorded on an Apple silicon Mac with macOS 26. The same network and hostile-guest suites run unchanged on Linux, and pass.

One new feature has not been through all of this yet. rc.3 adds domain grants: a VM with the internet off can reach a few named hosts, `koala create … --no-internet --domain pypi.org`, and nothing else. A new way out is a new thing to attack. The network helper checks both the TLS server name and the address that name resolved to, so a VM cannot present a granted name to an ungranted server, or point a granted name at the local network. Plain HTTP to a granted name is refused, and so is a grant that would cover a public suffix like `*.co.uk` or `*.github.io`.

On both platforms, real VMs tried an ungranted name at a granted address and were refused, and the parser was fuzzed. What has not happened yet is running domain grants through the root-in-the-guest suite above. Until it has, treat a grant as a narrower door, not a tested one. The [networking guide](https://kunalkushwaha.github.io/homebrew-koala/guide/networking/) says what a grant does and does not stop.

Koala is still a release candidate, and its source goes public with the 0.1.0 release. I have not yet had an independent security audit, or tested across other Linux distributions and hardware. Those are the next gaps, and saying so is part of the evidence.

## The questions to ask about any sandbox's tests

In [A Clean Room Is Not a Home](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/), I asked what any sandbox should answer. These five are about how it was tested, because "it is isolated" is not enough:

1. How was the boundary attacked?
2. Who judged the result: the guest, or the host?
3. Did the tests run against the real system, or a mock of it?
4. What happened when something crashed halfway through?
5. What has the team not tested yet?

Koala runs on Apple silicon Macs with macOS 26 or newer, and on x86-64 Ubuntu 24.04 with KVM. Read [how Koala is tested](https://kunalkushwaha.github.io/homebrew-koala/#hostile-guests-crashes-and-fuzzing-on-real-vms), or get the [releases and user guide](https://github.com/kunalkushwaha/homebrew-koala) on GitHub.

I trust the box after it has been attacked, not before.
