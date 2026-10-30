# Building Multi-Agent Workflows in Go: Simpler Than You Think

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2025-10-20
- Tags: AI Agents, AgenticGoKit, Go, Workflows
- Canonical: https://kunalkushwaha.github.io/2025/10/20/building-multi-agent-workflows/

**TL;DR**: Create a professional research-to-report pipeline with two AI agents in under 60 lines of Go code.

---

## The Challenge

You want to build an AI system that:

1. Researches a topic thoroughly
2. Synthesizes findings into a professional report
3. Handles the orchestration automatically

Traditionally, this means wrestling with:

- Complex state management
- Manual prompt chaining
- Error handling across multiple LLM calls
- Coordinating agent communication

**What if it could be as simple as this?**

```go
researcher := createAgent("researcher", "Gather key facts...", 0.7)
reporter := createAgent("reporter", "Create a report...", 0.5)

workflow.AddStep(vnext.WorkflowStep{Name: "research", Agent: researcher})
workflow.AddStep(vnext.WorkflowStep{Name: "report", Agent: reporter})

result := workflow.Run(ctx, "What is Kubernetes?")
```

Spoiler: **It is.**

---

## Introducing AgenticGoKit vNext

AgenticGoKit is a Go framework for building AI agent systems. The new **vNext API** makes multi-agent workflows ridiculously simple.

Let’s build a **Researcher → Reporter** pipeline that takes any topic and produces a professional report.

---

## The Complete Solution (Under 60 Lines!)

Here’s the entire program:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    "github.com/kunalkushwaha/agenticgokit/core/vnext"
    _ "github.com/kunalkushwaha/agenticgokit/plugins/llm/ollama"
)

func main() {
    ctx := context.Background()
    
    // Step 1: Create two specialized agents
    researcher := createAgent("researcher", 
        "Gather key facts and information about the topic.", 0.7)
    reporter := createAgent("reporter", 
        "Create a concise report with Summary, Key Points, and Conclusion.", 0.5)
    
    // Step 2: Create a sequential workflow
    workflow, _ := vnext.NewSequentialWorkflow(&vnext.WorkflowConfig{
        Mode:    vnext.Sequential,
        Timeout: 600 * time.Second,
    })
    
    // Step 3: Add agents as workflow steps
    workflow.AddStep(vnext.WorkflowStep{Name: "research", Agent: researcher})
    workflow.AddStep(vnext.WorkflowStep{
        Name:  "report",
        Agent: reporter,
        Transform: func(input string) string {
            return "Based on this research, create a professional report:\n\n" + input
        },
    })
    
    // Step 4: Run the workflow!
    workflow.Initialize(ctx)
    defer workflow.Shutdown(ctx)
    
    result, _ := workflow.Run(ctx, "What is Kubernetes?")
    
    // Display results
    fmt.Println("🔍 RESEARCH FINDINGS:")
    fmt.Println(result.StepResults[0].Output)
    
    fmt.Println("\n📄 FINAL REPORT:")
    fmt.Println(result.StepResults[1].Output)
    
    fmt.Printf("\n✅ Completed in %v\n", result.Duration)
}

// Helper to create an agent
func createAgent(name, prompt string, temperature float32) vnext.Agent {
    agent, err := vnext.NewBuilder(name).
        WithConfig(&vnext.Config{
            Name:         name,
            SystemPrompt: prompt,
            LLM: vnext.LLMConfig{
                Provider:    "ollama",
                Model:       "gemma3:1b",
                Temperature: temperature,
                MaxTokens:   400,
            },
            Timeout: 600 * time.Second,
        }).
        Build()
    
    if err != nil {
        log.Fatalf("Failed to create %s: %v", name, err)
    }
    
    agent.Initialize(context.Background())
    return agent
}
```

**That’s it.** Under 60 lines. No complex orchestration. No manual state management.

---

## Breaking It Down

### Step 1: Create Specialized Agents

```go
researcher := createAgent("researcher", 
    "Gather key facts and information about the topic.", 0.7)
reporter := createAgent("reporter", 
    "Create a concise report with Summary, Key Points, and Conclusion.", 0.5)
