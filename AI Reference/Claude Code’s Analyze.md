[[Claude Code Source Map]]

# Everyone Analyzed Claude Code’s Features. Nobody Analyzed Its Architecture.

## Five hundred thousand lines of leaked source code reveal that the moat in AI coding tools is not the model. It is the harness.

[Reference](https://medium.com/data-science-collective/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622)

[
![Han HELOIR YAN, Ph.D. ☕️](https://miro.medium.com/v2/resize:fill:64:64/1*QKpZJpnggjxenDvymZOQ_g@2x.jpeg)

](https://medium.com/@han.heloir?source=post_page---byline--1173470ab622---------------------------------------)

[Han HELOIR YAN, Ph.D. ☕️](https://medium.com/@han.heloir?source=post_page---byline--1173470ab622---------------------------------------)

Follow

14 min read
Apr 1, 2026

[link](https://medium.com/plans?dimension=post_audio_button&postId=1173470ab622&source=upgrade_membership---post_audio_button-----------------------------------------)

[Free link](https://medium.com/@han.heloir/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622?sk=b2c798cd1597ecc123f2356adf630348) =>**If this helped, I’d really appreciate your full 50 claps. It supports my work and helps others find it.**

On March 31, 2026, thousands of developers worldwide did the same thing: they fed Claude Code’s own source code back into Claude and asked it to explain itself.

Anthropic’s flagship CLI tool had just leaked its entire 512,000-line TypeScript codebase through a source map file accidentally bundled into an npm package. Within hours, the internet had cataloged 44 feature flags, a Tamagotchi pet system with 18 species and gacha mechanics, and internal codenames like “Tengu,” “Fennec,” and “Penguin Mode.”

But the feature list is not the story. Everyone wrote that article already. The real value of this leak is not what Claude Code can do. It is how Claude Code thinks. And the fact that developers paid Anthropic, per token, to understand Anthropic’s own product? That is not irony. That is the thesis.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1400/0*fJwyKc8hc5l9vm0D)

Photo by [Aneta Voborilova](https://unsplash.com/@anetvob?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

## Before we start!🦸🏻‍♀️

If this helps you ship better AI systems:

👏 **Clap 50 times** (yes, you can!) — Medium’s algorithm favors this, increasing visibility to others who then discover the article.

🔔 **Follow me on** [Medium](https://medium.com/@han.heloir), [LinkedIn](https://www.linkedin.com/in/hanheloiryan/) and [subscribe](https://medium.com/@han.heloir/about) to get my latest article

## The Harness Is the Product

Most people assumed Claude Code was a thin CLI wrapper around the Claude API. A terminal interface that sends your prompt, streams the response, and handles a few file operations.

The leaked source tells a different story. What Anthropic actually shipped is 512,000 lines of TypeScript: a custom React terminal renderer with double-buffered screen output and Yoga flexbox layout, 60+ permission-gated tools with deferred discovery, a multi-agent orchestration system that spawns and coordinates parallel worker agents, a background memory consolidation engine that runs while you sleep, and a self-healing query loop that refuses to crash.

This is not a wrapper. This is an operating system for an AI agent.

The Hacker News community split into two camps. One camp dismissed the leak with a casino metaphor: “The source code of the slot machine is not relevant to the casino manager.” The model is the money. The CLI is disposable. The other camp saw the opposite: the model is the dealer. The harness is the casino. And casinos(赌场) are very hard to build.

The numbers support the second camp. Opus 4.6 is available to anyone through the API at $5/$25 per million tokens. Yet VentureBeat reports Claude Code alone generates $2.5 billion in annualized recurring revenue, with 80% coming from enterprise adoption. Developers are not paying for the model. They are paying for the harness that makes the model useful: the tool orchestration, the permission system, the context management, the error recovery. Strip（除去） that away and you have an expensive autocomplete.

The self-referential（所指的） revenue loop from March 31 makes this concrete. One developer built an MCP server specifically so people could use Claude Code to explore leaked Claude Code interactively. Another used Claude Code at 4 AM to port Claude Code’s core architecture to Python before sunrise. A team in China built a 12-session reverse-engineering curriculum from the leaked source. Developer Jingle Bell captured the moment in one sentence: “Claude’s revenue today is coming from everyone using Claude to analyze Claude’s source code.”

The harness generates demand for the model. The model generates revenue for the harness. That flywheel is the product.

What follows are three architectural patterns extracted from the leaked source. Not features. Not codenames. The engineering primitives that make the flywheel spin.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*gtuJEtXRCtgC3gyRo0nqEA.png)

_Harness-as-product stack._

## Pattern 1: The Self-Healing Query Loop

The most expensive engineering in Claude Code is not the AI. It is the error handling.

At the core of Claude Code sits a query loop. Not a simple request-response cycle. A `while(true)` state machine that manages a mutable state object across iterations, with one overriding design goal: never surface a raw error to the user.

Each iteration follows the same sequence. First, it prefetches memory and skills in parallel (not sequentially, which would double latency). Then it applies message compaction if the context is growing too large. Then it calls the API with streaming. Then it executes whatever tools the model requested. Then it checks whether it should continue looping or return. At every step, the loop tracks why it did not terminate, storing the transition reason in state so the next iteration can adapt.

The error recovery cascade is where the real engineering lives. When something fails, the loop does not crash. It walks through a sequence of increasingly aggressive recovery strategies:

1. **Micro-compaction.** Trim low-value messages from context to free up tokens.
2. **Context collapse.** If compaction is not enough, collapse entire conversation segments into summaries.
3. **Token escalation.** If the model’s output budget is exhausted mid-task, inject an invisible meta-message (“Resume directly, no apology, no recap”) and continue. Maximum three consecutive attempts before surfacing the stop reason.
4. **Model fallback.** If the primary model is unavailable, fall back to an alternative.
5. **Surface error.** Only after all recovery paths are exhausted does the user see a failure.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*d4RauQEAoehCoEyHAnG9Ug.png)

_Query loop state machine_

One developer who reverse-engineered 12 versions of Claude Code from minified JavaScript found that 5.4% of all tool calls were silently orphaned: the model requested a tool, the tool ran, but the result never made it back. The self-healing loop is designed to absorb exactly this kind of failure without the user noticing.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*tTt7tTHk4FO896cEy-qHrQ.png)

_Tool concurrency batching._

Tool execution adds another layer. The loop does not run tools one at a time. It partitions tool calls into batches based on a concurrency safety classification. Each tool declares whether it is safe to run in parallel via an `isConcurrencySafe()` method. Read-only tools (grep, glob, file reads) run concurrently, up to 10 at once. Write tools (file edits, bash commands with side effects) run serially. The batches alternate: read batch, write batch, read batch. And a streaming tool executor can begin executing tools while the model is still generating output, overlapping computation and I/O to reduce latency.

The tool system itself is designed around the same principle of not wasting resources. Of Claude Code’s 60+ tools, only about 40 load on every request. The remaining 18 are marked deferred. They are invisible to the model until it searches for them via a dedicated ToolSearchTool. When the model needs a capability (LSP integration, background task creation, cron scheduling), it searches, receives the schema, and calls the tool in the same turn. The user sees nothing unusual. But the context window stayed 200K tokens lighter because tool schemas the model did not need never entered working memory.

Even the order of the tool list matters. Tools are sorted alphabetically before being sent to the API. This is not aesthetic. It is a cache optimization. Alphabetical ordering keeps the tool list identical across requests, which maximizes prompt cache hit rates. A cache miss on the tool list means re-processing thousands of tokens of schema definitions. Sorting by name turns that into a stable prefix.

The counterintuitive lesson for anyone building agent systems: reliability is not a feature you add after the core loop works. The core loop is the reliability system. Claude Code’s query engine is 46,000 lines not because the happy path is complex, but because the recovery paths are.

## Pattern 2: Sleep-Time Compute

The most important thing Claude Code does is what it does while you are not using it.

Every AI coding tool has the same problem. You work with it for hours, building context: architecture decisions, debugging patterns, build commands, personal preferences. Then you close the terminal. The next session starts from zero. The model has no memory of yesterday. You repeat yourself. It makes the same mistakes. The context you built is gone.

Claude Code’s answer is a system called autoDream. The name is deliberate. It is Claude, dreaming.

Between sessions, Claude Code spawns a forked subagent whose sole job is memory consolidation. The subagent reads your project’s memory directory, reviews recent session logs, identifies new information worth persisting, and rewrites the memory files to be cleaner, more accurate, and more useful for the next session. The system prompt for this subagent says exactly what it is: “You are performing a dream, a reflective pass over your memory files. Synthesize what you have learned recently into durable, well-organized memories so that future sessions can orient quickly.”

The dream does not run whenever it feels like it. It has a three-gate trigger, and all three gates must pass before consolidation begins.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*scFE3V8vfjhkCSvhCMPTig.png)

_Three-gate trigger system_

**Gate 1: Time.** At least 24 hours since the last dream. This prevents over-consolidation when you are actively working across multiple short sessions.

**Gate 2: Sessions.** At least 5 sessions since the last dream. This ensures enough new signal has accumulated to make consolidation worthwhile.

**Gate 3: Lock.** The subagent must acquire a consolidation lock. This prevents concurrent dreams if multiple Claude Code instances are active.

When all three gates pass, the dream executes four phases.

**Phase 1, Orient.** List the memory directory. Read the index file (MEMORY.md). Skim existing topic files to understand current state.

**Phase 2, Gather Signal.** Search recent sources for new information worth persisting. Priority order: daily logs first, then drifted memories (facts that have changed), then transcript search for patterns the model noticed during work.

**Phase 3, Consolidate.** Write or update memory files. Convert relative dates (“yesterday”) to absolute dates (“March 30, 2026”). Delete facts that contradict newer information. Merge redundant entries.

**Phase 4, Prune and Index.** Keep MEMORY.md under 200 lines and approximately 25KB. Remove stale pointers. Resolve contradictions. The index must stay small enough to load into every future session’s context without meaningful token cost.

The dream subagent gets read-only bash access. It can observe your project (read files, list directories, check git state) but cannot modify anything. This is a safety boundary: the consolidation process should never have side effects on your codebase.

This architecture maps onto a concept from UC Berkeley’s sleep-time compute research: using idle compute cycles to improve future inference efficiency. The paper proposes pre-inferring likely future queries from accumulated context. autoDream looks backward rather than forward. It organizes past memory, not future predictions. But the design philosophy is the same: invest compute when the user is not waiting, so the next session starts faster and with better context.

The result is a four-layer memory architecture that no other AI coding tool has shipped.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*n1NEmc40YLayXpuN-_K8rw.png)

_Four-layer memory architecture_

**Layer 1: CLAUDE.md.** Instructions you write. Static, human-authored, always loaded.

**Layer 2: Auto Memory.** Notes Claude writes during each session. Build commands, debugging insights, architecture decisions, your preferences. Accumulates automatically.

**Layer 3: Session Memory.** Conversation continuity within a single session. Standard context window management.

**Layer 4: Auto Dream.** Periodic consolidation of everything accumulated across layers 1 through 3. The garbage collector. The defragmenter. The REM sleep.

An instruction manual, a note-taker, short-term recall, and REM sleep. That is not a feature list. That is a cognitive architecture.

## Pattern 3: Compile-Time Feature Elimination

The source map leaked precisely because the security system worked. The code was eliminated from the build. It just was not eliminated from the map.

Anthropic ships one codebase to two audiences. Internal employees get KAIROS (the always-on persistent assistant), BUDDY (the Tamagotchi companion), Coordinator Mode (multi-agent orchestration), Voice Mode, Bridge Mode, and a dozen other experimental subsystems. External users get none of these. The external build does not contain dead code behind if-statements. The code is physically absent from the executable.

The mechanism is Bun’s `feature()` function, which evaluates at build time, not runtime. When Anthropic builds the external package, every `feature('KAIROS')` call resolves to `false` as a compile-time constant. Bun's bundler then dead-code-eliminates the entire branch. The resulting JavaScript file contains zero trace of KAIROS, BUDDY, or any other gated subsystem. No string references. No function bodies. No import paths. Gone.

This is the first tier of a two-tier feature system.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*eeyxe96PpQofEvViK6XFfg.png)

_Compile-time vs. runtime feature gating._

The second tier is runtime gating through GrowthBook, a feature flag platform. Runtime flags (prefixed `tengu_` in the codebase) control gradual rollouts, A/B tests, and kill switches for features that have already passed compile-time gates. The function that checks these flags is named `getFeatureValue_CACHED_MAY_BE_STALE()`. That name is a design decision encoded in code: stale data is acceptable for feature gates. Speed matters more than freshness. The agent should never block on a flag check.

The two tiers serve different purposes. Compile-time elimination is a security boundary: internal-only features must not exist in the external artifact at all, not even as unreachable code. Runtime gating is a rollout mechanism: features that are cleared for external release can be turned on gradually, monitored, and killed instantly if something breaks.

A third layer sits on top of both: `USER_TYPE === 'ant'` gates features exclusive to Anthropic employees. This includes staging API access, debug prompt dumping, Undercover Mode (which prevents Claude from revealing that it is an AI in public open-source contributions), and internal-only tools like ConfigTool and TungstenTool.

The irony that made this leak possible is structural. Source maps exist to bridge the gap between compiled output and original source. They contain the original source in a `sourcesContent` array, regardless of what the compiler removed. Dead-code elimination protects the executable. It does not protect the map. Bun generates source maps by default. Nobody configured it to stop. Nobody added `*.map` to `.npmignore`. And so the entire internal codebase shipped to npm in a 59.8MB JSON file.

This is not the first time. The same source map issue surfaced in February 2025 when Claude Code first launched. Anthropic removed it quietly. It came back in version 2.1.88, thirteen months later. The packaging pipeline was never fixed. Bun’s defaults were never overridden. An AI company with the world’s best language model, a company that built Undercover Mode specifically to prevent internal information from leaking, was brought down by a missing line in `.npmignore`.

The lesson for builders is structural, not anecdotal. If you are shipping an agent product, you need a two-tier feature system. Compile-time elimination for security boundaries (what must never exist in the external artifact). Runtime flags for rollout control (what exists but is toggled off). And a build pipeline that verifies what actually ends up in your published artifact. `npm pack --dry-run` before every publish. Every time.

## What This Means for the Harness vs. Model Debate

The internet spent March 31 cataloging features. A Tamagotchi pet system. Animal codenames. A frustration regex that detects when you swear at the AI. Those are entertaining. They are not important.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:2000/1*5idChHyenhvSGi349I1CkA.png)

