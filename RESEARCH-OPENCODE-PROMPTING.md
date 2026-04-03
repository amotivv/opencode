# OpenCode Prompting Architecture: Research Analysis

## Executive Summary

OpenCode's prompting system is architecturally distinctive because it treats **the model as a variable, not a constant**. Rather than writing one universal prompt and hoping it works across providers, OpenCode maintains provider-specific base prompts, layered instruction injection, dynamic mode switching via system reminders, and aggressive context management. This produces superior results by meeting each model where it performs best rather than forcing a lowest-common-denominator approach.

---

## 1. Provider-Specific Base Prompts

**Key Insight: Different models respond better to different prompting styles.**

OpenCode maintains separate base prompts for each major LLM family, selected at runtime in `packages/opencode/src/session/system.ts:20-34`:

| Provider | File | Core Philosophy |
|----------|------|-----------------|
| Claude/Anthropic | `anthropic.txt` | Professional objectivity, TodoWrite-driven planning, task delegation via Task tool |
| GPT-4/o1/o3 | `beast.txt` | Maximum autonomy, exhaustive internet research, never stop until done |
| GPT (general) | `gpt.txt` | Senior engineer mentality, minimal changes, direct communication |
| Gemini | `gemini.txt` | Action-oriented, AGENTS.md-centric, modular skill system |
| Kimi | `kimi.txt` | Convention-obsessive, ultra-concise output, read-before-edit enforcement |
| Default | `default.txt` | Minimal baseline with token optimization |

### Why This Matters

Each prompt is tuned to the model's tendencies:
- **Claude** gets explicit instructions about professional objectivity because it tends toward agreeableness. The prompt counteracts this: *"Prioritize technical accuracy and truthfulness over validating the user's beliefs."*
- **GPT-4/o1/o3** ("beast mode") gets heavy autonomy framing because these models benefit from persistence directives: *"You MUST iterate and keep going until the problem is solved."*
- **GPT** gets a "senior engineer" identity that encourages minimal, correct changes -- matching GPT's tendency to over-engineer when not constrained.
- **Kimi/Default** get extreme conciseness enforcement with 5+ redundant brevity directives, because these models are more verbose by default.

---

## 2. Layered Prompt Composition

Prompts are not monolithic. They are composed in strict layers at `packages/opencode/src/session/llm.ts:102-114`:

```
Layer 1: Agent prompt (if set) OR Provider base prompt
Layer 2: Custom system prompts passed into the call
Layer 3: User-specific system prompt from the last message
Layer 4: Environment info (model ID, working directory, platform, date)
Layer 5: Skills listing (dynamically loaded based on agent permissions)
Layer 6: User instructions (AGENTS.md, CLAUDE.md, CONTEXT.md -- walked up from cwd)
```

### Caching Optimization

The system maintains a 2-part structure for system messages (`llm.ts:122-127`). The first system message (the base prompt) acts as a stable cache header for providers like Anthropic that support prompt caching. Subsequent layers are joined into a second message. This means the expensive base prompt is cached across turns while dynamic content can change freely.

---

## 3. Agent-Based Behavioral Specialization

Rather than one monolithic agent, OpenCode defines specialized agents in `packages/opencode/src/agent/agent.ts:107-233`, each with distinct:
- **Permission rulesets** (what tools they can use)
- **Custom prompts** (behavioral overrides)
- **Temperature settings** (creativity vs. determinism)
- **Mode** (primary, subagent, or all)

| Agent | Purpose | Key Constraint |
|-------|---------|----------------|
| `build` | Default execution agent | Full tool access, can enter plan mode |
| `plan` | Read-only planning | Cannot edit files (except plan files) |
| `explore` | Fast codebase search | Read-only, search tools only |
| `general` | Parallel subtask execution | No TodoWrite (prevents recursive planning) |
| `compaction` | Context summarization | No tools at all |
| `title` | Conversation naming | No tools, temperature=0.5 |

### Why This Produces Better Results

By constraining agents to their purpose, OpenCode prevents the model from going off-track. The `explore` agent physically cannot edit files, so it focuses entirely on finding information. The `plan` agent cannot execute, so it focuses entirely on thinking. This constraint-based design channels model capability into the right behavior without relying on the model to self-regulate.

