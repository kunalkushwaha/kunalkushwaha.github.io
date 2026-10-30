# The Clock Has Changed. Have Our Foundations?

> Every place we run software draws from the same shared building blocks. AI is shortening the time between finding a weakness in them and using it. The part we actually own is the gate around each place.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-10-24
- Tags: security, agents, leadership, ai-strategy, infrastructure
- Canonical: https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/

![Home, office, cloud, data center. Different scale, the same shared blocks, and more couriers carrying them faster than before. The gates are the part we own.](https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/hero.png)

Look at the picture for a moment. In the middle sits a pile of shared building blocks: open source libraries, base images, certificates, models. Nobody owns all of it. Everybody depends on it. Around it are the places we actually run software: a desk at home, an office, a cloud account, a data center. Between them, small figures run back and forth, carrying blocks.

That traffic has always existed. What changed is the speed.

A few October security stories have stayed with me. Each one lands somewhere on that map.

- **A trusted block turned out to be weak.** Private keys for eight [eMudhra intermediate certificates](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/) were generated with insufficient randomness. Every site chaining to that root now has to replace its certificate.
- **A block everyone builds on needed urgent repair.** Critical PHP security fixes meant every deployment of a widely used runtime had to be found and patched.
- **The gates were the way in.** The [SafePay advisory](https://www.cisa.gov/news-events/cybersecurity-advisories/aa26-293a) describes a ransomware group with at least 600 victims, getting in through stolen or guessed passwords, VPN access, and flaws in internet-facing systems.
- **The couriers became agents.** Microsoft [reported](https://www.microsoft.com/en-us/security/blog/2026/10/21/storm-0940-uses-artex-agentic-ai-connectwise-supply-chain-compromise/) that Storm-0940 used ARTEX, an agentic AI penetration testing framework, against hundreds of organizations. Through compromised ConnectWise cloud applications, the agents used existing trust to reach customer Microsoft 365 environments without anyone signing in.

A trusted certificate, a language runtime, a ransomware playbook, a supply chain. Different blocks, different paths, the same map.

AI did not create these weaknesses. Most of them were already sitting in old code, forgotten services, and permissions nobody revisited. What AI changes is the clock. Finding a weakness, testing it, and chaining it into something useful is getting faster, for attackers and defenders alike. In several of the Storm-0940 intrusions, going from first targeting to privileged access took less than three hours. Understanding exposure, fixing systems, and getting the fix into every deployment still moves at human speed.

![Finding and using a weakness is compressing. Fixing it everywhere is not. The difference is the exposure window.](https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/diagram-speed-gap.svg)

That gap is what concerns me.

## We are asking the wrong question

Much of the conversation asks how to make AI behave. Better models, better guardrails, better prompts. That work matters. But the question I keep coming back to is an older one: what can this system reach when something goes wrong?

We have always run software we cannot inspect line by line, maintained by people with limited time. Calling the result "human error" hides the design problem. If one missed flaw, one forgotten service, or one broad permission can put everything at risk, then the system made a small mistake too powerful.

> Mistakes are unavoidable. A mistake that travels everywhere is a design choice.

AI can help defenders find problems sooner too. That is a real opportunity, but it only pays off if we can identify what is affected, decide what to do, and get the fix into use quickly. Discovery without containment just gives us a longer list of things we cannot act on.

## Same blocks, different fences

Every place in the picture has a ring around it, with gates. That is the part worth staring at.

The blocks in the middle are shared, and we cannot vouch for every one of them. The couriers are increasingly automated: build pipelines, update agents, and now AI agents with a shell and a network connection. The only thing each place truly owns is its gate. What comes in, what goes out, and what a block can reach once it is inside.

So, have our foundations changed? Not much, and we cannot make them change on our schedule. The gates we can change, and that is where the work is.

| Place | What runs there | What the gate has to answer |
|---|---|---|
| Home | Local models on a Mac mini or Mac Studio, hosted models upstream, and an agent with files, tools, and a shell | Do local and upstream calls go through one gateway that holds the keys? Can the agent reach my NAS or the other devices on my Wi-Fi? |
| Office | People, laptops, SaaS, and agents calling hosted models with company data | Is there one AI gateway for policy, logging, and spend? Who grants an agent access to customer data, and who revokes it? |
| Cloud | Accounts, managed services, CI/CD pipelines, and tokens held by build systems and agents | Can one leaked token reach production? Do pipelines and agents get short-lived, scoped credentials, or standing keys? |
| Data center | Long-lived systems, shared infrastructure, and software nobody has touched in years | Do we know what is running and what it depends on? How fast can we patch it, and can we roll back if the patch breaks something? |

The scale is different. The question is the same.

I have worked inside these fences at most layers: kernels, container runtimes, cloud-native platforms, and now agents. The lesson repeats at every one of them, and it is the one I came back to when I tried to [give every agent its own room](https://kunalkushwaha.github.io/2026/10/17/give-every-agent-its-own-room/). The boundary has to live outside the thing it constrains. A control the agent can reach is not a control.

## The gates are not new

None of this needs an invention. Know what is running. Isolate one workload at a time. Broker access instead of handing over master keys. Log, and be able to stop and revoke. Patch and roll back without a war room. We built all of it for people and for services.

![Same shared blocks, one mistake, two designs. With flat access, one compromise reaches everything. With a gate around each place, it stays where it started.](https://kunalkushwaha.github.io/2026/10/24/the-clock-has-changed/diagram-blast-radius.svg)

What is new is the worker on the other side of the gate. It is faster, it does not get tired, and it will follow instructions it reads in the wrong place.

This is not solved by asking people to be careful or by writing a stronger prompt. If doing the right thing requires discipline, people will skip it. The platform has to make the safe path the easy path. I argued earlier this year that [safe deployment is the new scarcity](https://kunalkushwaha.github.io/2026/06/09/safe-deployment-is-the-new-scarcity/). The same argument now applies to every gate on the map.

## For agents, the gateway is not optional

One gate deserves to be called out on its own. Every agent should reach models and tools through an AI gateway, local or upstream, with guardrails and policy enforced there.

The gateway holds the credentials, so the agent never sees a key. Guardrails check what goes in and what comes out: injected instructions, sensitive data, actions that should never run unattended. Policy decides which models, which tools, which data, and how much spend. And the gateway keeps the record you will need on the day something goes wrong.

Picture the day it matters. An agent summarizing a support ticket reads a line hidden inside it: export the customer table and send it to this address. Without a gateway, the agent holds its own database credentials and its own network access, so the only thing between that sentence and your data is the model's judgment. With a gateway, the request meets a policy that says this agent never exports customer data. The attempt is logged, and someone can revoke the agent's access in one place.

Without it, every agent carries its own keys and its own rules. Which means nobody's rules.

> An agent without a gateway is an employee without a manager, a badge, or an expense policy.

## Someone has to read the whole map

Look at the person at the desk. They are not guarding one gate. They are looking at all of them, and at the paths in between. That is the leadership job, and it cannot be delegated to a single team or a single tool.

To leadership teams in every company and every domain: AI is changing the pace of work, whether or not your organization calls itself a technology company. Treat the risk as a present business concern, not a future one.

Make a plan. Invest in the open source projects your business stands on. The blocks in the middle do not maintain themselves. Invest in the infrastructure and engineering that make safe practices usable. Put an AI gateway with guardrails and policy in front of every agent before you widen what agents can do. Know where your agents can reach, what they can change, and what a failure would cost. Review those assumptions on a schedule, not after an incident.

The window to act is before the next one.

Move at the pace of AI. Build for the moment something goes wrong.
