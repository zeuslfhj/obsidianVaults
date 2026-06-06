
### How Claude Code works in large codesbases?

RAG-powered AI coding tools work by embedding the entire codebase and retrieving relevant chunks at query time. But it can be outdate in active engineering teams.

Agentic search avoid the outdate problems. but it need enough context to know where to look.

### Claude code's harness
It includes five extensions: CLAUDE.md files, hooks, skills, plugins, and MCP servers.
Two capabilities: subagent, LSP

CLAUDE.md: root file for the big picture, subdirectory files for local conventions.
HOOKS: allow the system to self-improving. It can used to prevent doing something wrong, but their is more valuable use is auto improvement.
- How many hooks in claude code?