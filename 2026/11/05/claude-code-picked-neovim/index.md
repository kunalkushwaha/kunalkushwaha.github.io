# I Asked Claude Code to Build Me a Lighter VS Code. It Picked Neovim.

> VS Code used 6.7 GB across 10 windows on a machine that also runs local LLMs and VMs. Claude Code rebuilt the same workflow in Neovim, offline by default, for under 1 GB of editor memory.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2026-11-05
- Tags: Claude Code, Neovim, AI-assisted development, developer tools, local LLMs
- Canonical: https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/

![Claude Code, in Neovim. Same workflow, lighter editor: file tree, code and Claude side by side, with editor memory per window dropping from about 670 MB in VS Code to about 50 to 85 MB in Neovim.](https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/hero.png)

My editor was using **6.7 GB of RAM**.

That's VS Code with 10 windows open: just the windows, extension hosts and Electron helpers, not the language servers or Claude. My machine also runs local LLMs, a few VMs and a pile of containers. That's memory my models couldn't have.

I wanted it back without losing the one thing I use VS Code for. Claude Code works in auto mode, and I watch its changes land: file tree on one side, the changed file in the middle, Claude on the other.

So I asked Claude Code to build me something lighter. It picked Neovim, and *why* it picked it is the interesting part.

## What I couldn't give up

The plain Claude Code terminal is great for talking to Claude, but changes scroll past in a chat log. I want the tree, the file and the changed lines in front of me.

![What I needed to keep: VS Code, Helix and Neovim compared](https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/diagram-review-loop.svg)

I tried Zed, Helix and a few others. Each is good at what it does. None kept the workflow.

Helix shows the gap best. It's fast and lovely to edit in, but it doesn't reload files changed on disk (a long-standing open request), so every Claude edit meant pressing a key to see it. Diff review only happens in the terminal, and there's no file tree beside the code.

## The deciding factor was a protocol

Claude tried Helix first, showed me where it fell short, then suggested Neovim for one reason: [claudecode.nvim](https://github.com/coder/claudecode.nvim) speaks **the same protocol as the VS Code extension**.

So everything the extension gives me in VS Code works the same way in Neovim. When I want to look closer at a change, Claude's edits open as a real side-by-side diff: `:w` accepts, `:q` rejects.

Plenty of editors are lighter than VS Code. Neovim was the one that spoke the language the workflow runs on. The workflow was portable because the protocol was.

![Neovim with file tree, formatted Markdown and Claude Code side by side](https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/screenshot.png)

## The numbers, honestly

One snapshot on one Mac, same day, both running side by side: VS Code with 10 windows, Neovim with 4 sessions. Not a controlled benchmark.

|  | VS Code | Neovim |
| --- | --- | --- |
| Editor memory per window | ≈670 MB | ≈50–85 MB |
| Processes per window | ≈5 | 2 |
| Idle CPU | 3.4% across all 10 windows | 0.1–1.4% per session |

![Editor memory dropped about 10x; total memory about 2x](https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/diagram-memory.svg)

Scaled to 10 projects, that's roughly **6.7 GB down to 0.5–0.85 GB** of editor memory, about **6 GB back**.

But the editor was never the heavy part. Claude Code and the language servers cost the same in both, and gopls alone is about 1 GB per open Go project. Total memory dropped by about half, not to a tenth. VS Code's figure also slightly overstates, because Electron helpers share some memory.

Half was still the difference between my local LLM fitting comfortably and not.

## What I ended up with

| I wanted | What it does |
| --- | --- |
| My usual layout | `nvim .` opens tree left, code middle, Claude right. Cmd+\\ swaps sides; Claude keeps running. |
| See changes land | Every change, from Claude, a shell command, a formatter or git, shows up within about a second, with change bars in the margin. Diff review is there when I switch out of auto mode. |
| VS Code muscle memory | Cmd+C/V/S/Z, Cmd+P, Cmd+Shift+F, Cmd+B, Cmd+/. Each paste is its own undo step. Dark Modern colours, tabs, breadcrumbs, a right-click menu, and Space shows every shortcut. |
| Many projects at once | Ghostty tab title is repo plus worktree. A picker lists every git worktree, including the ones Claude Code creates. |
| My languages | Go, Python, Swift, TypeScript, Rust, HTML/CSS/JSON, with definitions, hover, completion, inline errors, format on save. |
| Markdown done properly | Formatted in the editor, Mermaid drawn inline as text, no Chromium. Live browser preview for the rest. |

The VS Code feel mattered more than I expected. Vim normally lumps a whole typing session into one undo, and that tripped me up immediately.

## Offline by default

I was swapping in a pile of third-party plugins, so I set one hard rule: **nothing in the editor sends data anywhere.**

![Offline by default: every automatic download off, zero connections measured](https://kunalkushwaha.github.io/2026/11/05/claude-code-picked-neovim/diagram-offline.svg)

Claude scanned the plugin source for network calls, turned off every automatic download it found, then checked it live: Neovim plus five language servers, **zero network connections**. Plugins download only when I update them, and every version is pinned.

Claude Code itself still talks to Anthropic, which is the point of it. I turned off its non-essential traffic too.

## What it was like to build

I didn't write this config. I described what I wanted, sent screenshots when things looked wrong, and Claude did the rest. It tested its own work headlessly, measured instead of guessing, and owned its mistakes.

The rough edges were the instructive part:

- **Swap sides broke after a few rapid presses.** Claude rebuilt it to hide and re-show the Claude pane rather than drag it, then tested that the Claude process survived every swap.
- **Inline diagrams didn't appear.** They were attached to code-fence lines the Markdown view hides. Claude found it by running a real Neovim in a hidden terminal and reading back the screen.
- **Homebrew quietly installed TypeScript 7**, which broke the usual TypeScript language server. Claude switched to TypeScript 7's built-in one.
- **A safety check blocked Claude from downloading plugin code itself**, so I ran those one-line installs. That's the right default.

## What I'd take from this, beyond editors

- **Name what you can't lose, then pick the tool.** The search ended when I stopped comparing editors and described the workflow.
- **Protocols make workflows portable.** The workflow moved because one plugin spoke the same protocol. That's worth checking before you commit to any AI tooling.
- **Say what didn't change.** The editor shrank 10×. The total shrank 2×. Both numbers are true; only one is honest on its own.
- **Make rules you can verify.** "Nothing phones home" became a test with a result: zero connections.

## Build your own

I'm not sharing my config, on purpose. It's tuned to my languages, my shortcuts and my machine. Yours should be tuned to yours, and the point of this post is that you don't need mine.

Open Claude Code in an empty folder and describe what you want. These are roughly the requirements I gave it:

```text
Set up Neovim so it feels like VS Code with the Claude Code extension.
- File tree left, code middle, Claude Code right, opened by `nvim .`
- Claude's edits visible as they land: auto-reload, change bars in the margin
- Claude's diffs reviewable inside the editor (claudecode.nvim)
- Mac shortcuts as in VS Code; each paste its own undo step
- Language servers and format on save for: <your languages>
- Nothing in the editor sends data anywhere; pin plugin versions
- Measure memory against VS Code before and after
```

Then iterate. Send screenshots when something looks wrong, and ask it to test its own changes. Mine runs on macOS and Linux, with Cmd becoming Ctrl on Linux, the same way VS Code does it.

## Was it worth it?

For me, yes. I kept what I liked about VS Code (watch Claude work, see every change land, step in when I need to) and gave about 6 GB back to the things that need it. I didn't have to become a Neovim expert. I had to be clear about what I wanted.

If you live in Claude Code across several projects and your editor is fighting your local models for RAM, it's worth an afternoon.
