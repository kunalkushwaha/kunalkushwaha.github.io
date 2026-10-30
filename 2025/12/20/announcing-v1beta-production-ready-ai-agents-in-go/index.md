# Announcing v1beta: Production-Ready AI Agents in Go

> We’re excited to announce v1beta, our next-generation API for building AI agents in Go. With streaming-first architecture, multimodal support, and powerful workflow orchestration, building production-ready AI systems has never been easier.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2025-12-20
- Tags: AI Agents, AgenticGoKit, Go, Release
- Canonical: https://kunalkushwaha.github.io/2025/12/20/announcing-v1beta-production-ready-ai-agents-in-go/

![AgenticGoKit v1beta](https://kunalkushwaha.github.io/2025/12/20/announcing-v1beta-production-ready-ai-agents-in-go/beta-release.png)

## TL;DR

**AgenticGoKit v1beta is here (v0.5.0).** Build AI agents in Go with first-class streaming, multimodal input, multi-agent workflows, and support for major LLM providers. Whether you’re building your first chatbot or orchestrating complex agent teams, v1beta keeps it simple.

```bash
go get github.com/agenticgokit/agenticgokit/v1beta
```

---

## Why This Matters

If you’re a Go developer looking to build AI-powered applications, you’ve probably hit these pain points:

- **Python Lock-in**: Most AI frameworks force you into Python, losing Go’s performance and type safety
- **Streaming Complexity**: Real-time token streaming requires complex async code
- **Multi-Agent Chaos**: Coordinating multiple agents means writing custom orchestration
- **Provider Limitations**: Switching between OpenAI, Ollama, or Azure means rewriting code

**v1beta solves all of this.** One unified API with explicit `Run()` and `RunStream()`, built-in orchestration, and seamless provider switching—all with the performance and simplicity you expect from Go.

### What’s in v0.5.0 / v1beta?

- A modern `v1beta` package (evolution of `core/vnext`) with a unified builder
- Streaming-first execution via `RunStream()` for agents and workflows
- Multimodal inputs (images, audio, video) via `RunWithOptions()`
- Workflow orchestration: sequential, parallel, DAG, and loop
- Better errors: typed, actionable, and easier to handle programmatically

> If you’re starting fresh: use `github.com/agenticgokit/agenticgokit/v1beta`.
> If you’re on `core/vnext`: update imports to `v1beta`.

---

## For the Impatient: 60 Seconds to Your First AI Agent

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    "github.com/agenticgokit/agenticgokit/v1beta"
)

func main() {
    // Create agent with Ollama (runs locally)
    agent, err := v1beta.NewBuilder("assistant").
        WithConfig(&v1beta.Config{
            Name:         "assistant",
            SystemPrompt: "You are a helpful coding assistant",
            Timeout:      30 * time.Second,
            LLM: v1beta.LLMConfig{
                Provider: "ollama",
                Model:    "gemma3:1b",
                BaseURL:  "http://localhost:11434",
            },
        }).
        Build()
    if err != nil {
        log.Fatal(err)
    }

    // Run and stream the response
    stream, err := agent.RunStream(context.Background(), 
        "Explain Go channels in 50 words")
    if err != nil {
        log.Fatal(err)
    }

    // Watch it think in real-time
    for chunk := range stream.Chunks() {
        if chunk.Type == v1beta.ChunkTypeDelta {
            fmt.Print(chunk.Delta)
        }
    }
    
    result, _ := stream.Wait()
    fmt.Printf("\n\nDone in %.2fs (%d tokens)\n", 
        result.Duration.Seconds(), result.TokensUsed)
}
```

**That’s it.** Real-time streaming AI in 30 lines of Go. No complex async. No callback hell. Just straightforward, readable code.

---

## The Power of Streaming-First

Traditional AI frameworks bolt streaming on as an afterthought. v1beta is **built so streaming is a first-class path**, not a special-case.

### Watch Your Agents Think

```go
// Traditional: Wait for complete response
ctx := context.Background()
prompt := "..."
result, _ := agent.Run(ctx, prompt)
fmt.Println(result.Content) // User waits for full completion

