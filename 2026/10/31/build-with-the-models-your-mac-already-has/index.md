# Build With the Models Your Mac Already Has

> Macs now ship strong on-device models for speech, translation and text. I built Kikitori, a private meeting translator, on them alone: no third-party model, no cloud. Here is what it costs, measured.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-10-31
- Tags: apple-silicon, edge-ai, llms, swift, performance, building
- Canonical: https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/

![A meeting in Tokyo, in two languages. Kikitori translates on the laptop, and nothing leaves the Mac.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/hero.jpg)

My company's official language is English, but we are in Tokyo, and plenty of discussions drift into Japanese. My colleagues are generous about it, and they help me learn. Still, we kept saying the same thing: we wish something translated English and Japanese in real time, without leaking anything.

That wish became Kikitori (聞き取り, "listening comprehension"): a macOS menu bar app that translates the meeting as people speak, then writes a transcript and a summary. It works both ways, so it helps my colleagues follow English too.

"Without leaking anything" was the hard requirement. Meeting audio is about as sensitive as data gets, so nothing leaves the Mac. Every step runs on models macOS already has. The download is 582 KB: no third-party model, no cloud.

> The overhead didn't disappear. It moved from my app into the operating system.

![I take notes in two thirds of the screen while Kikitori translates the meeting in the last third, newest line at the top.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/screenshot.png)

## Before: bring your own model

The first version wrapped whisper.cpp. Whisper is excellent, but everything around it was mine to carry: Homebrew packages, a 3.1 GB model, and a live pipeline that started over for every clip.

![Before, every 2-second clip launched ffmpeg twice and reloaded the Whisper model. After, one SpeechAnalyzer stream runs for the whole meeting, with partial text while people speak.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/diagram-before-after.svg)

## After: ask the OS

macOS 26 ships the models behind frameworks: `SpeechAnalyzer` for speech, Translation, Foundation Models for Apple's on-device language model, and Core Audio taps to capture a call. The OS downloads, updates, and runs them, and every app shares them.

![Kikitori keeps the audio, the language choice, and the UI. The models run in macOS services.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/diagram-architecture.svg)

So the app is mostly product. The glue for all three models is under 700 lines of Swift. The nicest part is structured output: describe the result as a Swift type, and the model fills it in.

```swift
@Generable
struct DiscussedTopic {
    @Guide(description: "What was first proposed")
    var firstProposal: String
    @Guide(description: "What everyone finally agreed, with the final numbers")
    var finalAgreement: String
}
let notes = try await session.respond(to: chunk, generating: GeneratedNotes.self)
```

Field order matters. Until I asked for the first proposal before the final agreement, the model reported the first number it heard as the decision.

## The overhead, measured

I benchmarked the same APIs and sampled every process on the machine, not just my own.

![The headline numbers. Live captions in two languages cost about 6% of one CPU core.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/diagram-numbers.svg)

## Where the cost actually goes

My process barely registers. The models run in system services, and that is where the memory is.

![Peak memory of the app versus the system service doing the work, for each job.](https://kunalkushwaha.github.io/2026/10/31/build-with-the-models-your-mac-already-has/diagram-cost.svg)

So it is not zero overhead. But the cost is shared by every app, loaded only while needed (`translationd` idles under 15 MB), and none of it ships in my installer.

## What isn't free

- **No language detection.** A transcriber needs a locale, so Kikitori runs Japanese and English side by side, and an arbiter picks per sentence. It is the hardest code in the app.
- **A 4,096-token context.** Long meetings are summarized in chunks. Those summaries are the part I trust least.
- **English first.** Summaries are written in English and then translated, because in Japanese the model lost track of revised decisions.
- **Opt-in and platform-bound.** Users download speech models, add translation languages, and turn on Apple Intelligence. It is macOS 26 and Apple silicon only, and an OS update can change the model under you.

## Try it

```bash
brew install --cask kunalkushwaha-reazon/kikitori/kikitori
```

It is not notarized yet, so the first launch needs **System Settings › Privacy & Security › Open Anyway**. And please ask before you record other people.

It started as a wish in our own meetings. If your team also works across two languages, tell me whether it helps: [issues are open](https://github.com/kunalkushwaha-reazon/homebrew-kikitori/issues).

The best model for your app may already be on your user's Mac.

You just have to ask the OS for it.