---

## 4. Runtime Mode Switching via System Reminders

OpenCode injects `<system-reminder>` tags into conversations at runtime (`packages/opencode/src/session/prompt.ts:252-300`). These are not part of user input -- they are system-level behavioral overrides.

Key uses:
- **Plan mode activation**: Injects read-only constraints and planning instructions
- **Build mode transition**: After planning, injects instructions to execute the plan
- **Max step enforcement**: When an agent hits its step limit, injects a directive to respond in text-only mode
- **Skills injection**: Attaches available skill descriptions contextually

This is powerful because it allows OpenCode to change the model's behavior mid-conversation without modifying the system prompt or starting a new session.

---

## 5. Aggressive Token Optimization

### Output Minimization
Multiple prompts include redundant brevity directives. The `default.txt` prompt has five separate instructions to minimize output:

1. *"You should minimize output tokens as much as possible"*
2. *"MUST answer concisely with fewer than 4 lines"*
3. *"One word answers are best"*
4. *"Avoid introductions, conclusions, and explanations"*
5. *"MUST avoid text before/after your response"*

The redundancy is intentional -- models sometimes ignore a single instruction but are far less likely to ignore five.

### Context Window Management
The compaction system (`packages/opencode/src/session/compaction.ts:35-37`) uses:
- **PRUNE_MINIMUM**: 20,000 tokens -- won't compact below this
- **PRUNE_PROTECT**: 40,000 tokens -- recent context within this range is preserved
- **Protected tools**: Skill tool results are never pruned (they contain critical instructions)

The pruning walks backward through message history, clearing old tool output while preserving recent user turns and skill results.

---

## 6. Convention Enforcement Through Read-Before-Edit

Every provider prompt includes some variant of: *"Before editing, always read the relevant file contents."* This is one of OpenCode's strongest design decisions because it:

1. **Prevents hallucination**: The model sees actual code before attempting changes
2. **Enforces style matching**: By reading surrounding code, the model naturally adopts existing patterns
3. **Reduces errors**: Understanding imports and dependencies prevents broken references

The Anthropic prompt goes further: *"NEVER assume that a given library is available, even if it is well known."* This forces verification of dependencies before use.

---

## 7. Task Planning with TodoWrite

The Anthropic prompt elevates TodoWrite from a convenience tool to a core workflow requirement:

*"Use these tools VERY frequently to ensure that you are tracking your tasks and giving the user visibility into your progress. These tools are also EXTREMELY helpful for planning tasks. If you do not use this tool when planning, you may forget to do important tasks - and that is unacceptable."*

This produces better results because:
- Complex tasks are decomposed before execution
- Progress is visible to the user
- The model has an external memory of what it needs to do (reducing drift)
- Tasks are marked complete incrementally (preventing premature termination)

---

## 8. Instruction System: AGENTS.md

OpenCode loads project-specific instructions from `AGENTS.md`, `CLAUDE.md`, and `CONTEXT.md` files (`packages/opencode/src/session/instruction.ts:19-23`). These are:
- Walked upward from the working directory to the worktree root
- Also loaded from global config (`~/.opencode/AGENTS.md`)
- Loaded from remote HTTP URLs if configured
- Deduplicated per assistant message to avoid redundant injection

The Gemini prompt explains the philosophy:
> *"README.md files are for humans: quick starts, project descriptions, and contribution guidelines. AGENTS.md complements this by containing the extra, sometimes detailed context coding agents need: build steps, tests, and conventions."*

---

## 9. Skills as Composable Prompt Extensions

Skills are modular instruction sets loaded on demand (`packages/opencode/src/session/system.ts:63-75`). They are presented in two ways:
1. **Verbose listing in system prompt** -- helps the model understand available capabilities
2. **Concise listing in tool descriptions** -- avoids context overload during execution

This dual-presentation approach is noted in code comments:
> *"the agents seem to ingest the information about skills a bit better if we present a more verbose version of them here and a less verbose version in tool description"*

---

## 10. Provider-Specific Message Transformation