// v1beta: Stream tokens as they're generated
stream, _ := agent.RunStream(ctx, prompt)
for chunk := range stream.Chunks() {
    fmt.Print(chunk.Delta) // Real-time output
}
```

This isn’t just about user experience—**it’s about debuggability**. When you can see exactly what your agent is generating in real-time, debugging becomes dramatically easier.

---

## Multi-Agent Workflows: From Simple to Sophisticated

Building one agent is easy. Coordinating **teams** of specialized agents? That’s where v1beta shines.

### Sequential Workflow: The Pipeline

Perfect for research, analysis, or content generation pipelines:

```go
// Create specialized agents
researcher, _ := v1beta.NewBuilder("researcher").
    WithConfig(&v1beta.Config{
        Name:         "researcher",
        SystemPrompt: "You are a technical research specialist. Gather facts.",
        LLM: v1beta.LLMConfig{Provider: "ollama", Model: "gemma3:1b"},
    }).Build()

analyzer, _ := v1beta.NewBuilder("analyzer").
    WithConfig(&v1beta.Config{
        Name:         "analyzer", 
        SystemPrompt: "You are a data analyst. Find patterns and insights.",
        LLM: v1beta.LLMConfig{Provider: "ollama", Model: "gemma3:1b"},
    }).Build()

writer, _ := v1beta.NewBuilder("writer").
    WithConfig(&v1beta.Config{
        Name:         "writer",
        SystemPrompt: "You are a technical writer. Create clear documentation.",
        LLM: v1beta.LLMConfig{Provider: "ollama", Model: "gemma3:1b"},
    }).Build()

// Build the pipeline
workflow, _ := v1beta.NewSequentialWorkflow(&v1beta.WorkflowConfig{
    Timeout: 300 * time.Second,
})
workflow.AddStep(v1beta.WorkflowStep{Name: "research", Agent: researcher})
workflow.AddStep(v1beta.WorkflowStep{Name: "analyze", Agent: analyzer})
workflow.AddStep(v1beta.WorkflowStep{Name: "write", Agent: writer})

// Execute and stream the entire workflow
ctx := context.Background()
stream, _ := workflow.RunStream(ctx, "Write a technical guide on Go concurrency")

for chunk := range stream.Chunks() {
    if chunk.Type == v1beta.ChunkTypeDelta {
        // Workflows annotate chunks with step metadata (e.g., "step_name").
        // If not present, fall back to stream-level metadata.
        label := stream.Metadata().AgentName
        if chunk.Metadata != nil {
            if stepName, ok := chunk.Metadata["step_name"].(string); ok && stepName != "" {
                label = stepName
            }
        }
        fmt.Printf("[%s] %s", label, chunk.Delta)
    }
}
```

**Output streaming shows you each agent’s contribution in real-time.** No more black-box workflows.

### Parallel Workflow: Speed Through Concurrency

When tasks are independent, run them concurrently:

```go
parallel, _ := v1beta.NewParallelWorkflow(&v1beta.WorkflowConfig{
    Timeout: 60 * time.Second,
})
parallel.AddStep(v1beta.WorkflowStep{Name: "technical", Agent: techAnalyst})
parallel.AddStep(v1beta.WorkflowStep{Name: "business", Agent: bizAnalyst})
parallel.AddStep(v1beta.WorkflowStep{Name: "security", Agent: secAnalyst})

// All three agents run simultaneously
ctx := context.Background()
stream, _ := parallel.RunStream(ctx, "Analyze this architecture proposal")
```

Lower end-to-end latency for independent tasks, with automatic result aggregation.

### DAG Workflow: Complex Dependencies Made Simple

For sophisticated workflows with dependencies:

```go
dag, _ := v1beta.NewDAGWorkflow(&v1beta.WorkflowConfig{
    Timeout: 120 * time.Second,
})