What matters is the three patterns.

A self-healing query loop that treats reliability as the core product, not a bolted-on concern. The loop is 46,000 lines because recovery paths (compaction, collapse, token escalation, model fallback) are more complex than the happy path. Tool execution is batched by concurrency safety. Deferred discovery keeps 18 tools out of working memory until needed. Alphabetical sorting stabilizes prompt caches. Every detail serves the same goal: the user should never see a failure.

A sleep-time compute system that invests idle cycles into memory consolidation. Four memory layers (instructions, notes, session recall, dream consolidation) form the first cognitive architecture shipped in a production coding tool. The three-gate trigger prevents both over-dreaming and under-dreaming. The dream subagent runs read-only. The index ceiling is 200 lines. The design is constrained, deliberate, and production-hardened.

A compile-time feature elimination system that enforces security boundaries at the artifact level. Two tiers (compile-time for security, runtime for rollout) serve distinct purposes. The structural irony is that the very system designed to keep internal features hidden is what made the source map so valuable when it leaked: everything the compiler removed was still in the map.

These are not features. They are primitives. The building blocks that any production agent system will eventually need, whether it is built on Claude, GPT, Gemini, or an open-source model. The leaked source did not reveal Claude Code’s competitive advantage. It revealed the engineering requirements for the entire category.

