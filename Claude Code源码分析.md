
### How Claude Code works in large codesbases?

RAG-powered AI coding tools work by embedding the entire codebase and retrieving relevant chunks at query time. But it can be outdate in active engineering teams.

Agentic search avoid the outdate problems. but it need enough context to know where to look.

### Claude code's harness
It includes five extensions: CLAUDE.md files, hooks, skills, plugins, and MCP servers.
Two capabilities: subagent, LSP

#### Five Extensions
**CLAUDE.md**: root file for the big picture, subdirectory files for local conventions.
**HOOKS**: allow the system to self-improving. It can used to prevent doing something wrong, but their is more valuable use is auto improvement.
- How many hooks in claude code?
**Skills**: present the expertise progressive disclosure.
**Plugins**: distribute installable packages
**MCP Servers**: used to extend everthing

Two Capabilities:
Subagents: split exploration from editing.

| Component                                                                                                                  | What it is                                                | When it loads                    | Best for                                                                 | Common confusion                                        |
| -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------- |
| CLAUDE.md                                                                                                                  | Context file Claude reads automatically                   | Every session                    | Project-specific conventions, codebase knowledge                         | Using it for reusable expertise that belongs in a skill |
| Hooks                                                                                                                      | Scripts that run at key moments                           | Triggered by events              | Automating consistent behavior, capturing session learnings              | Using prompts for things that should run automatically  |
| Skills                                                                                                                     | Packaged instructions for specific task types             | On demand, when relevant         | Reusable expertise across sessions and projects                          | Loading everything into CLAUDE.md instead               |
| Plugins                                                                                                                    | Bundled skills, hooks, MCP configs                        | Always available once configured | Distributing a working setup across the org                              | Letting good setups stay tribal                         |
| Language server protocol (LSP)*                                                                                            | Real-time code intelligence via language specific servers | Always available once configured | Symbol-level navigation and automatic error detection in typed languages | Assuming that it's automatic                            |
| MCP servers                                                                                                                | Connections to external tools and data                    | Always available once configured | Giving Claude access to internal tools it can't otherwise reach          | Building MCP connections before the basics are working  |
| Subagents*                                                                                                                 | Separate Claude instances for specific tasks              | When invoked                     | Splitting exploration from editing, parallel work                        | Running exploration and editing in the same session     |
| *LSP is accessed through the plugin layer. Subagents are a delegation capability rather than a configured extension point. |                                                           |                                  |                                                                          |                                                         |

Read `### Actively maintaining CLAUDE.md files as model intelligence evolves`  again

Getting Start CheckList 
1. Structure CLAUDE.md file
2. Wire up hooks for non-negotiables
3. install skills and plugins for domain expertise
4. make the codebase navigable
5. Schedule regular configuration reviews
6. Assign ownership