// Step 1: Code review (no dependencies)
dag.AddStep(v1beta.WorkflowStep{
    Name:  "review",
    Agent: codeReviewer,
})

// Step 2: Security scan (depends on review passing)
dag.AddStep(v1beta.WorkflowStep{
    Name:         "security",
    Agent:        secScanner,
    Dependencies: []string{"review"},
})

// Step 3: Performance analysis (depends on review passing)
dag.AddStep(v1beta.WorkflowStep{
    Name:         "performance",
    Agent:        perfAnalyzer,
    Dependencies: []string{"review"},
})

// Step 4: Final approval (depends on both security and performance)
dag.AddStep(v1beta.WorkflowStep{
    Name:         "approve",
    Agent:        approver,
    Dependencies: []string{"security", "performance"},
})
```

**Automatic dependency resolution** with optimal parallel execution wherever possible.

### Loop Workflow: Iterative Refinement

For tasks that need iteration until meeting quality criteria:

```go
loop, _ := v1beta.NewLoopWorkflow(&v1beta.WorkflowConfig{
    MaxIterations: 5,
    Timeout:       120 * time.Second,
})

loop.AddStep(v1beta.WorkflowStep{Name: "generate", Agent: codeGenerator})
loop.AddStep(v1beta.WorkflowStep{Name: "review", Agent: codeReviewer})

// Keep iterating until code is approved or max iterations reached
// Requires: import "strings"
loop.SetLoopCondition(func(ctx context.Context, iteration int, last *v1beta.WorkflowResult) (bool, error) {
    if last == nil {
        return true, nil
    }

    // LoopConditionFunc returns true to continue looping, false to exit.
    return !strings.Contains(last.FinalOutput, "APPROVED"), nil
})

ctx := context.Background()
stream, _ := loop.RunStream(ctx, "Generate a secure login handler in Go")
```

**Automatic iteration** with custom exit conditions—no manual retry logic needed.

---

## Multimodal: Beyond Text

Modern AI isn’t just about text. v1beta supports **images, audio, and video** as first-class citizens:

```go
opts := v1beta.NewRunOptions()

// Add images (from URL or Base64)
opts.Images = []v1beta.ImageData{
    {
        URL:      "https://example.com/architecture-diagram.png",
        Metadata: map[string]string{"type": "diagram"},
    },
}

// Add audio files
opts.Audio = []v1beta.AudioData{
    {
        URL:    "https://example.com/meeting-recording.mp3",
        Format: "mp3",
    },
}

// Add video
opts.Video = []v1beta.VideoData{
    {
        URL:    "https://example.com/demo.mp4",
        Format: "mp4",
    },
}

result, _ := agent.RunWithOptions(ctx, 
    "Analyze this architecture, summarize the meeting, and review the demo", 
    opts)
```

**Works with GPT-4o-class vision models** (and other multimodal-capable models exposed by your provider)—no special handling required.

> Note: Direct Google Gemini support is not available yet, but is planned.

---

## Provider Freedom: Your Choice, Zero Lock-in

One of v1beta’s superpowers: **switch LLM providers with a config change**:

```go
// Requires: import "os"

// Development: Free local models with Ollama
config := &v1beta.Config{
    LLM: v1beta.LLMConfig{
        Provider: "ollama",
        Model:    "gemma3:1b",
        BaseURL:  "http://localhost:11434",
    },
}

// Production: GPT-4 via OpenAI
config := &v1beta.Config{
    LLM: v1beta.LLMConfig{
        Provider: "openai",
        Model:    "gpt-4",
        APIKey:   os.Getenv("OPENAI_API_KEY"),
    },
}

