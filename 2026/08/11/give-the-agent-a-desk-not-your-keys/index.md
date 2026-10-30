# Give the Agent a Desk, Not Your Keys

> An autonomous agent with a shell is a coworker you cannot fully trust. Kagebox is an experiment in running one on a spare laptop: its own kernel, host held credentials, an egress latch it cannot reach, and a boundary that has to prove itself.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-08-11
- Tags: ai-native, agents, security, sandbox, building
- Canonical: https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/

![The desk is inside. The latch and the keys stay outside. That asymmetry is the entire design.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/hero.svg)

An autonomous agent can open a shell, install packages, edit files, and drive a browser for an hour without asking. That is what makes it useful, and it is also what makes running one on your own laptop a strange decision.

We have quietly onboarded a new kind of coworker: capable, tireless, and not fully trustworthy. Not because it is malicious, but because it makes mistakes, and because anything it reads can carry instructions it will dutifully follow. A prompt injection is just a coworker taking orders from the wrong person.

You would never give a new hire root on the machine holding your SSH keys. Why are we giving it to an agent?

## We are asking the wrong question

Most of the safety conversation is about making the agent *trustworthy*: better alignment, tighter prompts, more guardrails. That work matters, but you cannot prompt your way to safety when the thing on the other end has a shell. If it can run `curl`, it can exfiltrate.

The more useful question is an older one: **what can this thing reach when something goes wrong?**

We already solved this for people, and not by assuming the worst of them. Access follows the role: the systems the work touches, scoped credentials rather than the master password, and a record of what happened. A new hire is not trusted less than a director. They are scoped differently.

