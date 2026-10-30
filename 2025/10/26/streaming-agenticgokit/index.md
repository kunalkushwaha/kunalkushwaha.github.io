# Streaming AI Responses in Go with AgenticGoKit

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2025-10-26
- Tags: AI Agents, AgenticGoKit, Go, Streaming
- Canonical: https://kunalkushwaha.github.io/2025/10/26/streaming-agenticgokit/

Streaming makes AI feel alive—tokens show up instantly, long tasks feel responsive, and multi‑step workflows become explainable as they run. In this post, we’ll build two streaming experiences with AgenticGoKit:

- A minimal “simple-streaming” chat
- A sequential multi‑agent “streaming\_workflow” with step-by-step progress

We’ll also cover when to use streaming, why it helps, and a few gotchas and tips.

## What is streaming and why it matters

Instead of waiting for the full response, streaming lets you consume output as it’s generated (token‑by‑token or chunk‑by‑chunk). That enables:

- Real‑time feedback: Users see progress immediately
- Better UX for long tasks: No “blank screen” pause
- Step visibility in workflows: Know which agent/step is running
- Early assessment: Skim partial output and course‑correct sooner

Under the hood, AgenticGoKit vNext exposes a Stream you can iterate over, with multiple chunk types like text deltas, metadata, tool calls, and final completion signals.

## Prerequisites