// Enterprise: Azure OpenAI
config := &v1beta.Config{
    LLM: v1beta.LLMConfig{
        Provider: "azure",
        Model:    "gpt-4",
        BaseURL:  "https://your-resource.openai.azure.com",
        APIKey:   os.Getenv("AZURE_OPENAI_API_KEY"),
    },
}
```

**Supported providers:**

- OpenAI (e.g., GPT-4o / GPT-4.1-class, plus newer reasoning models)
- Azure OpenAI (same model families, enterprise deployment)
- Ollama (local models like Llama 3.x, Gemma, Mistral, Phi, etc.)
- HuggingFace (open-source + hosted inference)
- OpenRouter (multi-provider gateway)
- Any OpenAI-compatible API (self-hosted or third-party)

> Coming soon: direct integrations for **Anthropic (Claude)** and **Google Gemini**.

**Model examples (non-exhaustive):**

- OpenAI: GPT-4o, GPT-4.1, GPT-4-class, newer reasoning models
- Open-source: Llama 3.1/3.2-class, Mistral Large-class, Qwen 2.5-class

**Your code doesn’t change.** Just the config.

---

## Memory & RAG: Context-Aware Agents

Build agents that remember conversations and leverage knowledge bases:

```go
// Requires: import "os"

agent, _ := v1beta.NewBuilder("support-agent").
    WithConfig(&v1beta.Config{
        Name:         "support-agent",
        SystemPrompt: "You are a helpful customer support agent",
        LLM: v1beta.LLMConfig{
            Provider: "openai",
            Model:    "gpt-4",
            APIKey:   os.Getenv("OPENAI_API_KEY"),
        },
    }).
    WithMemory(
        v1beta.WithMemoryProvider("conversation-store"),
        v1beta.WithRAG(4000, 0.3, 0.7), // context size, min/max relevance
    ).
    Build()

// Agent automatically retrieves relevant context from memory
result, _ := agent.Run(ctx, "What was my previous issue about?")
```

**Built-in support** for conversation memory and RAG patterns—no need to wire up vector databases manually (though you can if you want).

---

## Extensibility: Custom Handlers (Bring Your Own Control Loop)

Prompts and presets are great for getting started, but most production systems need more control: input shaping, guardrails, deterministic pre/post processing, retries, and domain-specific logic.

v1beta supports **custom handlers** via `WithHandler()`. A handler gets a `Capabilities` object that can call the configured LLM, tools, and memory.

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/agenticgokit/agenticgokit/v1beta"
)

func main() {
    // Base handler: you fully control the request/response loop.
    base := func(ctx context.Context, input string, c *v1beta.Capabilities) (string, error) {
        system := "You are a strict senior Go reviewer. Reply with concise bullet points."
        return c.LLM(system, input)
    }

    // Compose behaviors: retries, tool augmentation, RAG augmentation, etc.
    handler := v1beta.WithLLMAugmentation(base, 3)

    agent, err := v1beta.NewBuilder("reviewer").
        WithConfig(&v1beta.Config{
            Name: "reviewer",
            LLM: v1beta.LLMConfig{
                Provider: "openai",
                Model:    "gpt-4",
                APIKey:   os.Getenv("OPENAI_API_KEY"),
            },
        }).
        WithHandler(handler).
        Build()
    if err != nil {
        panic(err)
    }

    result, err := agent.Run(context.Background(), "Review this function for correctness and security: ...")
    if err != nil {
        panic(err)
    }

    fmt.Println(result.Content)
}
```

You can also compose handlers using built-in utilities like `Retry(...)`, `WithToolAugmentation(...)`, and `WithRAGAugmentation(...)` to keep your agent logic modular and testable.

---

## Subworkflows: Compose at Scale

The real power move? **Use workflows as agents**:

