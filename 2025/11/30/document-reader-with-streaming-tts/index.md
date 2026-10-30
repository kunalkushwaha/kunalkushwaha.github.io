# Building a Local Document Reader with Streaming TTS

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2025-11-30
- Tags: AI Agents, Go, Python, TTS, Edge AI
- Canonical: https://kunalkushwaha.github.io/2025/11/30/document-reader-with-streaming-tts/

![KittenTTS demo](https://kunalkushwaha.github.io/2025/11/30/document-reader-with-streaming-tts/KittenTTS-blog.PNG)

Ever wished you could just press play on a long document and have it read to you without uploading anything to the cloud? That’s exactly what this project does. Feed it text, and within milliseconds, you’re hearing it narrated. No waiting, no cloud APIs, just your machine doing the work.

## Why Go Small?

Look, I get it. Everyone’s obsessed with massive language models these days. But here’s the thing: you don’t need a sledgehammer to crack a nut. Small language models (SLMs) are perfect for focused tasks like summarizing documents or prepping text for speech synthesis. They’re fast enough to run on your CPU, light on power consumption, completely private (nothing leaves your device), and available even when your WiFi isn’t.

For a document reader, that’s all you really need.

## The Magic of Streaming

Here’s where it gets interesting. Instead of synthesizing the entire document before playing anything (yawn), we stream the audio as it’s generated. The server sends chunks of audio data as soon as they’re ready, and the client starts playing immediately.

Think about it like buffering a video. You don’t wait for the whole movie to download before hitting play. Same principle, but with text-to-speech.

## Real Uses (That Actually Make Sense)

**The documentation listener**. Got a folder full of markdown docs? Technical notes? README files? Convert them to audio and listen while you work on something else. Perfect for absorbing knowledge when reading isn’t practical.

**The commute companion**. Long-form articles, blog posts, or research papers saved as text files. Listen during your drive instead of having them sit in your “read later” pile forever.

**The privacy-conscious reader**. Sensitive documents like medical records, legal notes, or personal journals. Synthesize and listen locally; your data never touches a server.

**The efficient multitasker**. Listen to meeting notes or documentation while you’re cooking dinner or doing chores. Your brain processes audio while your hands stay free.

**The quick scanner**. Use a small language model to summarize long content first, then have [KittenTTS](https://huggingface.co/KittenML/kitten-tts-mini-0.1) read just the highlights. Perfect for deciding if something’s worth a deep read.

> **Note on document formats**: This system works with plain text. If you have PDFs, Word docs, or other formats, you’ll want to extract the text first. Tools like `pdftotext`, `pandoc`, or Python libraries like `pypdf` work great for this. Many of us keep notes in markdown anyway, and those work perfectly out of the box.

## How It Actually Works

The architecture is surprisingly straightforward.

On the server side (Python + FastAPI), we take your text input, feed it to KittenTTS for synthesis, wrap the audio in a proper WAV format, and stream it chunk by chunk over HTTP.

On the client side (Go), we send your text to the server, receive the WAV header (those critical first 44 bytes), then stream the rest directly to both a file AND your speakers simultaneously. We use Go’s `io.Pipe` to avoid blocking issues.

The beauty is in the simultaneous save-and-play. One network read feeds two destinations, so you’re not choosing between “save for later” or “listen now.” You get both.

## Getting Started

Assuming have Python and Go installed, here’s how to fire things up.

Set up your Python environment:

```powershell
python -m venv kitten_env
.\kitten_env\Scripts\Activate.ps1
pip install -r requirements.txt
```

Start the server (h11 is important for reliable streaming):

```powershell
uvicorn server:app --host 127.0.0.1 --port 8000 --http h11
```

Run the client:

```powershell
go run client.go --save --stream --text "Your document text here"
```

Want to just test if it works? Hit it with curl:

```powershell
curl -X POST "http://127.0.0.1:8000/synthesize" \
  -H "Content-Type: application/json" \
  -d '{"text":"Testing one two three"}' \
  --output test.wav
```

## The Tricky Bits (So You Don’t Have To Learn The Hard Way)

**Problem: Client only gets 44 bytes then stops**

This one burned me. The culprit? The audio player’s `Play()` call was blocking everything. Solution: spin up `Play()` in its own goroutine and feed it via `io.Pipe`. Now the network read can continue while playback happens in parallel.

**Problem: Audio sounds like a robot gargling marbles**

Check your WAV header parameters. Sample rate, channels, and bit depth must match on both server and client. Mismatch equals chaos.

**Problem: Long documents eat all your RAM**

Don’t try to synthesize War and Peace in one go. Chunk it. Process pieces incrementally and stream each one. Your memory (and users) will thank you.

Pro tip for speed: If synthesis feels sluggish, try quantized models (int8) or switch to ONNX Runtime. You’d be surprised how much faster a compact, optimized model can be versus a bloated one.

## The Code (Conceptual Sketch)

Here’s the core idea in the Go client:

```go
// Grab the WAV header first
header := make([]byte, 44)
io.ReadFull(resp.Body, header)
outFile.Write(header)

// Set up pipe for simultaneous save + play
reader, writer := io.Pipe()
multiOut := io.MultiWriter(outFile, writer)

// Start playback in background
go player.Play(reader)

// Stream everything else
io.Copy(multiOut, resp.Body)
writer.Close()
```

That’s it. One read loop, two outputs, zero blocking.

## Why This Matters

Big models are cool, but they’re not always the answer. Sometimes you just need something that works: fast, private, and local. This setup proves you can build genuinely useful tools without cloud dependencies or heavyweight infrastructure.

Your documents, your device, your audio. Simple as that.

## Where I’m Headed: On-Device Podcasts

Here’s what I’m really excited about: turning this into something that feels less like a robotic document reader and more like an actual podcast.

Voice cloning with MelloTTS and OpenVoice lets you have your favorite writer’s articles read in a voice that sounds natural and engaging, or even clone your own voice for personal content. These models can do voice cloning with just a few seconds of reference audio, and they support way more than just English. Spanish technical docs? Japanese research papers? No problem.

The dream setup goes like this: take a long-form article or research paper, run it through a small LLM to restructure it into a conversational format (maybe even a two-host dialogue), then synthesize it with cloned voices. Suddenly your dense technical whitepaper becomes an engaging 20-minute “podcast episode” you can listen to on your morning run.

Picture this conversation:

- Host A: “So the key finding in this paper is…”
- Host B: “Wait, back up. What does that mean for practical applications?”
- Host A: “Great question! Let me break it down…”

All generated locally, all private, all in whatever language you need. The SLM handles the conversational restructuring, the TTS handles making it sound natural, and you get podcast-quality content from any document.

The technical challenges I’m exploring include getting voice cloning to work efficiently on-device (these models can be heavy), structuring prompts so the SLM generates natural dialogue instead of stiff Q&A, handling multiple voices in a single audio stream without glitches, supporting multilingual content smoothly (code-switching between languages mid-episode), and keeping latency reasonable. Nobody wants to wait 10 minutes for their “instant” podcast.

Here’s why this matters: Most podcast summaries or AI-narrated content still rely on cloud services. Your document goes up, audio comes back. But what if you could do all of this on your laptop? Privacy-sensitive content, offline operation, and zero recurring costs. That’s the goal.

## Beyond Text: Multimodal Agentic Workflows

While this project focuses on text-to-speech, I’m also working on something bigger in parallel: [multimodal support for AgenticGokit](https://github.com/AgenticGoKit/AgenticGoKit/tree/multi-model-support). The goal is to enable agentic workflows that go beyond just text. Think images, video, and audio all processed locally.

What this could unlock is pretty exciting. Document understanding where you feed in a PDF with diagrams and charts, and the agent extracts text, analyzes images, and generates an audio summary that references “the graph on page 5.” Video content processing that extracts audio from videos, transcribes it, summarizes key points, and generates a concise audio recap. Visual learning where you take a technical diagram or flowchart and have the agent explain it step-by-step in natural language audio. Meeting recordings where you process video calls to extract both speech and screen shares, then generate searchable summaries with audio playback.

The vision is to build agentic workflows where small models coordinate different modalities. Vision models process images, audio models handle speech, language models tie it all together, and everything runs on your device without external APIs.

This multimodal work is still early stage, but it’s the natural evolution of what started here with streaming TTS. I’ll be sharing progress as these pieces come together.

---

Complete Source code for this demo : <https://github.com/kunalkushwaha/kittentts-example>

Give it a spin and let me know what documents you end up listening to!