```

Each agent is configured for its specific role:

- **Researcher**: Higher temperature (0.7) for creative exploration
- **Reporter**: Lower temperature (0.5) for focused, structured output

The `createAgent()` helper encapsulates all the configuration:

- Connects to your LLM provider (Ollama, OpenAI, Azure, etc.)
- Sets the system prompt to define the agent’s role
- Configures temperature, token limits, timeouts

**One function call. One agent. Done.**

### Step 2: Create the Workflow

```go
workflow, _ := vnext.NewSequentialWorkflow(&vnext.WorkflowConfig{
    Mode:    vnext.Sequential,
    Timeout: 600 * time.Second,
})
```

AgenticGoKit supports multiple workflow patterns:

- **Sequential**: Steps run in order (what we’re using)
- **Parallel**: Steps run concurrently
- **DAG**: Complex dependency graphs
- **Loop**: Iterative processing

We chose sequential because we want: Research → Report

### Step 3: Add Your Agents as Steps

```go
workflow.AddStep(vnext.WorkflowStep{Name: "research", Agent: researcher})
workflow.AddStep(vnext.WorkflowStep{
    Name:  "report",
    Agent: reporter,
    Transform: func(input string) string {
        return "Based on this research, create a professional report:\n\n" + input
    },
})
```

This is where the magic happens:

1. **First step**: Researcher agent processes the input topic
2. **Transform function**: Reformats the research output into a prompt for the reporter
3. **Second step**: Reporter agent creates the final report

The `Transform` function is your secret weapon for **prompt engineering between steps**. It takes the output from one agent and shapes it perfectly for the next.

### Step 4: Run It!

```go
workflow.Initialize(ctx)
defer workflow.Shutdown(ctx)

result, _ := workflow.Run(ctx, "What is Kubernetes?")

fmt.Println(result.StepResults[0].Output)  // Research findings
fmt.Println(result.StepResults[1].Output)  // Final report
```

The workflow:

- ✅ Manages execution order
- ✅ Handles errors gracefully
- ✅ Passes data between steps
- ✅ Returns all results in one structure

You get access to:

- Individual step outputs
- Overall success/failure status
- Timing information
- Token usage

---

## What You Get

Running this with the topic **“What is Kubernetes and why is it important?”** produces:

```
🔍 RESEARCH FINDINGS:
Kubernetes is an open-source container orchestration platform that automates 
the deployment, scaling, and management of containerized applications. It was 
originally developed by Google and is now maintained by the Cloud Native 
Computing Foundation (CNCF).

Key aspects of Kubernetes:
• Automates container deployment across clusters
• Provides self-healing capabilities
• Enables horizontal scaling
• Supports rolling updates and rollbacks
• Manages service discovery and load balancing

📄 FINAL REPORT:

Summary:
Kubernetes is a powerful container orchestration system that has become the 
industry standard for deploying and managing cloud-native applications.

Key Points:
• Automates deployment and scaling of containerized applications
• Provides self-healing and resilience capabilities
• Enables microservices architecture at scale
• Supported by all major cloud providers
• Open-source with strong community backing

Conclusion:
Kubernetes has become essential infrastructure for modern cloud applications, 
enabling teams to build scalable, resilient systems with reduced operational 
complexity.

✅ Completed in 4.2s
```

**Professional results. Minimal code.**

---

## Extending the Workflow

Want to add more sophistication? It’s just as simple:

### Add a Third Agent (Editor)

```go
editor := createAgent("editor", "Proofread and improve the report", 0.3)
workflow.AddStep(vnext.WorkflowStep{Name: "edit", Agent: editor})
```

### Add Memory

```go
researcher := vnext.NewBuilder("researcher").
    WithConfig(config).
    WithMemory().  // Now remembers past interactions!
    Build()
```

### Add Conditional Logic

```go
workflow.AddStep(vnext.WorkflowStep{
    Name:  "summary",
    Agent: summarizer,
    Condition: func(ctx context.Context, wfCtx *vnext.WorkflowContext) bool {
        // Only summarize if research is lengthy
        return len(wfCtx.GetStepResult("research").Output) > 1000
    },
})
```

### Use Different Models per Agent

```go
// Heavy research with large model
researcher.LLM.Model = "llama3.1:8b"