```go
// Create a parallel research workflow
researchWorkflow, _ := v1beta.NewParallelWorkflow(&v1beta.WorkflowConfig{
    Timeout: 120 * time.Second,
})
researchWorkflow.AddStep(v1beta.WorkflowStep{Name: "web", Agent: webResearcher})
researchWorkflow.AddStep(v1beta.WorkflowStep{Name: "academic", Agent: academicResearcher})
researchWorkflow.AddStep(v1beta.WorkflowStep{Name: "news", Agent: newsResearcher})

// Wrap the entire workflow as a single agent
researchAgent := v1beta.NewSubWorkflowAgent("research-team", researchWorkflow)

// Use it in a larger workflow
mainWorkflow, _ := v1beta.NewSequentialWorkflow(&v1beta.WorkflowConfig{
    Timeout: 300 * time.Second,
})
mainWorkflow.AddStep(v1beta.WorkflowStep{
    Name:  "research",
    Agent: researchAgent, // 3-agent workflow acting as 1 agent!
})
mainWorkflow.AddStep(v1beta.WorkflowStep{Name: "write", Agent: contentWriter})
mainWorkflow.AddStep(v1beta.WorkflowStep{Name: "edit", Agent: editor})
```

**Hierarchical composition** lets you build complex systems from simple, testable components. Each subworkflow can be developed, tested, and versioned independently.

---

## Production-Ready from Day One

v1beta isn’t a toy—it’s built for production:

### Comprehensive Error Handling

```go
// Imports you’ll typically want here:
//   "context"
//   "errors"
//   "log"

result, err := agent.Run(ctx, prompt)
if err != nil {
    // v1beta returns structured errors you can inspect programmatically.
    var agentErr *v1beta.AgentError
    if errors.As(err, &agentErr) {
        log.Printf("agent error: code=%s message=%s details=%v", agentErr.Code, agentErr.Message, agentErr.Details)
        return
    }

    if errors.Is(err, context.DeadlineExceeded) {
        log.Printf("timed out: %v", err)
        return
    }

    log.Printf("unexpected error: %v", err)
    return
}

_ = result
```

**Typed errors** with actionable context—no more guessing what went wrong.

### Timeouts and Circuit Breaking

```go
config := &v1beta.Config{
    Timeout: 30 * time.Second, // Overall agent timeout
    Tools: &v1beta.ToolsConfig{
        Enabled:    true,
        MaxRetries: 3,
        Timeout:    10 * time.Second,
        CircuitBreaker: &v1beta.CircuitBreakerConfig{
            Enabled:          true,
            FailureThreshold: 5,
            SuccessThreshold: 2,
            Timeout:          30 * time.Second,
            HalfOpenMaxCalls: 1,
        },
    },
}
```

### Configuration Validation

```go
config := &v1beta.Config{
    Name: "", // Invalid!
    LLM:  v1beta.LLMConfig{Provider: "unknown"}, // Invalid!
}

agent, err := v1beta.NewBuilder("test").WithConfig(config).Build()
// err contains detailed validation errors
```

**Fail fast** with clear error messages before anything runs.

---

## Real-World Example: AI Code Reviewer

Here’s a complete example combining multiple features:

