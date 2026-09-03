REPOSITORY NAME:
nema

DESCRIPTION:
A lightweight runtime for persistent autonomous agents with memory, tools, checkpoints and resumable execution.

ABOUT:
Nema is an experimental runtime for building autonomous AI processes that persist beyond a single prompt. Agents receive memory, state, tools, tasks, permissions, checkpoints and a permanent execution history so they can stop, recover and continue working.

WEBSITE:
Leave blank for now.

TOPICS:
agents
ai
llm
autonomous-agents
typescript
runtime
memory
agent-framework
tool-use
automation
open-source

LICENSE:
MIT

LANGUAGE:
TypeScript

# Nema

A lightweight runtime for persistent autonomous agents.

Nema explores a simple question:

> What infrastructure does an AI need if it is expected to exist for longer than one prompt?

Most LLM applications follow roughly the same lifecycle:

user input → model → response → process ends

That model works well for assistants, but becomes limiting when building autonomous software expected to operate for minutes, hours, days or longer.

Nema treats an agent as a persistent computational process.

Instead of rebuilding the agent from scratch every time something happens, Nema maintains its state, memory, task queue, permissions, tool history and checkpoints.

The default execution loop is:

OBSERVE → REMEMBER → PLAN → ACT → VERIFY → CHECKPOINT → CONTINUE

## Why Nema

Long-running agents introduce infrastructure problems that are easy to ignore in small demos.

What happens if an agent crashes halfway through a task?

What happens when its context window fills?

What happens when a tool fails?

What happens if the model provider becomes unavailable?

What prevents an agent from repeatedly performing the same failed action?

How do we inspect exactly what an agent did six hours ago?

How do we limit what the agent is allowed to access?

Nema exists to explore those problems.

## Core Architecture

Every Nema agent is composed of:

Agent
State
Memory
Tasks
Tools
Model Router
Permission Engine
Event Store
Checkpoint Manager

Conceptually:

                       ┌──────────────┐
                       │    MODEL     │
                       └──────┬───────┘
                              │
                              ▼
┌──────────┐          ┌──────────────┐
│  MEMORY  │◄────────►│    AGENT     │◄────────► TOOLS
└──────────┘          └──────┬───────┘
                              │
                     ┌────────▼────────┐
                     │   TASK QUEUE    │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  EVENT STORE    │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  CHECKPOINTS    │
                     └─────────────────┘

## Agent State

An agent has persistent state.

Example:

{
  "agent_id": "nema-01",
  "status": "running",
  "current_task": "task_184",
  "step": 418,
  "started_at": "2026-09-03T19:00:00Z",
  "last_checkpoint": "cp_041",
  "budget_remaining": 7.42
}

Agent states include:

created
idle
planning
running
waiting
paused
blocked
failed
terminated

## Persistent Tasks

Tasks are first-class objects.

{
  "id": "task_184",
  "goal": "inspect repository and locate failing tests",
  "status": "running",
  "priority": 7,
  "attempts": 2,
  "parent": null
}

Tasks can spawn subtasks.

task_184
├── inspect_repository
├── run_tests
├── identify_failure
├── patch_code
└── verify_patch

This allows agents to perform structured work without putting an entire objective into one giant model call.

## Memory

Nema separates memory into multiple layers.

WORKING MEMORY

Information required for the current reasoning cycle.

EPISODIC MEMORY

Events the agent experienced.

SEMANTIC MEMORY

Facts learned during execution.

PROCEDURAL MEMORY

Reusable strategies.

ARCHIVE MEMORY

Compressed long-term history.

Example memory object:

{
  "id": "mem_8821",
  "type": "episodic",
  "content": "Build failed because Node 24 removed the --loader behavior.",
  "importance": 0.82,
  "created_at": "...",
  "related_task": "task_184"
}

## Memory Compression

Long-running agents cannot keep everything inside the model context forever.

Nema periodically compresses older events.

Raw events:

120,000 tokens

↓

summary generation

↓

9,400 tokens

↓

semantic extraction

↓

1,300 tokens of persistent knowledge

The original history remains available in storage while the compressed version is used for active reasoning.

## Event-Sourced Execution

Every meaningful action produces an event.

Examples:

agent.started
task.created
task.started
model.called
memory.created
tool.called
tool.completed
tool.failed
checkpoint.created
task.completed
agent.paused

Example:

{
  "event": "tool.completed",
  "agent": "nema-01",
  "task": "task_184",
  "tool": "shell.execute",
  "duration_ms": 842,
  "timestamp": "..."
}

This makes agent behavior inspectable and replayable.

## Tools

Tools implement a simple interface.

interface Tool {
  name: string
  description: string

  execute(
    input: unknown,
    context: ToolContext
  ): Promise<ToolResult>
}

Possible tools include:

filesystem
shell
browser
database
Git
HTTP
search
code execution
hardware
other agents

## Permission Engine

Autonomous systems should not receive unlimited privileges by default.

Example:

permissions:
  filesystem:
    read: true
    write: workspace

  shell:
    enabled: true
    dangerous_commands: false

  network:
    enabled: true

  spending:
    max_per_action_usd: 1
    max_daily_usd: 10

Permissions are enforced outside of the model.

The model cannot simply decide to override them.

## Checkpoints

Nema periodically creates checkpoints.

A checkpoint contains:

agent state
active tasks
working memory
plans
budgets
tool state
event cursor
configuration

Example:

checkpoints/
├── cp_001
├── cp_002
├── cp_003
└── cp_004

If the runtime dies after cp_004, execution can resume from that point.

## Model Routing

Nema does not require one model for every task.

Example:

planning → reasoning model
classification → small model
coding → coding model
summarization → fast model
embeddings → embedding model

Providers implement a common adapter.

interface ModelAdapter {
  generate(request: ModelRequest): Promise<ModelResponse>
}

## Example

const agent = await nema.createAgent({
  name: "Scout",
  objective: "Maintain this software project.",
  checkpointEvery: 10
})

agent.use(gitTool)
agent.use(shellTool)
agent.use(filesystemTool)

await agent.run()

## Proposed Structure

nema/
├── src/
│   ├── runtime/
│   │   ├── agent.ts
│   │   ├── loop.ts
│   │   └── scheduler.ts
│   ├── memory/
│   ├── tasks/
│   ├── tools/
│   ├── models/
│   ├── permissions/
│   ├── checkpoints/
│   └── events/
├── examples/
├── tests/
└── docs/

## Design Principles

Persistence over prompts.

Events over invisible state.

Explicit permissions over model judgment.

Recoverability over perfect execution.

Provider independence over model lock-in.

Inspectable systems over black boxes.

## Roadmap

Phase 01

Core runtime
SQLite persistence
Task queue
Tool registry
Event logging
Checkpoint system

Phase 02

Layered memory
Memory compression
Model routing
Budget controls
Retry strategies
Permission engine

Phase 03

Agent-to-agent messaging
Distributed workers
Execution debugger
Replay system
Remote runtimes

Phase 04

Long-duration experiments
Agent snapshots
Runtime migration
Collaborative agents
Autonomous scheduling

## Status

Experimental.

Nema is being built as a research project around persistent autonomous software.

The goal is not to create the largest agent framework.

The goal is to discover the smallest reliable infrastructure required for an AI process to:

remember
act
fail
recover
learn
and continue.

## Contributing

Experiments, issues and pull requests are welcome.

## License

MIT