Beyond prompts, OpenCode adapts the entire message format per provider (`packages/opencode/src/session/message-v2.ts`):
- **Tool result media**: Anthropic supports images in tool results; OpenAI requires separate user messages
- **Tool call IDs**: Claude needs alphanumeric+dashes; Mistral needs 9-char numeric padding
- **Reasoning parts**: Filtered from content and placed in provider-specific fields
- **Prompt caching**: Anthropic gets ephemeral cache control headers on system messages

---

## Architectural Principles That Drive Superior Results

### 1. Constraint Over Instruction
Rather than telling the model "don't edit files during planning," OpenCode physically removes edit permissions from the plan agent. Constraints are more reliable than instructions.

### 2. Redundancy for Critical Behaviors
Important behaviors (conciseness, read-before-edit, convention following) are stated multiple times in different ways. This mirrors best practices in prompt engineering where critical instructions need reinforcement.

### 3. Model-Aware Adaptation
The system adapts everything -- prompt style, message format, parameter configuration, tool presentation -- based on which model is being used. This is fundamentally different from a one-size-fits-all approach.

### 4. External Memory via Tools
TodoWrite and plan files give the model external memory that persists across turns and survives context compaction. This prevents the "forgetting the plan" failure mode common in long conversations.

### 5. Layered Authority
System reminders can override normal behavior, skills extend capabilities, and user instructions customize per-project. This clear hierarchy prevents conflicts and gives each layer appropriate authority.

### 6. Minimal Diff Philosophy
The GPT prompt captures this well: *"The best changes are often the smallest correct changes."* By encouraging minimal modifications, OpenCode reduces the blast radius of errors and keeps changes reviewable.

---

## Claude-Specific Deep Dive: How OpenCode Optimizes for Anthropic Models

This section traces the exact path OpenCode takes when a Claude model is selected, from prompt selection through message formatting, parameter tuning, and context management. Understanding these Claude-specific choices reveals why OpenCode produces superior results with Anthropic models compared to generic coding agent implementations.

### A. The Anthropic Base Prompt: Counteracting Claude's Tendencies

The `anthropic.txt` prompt (`packages/opencode/src/session/prompt/anthropic.txt`) is purpose-built to address Claude's known behavioral patterns:

**1. Counteracting Sycophancy**

Claude models tend toward agreeableness and validation. The Anthropic prompt explicitly fights this:

> *"Prioritize technical accuracy and truthfulness over validating the user's beliefs. Focus on facts and problem-solving, providing direct, objective technical info without any unnecessary superlatives, praise, or emotional validation."*

> *"Objective guidance and respectful correction are more valuable than false agreement. Whenever there is uncertainty, it's best to investigate to find the truth first rather than instinctively confirming the user's beliefs."*

This is absent from every other provider prompt. GPT doesn't need it (it's naturally more terse). Beast mode doesn't need it (persistence overrides politeness). This directive exists because Claude specifically benefits from permission to disagree.

**2. TodoWrite as External Memory**

Claude's Anthropic prompt treats TodoWrite not as optional but as mandatory scaffolding:

> *"Use these tools VERY frequently... These tools are also EXTREMELY helpful for planning tasks, and for breaking down larger complex tasks into smaller steps. If you do not use this tool when planning, you may forget to do important tasks - and that is unacceptable."*

This is uniquely important for Claude because:
- Claude tends to be thorough but can lose track of multi-step plans in long conversations
- The external todo list acts as a forcing function for decomposition
- Marking tasks complete incrementally prevents Claude's tendency to summarize work as done before it actually is
- The todo list survives context compaction, providing persistent memory

Compare this with the GPT prompt, which never mentions TodoWrite. GPT is instead told to "persist until the task is fully handled end-to-end" -- a different strategy for the same problem (premature termination), tuned to what works for each model.

**3. Task Tool for Context Preservation**

The Anthropic prompt strongly pushes Claude toward the Task tool (subagent delegation):

> *"When doing file search, prefer to use the Task tool in order to reduce context usage."*
> *"You should proactively use the Task tool with specialized agents when the task at hand matches the agent's description."*
> *"VERY IMPORTANT: When exploring the codebase to gather context or to answer a question that is not a needle query... it is CRITICAL that you use the Task tool."*

This is a Claude-specific optimization because Claude is particularly effective at:
- Formulating clear subagent prompts (its instruction-following is precise)
- Synthesizing results from multiple parallel subagent responses
- Maintaining high-level context while delegating detail work

