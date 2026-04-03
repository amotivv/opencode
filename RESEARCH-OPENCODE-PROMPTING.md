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
