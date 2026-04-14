# Hermes Agent: The Full Picture

*A first-principles tutorial on what Hermes Agent is, how it works, and why it's built the way it is.*

---

## The One-Paragraph Version

Hermes Agent is an open-source AI agent built by [Nous Research](https://nousresearch.com). You give it a language model, it gives the model tools (terminal, file system, web browser, code execution), and then it runs a loop: the model thinks, calls a tool, sees the result, thinks again. What makes it unusual is the learning loop — it persists memory across sessions, creates reusable skills from experience, and searches its own conversation history. And the whole thing can be used to generate training data for making the *next* model better at using tools. It's an agent framework that feeds back into model training.

Now let's build up to that understanding from scratch.

---

## Part 1: The Core Insight — LLMs That Do Things

Here's the foundational idea. A language model, by itself, can only generate text. It takes in text, it produces text. That's it. But if you do three things — (1) describe some tools in the system prompt, (2) let the model output a special "I want to call this tool" message instead of regular text, and (3) actually execute that tool and feed the result back — then suddenly the model can *do things*. It can run shell commands, edit files, search the web, control a browser.

This is the **agentic loop**, and it's the beating heart of Hermes Agent. Let's look at how it actually works in code.

### The Agent Loop

Open `run_agent.py` and find the `run_conversation()` method on `AIAgent`. The logic is roughly:

```
while we haven't hit the iteration limit:
    response = call the LLM with (messages + tool_schemas)
    
    if response contains tool_calls:
        for each tool_call:
            result = execute the tool
            append the result to messages
        continue the loop
    else:
        # The model chose to respond with text — we're done
        return response.content
```

That's it. That's the core loop. Everything else in the codebase is either *making this loop work better* or *making it work in more places*.

The messages follow the OpenAI format: `system`, `user`, `assistant`, `tool` roles. The model sees the full conversation history plus all available tool schemas, decides whether to call a tool or respond, and the loop continues until it responds with text.

### How Tools Are Registered

The tool system uses a **registry pattern** (see `tools/registry.py`). Each tool file in `tools/` registers itself at import time:

```python
# In tools/web_tools.py (simplified)
from tools.registry import registry

registry.register(
    name="web_search",
    toolset="web",
    schema={"name": "web_search", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: web_search(query=args["query"]),
    check_fn=lambda: bool(os.getenv("TAVILY_API_KEY")),
)
```

Every tool follows this exact pattern:
1. Define a function that does the work and returns a JSON string
2. Register it with a name, toolset membership, JSON schema, handler, and optional availability check

The orchestration layer (`model_tools.py`) triggers discovery by importing all tool modules, then provides `get_tool_definitions()` (returns schemas for the LLM) and `handle_function_call()` (dispatches execution). The `AIAgent` class only talks to these two functions — it never touches individual tools directly.

This is a clean separation. The agent loop doesn't know what tools exist. The tools don't know about the agent loop. The registry sits in the middle.

### Toolsets: Grouping Tools

Not every conversation needs every tool. The **toolset system** (`toolsets.py`) groups tools into named sets:

- `web` — web_search, web_extract
- `terminal` — terminal, process management
- `file` — read_file, write_file, patch, search_files
- `browser` — full browser automation suite
- `skills` — skill listing, viewing, management

The shared list `_HERMES_CORE_TOOLS` defines the default set that's enabled for both CLI and messaging platforms. You can enable/disable toolsets at runtime (`hermes tools`), and the agent adjusts what schemas it sends to the model.

This matters more than you might think. Sending 40+ tool schemas to a model eats context window and can confuse smaller models. Being able to scope down to just `terminal` + `file` for a coding task is a practical optimization.

---

## Part 2: The Learning Loop — What Makes It "Self-Improving"

Most agent frameworks are stateless. Each conversation starts from zero. Hermes Agent does something different: it learns from experience and carries that knowledge forward.

### Memory: Persistent Notes Across Sessions

The memory system (`tools/memory_tool.py`) maintains two files:

- **MEMORY.md** — the agent's personal notes. Environment facts, project conventions, tool quirks, things it learned. ("This project uses pytest, not unittest." "The Docker daemon needs to be started manually on this machine.")
- **USER.md** — what the agent knows about the user. Preferences, communication style, workflow habits. ("User prefers concise responses." "User works primarily in Python and Go.")

Both files are injected into the system prompt as a frozen snapshot at session start. When the agent writes a new memory mid-session, the file on disk updates immediately, but the system prompt doesn't change — this preserves the **prompt cache** (critical for cost). The snapshot refreshes on the next session.

The agent doesn't just passively record things. The system includes periodic **nudges** that prompt the agent to persist knowledge it's accumulated during a conversation. This is the "closed learning loop" — the agent is incentivized to notice what it learned and write it down.

### Skills: Reusable Procedural Knowledge

Skills are markdown files with instructions that the agent can invoke on demand. They live in `~/.hermes/skills/` and can be browsed with `/skills`. But the interesting part is **autonomous skill creation**: after completing a complex multi-step task, the agent can create a skill that captures the procedure for next time.

Skills self-improve during use. If a skill's instructions lead to a suboptimal outcome, the agent can update the skill with better instructions. Over time, skills get refined by actual usage.

The skill system is compatible with the [agentskills.io](https://agentskills.io) open standard, so skills can be shared across different agent frameworks.

### Session Search: Cross-Conversation Recall

The state store (`hermes_state.py`) is a SQLite database with **FTS5 full-text search**. Every conversation is stored — messages, tool calls, results, metadata. The agent has a `session_search` tool that can search across all past sessions.

This means the agent can answer questions like "how did I set up that Docker container last week?" by searching its own history. Combined with persistent memory, this gives the agent a form of long-term recall that spans conversations.

---

## Part 3: Architecture — How the Pieces Connect

Here's the dependency chain, from the bottom up:

```
tools/registry.py          ← Central registry (no deps, imported by all tools)
    ↑
tools/*.py                 ← Each tool self-registers at import time
    ↑
model_tools.py             ← Orchestration: triggers discovery, provides public API
    ↑
run_agent.py               ← AIAgent class: the conversation loop
    ↑
cli.py / gateway/run.py    ← Entry points: terminal UI or messaging platforms
```

### Key Classes

**`AIAgent`** (`run_agent.py`) — The core. Holds the conversation loop, manages message history, handles context compression, builds the system prompt. Key methods:
- `chat(message)` — simple interface, returns a string
- `run_conversation(user_message, ...)` — full interface, returns messages + metadata

**`SessionDB`** (`hermes_state.py`) — SQLite session store. Stores every conversation with FTS5 search. Used by both CLI and gateway.

**`ContextCompressor`** (`agent/context_compressor.py`) — When the conversation gets too long for the context window, this summarizes the middle turns while protecting the head (system prompt, early context) and tail (recent messages). Uses an auxiliary model for summarization.

**`MemoryManager`** (`agent/memory_manager.py`) — Orchestrates memory providers. Always has the built-in provider (MEMORY.md + USER.md), and can plug in one external provider (like [Honcho](https://github.com/plastic-labs/honcho) for dialectic user modeling).

### System Prompt Assembly

The system prompt is assembled by `agent/prompt_builder.py` from multiple pieces:
1. **Identity** — who the agent is, what it can do
2. **Platform hints** — CLI vs. messaging differences
3. **Skills index** — list of available skills
4. **Context files** — project-specific instructions (AGENTS.md, .cursorrules)
5. **Memory snapshot** — frozen MEMORY.md + USER.md content

The prompt builder also includes **injection detection** — it scans context files for patterns like "ignore previous instructions" and strips them. This is a real security concern when the agent reads files from untrusted sources.

### Prompt Caching

Anthropic's API supports caching the system prompt prefix. Since the system prompt is stable across turns, this saves ~75% on input token costs. The caching logic (`agent/prompt_caching.py`) places cache breakpoints at the system prompt and the last 3 messages (rolling window).

This is why memory writes don't update the system prompt mid-session — doing so would **invalidate the cache** and dramatically increase costs.

---

## Part 4: Where It Runs — Terminal Backends and the Gateway

### Terminal Backends

Hermes Agent doesn't just run on your laptop. The `tools/environments/` directory provides six terminal backends:

| Backend | What it is |
|---------|-----------|
| **local** | Your machine. The default. |
| **docker** | Commands run in a Docker container. Isolated but stateful. |
| **ssh** | Commands run on a remote machine over SSH. |
| **daytona** | Serverless dev environment. Hibernates when idle, wakes on demand. |
| **modal** | Serverless cloud compute. Spin up GPU instances for ML tasks. |
| **singularity** | HPC container runtime. For cluster environments. |

The `terminal` tool is the same regardless of backend — the environment abstraction means the agent doesn't know or care where commands execute. This is powerful: you can develop locally, then switch to a cloud backend for heavy compute, without changing anything about how you interact with the agent.

### The Messaging Gateway

The gateway (`gateway/`) is a single process that connects Hermes to messaging platforms:

- Telegram
- Discord
- Slack
- WhatsApp
- Signal
- Home Assistant

Each platform has an adapter in `gateway/platforms/`. The gateway translates platform-specific events (Telegram messages, Discord interactions, Slack commands) into a uniform format, runs them through the same `AIAgent` loop, and sends responses back.

This means you can start a conversation in the CLI, continue it from Telegram on your phone, and the agent has the same memory, skills, and session history across both. The gateway supports voice memo transcription, so you can literally talk to it.

### Subagent Delegation

The delegate tool (`tools/delegate_tool.py`) spawns child `AIAgent` instances for parallel work. Each child gets:
- A fresh conversation (no parent history)
- Its own terminal session
- A restricted toolset (no recursive delegation, no user interaction, no memory writes)

The parent only sees the delegation call and the summary result. This keeps the parent's context clean while allowing complex multi-step work to happen in parallel.

---

## Part 5: The Research Angle — Closing the Loop

This is where Hermes Agent goes beyond a typical agent framework. The `environments/` directory contains integration with the **Atropos** RL training framework.

### The Idea

If you have an agent framework that:
1. Can run any model through multi-turn tool-calling tasks
2. Can score the results (did the model solve the task?)
3. Can capture full trajectories (every message, every tool call, every result)

...then you have everything you need to **train better models**. This is the loop:

```
Model generates trajectories → Score with reward functions → Train model with RL → Better model → Repeat
```

### How It Works

**`HermesAgentBaseEnv`** (`environments/hermes_base_env.py`) extends Atropos's `BaseEnv` to add hermes-agent tool calling. Concrete environments inherit from it:

- **HermesSweEnv** — SWE-bench style coding tasks. Model gets an instruction, uses terminal + file tools, reward function runs tests.
- **TerminalBench2** — 89 terminal tasks with Docker sandboxes. Each task has a pre-built image and test suite.
- **TerminalTestEnv** — Stack validation. Simple tasks to verify the full pipeline works.

**`ToolContext`** gives reward functions direct access to the agent's sandbox. After a rollout, the verifier can run commands, read files, and check results in the *same environment* the model used. This enables rich reward signals beyond just "did the output match?"

### Two Phases

- **Phase 1 (OpenAI server)**: For evaluation and SFT data generation. The server handles tool call parsing natively.
- **Phase 2 (VLLM ManagedServer)**: For full RL training. Client-side parsers in `environments/tool_call_parsers/` reconstruct tool calls from raw model output, giving you exact token IDs and logprobs for the training signal.

The `batch_runner.py` at the root provides parallel batch processing for generating trajectories at scale, with checkpointing for fault tolerance.

---

## Part 6: The Big Picture

Let's zoom out. Here's what Hermes Agent actually is, as a system:

1. **An agent runtime** — the core loop that connects LLMs to tools
2. **A learning system** — memory, skills, and session search that improve over time
3. **A multi-platform interface** — CLI, Telegram, Discord, Slack, WhatsApp, Signal
4. **A multi-backend executor** — local, Docker, SSH, Daytona, Modal, Singularity
5. **A research platform** — trajectory generation, RL training, benchmarking

Most agent frameworks give you (1). Some give you (1) + (3). Hermes gives you all five, and the key insight is that they reinforce each other. The multi-platform interface generates more conversations. More conversations generate more memories and skills. Better memories and skills generate better trajectories. Better trajectories train better models. Better models run the agent loop more effectively.

It's a flywheel, and the codebase is designed to spin it.

### What It's Not

- It's not tied to one model provider. Use OpenAI, Anthropic, OpenRouter (200+ models), or your own endpoint.
- It's not a hosted service. It runs on your hardware, your VPS, your cluster.
- It's not a framework you build *on top of*. It's a complete agent you configure and extend with skills and tools.

---

## Where to Go from Here

| If you want to... | Start here |
|---|---|
| Use it | `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh \| bash` |
| Understand the agent loop | `run_agent.py` → `run_conversation()` |
| Understand the tool system | `tools/registry.py` → `model_tools.py` → any `tools/*.py` |
| Understand memory | `tools/memory_tool.py` + `agent/memory_manager.py` |
| Understand the gateway | `gateway/run.py` + `gateway/platforms/` |
| Train models with it | `environments/README.md` + `environments/hermes_base_env.py` |
| Add a new tool | Follow the 3-file pattern: `tools/your_tool.py` → `model_tools.py` → `toolsets.py` |
| Add a new skill | Drop a markdown file in `~/.hermes/skills/` |

The code is MIT-licensed, the community lives on [Discord](https://discord.gg/NousResearch), and the full docs are at [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).

---

*Built by [Nous Research](https://nousresearch.com). The only agent that learns from itself and feeds back into training the next generation of models.*