So the experiment was not "how do I make an agent safe enough to hold my keys," but "how do I build a place where it never needs to." I called it **Kagebox**, and the agent inside is [Hermes Agent](https://github.com/NousResearch/hermes-agent).

## Running it at home is the dangerous option, which is the point

In the cloud, a misbehaving agent is a rented box far from anything of mine. On the spare laptop in the other room, it sits on the same Wi-Fi as my phone, my NAS, and the machine I actually work on. Every property that makes local attractive, my hardware and my data and my network, is now a property I have to defend against. So "run it locally" and "isolate it hard" were never two choices. They are one.

## The design, in one sentence

Give the agent a full machine to work on, and possibly wreck, that simply is not yours. Let it reach models and the web through a single door you control. Keep every credential, and the final say, on your side of that door.

![What crosses the boundary and what does not. The agent gets a whole machine; the keys, the latch, the budget, the log, and the memory all live on my side.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/diagram-1-boundary.svg)

**Its own kernel, not your kernel.** The agent runs in a KVM microVM, not a container. A container shares your kernel, and container escape is a regularly discovered class of bug. A microVM makes the boundary a hypervisor rather than a namespace.

**The keys never cross the desk, whichever model you pick.** Every call routes through a bridge that runs as *me*, on the host, and injects the credential. Providers are a runtime registry, so the same broker fronts four different arrangements:

| Backend | What it is | Where the credential lives |
|---|---|---|
| `ollama` | a local model on the host GPU or CPU | no credential at all |
| `gemini`, `anthropic`, `openrouter` | hosted APIs through the bridge | on the host, in `secrets.env` |
| anything OpenAI compatible | an entry in `bridge/providers.json` | same |
| `claude` | the host `claude -p`, existing auth, tools disabled | no API key at all |

Switching is one command, and the agent cannot tell the difference: from inside the box, all four are the same plain endpoint. Compromise the sandbox completely and you get a virtual machine, not my keys.

**The bill is a boundary too.** Taking the credential away from the agent does not take away its ability to *spend* it. The VM bounds CPU, memory, and disk by construction, because those have physical edges. Money does not. So the bridge caps requests per hour, requests per day, and dollars per day, and replays the last twenty four hours on startup so a restart is not a fresh day.

**It remembers, but the memory is mine.** State is snapshotted to the host every ten minutes, so I can delete the VM entirely and the agent returns remembering our history.

## Where the latch lives

An earlier version enforced the network policy inside the VM, which is theater: the agent has root in there and can flush its own firewall. So the rule moved to the host, onto the interface the VM sits behind. Now the agent can be compromised down to root and *still* cannot open the door.

![Same rule, two places. Only one of them is a control.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/diagram-2-latch.svg)

**A control the agent can reach is not a control.** The same principle applies in time as well as layers: containment installs as a root unit ordered *before* the VM manager, so after a reboot the allowlist is up before the guest can pass a packet. A boundary that arrives second is not a boundary.

## The door is a profile, not a switch

![Four profiles, four blast radii. The default profile is deliberately not the tightest one.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/diagram-3-profiles.svg)

| Profile | Reaches | For |
|---|---|---|
| `sealed` | the model bridge and nothing else | untrusted input, forced for throwaway tasks |
| `research` (the default profile) | search, Wikipedia, arXiv, Crossref, Hacker News | reading and writing |
| `dev` | research plus PyPI, npm, crates, Go, GitHub | coding agents |
| `open` | everything | you have decided |

Containment starts at `research` rather than `sealed`, on purpose. A sealed starting point blocks every useful task, so it gets opened once and left open, including by the person who wrote it. A curated one that survives contact with real use is more contained in practice than a strict one nobody keeps. Containment itself is still something you switch on rather than something already on, which is worth saying plainly rather than burying: until you run `./kagebox egress on`, the box reaches the open internet.

No allowlist can express "follow that search result wherever it points," which is most of what research is. So the sandbox can also *ask*. It names the hosts it wants, my phone shows the destination and a duration, and if I approve, exactly those addresses open. The host resolves the names, and the window carries a kernel side timeout, so it closes even if the process that opened it is killed outright.

![The agent can ask. Only I can grant. The approval bot runs on my side because the agent's own bot token lives inside the box.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/diagram-4-approval.svg)

That asymmetry meant the approval bot had to be a *different* bot from the one the agent drives. A guard whose badge sits in the prisoner's pocket is not a guard.

None of this seals every crack, and the repo says so in the open. DNS still resolves. An allowlisted name on a shared CDN drags its neighbors along. The shared folder is a two way door.

## Every layer fails open

One property shaped this project more than any other. Safety systems do not announce their own absence. Features do. A firewall rule that silently did not load looks exactly like one that is working, right up until it matters, and so does a status banner, a default nobody keeps, and a test suite people quietly stopped believing.

![Four layers, one failure direction. Each one looks correct from outside and has to be checked from a place it does not control.](https://kunalkushwaha.github.io/2026/08/11/give-the-agent-a-desk-not-your-keys/diagram-5-fails-open.svg)

I learned this from the reporting layer. Setup used to print `✓ Sandbox ready. (egress: contained)` unconditionally, so declining a sudo prompt still ended in a green checkmark and the word "contained." The truthful line was one row above, in yellow. It just lost to a checkmark. A boundary you *believe* in but do not have is worse than one you know is open, because you will point untrusted work at it.

So Kagebox does not describe its properties in prose. They are ten numbered invariants, each paired with the attack that tests it, plus an honest list of what is not covered. `./kagebox verify` runs them against a live sandbox and reports what it observed rather than what was intended. Never claiming an unverified posture is itself invariant ten.

## What is actually in the box

Shell and standard library Python, no build step. It wants a Linux host with KVM, [multipass](https://multipass.run), and [Ollama](https://ollama.com) for local models.

```bash
git clone https://github.com/kunalkushwaha/Kagebox.git && cd Kagebox
./kagebox doctor          # preflight the host
./kagebox setup           # build the VM, the bridge, and the agent
./kagebox shell           # step inside, then run: hermes
```

| Area | Commands |
|---|---|
| Lifecycle | `setup`, `destroy`, `up`, `down`, `status`, `shell`, `autostart on` |
| Models | `backend <name>`, `providers`, `usage` |
| Containment | `egress on` / `off`, `egress profile <name>`, `verify` |
| Bounded access | `research "<q>"`, `skills <...>`, `warden setup` |
| Isolation per task | `task "<prompt>"` in a throwaway clone, egress forced sealed |
| Memory and rollback | `backup`, `restore`, `snapshot`, `rollback` |
| Review | `audit`, `verify` |
| Reach it from anywhere | `telegram` |

## Why this is the scarce thing now

On an ordinary afternoon I message a bot from my phone. In the other room the agent searches, reads, asks for one site I had not already allowed, and drops a finished document in the shared folder on my main machine. It used my hardware and my accounts, and it could not read a file of mine, see a key, touch my network, open that door on its own, or spend past a number I set.

Building with agents got cheap. Capability is not the constraint anymore. I have argued before that [safe deployment is the new scarcity](https://kunalkushwaha.github.io/2026/06/09/safe-deployment-is-the-new-scarcity/), and agents are where that argument goes next. The constraint is somewhere safe to *run* it: a sandbox, a credential broker, a spend cap, an egress latch you hold from outside, and an audit log. For an organization it is the same list multiplied. These are the controls we already built for people and for microservices, pointed at a new kind of worker.

Kagebox is a small, single machine experiment in that direction, MIT licensed and rough in the places experiments are rough: [github.com/kunalkushwaha/Kagebox](https://github.com/kunalkushwaha/Kagebox).

Stop trying to make the agent trustworthy enough to hold your keys. Build a place where it never has to. Give it a desk, not your keys, and then you can actually let it work.