The GPT prompt takes the opposite approach: *"When searching for text or files, prefer using Glob and Grep tools"* -- because GPT works better with direct tool calls than delegation.

**4. Communication Style: CLI-Optimized Output**

> *"Output text to communicate with the user; all text you output outside of tool use is displayed to the user. Only use tools to complete tasks. Never use tools like Bash or code comments as means to communicate with the user during the session."*

This targets Claude's specific habit of using code comments or bash echo statements to communicate thoughts -- something GPT rarely does but Claude does when it wants to "show its work."

### B. Claude-Specific Message Handling

At the protocol level, OpenCode applies several Claude-specific transformations in `packages/opencode/src/provider/transform.ts`:

**1. Empty Content Filtering (lines 56-74)**

Anthropic's API rejects messages with empty content. OpenCode proactively filters these:
```typescript
if (model.api.npm === "@ai-sdk/anthropic" || model.api.npm === "@ai-sdk/amazon-bedrock") {
  msgs = msgs.filter(msg => {
    // Remove empty string messages and empty text/reasoning parts
  })
}
```
This prevents a class of API errors that other providers silently handle.

**2. Tool Call ID Sanitization (lines 76-103)**

Claude requires tool call IDs to be alphanumeric with underscores and dashes only:
```typescript
if (model.api.id.includes("claude")) {
  const scrub = (id: string) => id.replace(/[^a-zA-Z0-9_-]/g, "_")
  // Applied to all tool-call and tool-result parts
}
```
This is a strict API requirement that would cause silent failures without intervention.

**3. Prompt Caching with Ephemeral Cache Control (lines 192-238)**

OpenCode applies Anthropic-specific prompt caching when it detects a Claude model:
```typescript
if (model.providerID === "anthropic" || model.api.id.includes("claude") || ...) {
  msgs = applyCaching(msgs, model)
}
```

The caching strategy places `ephemeral` cache control on:
- The first 2 system messages (base prompt = stable cache header)
- The last 2 messages in the conversation (recent context)

This is significant because Anthropic charges differently for cached vs. uncached prompt tokens. By keeping the base prompt as a stable first system message and joining dynamic content into a second message (`llm.ts:122-127`), OpenCode maximizes cache hits across turns.

For Anthropic specifically, the cache control uses message-level options rather than content-level:
```typescript
const useMessageLevelOptions =
  model.providerID === "anthropic" || model.providerID.includes("bedrock")
```

### C. Claude Temperature and Parameters

OpenCode makes deliberate parameter choices for Claude in `packages/opencode/src/provider/transform.ts`:

**Temperature: `undefined` (line 327)**
```typescript
if (id.includes("claude")) return undefined
```
Claude's temperature is left at its API default rather than being explicitly set. This is notably different from Gemini (`1.0`), Qwen (`0.55`), and Kimi (`0.6`). The reasoning: Claude's default temperature already produces the right balance of creativity and determinism for coding tasks, and overriding it can degrade tool-calling reliability.

**Top-P: `undefined`**
Similarly unset for Claude, while other models get explicit values (Gemini: `0.95`, Qwen: `1`).

**Extended Thinking Configuration (lines 551-583)**

For Claude models accessed via `@ai-sdk/anthropic`, OpenCode configures thinking variants:
```typescript
// Adaptive thinking (newer models)
{ thinking: { type: "adaptive" }, effort: "low"|"medium"|"high" }

// Budget-based thinking (fallback)
{ thinking: { type: "enabled", budgetTokens: 16000 } }  // "high"
{ thinking: { type: "enabled", budgetTokens: 31999 } }  // "max"
```

The thinking budget is capped at `min(16000, floor(output_limit/2 - 1))` for "high" and `min(31999, output_limit - 1)` for "max". This ensures thinking never consumes more than half the output budget, leaving room for actual code generation.

### D. The Plan Mode Pipeline: Designed Around Claude's Strengths

The plan mode system (`packages/opencode/src/session/prompt.ts:302-386`) is architecturally tailored to Claude's capabilities:

**Phase 1 -- Parallel Exploration**: Claude is instructed to launch up to 3 explore subagents in parallel. This leverages Claude's strength at formulating clear, scoped prompts for delegated work.

