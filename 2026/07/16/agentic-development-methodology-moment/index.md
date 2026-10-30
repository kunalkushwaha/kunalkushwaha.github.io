# Agentic Development Needs Its Methodology Moment

> Every leap in who writes software was unlocked by process, not talent. AI agents are the next leap, and the teams winning with them gave their agent a methodology.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-07-16
- Tags: ai-native, agentic, building, engineering, process
- Canonical: https://kunalkushwaha.github.io/2026/07/16/agentic-development-methodology-moment/

![The model is the water. The methodology is the canal: a lock gate at every phase, lifting the work from idea to shipped.](https://kunalkushwaha.github.io/2026/07/16/agentic-development-methodology-moment/hero.svg)

Waterfall gave us predictability we could not afford. Agile gave us feedback loops. Sprints and CI gave us rhythm and a referee. Every leap in who writes software, and how fast, has been unlocked not by talent but by process. AI agents are the next leap. And the teams winning with them are not the ones with the cleverest prompts. They are the ones who gave their agent a methodology.

This is a field guide from a real project: AnimStudio, a Rust and WASM animation studio with a creator mode built for kids, where an AI coding assistant ran the loop end to end. Product definition, design docs, GitHub issues, implementation, tests, browser verification, adversarial code review, and merges. Around thirty PRs, CI green throughout, a designer in the loop, and real users breaking things.

We spent two decades learning that human developers ship reliably only inside a harness: a definition of done, small reviewable units, tests as gates, retros that turn incidents into rules. Then we handed AI assistants a chat box and expected senior engineer outcomes from vibes. The result is familiar. Dazzling demos, unshippable diffs.

The correction is the same one it was for humans. The model writes the code. The methodology is what makes it shippable. I have argued before that [you do not trust the output, you trust the loop](https://kunalkushwaha.github.io/2026/06/24/vibe-assisted-agentic-who-reviews-the-work/). This post is what that loop looks like when you actually write it down: the concrete harness one project ran on, and the specific failures it caught that raw capability never would have.

## Phase zero: define the product before you automate anything

Waterfall got one thing right. You cannot iterate toward a destination you never named. Before the first issue existed, this project wrote down four things, in versioned documents that carry real weight, not in a chat scrollback.

**The problem statement and the non goals** (`DESIGN.md`). An animation studio that lives in the browser: 2D game style animation, Markdown slide decks, simple 3D, fully client side, no server required. The things v1 would not do were listed with the same care as the things it would (no cloud accounts, no general 3D engine), because an autonomous agent will happily build anything you forgot to exclude.

**How it ships, hosts, and is accessed.** Decided up front, in scope. Running fully client side is an architecture decision with hosting consequences: the app builds to a static bundle (wasm-pack plus Vite), deploys to any static host, needs no backend, and keeps user work locally in the browser (OPFS). The question of how a user gets this and where their data lives was answered before feature one, so no later feature could quietly assume a server that did not exist.

**The invariants that carry the whole design, as Architecture Decision Records.** One JSON scene document as the single source of truth. Every edit an invertible command, which buys undo now and collaboration someday, for free. A deliberately coarse boundary between JS and WASM. Deterministic sampling: what you scrub is what you export.

**A milestone roadmap** (`ROADMAP.md`). From M0, the walking skeleton that proves the whole toolchain end to end (the first deliverable was a spinning quad, not a feature), through shipped versions, each milestone ending in a tag.

Only then did work exist as GitHub issues: the roadmap decomposed into tasks with context, acceptance criteria, and fences around what is out of scope. The agent never works from chat instructions. It works from the queue. Chat is for changing the queue.

This is the waterfall shaped ten percent that agile never actually killed: name the product, fix the invariants, decide how it reaches users. Everything after runs as tight iterative loops.

## The loop is law: five phases with exit gates

If phase zero is the waterfall residue, the working loop is pure iteration: a sprint per issue, with a definition of done a machine can check. The repo carries an `AGENTS.md` that defines how any contributor, human or agent, works:

```
1. DESIGN → 2. IMPLEMENT → 3. TEST → 4. VERIFY → 5. REVIEW → merge
```

![The five phase loop. Each phase ends at a gate a machine can check, and failures return to the phase that owns them.](https://kunalkushwaha.github.io/2026/07/16/agentic-development-methodology-moment/diagram-1-loop.svg)

Two properties make this work for an agent.

**Each phase has an exit gate you can check mechanically.** Design is not done until a design note is posted on the issue. Implementation is not done until the build passes. Test is not done until the suites are green locally. Verify is not done until every acceptance criterion has been observed in the running app. Review is not done until findings are fixed or filed. Agents are relentless executors of explicit rules and unreliable interpreters of vibes. Gates convert quality from a vibe into a rule.

**Failures return to the phase that owns them.** A failing test sends you back to implementation. A wrong approach sends you back to design, and the design note on the issue gets amended, in public. The audit trail is the issue thread.

## The issue queue is the plan

Before any code: strategy, then milestone, then issues. Every issue follows one template: context (why this exists, with evidence), acceptance criteria (checkboxes a verifier can execute), and out of scope (the fence). Two rules earn their keep many times over.

**One issue, one branch, one PR.** When the agent discovers a different problem mid task, and it will, constantly, the rule is simple: file a new issue, do not bundle. In this project that turned incidental discoveries (a favicon 404, a global hotkey eating keystrokes from text fields, undo stacks surviving document swaps) into a clean backlog instead of scope creep.

**Design notes before code, as issue comments.** Five to fifteen lines: approach, files touched, API changes, risks, out of scope. When a design decision changed mid flight (bit exact golden fixtures became a tolerance of 1e-12, because elastic easings go through platform libm), the amendment was posted with the reasoning before the code diverged from the spec.

For bigger scope shifts we added a journey walk: play through the product as the actual user, in this case a twelve year old making a Mario level and a 3D drone video, write down every wall they hit, then review the resulting scope three times. Once for architecture fit, once for user value, once to cut. The three passes killed the nice to haves (motion path ghosts, sound) and surfaced the things that actually mattered: you could not delete anything, the movie length was a hard wall, and switching templates silently destroyed yesterday's work.

## Verification is not testing, and it catches different bugs

The single highest value rule in the repo: phase 4 means running the real app and observing the acceptance criteria, not inferring them from passing tests. The agent drives the actual dev server with a headless browser, clicks the real buttons, screenshots the real canvas, and reads the screenshots.

![What the tests said, next to what the screen showed. Only one of these columns tells the truth about the product.](https://kunalkushwaha.github.io/2026/07/16/agentic-development-methodology-moment/diagram-2-verify.svg)

What only verification caught, in one project:

**A transposed view matrix.** The fly camera feature added a quaternion camera path. Unit tests passed, because identity and 180 degree rotations are symmetric and literally cannot distinguish a rotation from its transpose. The screenshot showed the camera staring into empty sky. The fix came with a 90 degree pitch test that pins the transpose forever. The lesson: write the test that can tell your failure modes apart, and you often only learn what those are by watching the real thing misbehave.

**A camera path that flew through buildings.** The generated city template validated, parsed, and sampled deterministically. And the flight clipped straight through towers. Only eyes on pixels catch technically correct, visually absurd.

**A drag that committed but looked dead.** The selection box only redrew when the playhead moved. With playback paused, drags landed in the document but nothing moved on screen. Every unit test of the command path passed. The user facing symptom, "the studio is broken," was pure presentation.

**3D cubes rendering as a screen filling circle in 2D scenes.** Found because the verification habit is to look.

Verification scripts do not get thrown away either. The ad hoc checks grew into `sweep-creator.mjs`, a script that lives in the repo: thirteen end to end assertions (select, drag, action, record, fly, undo, both UI modes) that run against the dev server and exit with an error when anything fails. Executable acceptance criteria, reused as the regression gate for every later PR.

## Review as an adversarial pass, not a formality

Every PR gets a self review before merge, but structured as independent finder passes with different angles: line by line, removed invariants, cross file tracing, conventions. Each pass produces candidates with a concrete failure scenario, and each candidate is verified before being accepted. Findings are either fixed in the PR or filed as issues. Nothing is silently dropped.

Real catches from this structure, all before merge:

**CI's clippy job never compiled the wasm target**, so every `js_sys` and `web_sys` lint in a new determinism guard was unenforceable. The reviewer angle "trace the invariant to where it is enforced" found it. The fix added a second clippy pass and proved both with canary compiles: deliberately committing a violation to watch the lint fail, then reverting.

**`zip()` in a golden fixture comparison silently truncates.** A grid size change would have passed green with half the samples unvalidated.

**The AI copilot could poison its own session.** A failed turn left a dangling `tool_use` in the conversation history, making every subsequent API call fail with a 400. Found by asking what state survives an exception.

**Quaternion keys with flipped hemispheres in already committed data.** The reviewer verified the defect in the artifact, not just the code, and the fix included sign alignment plus regeneration of the data.

## Make invariants enforced, not assumed. Then add self repair.

The project's worst bug reached the user: a corrupted save file that boot looped the app. The chain is instructive. An action generator placed a keyframe exactly on an existing one. The engine's `apply` accepted the duplicate, because only the loader validated strict ordering. The autosave persisted it. Reload rejected the whole document. Permanent failure, no fallback.

Three lessons, each now enforced in code.

**An invariant checked in one place is a bug factory.** If commands promise that valid documents stay valid, the command layer must reject violations, not rely on generators being polite. `AddKeyframe` now refuses duplicate times.

**Fix the producer too.** The generator now chains one frame past existing keys. But never fix only the producer.

**User data deserves self repair.** The boot path now attempts an automatic repair (drop duplicate time keys, newest wins), quarantines what it cannot fix, and starts clean instead of dying. The user's broken world loaded again on their next reload with a "fixed a hiccup 🔧" toast.

The same philosophy powered the determinism guard. The engine's core promise, that sampling is a pure function of document and time, went from a comment to a clippy disallowed methods list plus golden value fixtures over every example file, with the fixtures globbed from disk so a new example cannot silently escape coverage.

## The supporting cast

**Docs carry weight.** `DESIGN.md` (architecture and scene format) and `ROADMAP.md` (milestones) are declared the source of truth, and the loop forbids silently diverging from them. When a designer joined, the agent produced a handoff spec (tokens, components, states, open questions), the designer answered with screens, and the adopted decisions were written back onto the issues before implementation. Same loop, new participant.

**CI is the arbiter, merges are earned.** Clippy with warnings treated as errors on two targets, all test suites, typecheck, build. Squash merges, milestone tags, and an explicit human grant before the agent was allowed to merge its own green PRs.

**Small PRs, honest checklists.** Every PR body fills the gate checklist truthfully, including entries like "docs only, no runtime surface to verify" when that is the truth. An agent that reports "tests pass but I did not run the app" is more useful than one that rounds up.

## The actual files: CLAUDE.md and AGENTS.md

Two small documents carry the whole system. The division of labor matters. `CLAUDE.md` is what every session must know about the repo, loaded into the assistant's context automatically. `AGENTS.md` is the process contract, referenced from `CLAUDE.md` and read when work starts. Keep both short enough that they are always true.

The project's real `CLAUDE.md`, near verbatim:

```markdown
# AnimStudio repo guide

Browser based animation studio: Rust/WASM engine + TypeScript/Svelte studio UI.
Read **DESIGN.md** (architecture, scene format, ADRs) and **ROADMAP.md**
(milestones) before making structural changes. They are the source of
truth for scope and module boundaries.

## Development process (mandatory)

All feature work follows the five phase loop in **AGENTS.md**:
**design → implement → test → verify → review**, one GitHub issue at a
time, on a branch, ending in a PR that closes the issue. Each phase has
an exit gate; never skip one (in particular: phase 4 means running the
app in a browser and observing the acceptance criteria, not just
passing tests).

## Commands

- `npm run dev`: build engine (wasm-pack) + Vite dev server
- `npm run build:engine`: rebuild WASM only (Vite does NOT watch Rust)
- `cargo test --manifest-path engine/Cargo.toml`: engine tests
- `cargo clippy ... -- -D warnings`: CI enforces clippy clean

## Architecture rules

- `studio/src/bridge.ts` is the ONLY place the studio imports the WASM
  module.
- JS to WASM boundary stays coarse grained: full document load plus
  incremental commands. No fine grained per property setters across
  the boundary.
- The JSON scene document is the single source of truth; every edit is
  an invertible Command (undo/redo, future collaboration).
```

Note what it contains: the one paragraph mental model, the process pointer, the exact commands including the trap (Vite does not watch Rust), and the two or three architecture rules that must never be violated. Nothing else.

And the heart of the real `AGENTS.md`:

```markdown
## The loop

  1. DESIGN → 2. IMPLEMENT → 3. TEST → 4. VERIFY → 5. REVIEW → merge
  (failures return to the phase that owns them)

### 1. Design, before writing any code
- Write a short design note (5 to 15 lines) and post it as a comment
  on the issue: approach, files touched, public API changes, risks,
  out of scope.
- If the design conflicts with DESIGN.md, STOP and raise it. Never
  silently diverge.
**Exit gate:** note posted on the issue.

### 4. Verification: tests passing is not "it works"
- Run the real app and exercise the issue's acceptance criteria end
  to end in the browser. Actual behavior, not inference from tests.
- Record evidence in the PR (what you ran, what you observed,
  screenshots).
**Exit gate:** every acceptance criterion observed working.

## Rules of engagement
- **The issue queue is the plan.** Don't invent unscheduled work.
  File an issue instead.
- **One issue, one branch, one PR.** Discover something else? File it.
- **Never merge with a red or skipped gate.** "Tests pass but I didn't
  run the app" fails phase 4. "It works but has no tests" fails
  phase 3.
- **Acceptance criteria are the contract.** Wrong or ambiguous? Get
  them corrected on the issue before building.
```

Everything in the sections above is just these two files, obeyed without exception.

## How my own workflow matured into this

I have been running agent loops on nearly everything I build, side projects and main work alike, for a while now. The harness in this post did not arrive fully formed. It is the current rung of a ladder, and naming the rungs might save you some climbing.

![Three rungs: chat driven coding, goals with loops, and a process that lives in the repo and ratchets.](https://kunalkushwaha.github.io/2026/07/16/agentic-development-methodology-moment/diagram-3-ladder.svg)

**Rung one was chat driven coding.** Ask, paste, run, ask again. Brilliant moments, zero consistency. Every session started from nothing, quality depended on my attention that day, and nothing an incident taught us survived the scrollback.

**Rung two was goals and loops.** I started giving the agent a standing objective, keep picking up tasks one by one until everything is finished, with a basic dev loop attached. Throughput jumped. But the failure modes just moved: scope bundling, skipped verification when the agent felt confident, fixes that regressed quietly, and a process that lived in my prompts, so it evaporated between sessions.

**The current rung, the one this whole post describes, came from a single realization: the process has to live in the repo and ratchet, not live in me.** Concretely, what changed:

- `CLAUDE.md` and `AGENTS.md` made the methodology ambient. Every session, every machine, starts at full maturity instead of deriving it again from chat.
- The issue queue became the only source of work. My chat messages change the queue, they do not bypass it. Standing goals now mean drain the queue through the loop, with merges earned by green gates rather than by my mood.
- Verification became executable and cumulative. The ad hoc browser checks of each feature hardened into a sweep in the repo that every later change must pass. Quality stopped being argued again on every PR.
- Incidents became invariants. A corrupted save file did not end as a hotfix. It ended as an engine level rejection, a producer fix, and a self repair path, plus a new sweep check. The process only moves one direction.
- Collaborators who do not write code plugged into the same loop. A designer received a generated spec, returned screens, and the adopted decisions were written onto the issues before implementation. No special ceremony.
- Persistent memory across sessions means the agent itself remembers the conventions, the gotchas, and where the bodies are buried.

The compounding effect is the point. Each project I run this way starts more mature than the last one ended, and the results have become boringly consistent, which is the highest compliment a process can earn. My role has drifted upward. I define products, set direction, react to mocks, arbitrate scope, and read honest gate reports. The agent does the engineering. The harness does the trust.

## A starter checklist

If you want autonomous development you can actually trust:

1. Write the loop down (`AGENTS.md`) with mechanically checkable exit gates. Make phase 4 "drive the real app," never "tests pass."
2. Plan in issues: context, acceptance criteria, out of scope. One issue, one PR. File discoveries instead of bundling them.
3. Post design notes on issues before code. Amend them publicly when reality disagrees.
4. Keep an executable end to end sweep next to the code and run it for every change.
5. Review with adversarial finder passes. Require concrete failure scenarios. Fix or file every finding.
6. Enforce invariants where the data is written, not where it is read. And give user data a self repair path.
7. Let CI be the judge, keep PRs small, and make the agent report honestly against the gates.

None of this is exotic. It is the discipline most teams aspire to, the definition of done, the small PR, the honest retro, applied without exception, which is precisely what an agent is good at and humans find exhausting. We did not get reliable human engineering from smarter humans. We got it from waterfall's respect for definition, agile's respect for feedback, and CI's refusal to negotiate.

Agentic development is now having the same methodology moment. The teams that treat the harness as a first class product of their engineering culture, with problem statements, design docs that say how it ships, an issue queue as the plan, and gates that cannot be skipped, will ship with agents. The teams prompting harder will keep making demos.