- Go installed and working in this repo
- Ollama running locally (default http://localhost:11434)
- Model: gemma3:1b
- Repo paths in this post are relative to the project root

Pull the model if needed:

```powershell
ollama pull gemma3:1b
```

Alternatively, you can use OpenAI or Azure OpenAI instead of Ollama by setting API keys and pointing your agent config to those providers:

```powershell
# OpenAI
$env:OPENAI_API_KEY = "<your-openai-api-key>"

# Azure OpenAI
$env:AZURE_OPENAI_KEY = "<your-azure-openai-key>"
# Your Azure Base URL typically looks like:
# https://<your-resource-name>.openai.azure.com/
```

For a deeper dive into APIs and options, see `core/vnext/STREAMING_GUIDE.md`.

## Part 1: Minimal simple-streaming

The example lives here: [examples/vnext/simple-streaming/main.go](https://github.com/AgenticGoKit/AgenticGoKit/blob/master/examples/vnext/simple-streaming/main.go).

It creates a small chat agent and prints tokens as they arrive:

```go
// Start streaming
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

stream, err := agent.RunStream(ctx, prompt)
if err != nil {
    log.Fatalf("Failed to start streaming: %v", err)
}

// Print tokens as they arrive
for chunk := range stream.Chunks() {
    if chunk.Error != nil { fmt.Printf("\n❌ Error: %v\n", chunk.Error); break }
    switch chunk.Type {
    case vnext.ChunkTypeDelta:
        fmt.Print(chunk.Delta) // token-by-token
    case vnext.ChunkTypeDone:
        fmt.Println("\n\n✅ Streaming completed!")
    }
}

// Always check the final result
_, _ = stream.Wait()
```

Run it:

```powershell
cd examples/vnext/simple-streaming
go run .
```

You’ll see something like:

![Simple streaming (terminal)](https://kunalkushwaha.github.io/2025/10/26/streaming-agenticgokit/simple-streaming.gif)

Notes:

- The example uses Ollama with `gemma3:1b`. Adjust the model/provider in the config if needed.
- Always call `stream.Wait()` after consuming chunks to surface any trailing errors.

## Part 2: Multi‑agent streaming workflow

Now, let’s level up with a sequential, two‑agent workflow that streams each step in real time. Code: [examples/vnext/streaming\_workflow/main.go](https://github.com/AgenticGoKit/AgenticGoKit/tree/master/examples/vnext/streaming_workflow/main.go).

We’ll create two specialized agents—Researcher and Summarizer—and wire them into a vNext Workflow. Each step streams its own tokens and emits metadata so you know what’s happening.

### Defining agents

```go
func CreateResearcherAgent() (vnext.Agent, error) {
    return vnext.QuickChatAgentWithConfig("Researcher", &vnext.Config{
        Name:         "researcher",
        SystemPrompt: "You are a Research Agent...",
        Timeout:      60 * time.Second,
        LLM: vnext.LLMConfig{ Provider: "ollama", Model: "gemma3:1b", Temperature: 0.2, MaxTokens: 300, BaseURL: "http://localhost:11434" },
    })
}

func CreateSummarizerAgent() (vnext.Agent, error) {
    return vnext.QuickChatAgentWithConfig("Summarizer", &vnext.Config{
        Name:         "summarizer",
        SystemPrompt: "You are a Summarizer Agent...",
        Timeout:      60 * time.Second,
        LLM: vnext.LLMConfig{ Provider: "ollama", Model: "gemma3:1b", Temperature: 0.3, MaxTokens: 150, BaseURL: "http://localhost:11434" },
    })
}
```

#### Using OpenAI or Azure OpenAI

Where the Ollama LLM config is defined above, you can swap in OpenAI or Azure OpenAI with minimal changes:

```go
// OpenAI
LLM: vnext.LLMConfig{
    Provider: "openai",
    Model:    "gpt-4",
    APIKey:   os.Getenv("OPENAI_API_KEY"),
}

// Azure OpenAI
LLM: vnext.LLMConfig{
    Provider: "azure",
    Model:    "gpt-4",
    BaseURL:  "https://your-resource.openai.azure.com/",
    APIKey:   os.Getenv("AZURE_OPENAI_KEY"),
}
```

Notes:

- Keep the rest of the streaming code exactly the same; provider selection is handled via `LLMConfig`.
- Ensure the appropriate environment variables are set in your shell before running the examples.

### Building a sequential workflow

```go
workflow, err := vnext.NewSequentialWorkflow(&vnext.WorkflowConfig{ Mode: vnext.Sequential, Timeout: 180 * time.Second })
if err != nil { log.Fatal(err) }

_ = workflow.AddStep(vnext.WorkflowStep{
    Name:  "research",
    Agent: researcher,
    Transform: func(input string) string {
        return fmt.Sprintf("Research the topic: %s. Provide key information, benefits, and current applications.", input)
    },
})

_ = workflow.AddStep(vnext.WorkflowStep{
    Name:  "summarize",
    Agent: summarizer,
    Transform: func(input string) string {
        return fmt.Sprintf("Please summarize this research into key points:\n\n%s", input)
    },
})
```

### Running with streaming

```go
ctx := context.Background()
stream, err := workflow.RunStream(ctx, topic)
if err != nil { log.Fatal(err) }

for chunk := range stream.Chunks() {
    switch chunk.Type {
    case vnext.ChunkTypeMetadata:
        if stepName, ok := chunk.Metadata["step_name"].(string); ok {
            fmt.Printf("\n🔄 [STEP: %s] %s\n", strings.ToUpper(stepName), chunk.Content)
            fmt.Println("─────────────────────")
        }
    case vnext.ChunkTypeDelta:
        fmt.Print(chunk.Delta)
    case vnext.ChunkTypeDone:
        fmt.Println("\n✅ Workflow step completed!")
    }
}

result, err := stream.Wait() // final success/error
```

Run it:

```powershell
cd examples/vnext/streaming_workflow
go run .
```

What you’ll see:

```
🚀 vnext.Workflow Streaming Showcase
====================================

🔍 Testing Ollama connection...
✅ Ollama connection successful

🌟 vnext.Workflow Sequential Streaming
=====================================
🎯 Topic: Benefits of streaming in AI applications

💬 Real-time Workflow Streaming:
─────────────────────────────────

🔄 [STEP: RESEARCH] Step 1/2: research
Streaming is a really cool way to access content...

🔄 [STEP: SUMMARIZE] Step 2/2: summarize
Based on the research findings, here are the key points:

🎉 vnext.WORKFLOW STREAMING COMPLETED!
```

## When to use streaming vs. non‑streaming

Without streaming:

```
User: "Research AI streaming benefits"
System: [Working... 60–90s of silence]
System: [Full results appear all at once]
```

With streaming:

```
User: "Research AI streaming benefits"
System: 🔄 [STEP: RESEARCH] … tokens stream live …
System: 🔄 [STEP: SUMMARIZE] … tokens stream live …
System: ✅ Workflow completed
```

Streaming shines when:

- The task takes more than ~1–2 seconds
- You want visibility into multi‑step progress
- You’re building chat UIs or CLIs where responsiveness matters

## Tips, options, and best practices

- Always use contexts with timeouts for cancellation:

  ```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```
- After consuming chunks, call `stream.Wait()` to catch final errors and access the final result
- Need only text? Use text‑only streaming to reduce noise
- For UIs, include metadata to display current step/agent
- Tune buffer size and flush intervals for your UX/perf needs

See `core/vnext/STREAMING_GUIDE.md` for:

- Chunk types (Text, Delta, Thought, ToolCall, ToolResult, Metadata, Error, Done)
- Stream options: buffer size, thoughts/tool calls, metadata, flush interval
- Utilities: `CollectStream`, `PrintStream`, `StreamToChannel`, `AsReader`

## Troubleshooting

- Stream hangs or never finishes: use a context with timeout and ensure you read all chunks
- Missing output: verify you’re handling `ChunkTypeDelta` (token deltas) and/or `ChunkTypeText`
- Slow UI updates: try a larger buffer or longer flush interval
- Provider issues: confirm Ollama is running and the model is pulled

## Wrap‑up

You now have two paths:

- Start simple with `examples/vnext/simple-streaming` to understand token streaming
- Build richer, explainable systems with `examples/vnext/streaming_workflow`

Both rely on the same Stream primitives, so once you’re comfortable with one, the other feels natural.

If you want to go deeper, open `core/vnext/STREAMING_GUIDE.md` and explore advanced options like tool‑call streaming, thought visibility, and custom stream builders.
