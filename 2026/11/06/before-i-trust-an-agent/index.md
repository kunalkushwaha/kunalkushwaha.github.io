# Before I Trust an Agent, I Try to Break Its Box

> Koala gives every agent its own room. A room is only a claim until something hostile tries the door. This is how I test the boundary from the inside, what broke, and what I have not tested yet.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-11-06
- Tags: agents, security, sandbox, testing, koala, building
- Canonical: https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/

![Untrusted code runs inside the VM. The host stays outside the guest boundary.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/hero.jpg)

In October I wrote that Koala should [give every agent its own room](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/): one Linux VM per workload, with the network check and the credentials kept on the host. That is a design claim. Claims are cheap.

The question that matters is simpler. If the code inside that VM is hostile, what can it reach, and what breaks when something goes wrong?

I do not treat an agent as a predictable app. It reads web pages, messages, and files, and it may act on instructions hidden in any of them. The safer assumption is that, sooner or later, something untrusted will run inside its environment. So the boundary has to be tested from the inside, by something trying to get out.

## Tested by attacking it

Every test runs on real virtual machines, on both supported platforms. After each run, the host and a neighboring control VM are checked, not just the VM under attack.

![Root in the guest attacks the boundary, fuzzers hit every parser of guest data, and crash checks kill the manager, VM processes, and helpers at chosen moments. A canary key never shows up inside the VM.](https://kunalkushwaha.github.io/2026/11/06/before-i-trust-an-agent/diagram-tested-by-attacking.svg)

Three numbers set the scale: more than 40 fuzz targets, at least 100,000 generated inputs for each, and about 1,400 crash checks on each platform. The numbers matter less than where they point. Every one of them aims at the boundary.

## An attacker, a bystander, and witnesses

Each hostile-guest run puts three things together.

The attacker is a VM running as root, the most powerful user inside it. That is the worst case for an agent: fully taken over. The question is whether it can get out.

The bystander is a healthy VM running beside it, answering a check every half second for the whole run. If the attack slows or breaks the neighbor, the test fails, even if nothing escaped.

The witnesses live on the host, where the attacker cannot reach them. Listeners record every packet that arrives. Files are hashed before and after. Fake secrets, canaries, are planted on the host, so any leak has a name.

> Never believe the attacker. Judge every result by what the host saw.

This is where sandbox tests most often go wrong. Code inside the VM can print "access denied" while it gets through. A test that reads the guest's own report is testing the guest's honesty, not the boundary.

The attacks are small programs copied in with `koala cp`, run against the real, installed, signed Koala. Not a special test build.

## What attacking it looks like

| What the attacker tried | What the host saw |
|---|---|
| Knock on every host port the VM could reach | 0 of 10 answered |
| Race Koala for its private reply channels, 80,000 to 86,000 attempts per channel | Won 0 times |
| Flood the console to fill the disk | The log stopped at exactly 4 MiB, as designed |
| Rewrite Koala's shared, read-only system files | Refused; hashes identical afterwards |
| Read a victim VM's disk, or attach its volume | Only its own four disks visible, no victim marker in any byte; attach refused at creation |
| Plant a symlink in a shared folder pointing at a host credential, then wait for `koala cp` | Symlink refused, credential untouched. Swapping the file mid-copy returned only safe content |
| Spoofed, fragmented, oversized, and IPv6 packets; claiming to be the router; DNS rebinding | Only the two allowed listeners received anything, and only expected test messages |
| 2,000-query DNS flood | Capped at 486 answers; the network helper survived |

The symlink row is the one I keep thinking about. It does not attack Koala's code. It attacks Koala's helpfulness: wait for the host to reach into the guest, and hand it something poisoned.

Boundaries rarely fail where the guest reaches out. They fail where the host reaches in.

## Break things while it is under attack

Next, Koala itself is stressed while the attacker floods every channel at once. During the flood the test cancels a command, stops a VM, kills the network helper, and kills the manager outright with SIGKILL before restarting it. A sampler records memory, open files, and disk use every second.

Resources stayed bounded. Memory peaked at 325 MiB, open files went from 30 to 18, and logs stayed within quota. When its network helper died, the attacker VM was marked degraded instead of quietly carrying on. The bystander kept answering: 97% of its checks succeeded across the run. Not 100%, and that gap is worth knowing too.

The round also found two real problems. Killing the manager took down every VM's network until restart, and one VM's process was left orphaned. Neither was a security hole. Both failed safe, and both are recorded for fixing.

## Assume the box already failed

The Linux suite asks a harder question. What if the attacker gets out anyway, through a hypervisor bug for example?

On Linux, every Koala process that touches a VM runs in its own locked-down sandbox: the hypervisor, the shared-folder server, the network helper, and the VM's worker. So an escape probe runs inside each of those four, as if the attacker were already there. It tries to read the home folder and a planted SSH key, read Koala's secret store or another VM's disk, talk to or freeze the manager, open network connections, become root or start a shell, and create devices or set-root files in the shared folder.

Every attempt was refused, in all four sandboxes: 19, 19, 22, and 19 attempts. The same probe run without the sandbox succeeded at 17 of them. That control is what gives the refusals meaning. They came from Koala's sandbox, not from a machine that simply lacked those things.

Defense in depth is easy to draw on a slide. This is what it looks like as a test.

## The awkward paths

Normal success paths are the easy part. The bugs live where nobody looks on a good day.

| Check | What it does | What passing means |
|---|---|---|
| Fuzzing | 40+ targets in Swift, Go, and C hit every parser of guest data, each with at least 100,000 generated inputs and at least five minutes of run time. Swift and C targets run under AddressSanitizer. | No crashes, hangs, or memory errors |
| Crash recovery | About 1,400 checks per platform kill the manager, each VM's process, and every helper (image extractor, file transfer, network helper, broker) at chosen moments | No lost or duplicated VMs, no orphaned processes, no damaged state, and the VM starts again |
| Secret handling | A canary key stays on the host and is used through the broker. The guest disk, volumes, and image are searched for it. | The key appears only in the approved upstream request, outside the VM |
| Release testing | Clean machines, upgrades from the published release, and removal | What users download works, not just what passed on a developer's machine |

## A test suite that never fails is testing the wrong things

These checks have found real bugs. Fuzzing found deeply nested input that could crash the manager. Crash testing exposed recovery cases where a VM could be left running after its manager stopped. The Linux escape probe caught something subtler: after one refused connection, the sandbox wrongly refused every later legitimate one. The cause was a number passed as 32 bits where 64 were needed, and an earlier unit test had missed it.

That bug failed closed. More blocking, not less. Safe, but it would have broken real features. Failing closed is the right direction. It is still a bug.

None of those would show up in a demo. All of them would show up in production.

> Finding a bug is only useful when the failure becomes a regression test.

Each fix now has a test that fails if the problem comes back.

## Two platforms, one set of expectations

Koala now runs on Linux too. It supports Apple silicon Macs through Apple's Virtualization framework, and x86-64 Linux through KVM and Cloud Hypervisor. The same acceptance suites run on real machines on both: command-line workflows, networking, hostile guests, the broker, and crash recovery.

## What the numbers do not say

These numbers describe specific checks. They are not a promise that every failure is impossible.

One limit is by design. A VM can fake the result of its own command, and claim a failed command succeeded. The test confirmed it. A VM can always lie about itself; it cannot lie about anything else. That is recorded as a known limit, and it is why nothing on the host should trust a guest's report about itself.

The detailed hostile-guest runs above were recorded on an Apple silicon Mac with macOS 26. The same network and hostile-guest suites run unchanged on Linux.

Koala is still a preview. I have not yet run long-duration tests under sustained resource pressure, had an independent security audit, or tested across other Linux distributions and hardware. Those are the next gaps, and saying so is part of the evidence.

## The questions to ask about any sandbox's tests

When software runs code on your behalf, "it is isolated" is not enough. Five questions get closer to the truth:

Last week, in [A Clean Room Is Not a Home](https://kunalkushwaha.github.io/2026/10/30/a-clean-room-is-not-a-home/), I asked what any sandbox should answer. These are the questions to ask about how it was tested.

1. How was the boundary attacked?
2. Who judged the result: the guest, or the host?
3. Did the tests run against the real system, or a mock of it?
4. What happened when something crashed halfway through?
5. What has the team not tested yet?

In [The Clock Has Changed](https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/) I argued that mistakes are unavoidable, and a mistake that travels everywhere is a design choice. Testing from the inside is how you find out which one you built.

Koala runs on Apple silicon Macs with macOS 26 or newer, and on x86-64 Ubuntu 24.04 with KVM. Read [how Koala is tested](https://kunalkushwaha.github.io/homebrew-koala/#hostile-guests-crashes-and-fuzzing-on-real-vms), the [release notes](https://github.com/kunalkushwaha/homebrew-koala/releases), or the [project on GitHub](https://github.com/kunalkushwaha/homebrew-koala).

An isolation claim is a hypothesis. Root in the guest is the experiment.

I trust the box after it has been attacked, not before.