**Phase 2 -- Design Agent**: A general subagent designs the implementation. Claude's strength at structured reasoning makes this delegation effective.

**Phase 3 -- Review**: Claude reads critical files identified by agents. This read-then-plan pattern aligns with the objectivity directive.

**Phase 4 -- Plan File**: The plan is written to a file, not kept in context. This is critical for Claude because:
- It survives context compaction
- It forces explicit, reviewable planning (countering Claude's tendency to plan implicitly)
- It creates an artifact the user can inspect and approve

**Phase 5 -- Plan Exit**: Claude must explicitly call `plan_exit` to transition. The prompt warns: *"Do NOT use question tool to ask 'Is this plan okay?' -- that's what plan_exit does."* This prevents a Claude-specific failure mode where it asks permission to proceed rather than using the designated tool.

**Build Mode Transition**: When switching from plan to build, a system reminder is injected:
> *"Your operational mode has changed from plan to build. You are no longer in read-only mode. You are permitted to make file changes, run shell commands, and utilize your arsenal of tools as needed."*

This clear mode boundary prevents Claude from carrying over the read-only constraint. Without it, Claude's tendency to follow instructions conservatively means it might continue avoiding edits even after plan approval.

### E. Context Compaction: Preserving What Claude Needs

The compaction system has implicit Claude optimizations:

**Skill results are never pruned** (`compaction.ts:37`):
```typescript
const PRUNE_PROTECTED_TOOLS = ["skill"]
```
This matters for Claude because skill instructions are essentially injected prompt extensions. If pruned, Claude loses access to specialized workflows mid-conversation.

**Compaction uses the same model** (`compaction.ts:179-182`): The compaction agent uses the same Claude model that generated the conversation, ensuring the summary captures what Claude would consider important for continuation.

**The compaction prompt** (`agent/prompt/compaction.txt`) is deliberately model-agnostic but optimized for Claude's summarization strengths:
> *"Focus on information that would be helpful for continuing the conversation, including: What was done, what is currently being worked on, which files are being modified, what needs to be done next, key user requests/constraints/preferences that should persist, important technical decisions and why they were made."*

### F. Why This Matters: Claude vs. Generic Approaches

Most coding agents use a single prompt across all models and hope for the best. OpenCode's Claude-specific approach produces better results because it:

1. **Addresses sycophancy head-on**: The objectivity directive causes Claude to push back on wrong assumptions rather than implementing bad ideas agreeably
2. **Leverages Claude's delegation strength**: The Task tool emphasis plays to Claude's ability to formulate precise subagent prompts
3. **Uses external memory strategically**: TodoWrite and plan files compensate for context window limits without degrading Claude's reasoning quality
4. **Optimizes API-level details**: Prompt caching, empty content filtering, and tool ID sanitization prevent silent failures that degrade output quality
5. **Leaves temperature alone**: By not overriding Claude's default, OpenCode avoids the tool-calling reliability issues that come with explicit temperature settings
6. **Structures the thinking pipeline**: Plan mode's phased approach (explore -> design -> review -> write -> exit) matches Claude's strength at structured, methodical reasoning

The net effect is that OpenCode doesn't just use Claude -- it uses Claude in the way Claude works best, with guardrails where Claude is weak (sycophancy, premature termination) and freedom where Claude is strong (delegation, structured reasoning, instruction following).

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/session/system.ts` | Provider prompt selection + environment/skills injection |
| `src/session/prompt.ts` | Main prompt orchestration (1900+ lines) |
| `src/session/llm.ts` | LLM streaming, system prompt composition, parameter config |
| `src/session/prompt/*.txt` | Provider/mode-specific base prompts |
| `src/agent/agent.ts` | Agent definitions with permissions and custom prompts |
| `src/session/instruction.ts` | AGENTS.md/CLAUDE.md instruction loading |
| `src/session/compaction.ts` | Context window management and pruning |
| `src/session/message-v2.ts` | Provider-specific message transformation |
| `src/provider/transform.ts` | Provider-specific parameter defaults |
| `src/tool/skill.ts` | Skill system implementation |

All paths relative to `packages/opencode/`.