```go
package main

import (
    "context"
    "fmt"
    "os"
    "time"
    
    "github.com/agenticgokit/agenticgokit/v1beta"
)

func main() {
    // Security reviewer
    secAgent, _ := v1beta.NewBuilder("security").
        WithConfig(&v1beta.Config{
            Name:         "security",
            SystemPrompt: "You are a security expert. Find vulnerabilities.",
            LLM: v1beta.LLMConfig{Provider: "openai", Model: "gpt-4", APIKey: os.Getenv("OPENAI_API_KEY")},
        }).Build()

    // Performance reviewer
    perfAgent, _ := v1beta.NewBuilder("performance").
        WithConfig(&v1beta.Config{
            Name:         "performance",
            SystemPrompt: "You are a performance expert. Find bottlenecks.",
            LLM: v1beta.LLMConfig{Provider: "openai", Model: "gpt-4", APIKey: os.Getenv("OPENAI_API_KEY")},
        }).Build()

    // Best practices reviewer
    practicesAgent, _ := v1beta.NewBuilder("practices").
        WithConfig(&v1beta.Config{
            Name:         "practices",
            SystemPrompt: "You are a Go expert. Check best practices.",
            LLM: v1beta.LLMConfig{Provider: "openai", Model: "gpt-4", APIKey: os.Getenv("OPENAI_API_KEY")},
        }).Build()

    // Summarizer
    summaryAgent, _ := v1beta.NewBuilder("summarizer").
        WithConfig(&v1beta.Config{
            Name:         "summarizer",
            SystemPrompt: "Synthesize review feedback into actionable items.",
            LLM: v1beta.LLMConfig{Provider: "openai", Model: "gpt-4", APIKey: os.Getenv("OPENAI_API_KEY")},
        }).Build()

    // Step 1: Parallel review (3 agents at once)
    reviewWorkflow, _ := v1beta.NewParallelWorkflow(&v1beta.WorkflowConfig{
        Timeout: 60 * time.Second,
    })
    reviewWorkflow.AddStep(v1beta.WorkflowStep{Name: "security", Agent: secAgent})
    reviewWorkflow.AddStep(v1beta.WorkflowStep{Name: "performance", Agent: perfAgent})
    reviewWorkflow.AddStep(v1beta.WorkflowStep{Name: "practices", Agent: practicesAgent})

    // Step 2: Sequential pipeline (review -> summarize)
    mainWorkflow, _ := v1beta.NewSequentialWorkflow(&v1beta.WorkflowConfig{
        Timeout: 300 * time.Second,
    })
    reviewAgent := v1beta.NewSubWorkflowAgent("reviewers", reviewWorkflow)
    mainWorkflow.AddStep(v1beta.WorkflowStep{Name: "review", Agent: reviewAgent})
    mainWorkflow.AddStep(v1beta.WorkflowStep{Name: "summarize", Agent: summaryAgent})

    // Execute with streaming
    codeToReview := `
    func ProcessPayment(amount float64) error {
        db.Exec("UPDATE accounts SET balance = balance - " + fmt.Sprintf("%.2f", amount))
        return nil
    }
    `

    stream, _ := mainWorkflow.RunStream(context.Background(), 
        "Review this Go code:\n"+codeToReview)

    fmt.Println("AI Code Review in Progress...\n")
    for chunk := range stream.Chunks() {
        if chunk.Type == v1beta.ChunkTypeDelta {
            label := stream.Metadata().AgentName
            if chunk.Metadata != nil {
                if stepName, ok := chunk.Metadata["step_name"].(string); ok && stepName != "" {
                    label = stepName
                }
            }
            fmt.Printf("[%s] %s", label, chunk.Delta)
        }
    }

    result, _ := stream.Wait()
    fmt.Printf("\n\nReview complete in %.2fs\n", result.Duration.Seconds())
}
```

**This example demonstrates:**

- Parallel execution (3 reviewers at once)
- Subworkflows (review team as single agent)
- Sequential pipeline (review → summarize)
- Real-time streaming output
- Production-ready error handling

---

## Getting Started

### Installation

```bash
go get github.com/agenticgokit/agenticgokit/v1beta
```

### Quick Start with Ollama (Local, Free)

