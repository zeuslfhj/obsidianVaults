# Claude Code Source Analysis

## How Claude Code Works in Large Codebases

Most RAG-powered AI coding tools work by embedding the entire codebase, then retrieving relevant chunks at query time. This approach is useful for static repositories, but it can become stale in active engineering teams because embeddings may lag behind recent code changes.

Claude Code relies more heavily on agentic search. Instead of depending only on a prebuilt index, the agent explores the repository with tools such as file search, symbol lookup, shell commands, and project-specific context. This avoids some staleness problems, but it introduces a different requirement: the agent needs enough context to know where to look and enough guardrails to avoid unsafe or irrelevant actions.

The key idea is that Claude Code is not just a chat interface around a model. Its value comes from the harness around the model: context loading, tool orchestration, permissions, hooks, skills, plugins, MCP integrations, subagents, and code-intelligence support.

## Claude Code's Harness

Claude Code can be understood as a model plus a harness. The model reasons and writes. The harness decides what context is available, what tools can be used, when automation runs, and how project-specific knowledge is loaded.

### Five Extension Points

| Extension | What It Does | Best Used For | Common Mistake |
| --- | --- | --- | --- |
| `CLAUDE.md` | Provides project instructions that Claude reads automatically. Root files describe the big picture; nested files can describe local conventions. | Repository conventions, architecture notes, build commands, testing rules. | Putting reusable task expertise here instead of in a skill. |
| Hooks | Run scripts at specific lifecycle events. | Enforcing non-negotiable rules, validating commands, capturing repeated session learnings. | Treating hooks only as blockers instead of also using them for automation and feedback. |
| Skills | Package task-specific instructions and resources for on-demand loading. | Reusable expertise across sessions, repositories, or teams. | Loading all possible guidance into `CLAUDE.md`, which increases context noise. |
| Plugins | Bundle and distribute reusable Claude Code setup. | Sharing skills, hooks, MCP configuration, and team defaults. | Letting useful local setup remain undocumented or tribal. |
| MCP Servers | Connect Claude Code to external tools and data sources. | Internal APIs, issue trackers, documentation systems, databases, and custom tools. | Building MCP integrations before the local project basics are working. |

### Core Capabilities

| Capability | What It Is | When It Helps |
| --- | --- | --- |
| Subagents | Separate Claude instances delegated to specific tasks. | Splitting exploration from editing, running parallel investigations, or isolating specialized work. |
| Code intelligence / LSP | Language-specific symbol navigation and diagnostics when configured by the environment. | Jumping to definitions, understanding typed code, and detecting errors faster than plain text search. |

Subagents and LSP-style code intelligence are capabilities rather than ordinary extension points. Subagents are invoked when a task benefits from delegation. LSP support depends on the language and the local setup, so it should not be assumed to work automatically in every repository.

## Component Comparison

| Component | What It Is | When It Loads | Best For | Common Confusion |
| --- | --- | --- | --- | --- |
| `CLAUDE.md` | Automatically loaded project context. | Every relevant session. | Project-specific conventions and durable codebase knowledge. | Using it for reusable task expertise that belongs in a skill. |
| Hooks | Scripts triggered by Claude Code events. | When the configured event occurs. | Consistent automation, validation, logging, and learning capture. | Using prompts for behavior that should run automatically. |
| Skills | Packaged task instructions. | On demand, when relevant. | Reusable expertise across sessions and projects. | Loading everything into `CLAUDE.md`. |
| Plugins | Bundled Claude Code configuration. | Once installed and configured. | Distributing a working setup across a team or organization. | Keeping good setups local and informal. |
| MCP servers | External tool and data integrations. | Once configured and available. | Giving Claude access to systems it cannot otherwise reach. | Adding external integrations before project context is clean. |
| Subagents | Delegated Claude instances. | When invoked. | Parallel exploration, focused analysis, and separation of concerns. | Mixing exploration and editing in one overloaded session. |
| LSP / code intelligence | Language-aware navigation and diagnostics. | When the environment supports it. | Symbol-level code understanding and early error detection. | Assuming it works automatically without configuration. |

## Practical Setup Checklist

1. Structure `CLAUDE.md` files.
   - Put repository-wide guidance in the root `CLAUDE.md`.
   - Put local conventions near the code they describe.
   - Keep instructions concrete: commands, paths, constraints, and examples.

2. Wire up hooks for non-negotiables.
   - Use hooks for checks that should run consistently.
   - Prefer automation for repeated rules instead of relying on the model to remember them.
   - Review hook output so it is actionable and not just noisy.

3. Install skills and plugins for reusable expertise.
   - Use skills for domain-specific workflows.
   - Use plugins when the setup should be shared across multiple projects or team members.
   - Keep `CLAUDE.md` focused on the current repository.

4. Make the codebase navigable.
   - Ensure search, tests, build commands, and language tooling work locally.
   - Configure language servers or equivalent tooling where useful.
   - Document important entry points and architectural boundaries.

5. Add MCP servers only where they create real leverage.
   - Start with local repository context first.
   - Add external systems when the agent needs data or actions outside the codebase.
   - Keep permission boundaries explicit.

6. Schedule regular configuration reviews.
   - Remove stale instructions.
   - Promote repeated session learnings into durable docs or skills.
   - Re-check hooks, plugins, and MCP servers as the model and toolchain evolve.

7. Assign ownership.
   - Treat Claude Code configuration as part of the engineering system.
   - Give someone responsibility for keeping instructions, hooks, skills, and integrations current.

## Key Takeaway

For large, fast-moving codebases, the central problem is not only retrieval. It is maintaining a reliable agent harness: fresh context, searchable structure, safe automation, reusable expertise, and clear integration boundaries.

RAG helps an agent remember where relevant information might be. Agentic search helps it discover what changed. The harness determines whether either approach is useful in real engineering work.