Competitors now have a reference architecture. One developer ported the core patterns to Python before sunrise. A team built a 12-session reverse-engineering curriculum. Rust ports are already in progress. The patterns will transfer. They always do.

But knowing the architecture is not the same as executing it. Claude Code has 13 months of production hardening behind these patterns. Recovery paths refined across 12 shipped versions. Permission rules tuned against real enterprise workloads. A permission system with five public modes, an ML classifier for auto-approval, and pre-execution hooks that can silently modify tool parameters before execution (the model does not know its input was changed). The source code is a snapshot. The operational knowledge is not in the repository.

The self-referential loop from March 31 closes the argument. Developers paid for the model to understand the harness. The harness is what makes the model useful. The model is what makes the harness profitable. That flywheel is not something you can copy from a source map. It is the product.

## Credits & Further Reading

**Kuberwastaken/claude-code** (GitHub): The most complete feature catalog and breakdown of the leaked source. Start here for the “what.” [https://github.com/Kuberwastaken/claude-code](https://github.com/Kuberwastaken/claude-code)

**sathwick.xyz, “Reverse-Engineering Claude Code”**: 38-minute architectural deep dive covering the query engine, tool system, permission pipeline, and terminal renderer. The best technical analysis published. [https://sathwick.xyz/blog/claude-code.html](https://sathwick.xyz/blog/claude-code.html)

**VentureBeat, “Claude Code’s source code appears to have leaked”**: Business context including $19B Anthropic ARR, $2.5B Claude Code ARR, and the axios supply-chain attack that coincided with the leak. [https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)

**Penligent, “Claude Code Source Map Leak”**: The most measured security assessment. Distinguishes what the evidence supports from what it does not. [https://www.penligent.ai/hackinglabs/claude-code-source-map-leak-what-was-exposed-and-what-it-means/](https://www.penligent.ai/hackinglabs/claude-code-source-map-leak-what-was-exposed-and-what-it-means/)

**thehuman2ai.com, “Claude Code source has been available for 13 months”**: The historical timeline showing this source map has shipped since launch day, February 24, 2025. [https://thehuman2ai.com/blog/claude-code-source-leak](https://thehuman2ai.com/blog/claude-code-source-leak)

**shareAI-lab/learn-claude-code** (GitHub): 12-session harness engineering curriculum built from the leaked architecture. Best resource for builders who want to implement the patterns. [https://github.com/shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)