1. **Install Ollama**: [ollama.ai](https://ollama.ai)
2. **Pull a model**: `ollama pull gemma3:1b`
3. **Run the example**: See “60 Seconds to Your First AI Agent” above

### Quick Start with OpenAI

```go
// Requires: import "os"

agent, _ := v1beta.NewBuilder("assistant").
    WithConfig(&v1beta.Config{
        Name:         "assistant",
        SystemPrompt: "You are a helpful assistant",
        LLM: v1beta.LLMConfig{
            Provider: "openai",
            Model:    "gpt-4",
            APIKey:   os.Getenv("OPENAI_API_KEY"),
        },
    }).
    Build()
```

---

## Learning Resources

### Documentation

- [v1beta API Reference](https://github.com/kunalkushwaha/agenticgokit/blob/main/v1beta/README.md)
- [Migration Guide](https://github.com/kunalkushwaha/agenticgokit/blob/main/docs/MIGRATION.md)
- [Full Documentation](https://docs.agenticgokit.com)

### Examples

- [19+ Working Examples](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples)
- [Story Writer Chat v2](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/story-writer-chat-v2) - Real-time multi-agent workflow
- [Streaming Workflow](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/streaming_workflow) - Advanced streaming patterns
- [MCP Integration](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/mcp-integration) - Tool integration

### Provider Quickstarts

- [Ollama Quickstart](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/ollama-quickstart)
- [HuggingFace Quickstart](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/huggingface-quickstart)
- [OpenRouter Quickstart](https://github.com/kunalkushwaha/agenticgokit/tree/main/examples/openrouter-quickstart)

---

## What’s Next: The Road to v1.0

v1beta is **stable and production-ready**, but we’re not stopping here. Based on community feedback, we’re working towards v1.0 with:

- **Tool registry upgrades** - Rich tool schemas, lifecycle management (register/unregister), and smoother discovery across internal + MCP tools
- **Advanced RAG patterns** - Deeper retrieval pipelines and more native vector DB integrations (pgvector today; expanding beyond)
- **Turn-key production observability** - Built-in metrics/tracing exports and better correlation (run/workflow/tool IDs) out of the box
- **Performance optimizations** - Even faster execution and lower latency
- **New CLI tool** (`agk`) - Scaffolding, testing, and deployment helpers
- **Workflow pattern recipes** - Higher-level helpers and examples for routing/branching/conditionals on top of existing sequential/parallel/DAG/loop workflows

**Your feedback shapes the future.** Join us:

- [GitHub Discussions](https://github.com/kunalkushwaha/agenticgokit/discussions)
- [Issues & Feature Requests](https://github.com/kunalkushwaha/agenticgokit/issues)
- [Contributing Guide](https://github.com/kunalkushwaha/agenticgokit/blob/main/docs/contributors/ContributorGuide.md)

---

## Why Now?

AI agent frameworks have matured rapidly in 2025. The patterns are proven. The use cases are clear. What’s been missing is a **Go-native solution** that matches the language’s philosophy:

- **Simple, not easy**: Powerful features without magic
- **Explicit over implicit**: Clear control flow and error handling
- **Performance matters**: Compiled binaries, efficient concurrency
- **Production-ready**: Type safety, comprehensive testing, real-world deployment

**v1beta delivers all of this.** Whether you’re building your first chatbot or orchestrating dozens of specialized agents, you get the same clean, performant API.

---

## Try It Today

```bash
# Install
go get github.com/agenticgokit/agenticgokit/v1beta

# Run a complete example
git clone https://github.com/kunalkushwaha/agenticgokit.git
cd agenticgokit/examples/ollama-quickstart
go run .
```

**Join the community:**

- [Star on GitHub](https://github.com/kunalkushwaha/agenticgokit)
- [Join Discussions](https://github.com/kunalkushwaha/agenticgokit/discussions)
- [Report Issues](https://github.com/kunalkushwaha/agenticgokit/issues)
- [Read the Docs](https://docs.agenticgokit.com)

---

## Final Thoughts

Building AI agents should feel like building any other Go application: **straightforward, type-safe, and efficient**. v1beta makes that vision a reality.

Whether you’re experimenting with local models via Ollama, deploying production services with GPT-4, or building complex multi-agent systems—**you’re writing the same clean Go code**.

We can’t wait to see what you build.

Happy coding.

---

*AgenticGoKit is Apache 2.0 licensed and welcomes contributions from the community.*
