# The AI Writes the Screenplay, the Renderer Never Lies

> Notes from building vsim, a tool for AI assisted video that is completely reproducible. The model writes the screenplay as a validated document; a deterministic renderer turns it into the same MP4, byte for byte, forever.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-07-08
- Tags: ai-native, building, video, determinism, typescript
- Canonical: https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/

![vsim: code in, deterministic 3D video out. TypeScript in, MP4 out, byte identical everywhere, no GPU required.](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/banner.svg)

I've been chewing on an idea for a while now, and it turned into my favorite side project: AI assisted video that is completely reproducible.

Generative video feels like magic. You type a sentence and get moving pictures. But as a builder it frustrates me in a very specific way: the output is a slot machine. Run the same prompt twice and you get two different videos. Want to nudge one detail, say the fox turns left instead of right? You can't. There is no source to edit, no diff to review, no way to render the same frames again next year. The pixels are the only artifact, and pixels are a terrible source format.

Meanwhile, on the code side of the world, we solved this decades ago. Code is text. Text is diffable, reviewable, versionable, reproducible. Remotion proved that video as code works beautifully for motion graphics. So my question became:

> What if video was a build artifact? Prompt in, deterministic MP4 out, with a human readable document in between that git can diff?

That question turned into [vsim](https://github.com/kunalkushwaha/vsim), and this post is the story of what I built and what I learned.

## The bet: determinism first, AI second

Most AI video projects put the model in the middle of the pixel pipeline. I did the opposite. The foundation of vsim is a rendering stack that is boring on purpose, with one contract that never bends:

**The same scene renders to the same bytes. Everywhere. Forever.**

Concretely, that means:

- Time is counted in frames, not by the clock. Frame 47 depends only on the scene document, so it comes out the same on every render. There is no `Date.now()` anywhere near the render loop.
- All randomness flows through a seeded RNG. The linter literally bans `Math.random` in runtime code. Particles, flames, grass moving in the wind: all closed form or seeded.
- The reference renderer is pure TypeScript. No GPU, no native modules, no headless Chrome for the 3D path. A software rasterizer with per pixel PBR lighting, shadow maps, mipmapped textures, and supersampling. It is not the fastest renderer in the world, but it produces identical bytes on my laptop, in CI, and in a browser tab.
- CI enforces the contract with golden frame hashes. If a refactor changes even one pixel, the build goes red.

Every scene is a plain JSON document validated with zod, the SceneDocument, describing meshes, materials, lights, cameras, keyframes, skeletal animation clips, even deterministic Rapier physics. A builder API lets you author it fluently from TypeScript:

```ts
export default scene({ fps: 30, duration: 90, width: 640, height: 360 })
  .mesh("cube", { geometry: { kind: "box", size: [1.4, 1.4, 1.4] }, material: "red" })
  .animate("cube", "rotation.y", [
    { frame: 0, value: 0 },
    { frame: 90, value: Math.PI * 2 },
  ])
  .build();
```

`npx vsim render scene.ts -o out.mp4` and you have a video. Run it again in five years and you have the same video.

![The exact scene.ts above, rendered: a red cube rotating over a floor with a moving shadow](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/quickstart.gif)

*That file, rendered. Nothing else involved.*

And boring on purpose does not mean ugly. The whole renderer is under my control, so every feature had to earn its place in the determinism contract, and it adds up. This is the pure TypeScript rasterizer, no GPU anywhere:

![A low poly fox running through a golden hour park: sun disc, long shadows, grass bending in the wind, distance fog](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/fox.gif)

*Golden hour sun and glow, ambient light derived from the sky, distance fog, 700 blades of grass moving in the wind.*

![A fox resting by a flickering campfire at dusk, firelight pooling around the pit, breathing shadows, rising sparks and smoke](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/campfire.gif)

*One flickering point light with inverse square falloff carries this whole scene: breathing shadows, spark and smoke particles, ACES tone mapping rolling the flames off filmically. Every flicker is seeded, so this GIF re-renders byte identical every time.*

| ![deterministic physics tower collapse](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/physics.gif) | ![manga cel shaded scene](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/manga.gif) | ![cartoon mouse waving](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/mouse.gif) |
|:---:|:---:|:---:|
| **Deterministic physics.** Rapier rigid bodies, the same collapse every run | **Manga mode.** Cel shading plus ink outlines with one flag | **Characters from primitives.** No rig, just pivoted node groups |

None of this involves AI yet. That is the point. The renderer is the part of the system that never lies, so it is the part the AI is never allowed to touch.

## Where the AI goes: authoring time, not render time

Once the deterministic substrate existed, the AI question answered itself. The model does not generate pixels. It does not even generate keyframes. It generates documents, and only documents the schema will accept.

The mechanism is tool use constrained by the schema: the model's output is forced through zod validation, grounded in the scene's actual objects. The result is a property I have come to think of as the core design rule of the whole project:

> The AI can propose a bad film. It cannot emit an invalid one.

A bad film is a taste problem. You iterate. An invalid film is a runtime crash, a fox walking through a tree, a camera pointing at nothing for eight seconds. The schema and the compiler make the second category unrepresentable.

This shipped in three escalating layers.

**Layer 1: editing scenes in natural language.** `vsim edit scene.ts --prompt "make the cube blue and add a point light"` turns the prompt into validated edit operations against the existing document, applies them deterministically, and hands you a new document you can render like any other.

**Layer 2: 2D films from a screenplay.** `vsim film -p "how a CDN makes websites fast"` asks the model for a FilmDoc: stage entities, beats, actions, and camera moves over a library of explainer primitives. Servers, packets flying along paths, arrows that draw themselves on, kinetic titles. An interpreter plays the document on a frame pure SVG timeline; a recorder steps it frame by frame into an MP4. Narration comes from timed lines through TTS with a mouth envelope computed per frame, so even the lip sync is deterministic.

| ![how a web request works](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/web-request.gif) | ![Pip the mouse, talking](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/pip-hello.gif) | ![what is a message queue](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/message-queue.gif) |
|:---:|:---:|:---:|
| **Technical explainer.** DNS to TLS to request to render, with karaoke captions | **Voiced character short.** Pip's mouth is driven by the narration's audio envelope | **The template proof.** A second explainer from the same kit, one screenplay later |

And here is the payoff. The AI wrote this entire film from the prompt above. I did not touch a keyframe; the model wrote the screenplay document and the interpreter did the rest:

![An AI written explainer film: how a CDN makes websites fast](https://kunalkushwaha.github.io/2026/07/08/deterministic-video-ai-writes-the-screenplay/cdn.gif)

*`vsim film -p "how a CDN makes websites fast"`. The committed screenplay re-renders this exact film forever, no AI required.*

**Layer 3 is the one I am most excited about: real 3D films directed by the AI.**

## Prompt to screenplay to keyframes to MP4

For 3D, the model writes a Film3DDoc: a screenplay at story altitude. It picks a set preset (meadow, dusk, night, snow), places props, casts actors from a bundled character library, and writes beats of `move`, `play`, and `face` actions plus a shot list of wide, close, follow, and orbit cuts.

Here is an entire beat from a film in the repo. This is the level the AI operates at:

```json
{
  "id": "dash",
  "start": 12,
  "end": 17,
  "caption": "Then it is gone — a streak of orange into the trees.",
  "actions": [
    { "do": "move", "actor": "fox", "to": [-7.5, -5], "at": 0.5, "dur": 3.4, "gait": "run" }
  ]
}
```

No keyframes. No quaternions. No camera matrices. Just what happens in the story.

A compiler then lowers this to a plain SceneDocument, and the compiler is where all the film craft actually lives:

- It walks the beat list tracking each actor's position and heading, and turns `move` into position tracks plus eased rotations that swing the actor into its heading, with angle unwrapping so a fox never spins 350 degrees the long way around.
- It picks a gait from travel speed, distance over duration, crossfades the rig's walk or run cycle in, then settles back to idle on arrival.
- Cameras do not aim at an actor's feet. Every character gets an aim node at head height, and follow shots track it.
- The schema validates that clips actually exist on each character's rig, that beats are contiguous, that actions do not overrun their beats, and that the shot list covers the whole film. The model literally cannot hand back a screenplay that does not run.

The best part: the generated `*.film3d.json` gets committed to the repo. From that moment the AI is optional. The film re-renders byte identical forever, on any machine, with no API key. The prompt was scaffolding; the document is the source of truth.

## What I learned

**Determinism is a discipline, not a feature.** You do not add it at the end. Every subsystem had to be designed under the contract: particles, physics, text rasterization, audio muxing, even multicore rendering, where workers split the frames but must produce bytes identical to the single threaded path. The lint rule banning `Math.random` did more for the project than any single algorithm.

**Give the model a language, not a canvas.** The single biggest quality jump came from raising the altitude of what the AI writes. Ask a model for keyframes and you get mush. Ask it for beats with captions, actions, and a shot list, and you get something that reads like a director's treatment, because that is the shape of the data. The schema is not just validation. It is a creative constraint that keeps the model in the part of the problem where it is actually good: story, pacing, staging.

**Put the craft in the compiler.** Everything that requires judgment applied consistently belongs in deterministic code, written once and tested: gait selection, turn easing, camera framing. The AI decides that the fox dashes into the trees; the compiler decides what a dash looks like. I think this split is the general pattern for AI assisted creative tools: models for intent, compilers for execution.

**The document in the middle changes everything.** Because the intermediate artifact is human readable JSON, I can review an AI written film like a pull request, tweak one number by hand, diff two versions of a shot, and bisect a regression. Try doing any of that with a video model's latent noise.

And one more lesson, a very meta one: the project itself was built the way it works. A huge amount of vsim was written pair programming with an AI coding agent, under the same contract the renderer applies to the pixels. The agent proposes; the type checker, the tests, and the golden frame hashes decide. Determinism turns out to be exactly what makes AI assistance safe to accept, in both places.

## Where this goes

vsim today is an MIT licensed monorepo: the deterministic renderer, a Three.js engine for GPU fidelity preview, physics, a character library, a browser player that shares the exact runtime with the exporter, a fledgling visual editor, and the three AI authoring layers. Next on my list: richer lighting per shot, better set dressing, and letting the copilot iterate on a film it can see. Render, look, revise, all through the same validated document.

If the idea of films you can `git diff` appeals to you, the repo is at [github.com/kunalkushwaha/vsim](https://github.com/kunalkushwaha/vsim). Clone it and `pnpm showreel` rebuilds every demo from source, byte for byte identical to mine. That reproducibility is not a footnote. It is the whole idea.

```bash
pnpm install
pnpm film3d:fox                                # a hand written 3D screenplay, rendered to MP4
vsim film -p "your story here" --template 3d   # or let the AI direct one
```

The AI writes the screenplay. The renderer never lies.