// Fast reporting with small model  
reporter.LLM.Model = "gemma3:1b"
```

---

## Getting Started

### Prerequisites

1. **Install Go 1.23+**
2. **Install Ollama** (or use OpenAI/Azure)

   ```bash
# macOS/Linux
curl https://ollama.ai/install.sh | sh

# Pull a model
ollama pull gemma3:1b
```
3. **Install AgenticGoKit**

   ```bash
go get github.com/kunalkushwaha/agenticgokit
```

### Run the Example

```bash
git clone https://github.com/kunalkushwaha/AgenticGoKit
cd AgenticGoKit/examples/vnext/researcher-reporter
go run main.go
```

### Customize for Your Use Case

Change the topic:

```go
result, _ := workflow.Run(ctx, "Your topic here")
```

Change the agents:

```go
analyst := createAgent("analyst", "Analyze market trends", 0.6)
strategist := createAgent("strategist", "Create strategy recommendations", 0.5)
```

Change the workflow:

```go
workflow, _ := vnext.NewParallelWorkflow(config)  // Run steps concurrently
// or
workflow, _ := vnext.NewDAGWorkflow(config)      // Complex dependencies
```

---

## The Bottom Line

Multi-agent AI systems don’t have to be complex. With AgenticGoKit vNext:

✅ **Under 60 lines of code** instead of 200+  
✅ **4 simple steps** instead of complex orchestration  
✅ **Production-ready** error handling and state management  
✅ **Flexible** - extend with memory, tools, streaming, and more  
✅ **Type-safe** Go with excellent IDE support

### The Philosophy

> **“Simple things should be simple. Complex things should be possible.”**

AgenticGoKit makes the simple case trivial (as you’ve seen), while still supporting:

- RAG with vector databases
- MCP tool integration
- Custom memory providers
- Complex DAG workflows
- And much more…

---

## Try It Yourself

The complete example is available in the [AgenticGoKit repository](https://github.com/kunalkushwaha/AgenticGoKit/tree/main/examples/vnext/researcher-reporter).

**What will you build?**

- Research assistants?
- Content pipelines?
- Data analysis workflows?
- Something entirely new?

Share your creations! We’d love to see what you build.

---

## Resources

- **GitHub**: [kunalkushwaha/AgenticGoKit](https://github.com/kunalkushwaha/AgenticGoKit)
- **Documentation**: [Full vNext Guide](https://github.com/kunalkushwaha/AgenticGoKit/tree/master/docs)
- **Examples**: [More vNext Examples](https://github.com/kunalkushwaha/AgenticGoKit/tree/master/examples/vnext)
- **Community**: [Discussions](https://github.com/kunalkushwaha/AgenticGoKit/discussions)

---

**Questions?** Open an issue or start a discussion on GitHub.

**Want to contribute?** PRs are always welcome!

---

## Appendix: Quick Reference

### Agent Creation

```go
agent, err := vnext.NewBuilder(name).
    WithConfig(&vnext.Config{
        Name:         name,
        SystemPrompt: prompt,
        LLM: vnext.LLMConfig{
            Provider:    "ollama",
            Model:       "gemma3:1b",
            Temperature: 0.7,
            MaxTokens:   400,
        },
        Timeout: 600 * time.Second,
    }).
    Build()
```

### Workflow Types

```go
vnext.NewSequentialWorkflow(config)  // Steps in order
vnext.NewParallelWorkflow(config)    // Steps concurrently  
vnext.NewDAGWorkflow(config)         // Dependency graph
vnext.NewLoopWorkflow(config)        // Iterative execution
```

### Workflow Step

```go
vnext.WorkflowStep{
    Name:      "step-name",
    Agent:     agent,
    Transform: func(input string) string { return modified },
    Condition: func(ctx, wfCtx) bool { return true },
}
```

### Result Access

```go
result.Success              // bool
result.FinalOutput          // string  
result.StepResults[i]       // Individual step results ([]StepResult)
result.Duration             // time.Duration
result.TotalTokens          // int
result.ExecutionPath        // []string - order of executed steps
```

---

**Happy building!** 🚀
