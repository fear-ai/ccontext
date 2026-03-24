# Claude Code Prompt System: Structure, Catalog, Reduction, and Reference

*Maintained: 2026-03-14. Analysis of the Claude Code prompt system as extracted from the v2.1.74 native binary (`/opt/homebrew/Caskroom/claude-code/2.1.74/claude`), with installation-specific reduction strategy for the Spank project.*

*Cross-references: `CInternals.md` — binary internals and the decompiled runtime assembly function pseudocode. `CCLaunch.md` — **External reference index** (canonical URLs for env vars, interactive shortcuts, API compaction, advanced tool use).

---

## Introduction

### Goals and Intent of This Investigation

Claude Code's context window is its most constrained resource. At 200K tokens, every session starts with a baseline load of system prompt fragments, tool descriptions, injected project files, and cached sections — before any conversation has occurred. Hitting the **in-app** autocompaction threshold mid-task produces a context summary that discards tool results, loses intermediate reasoning, and can desynchronize the model's understanding of the current state. The threshold is **percentage-based** in official docs (**~95%** default, overridable lower via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`; see [Claude Code environment variables](https://code.claude.com/docs/en/env-vars)). Binary extraction in this document also records `effectiveWindow − 13000` (93.5% on a 200K window). The `/context` UI may show a fixed **Autocompact buffer** slice (~16.5% in observed builds) — a third accounting layer. **Do not confuse** with **Messages API** compaction (`compact_20260112`), whose documented default **trigger** is **150,000 input tokens** ([Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)). For a project like Spank, which runs multi-step agentic sessions with numerous tool calls, baseline bloat is the primary cause of premature compaction.

This investigation has four goals:

1. **Understand the complete prompt assembly pipeline** — what is loaded, when, by what mechanism, in what order, and why. This is prerequisite knowledge for any safe modification.
2. **Identify all token costs** — catalog every prompt fragment with its conditional load state, so that both baseline and worst-case context sizes are known before any model call.
3. **Find and eliminate wasteful or contradictory content** — built-in fragments that are superseded by project-level rules, duplicate behavioral guidance spread across multiple files, settings that nullify tool descriptions, and irrelevant domain examples.
4. **Produce actionable, reversible reduction steps** — specific tweakcc patches, `settings.json` edits, and CLAUDE.md / MEMORY.md amendments, each with a justification tied to the duplication or contradiction analysis.

The secondary goal is to create a persistent reference document for future prompt maintenance: understanding how the system responds to Claude Code version updates, what requires revalidation when fragments change, and how the symbol table of obfuscated binary identifiers maps to human-readable function names.

### What This Document Is Not

This is not a guide to using Claude Code generally. It is a technical reference for a specific installation. The Spank project context is used as a concrete case study for reduction decisions. The token counts, file paths, and settings are installation-specific. The binary analysis (§2, §A) and the catalog (§3) are version-specific to v2.1.74 and may shift with minor releases.

---

## 1. System Architecture and Terminology

### 1.1 Key Concepts

**Context window** — The total token budget available to the model per API call. For the Anthropic Claude API, this is 200K tokens for Claude 3/4 Sonnet/Opus models. The window is shared between input (system prompt + conversation) and output (model response).

**Compaction (Claude Code client)** — When the conversation history grows to the auto-compaction threshold, Claude Code invokes a summarization path and replaces conversation history with a structured summary. **Threshold sources (reconcile per version):** (1) Official documentation: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` as a percentage of context capacity; default **approximately 95%**; values above the default threshold have no effect ([env-vars](https://code.claude.com/docs/en/env-vars)). (2) Binary-derived formula from v2.1.74 analysis: `effectiveWindow − 13000` (e.g. 187K on a 200K window = 93.5%); `CLAUDE_CODE_AUTO_COMPACT_WINDOW` sets the token denominator for that percentage math. Set overrides in `settings.json` → `"env"` so the **Claude Code process** sees them — not only in `CLAUDE_ENV_FILE` unless that is how you intentionally load them. Telemetry may emit `context_limit_event`. The compaction process uses its own system prompt (`compact-prompt-*.md` fragments) and does not operate on the system prompt block — MEMORY.md and CLAUDE.md survive compaction because they are in the system prompt region, not conversation history.

**Compaction (Messages API, separate feature)** — Beta server-side strategy `compact_20260112` with default **trigger** **150,000 input tokens** (minimum 50,000), documented at [Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction). This is **not** the same knob as Claude Code’s `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`.

**Advanced tool use (platform pattern)** — For large MCP/tool libraries, Anthropic documents **Tool Search Tool**, deferred loading (`defer_loading`), and related patterns ([Advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use)). In Claude Code, `ENABLE_TOOL_SEARCH` controls MCP tool search behavior ([env-vars](https://code.claude.com/docs/en/env-vars)).

**Fragment** — A single `.md` file from the `system-prompts/` extraction directory. Fragments are string literals embedded in the Claude Code binary. Each fragment maps to a specific behavioral rule or section of the system prompt.

**Injection point** — The position in the assembled system prompt where a fragment or file content is inserted. Positions are partially ordered: identity (1) → tool rules (2) → coding discipline (3) → action safety (4) → tool prefs + CLAUDE.md (5) → git rules (6) → PR rules (7) → cached sections (8a–8j).

**Override declaration** — The header prepended to every injected user file: "Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written." This makes the model prioritize project-file rules over built-in fragments when they conflict.

**Cached section** — A section in the assembled system prompt that is computed once per session and keyed for cache reuse across turns. Cached via `cachedSection()` or `conditionalCachedSection()`. Examples: memory block, environment info, output style.

**Tool description** — The `description` field of a tool schema. Tool descriptions are loaded separately from the system prompt — they are not part of the `systemPromptAssembler()` output. Every tool Claude Code provides has a description loaded when the tool is active, regardless of permission state.

**System reminder** — An ephemeral `<system_reminder>` block injected mid-conversation when a specific runtime event fires (file opened in IDE, compaction occurred, hook ran, etc.). Reminders are not in the baseline system prompt and do not persist across turns.

**Permission mode** — The approval behavior for tool calls. Modes: `default` (prompts on first use per session), `acceptEdits` (auto-approves file edits), `plan` (analysis-only), `dontAsk` (auto-denies unless pre-approved), `bypassPermissions` (skips all checks). Set in `settings.json` as `permissions.defaultMode`.

**Toolset** — A tweakcc feature. A named subset of tools that Claude is allowed to call. The UI-level filter runs at `toolFetchingMemoFilter()` before the API request, so tools outside the toolset don't appear in the tools array and have zero token cost.

**Skill** — A `.md` file in the skills directory invocable via slash command (e.g., `/loop`, `/debug`). Skills are loaded on demand by the Skill tool and are not in the baseline context. After first invocation in a session, `system-reminder-invoked-skills.md` persists the skill as active for that session.

**Data file** — A `data-*.md` file referenced within a skill and loaded when the skill reads it. Neither skills nor data files contribute to baseline context.

**Subagent** — A model call spawned by the Agent/Task tool. Two types: (a) *fresh* — spawned with `subagent_type`, runs its own `systemPromptAssembler()` call with the same project CLAUDE.md and MEMORY.md, starts with empty conversation history; (b) *context-inheriting* — no `subagent_type`, inherits the parent's assembled system prompt verbatim and does not re-run `systemPromptAssembler()`.

**tweakcc** — The tool at `/Users/walter/Work/Claude/tweakcc` (Piebald AI) that patches the local Claude Code binary. It modifies string literals in the compiled Bun binary to replace built-in prompt fragments with user-edited versions, or to inject new behaviors. Patches are applied to `cli.js` (extracted from the native binary), then the binary is rebuilt. The `systemPrompts.ts` patch engine finds each prompt via regex match on the binary content and replaces it with the content of the corresponding `.md` file.

### 1.2 File Locations

| File / Directory | Purpose |
|---|---|
| `/opt/homebrew/Caskroom/claude-code/2.1.74/claude` | Native binary (Bun-compiled, arm64, 182MB) |
| `/opt/homebrew/bin/claude` | Symlink to above |
| `~/.claude/settings.json` | Global user settings (permissions, model, hooks, env) |
| `.claude/settings.json` | Project settings (committed to git, shared with team) |
| `.claude/settings.local.json` | Local project settings (gitignored, personal overrides) |
| `~/.claude.json` | OAuth session, MCP configs, per-project state, caches |
| `~/.claude/CLAUDE.md` | Global user instructions (lowest project-file priority) |
| `<project>/CLAUDE.md` | Project instructions (highest project-file authority) |
| `~/.claude/projects/<encoded-path>/memory/` | Memory directory for this project |
| `~/.claude/projects/<encoded-path>/memory/MEMORY.md` | Primary project memory file |
| `~/.claude/projects/<encoded-path>/memory/summary.md` | Auto-managed session notes (distinct from MEMORY.md) |
| `/Users/walter/Work/Claude/claude-code-system-prompts/` | Extracted prompt fragments (all 245 files) |
| `/Users/walter/Work/Claude/tweakcc/` | tweakcc source and config |

### 1.3 Configuration Scope Hierarchy

Settings apply in a layered hierarchy. More specific scopes take precedence:

| Priority | Scope | Location | Shared? |
|---|---|---|---|
| 5 (highest) | Managed | Server-managed / MDM / registry | IT-deployed |
| 4 | Local | `.claude/settings.local.json` | No (gitignored) |
| 3 | Project | `.claude/settings.json` | Yes (committed) |
| 2 | User | `~/.claude/settings.json` | No |
| 1 (lowest) | Default | Hard-coded binary defaults | — |

Managed settings (MDM, registry, or system-level `managed-settings.json`) cannot be overridden by any other scope. For enterprise deployments, `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`, and `allowManagedMcpServersOnly` lock down user/project overrides.

---

## 2. The Five Context Streams

Claude Code builds the context it sends to the Anthropic API from five distinct streams, concatenated in order. They are not deduplicated against each other.

```
[Stream 1: System Prompt]
  ↓  assembled by systemPromptAssembler() from fragment array
  ↓  includes: identity, rules, coding discipline, action safety,
  ↓            CLAUDE.md injection, git/PR rules, cached sections
  ↓            (memory, env, output style, MCP, scratchpad, etc.)

[Stream 2: Tool Definitions]
  ↓  one description per active tool, loaded alongside tool schema
  ↓  NOT part of systemPromptAssembler() output
  ↓  Bash: assembled from 35 modular fragments

[Stream 3: Conversation History]
  ↓  all user and assistant turns so far


[Stream 4: System Reminders]
  ↓  <system_reminder> blocks injected on runtime events
  ↓  ephemeral: present for one turn then gone

[Stream 5: (API-level) SDK System Prompt]
  ↓  --append-system-prompt / SDK system field
  ↓  appended after the assembled system prompt
  ↓  triggers IDENTITY_SDK identity string variant
```

### 2.1 Stream 1: System Prompt Assembly

The main system prompt is assembled by `systemPromptAssembler()` once at session start and reassembled after each compaction. The assembly is an ordered array of string sections, filtered for `null`:

```javascript
// Pseudocode reconstruction from binary analysis (CPrompts.md §System Prompt Construction)
async function systemPromptAssembler(_, toolNames, envArgs, mcpServers) {
  if (simpleModeCheck(process.env.CLAUDE_CODE_SIMPLE))
    return [`You are Claude Code...\nCWD: ${cwdGetter()}\nDate: ${dateGetter()}`];

  let cwd = cwdGetter();
  let [claudemdContent, outputStyle, envInfo] =
    await Promise.all([claudemdReader(cwd), outputStyleReader(), envInfoBuilder(toolNames, envArgs)]);

  // Cached sections computed once per session
  let cachedSections = [
    cachedSection("memory",              () => memoryPromptBuilder()),
    cachedSection("ant_model_override",  () => modelOverrideSection()),
    cachedSection("env_info_simple",     () => envInfoBuilder(toolNames, envArgs)),
    cachedSection("language",            () => languageSection(appState.language)),
    cachedSection("output_style",        () => outputStyleSection(outputStyle)),
    conditionalCachedSection("mcp_instructions", () => hasMcpServers() ? null : mcpInstructionsSection(mcpServers), "..."),
    cachedSection("scratchpad",          () => scratchpadSection()),
    cachedSection("frc",                 () => featureFlagsSection(toolNames)),
    cachedSection("summarize_tool_results", () => summarizeToolResultsText),
    cachedSection("brief",               () => briefModeSection()),
  ];
  let resolvedCached = await cachedSectionsEvaluator(cachedSections);

  return [
    identitySection(outputStyle),          // pos 1
    toolRulesSection(toolNames),           // pos 2
    outputStyle?.keepCodingInstructions !== false ? codingDisciplineSection() : null,  // pos 3
    actionSafetySection(),                 // pos 4
    claudemdInjector(toolNames, claudemdContent),  // pos 5 — tool prefs + CLAUDE.md
    gitRulesSection(),                     // pos 6
    prRulesSection(),                      // pos 7
    ...resolvedCached                      // pos 8a–8j
  ].filter(s => s !== null);
}
```

The injection order matters for authority: position 5 (`claudemdInjector`) injects CLAUDE.md with the OVERRIDE declaration, placing it after all the built-in rule sections. The model encounters the built-in rules first, then the project's override — which is why "OVERRIDE any default behavior" is necessary and effective.

**Position 8a** (`memory`) is the first cached section: this is where MEMORY.md content appears, built by `memoryPromptBuilder()`. The MEMORY block is: auto-memory header from `autoMemoryHeaderText()` + MEMORY.md file content (truncated at `MEMORY_LINE_LIMIT = 200` lines).

### 2.2 Stream 2: Tool Definitions

Tool descriptions are loaded alongside each tool's JSON schema when the tool is registered for a session. They are completely separate from the `systemPromptAssembler()` pipeline. The model sees tool descriptions alongside the `tools` array in the API request, not inside the `system` field.

Critical implication: **pre-approving a tool in `settings.json` does not reduce the tool description's token cost.** The tool description is always loaded when the tool is available, regardless of whether use of the tool is automatically approved.

The Bash tool is unique: its description is assembled from 35 modular fragments concatenated at runtime. All other tools have 1–3 description files. Team tools (TeammateTool, SendMessageTool, TeamDeleteTool) have their descriptions loaded only when `teamToolsEnabled()` returns true — i.e., only in active team/swarm sessions with team name, worker ID, and team context all present. Solo sessions pay zero tokens for team tool descriptions.

### 2.3 Stream 3: Conversation History

Includes all user messages, assistant responses, and tool results from the current session.

### 2.4 Stream 4: System Reminders

`<system_reminder>` blocks are injected into the conversation context at specific runtime events, appearing as assistant-role blocks or synthetic user messages (the exact mechanism is internal). They fire once per event and do not recur unless the event recurs. The largest — `system-reminder-plan-mode-is-active-5-phase.md` at 1297 tokens — fires every turn while plan mode is active.

### 2.5 Compaction Prompt (Stream 1 variant)

At compaction time, `systemPromptAssembler()` is called with a different fragment set. The compaction-specific fragments (`system-prompt-context-compaction-summary.md`, `analysis-instructions-for-full-compact-prompt-*.md`) replace normal content. The compact prompt also contains the `compactInstructionsInjectionPoint` placeholder (`${aJT}`) where `/compact [custom instructions]` arguments are interpolated.

The compact prompt structure:
```
Your task is to create a detailed summary of the conversation so far...
${compactInstructionsInjectionPoint}           ← custom /compact argument injected here
[9 structured sections: Primary Request, Key Concepts, Files, Errors,
 Problem Solving, User Messages, Pending Tasks, Current Work, Next Step]
```

A `pre_compact` hook can return exit code 2 to block compaction or output additional instructions merged into `${compactInstructionsInjectionPoint}`.

---

## 3. Complete Prompt Catalog

The 245 extracted files divide into seven categories. Token counts are from the extraction repository's `README.md`. Files marked **conditional** load only when the specified feature is active. Template variables (e.g., `${BASH_TOOL_NAME}`) are not interpolated in extracted files; actual session counts are ±20 tokens.

**Estimated baseline token cost (standard solo session, no optional features):**
- Always-loaded system prompt fragments: ~4,200 tokens
- Standard tool descriptions (Bash + core tools, no team): ~6,000 tokens
- CLAUDE.md injection (project-specific): ~800–2,000 tokens
- MEMORY.md injection (header + content): ~400–800 tokens
- **Total baseline: ~11,400–13,000 tokens** of a 200K budget

### 3.1 System Prompt Fragments (67 files)

| File | Tokens | Load Condition |
|------|--------|----------------|
| `system-prompt-system-section.md` | 156 | always |
| `system-prompt-executing-actions-with-care.md` | 541 | always |
| `system-prompt-doing-tasks-no-unnecessary-additions.md` | 78 | always |
| `system-prompt-doing-tasks-avoid-over-engineering.md` | 30 | always |
| `system-prompt-doing-tasks-no-premature-abstractions.md` | 60 | always |
| `system-prompt-doing-tasks-minimize-file-creation.md` | 47 | always |
| `system-prompt-doing-tasks-no-compatibility-hacks.md` | 52 | always |
| `system-prompt-doing-tasks-no-time-estimates.md` | 47 | always |
| `system-prompt-doing-tasks-no-unnecessary-error-handling.md` | 64 | always |
| `system-prompt-doing-tasks-read-before-modifying.md` | 46 | always |
| `system-prompt-doing-tasks-security.md` | 67 | always |
| `system-prompt-doing-tasks-software-engineering-focus.md` | 104 | always |
| `system-prompt-doing-tasks-help-and-feedback.md` | 24 | always |
| `system-prompt-doing-tasks-ambitious-tasks.md` | 47 | always |
| `system-prompt-doing-tasks-blocked-approach.md` | 90 | always |
| `system-prompt-tone-and-style-concise-output-detailed.md` | 89 | one of three output style variants |
| `system-prompt-tone-and-style-concise-output-short.md` | 16 | one of three output style variants |
| `system-prompt-output-efficiency.md` | 177 | one of three output style variants |
| `system-prompt-tone-and-style-code-references.md` | 39 | always |
| `system-prompt-tool-usage-read-files.md` | 29 | always |
| `system-prompt-tool-usage-edit-files.md` | 26 | always |
| `system-prompt-tool-usage-create-files.md` | 30 | always |
| `system-prompt-tool-usage-search-files.md` | 26 | always |
| `system-prompt-tool-usage-search-content.md` | 30 | always |
| `system-prompt-tool-usage-reserve-bash.md` | 75 | always |
| `system-prompt-tool-usage-direct-search.md` | 39 | always |
| `system-prompt-tool-usage-delegate-exploration.md` | 95 | always |
| `system-prompt-tool-usage-subagent-guidance.md` | 103 | always |
| `system-prompt-tool-usage-task-management.md` | 73 | always |
| `system-prompt-tool-usage-skill-invocation.md` | 102 | always |
| `system-prompt-parallel-tool-call-note-part-of-tool-usage-policy.md` | 102 | always |
| `system-prompt-writing-subagent-prompts.md` | 365 | always |
| `system-prompt-subagent-delegation-examples.md` | 588 | always |
| `system-prompt-agent-thread-notes.md` | 216 | always |
| `system-prompt-fork-usage-guidelines.md` | 339 | always |
| `system-prompt-git-status.md` | 97 | session start only (once) |
| `system-prompt-scratchpad-directory.md` | 170 | conditional: scratchpad feature |
| `system-prompt-auto-mode.md` | 188 | conditional: auto mode |
| `system-prompt-worker-instructions.md` | 272 | conditional: worker/fork role |
| `system-prompt-agent-memory-instructions.md` | 337 | conditional: memory enabled |
| `system-prompt-description-part-of-memory-instructions.md` | 148 | conditional: memory enabled |
| `system-prompt-memory-system-private-feedback.md` | 112 | conditional: memory enabled |
| `system-prompt-team-memory-content-display.md` | 52 | conditional: team memory |
| `system-prompt-teammate-communication.md` | 130 | conditional: team/swarm mode |
| `system-prompt-how-to-use-the-sendusermessage-tool.md` | 283 | conditional: team/swarm mode |
| `system-prompt-hooks-configuration.md` | 1482 | conditional: hooks configured |
| `system-prompt-agent-summary-generation.md` | 178 | conditional: agent summary |
| `system-prompt-tool-use-summary-generation.md` | 171 | conditional: tool summary |
| `system-prompt-phase-four-of-plan-mode.md` | 142 | conditional: plan mode |
| `system-prompt-one-of-six-rules-for-using-sleep-command.md` | 23 | conditional: Sleep tool present |
| `system-prompt-censoring-assistance-with-malicious-activities.md` | 98 | always |
| `system-prompt-chrome-browser-mcp-tools.md` | 156 | conditional: browser MCP |
| `system-prompt-claude-in-chrome-browser-automation.md` | 759 | conditional: browser MCP |
| `system-prompt-learning-mode.md` | 1042 | conditional: learning mode |
| `system-prompt-learning-mode-insights.md` | 142 | conditional: learning mode |
| `system-prompt-skillify-current-session.md` | 1882 | conditional: skillify feature |
| `system-prompt-option-previewer.md` | 151 | conditional: option preview |
| `system-prompt-context-compaction-summary.md` | 278 | compaction only (SDK variant) |
| `system-prompt-analysis-instructions-for-full-compact-prompt-full-conversation.md` | 182 | compaction only |
| `system-prompt-analysis-instructions-for-full-compact-prompt-recent-messages.md` | 178 | compaction only |
| `system-prompt-analysis-instructions-for-full-compact-prompt-minimal-and-via-feature-flag.md` | 157 | compaction only |
| `system-prompt-insights-at-a-glance-summary.md` | 569 | conditional: insights feature |
| `system-prompt-insights-friction-analysis.md` | 139 | conditional: insights feature |
| `system-prompt-insights-on-the-horizon.md` | 148 | conditional: insights feature |
| `system-prompt-insights-session-facets-extraction.md` | 310 | conditional: insights feature |
| `system-prompt-insights-suggestions.md` | 748 | conditional: insights feature |

**Note on `includeGitInstructions`.** The `settings.json` key `includeGitInstructions` (default: `true`) controls whether the git commit/PR workflow fragments are included. Setting it to `false` suppresses `tool-description-bash-git-commit-and-pr-creation-instructions.md` (1558 tks) without requiring a tweakcc patch. This is the preferred removal method for that fragment.

### 3.2 Tool Descriptions (73 files + 1 parameter)

Tool descriptions are loaded when their tool is in the active tool list. Team tools are gated by `teamToolsEnabled()` and absent from solo sessions.

**Always loaded — standard solo session:**
Bash, Edit, Write, Glob, Grep, Read, WebFetch, WebSearch, AskUserQuestion, Agent, TodoWrite, EnterPlanMode, ExitPlanMode, ToolSearch, Skill.

**Conditionally loaded:**

| Tool(s) | Condition |
|---|---|
| TeammateTool, SendMessageTool, TeamDeleteTool, TaskList | `teamToolsEnabled()` — team/swarm session only |
| Computer | Browser automation MCP or computer-use mode |
| LSP | LSP integration enabled |
| NotebookEdit | Jupyter notebook support active |
| Sleep | Sleep tool enabled |
| CronCreate | Cron scheduling feature active |
| TaskCreate | Task creation feature active |
| EnterWorktree, ExitWorktree | Worktree mode enabled |

**Deferred loading — MCP tools only:**
The Anthropic API supports `defer_loading: true` as a tool definition property. In Claude Code, this applies only to MCP-sourced tools (`tool.isMcp === true`). Built-in tools listed above are NOT deferred — their schemas are sent to the API at session start. MCP tool schemas are withheld from the initial API request and listed as available-deferred-tools; the model fetches them on demand via ToolSearch.

**Tools accessible via ToolSearch (on-demand):**
When a built-in tool's description is removed via tweakcc, the tool schema remains registered and the model can still call it — but it won't spontaneously do so because it has no description. ToolSearch (`tool-description-toolsearch-second-part.md`, 202 tks) lets the model fetch any tool's description at runtime. This is the safe removal path for tweakcc: description token cost eliminated at session start, access preserved on demand. The tradeoff: the model must be explicitly prompted to use the tool or must run a ToolSearch to discover it.

**ToolSearch query syntax** (from `tool-description-toolsearch-second-part.md`):
```
select:Read,Edit,Grep   — fetch exact tools by name (comma-separated)
notebook jupyter        — keyword search; returns up to max_results best matches
+slack send             — require "slack" in tool name, rank by remaining terms
```
Recommended use: invoke ToolSearch by name when a specific tool's description has been removed and you need to restore it for the current session (`select:TodoWrite`), or to explore what tools are available for a new task domain (`notebook jupyter`). For most standard tasks, built-in tool schemas are already in context and ToolSearch is not needed.

| File | Tokens | Tool |
|------|--------|------|
| `tool-description-todowrite.md` | 2161 | TodoWrite |
| `tool-description-teammatetool.md` | 1645 | TeammateTool (team only) |
| `tool-description-bash-git-commit-and-pr-creation-instructions.md` | 1558 | Bash (git workflow) |
| `tool-description-sendmessagetool.md` | 1205 | SendMessageTool (team only) |
| `tool-description-agent-usage-notes.md` | 931 | Agent/Task tool |
| `tool-description-enterplanmode.md` | 878 | EnterPlanMode |
| `tool-description-exitplanmode.md` | 417 | ExitPlanMode |
| `tool-description-readfile.md` | 440 | ReadFile |
| `tool-description-webfetch.md` | 297 | WebFetch |
| `tool-description-websearch.md` | 321 | WebSearch |
| `tool-description-grep.md` | 300 | Grep |
| `tool-description-askuserquestion.md` | 287 | AskUserQuestion |
| `tool-description-glob.md` | 122 | Glob |
| `tool-description-edit.md` | 246 | Edit |
| `tool-description-skill.md` | 326 | Skill |
| `tool-description-write.md` | 129 | Write |
| `tool-description-croncreate.md` | 738 | CronCreate |
| `tool-description-taskcreate.md` | 528 | TaskCreate |
| `tool-description-teamdelete.md` | 154 | TeamDelete (team only) |
| `tool-description-enterworktree.md` | 359 | EnterWorktree |
| `tool-description-exitworktree.md` | 527 | ExitWorktree |
| `tool-description-lsp.md` | 255 | LSP |
| `tool-description-notebookedit.md` | 121 | NotebookEdit |
| `tool-description-sleep.md` | 154 | Sleep |
| `tool-description-computer.md` | 161 | Computer (browser) |
| `tool-description-agent-when-to-launch-subagents.md` | 186 | Agent/Task tool |
| `tool-description-askuserquestion-preview-field.md` | 134 | AskUserQuestion |
| `tool-description-sendmessagetool-non-agent-teams.md` | 133 | SendMessageTool |
| `tool-description-tasklist-teammate-workflow.md` | 133 | TaskList (team only) |
| `tool-description-toolsearch-second-part.md` | 202 | ToolSearch |
| `tool-parameter-computer-action.md` | 251 | Computer (parameter) |
| **Bash description (35 modular pieces):** | | |
| `tool-description-bash-overview.md` | 19 | Bash |
| `tool-description-bash-prefer-dedicated-tools.md` | 71 | Bash |
| `tool-description-bash-built-in-tools-note.md` | 53 | Bash |
| `tool-description-bash-alternative-communication.md` | 18 | Bash |
| `tool-description-bash-alternative-content-search.md` | 27 | Bash |
| `tool-description-bash-alternative-edit-files.md` | 27 | Bash |
| `tool-description-bash-alternative-file-search.md` | 26 | Bash |
| `tool-description-bash-alternative-read-files.md` | 27 | Bash |
| `tool-description-bash-alternative-write-files.md` | 29 | Bash |
| `tool-description-bash-command-description.md` | 71 | Bash |
| `tool-description-bash-parallel-commands.md` | 72 | Bash |
| `tool-description-bash-sequential-commands.md` | 42 | Bash |
| `tool-description-bash-semicolon-usage.md` | 29 | Bash |
| `tool-description-bash-no-newlines.md` | 24 | Bash |
| `tool-description-bash-maintain-cwd.md` | 41 | Bash |
| `tool-description-bash-working-directory.md` | 37 | Bash |
| `tool-description-bash-verify-parent-directory.md` | 38 | Bash |
| `tool-description-bash-quote-file-paths.md` | 35 | Bash |
| `tool-description-bash-timeout.md` | 83 | Bash |
| `tool-description-bash-sleep-keep-short.md` | 29 | Bash |
| `tool-description-bash-sleep-no-polling-background-tasks.md` | 37 | Bash |
| `tool-description-bash-sleep-run-immediately.md` | 21 | Bash |
| `tool-description-bash-sleep-use-check-commands.md` | 34 | Bash |
| `tool-description-bash-git-avoid-destructive-ops.md` | 58 | Bash |
| `tool-description-bash-git-never-skip-hooks.md` | 59 | Bash |
| `tool-description-bash-git-prefer-new-commits.md` | 22 | Bash |
| `tool-description-bash-sandbox-default-to-sandbox.md` | 38 | Bash (sandbox) |
| `tool-description-bash-sandbox-mandatory-mode.md` | 34 | Bash (sandbox) |
| `tool-description-bash-sandbox-per-command.md` | 52 | Bash (sandbox) |
| `tool-description-bash-sandbox-no-exceptions.md` | 17 | Bash (sandbox) |
| `tool-description-bash-sandbox-no-sensitive-paths.md` | 36 | Bash (sandbox) |
| `tool-description-bash-sandbox-explain-restriction.md` | 36 | Bash (sandbox) |
| `tool-description-bash-sandbox-failure-evidence-condition.md` | 48 | Bash (sandbox) |
| `tool-description-bash-sandbox-evidence-list-header.md` | 15 | Bash (sandbox) |
| `tool-description-bash-sandbox-evidence-access-denied.md` | 15 | Bash (sandbox) |
| `tool-description-bash-sandbox-evidence-network-failures.md` | 17 | Bash (sandbox) |
| `tool-description-bash-sandbox-evidence-operation-not-permitted.md` | 18 | Bash (sandbox) |
| `tool-description-bash-sandbox-evidence-unix-socket-errors.md` | 11 | Bash (sandbox) |
| `tool-description-bash-sandbox-response-header.md` | 17 | Bash (sandbox) |
| `tool-description-bash-sandbox-retry-without-sandbox.md` | 33 | Bash (sandbox) |
| `tool-description-bash-sandbox-user-permission-prompt.md` | 14 | Bash (sandbox) |
| `tool-description-bash-sandbox-tmpdir.md` | 102 | Bash (sandbox) |
| `tool-description-bash-sandbox-adjust-settings.md` | 26 | Bash (sandbox) |

### 3.3 Agent Prompts (32 files)

Agent prompts are the system prompts for specific subagent types. They load only when that subagent is invoked and do not contribute to the parent session's baseline. Notable large ones:

| File | Tokens | Invocation |
|------|--------|------------|
| `agent-prompt-security-monitor-2.md` | 2966 | security monitor subagent |
| `agent-prompt-security-monitor.md` | 2675 | security monitor subagent |
| `agent-prompt-session-memory-update-instructions.md` | 756 | auto-memory update subagent |
| `agent-prompt-agent-creation-architect.md` | ~600 | agent creation helper |
| All others | < 500 | various |

The 32 types include: Explore, Plan mode, security monitor, agent creation architect, CLAUDE.md generator, session title/branch, quick git commit, quick PR, conversation summarization, memory update, session search, hook condition evaluator, code reviewer, batch command, worker fork, webfetch summarizer, bash command description writer, bash prefix detection, status line setup, prompt suggestion generator, claude-guide, update-magic-docs, verification specialist, PR comments, review PR, security review, coding session title, common suffix response format, worker fork execution, recent message summarization, session memory update, determine memory files to attach.

### 3.4 System Reminders (37 files)

| File | Tokens | Trigger Event |
|------|--------|---------------|
| `system-reminder-plan-mode-is-active-5-phase.md` | 1297 | plan mode active (each turn) |
| `system-reminder-plan-mode-is-active-iterative.md` | 919 | plan mode variant (each turn) |
| `system-reminder-team-coordination.md` | 250 | team/swarm mode active |
| `system-reminder-plan-mode-is-active-subagent.md` | 307 | plan mode in subagent |
| `system-reminder-plan-mode-re-entry.md` | 236 | re-entering plan mode |
| `system-reminder-btw-side-question.md` | 244 | `/btw` command used |
| `system-reminder-todowrite-reminder.md` | 98 | TodoWrite reminder |
| `system-reminder-task-tools-reminder.md` | 123 | task tracking reminder |
| `system-reminder-team-shutdown.md` | 136 | team shutdown event |
| `system-reminder-token-usage.md` | 39 | token stats display |
| `system-reminder-usd-budget.md` | 42 | budget stats display |
| `system-reminder-memory-file-contents.md` | 38 | memory file loaded |
| `system-reminder-nested-memory-contents.md` | 33 | nested memory loaded |
| `system-reminder-invoked-skills.md` | 33 | skill invoked (persists for session) |
| `system-reminder-compact-file-reference.md` | 57 | post-compaction |
| `system-reminder-file-opened-in-ide.md` | 37 | IDE file open event |
| `system-reminder-lines-selected-in-ide.md` | 66 | IDE selection event |
| `system-reminder-file-modified-by-user-or-linter.md` | 97 | external file modification |
| `system-reminder-new-diagnostics-detected.md` | 35 | linter error detected |
| `system-reminder-hook-success.md` | 29 | hook executed successfully |
| `system-reminder-hook-blocking-error.md` | 52 | hook blocked tool call |
| `system-reminder-hook-stopped-continuation.md` | 30 | hook stopped continuation |
| `system-reminder-hook-additional-context.md` | 35 | hook injected context |
| `system-reminder-hook-stopped-continuation-prefix.md` | 12 | hook prefix variant |
| `system-reminder-exited-plan-mode.md` | 73 | plan mode exit |
| `system-reminder-verify-plan-reminder.md` | 47 | plan verification reminder |
| `system-reminder-plan-file-reference.md` | 62 | plan file reference |
| `system-reminder-task-status.md` | 18 | task status update |
| `system-reminder-session-continuation.md` | 37 | session resumed |
| `system-reminder-agent-mention.md` | 45 | agent mentioned in message |
| `system-reminder-output-style-active.md` | 32 | output style activated |
| `system-reminder-malware-analysis-after-read-tool-call.md` | 87 | malware context after read |
| `system-reminder-file-truncated.md` | 74 | file too large to read fully |
| `system-reminder-file-shorter-than-offset.md` | 59 | read offset error |
| `system-reminder-file-exists-but-empty.md` | 27 | empty file read |
| `system-reminder-mcp-resource-no-content.md` | 41 | MCP resource empty |
| `system-reminder-mcp-resource-no-displayable-content.md` | 43 | MCP resource not displayable |

### 3.5 Skills (9 files, on-demand)

| File | Tokens | Invocation Command |
|------|--------|--------------------|
| `skill-build-with-claude-api.md` | 5144 | `build-with-claude-api` |
| `skill-create-verifier-skills.md` | 2625 | `create-verifier-skills` |
| `skill-verification-specialist.md` | 2472 | `verification-specialist` |
| `skill-update-claude-code-config.md` | 1232 | `update-claude-code-config` |
| `skill-loop-slash-command.md` | 984 | `/loop` |
| `skill-stuck-slash-command.md` | 865 | `/stuck` |
| `skill-simplify.md` | 822 | `simplify` |
| `skill-debugging.md` | 412 | `/debug` |
| `skill-build-with-claude-api-reference-guide.md` | 410 | (referenced by build-with-claude-api) |

### 3.6 Data Files (26 files, on-demand)

Data files range from 292 tokens (`data-session-memory-template.md`) to 5106 tokens (`data-tool-use-reference-python.md`). Total ≈56,000 tokens. None enters baseline context; all are loaded on demand by skill invocations.

---

## 4. Execution and Permission Layering

Understanding the precise sequence of events for a tool call is necessary to understand why settings.json and tool descriptions interact the way they do.

### 4.1 Tool Call Execution Sequence

```
Step 1: Model reasoning
  ↳ Model reads assembled context (system prompt + tool descriptions + conversation)
  ↳ Model decides to emit a tool call, e.g.: Bash("grep -r 'foo' src/")

Step 2: Tool call emitted
  ↳ Model outputs structured tool_use block with tool name + parameters

Step 3: Claude Code process intercepts
  ↳ Tool call is not executed yet; it is held for permission evaluation

Step 4: Permission check (settings.json)
  ↳ Deny rules checked first — if match, block immediately (no dialog)
  ↳ Ask rules checked — if match, require confirmation per use
  ↳ Allow rules checked — if match, auto-approve (no dialog)
  ↳ No match → defaultMode behavior (prompt, acceptEdits, etc.)
  ↳ NOTE: This step operates on the already-emitted tool call; it does not
    affect what tool descriptions were loaded in Step 1

Step 5: PreToolUse hook (if configured)
  ↳ Hook script runs with tool name, parameters, and session context
  ↳ Can return {continue: false} or {permissionDecision: "deny"} to block
  ↳ Can return {stopReason: "..."} to halt session
  ↳ Hook runs AFTER permission approval — can override a Step 4 allow
  ↳ Hooks have higher enforcement authority than settings.json permissions

Step 6: Tool executes
  ↳ Subprocess spawned, or file operation performed

Step 7: PostToolUse hook (if configured)
  ↳ Hook script receives tool result
  ↳ Can inject additional context into the conversation

Step 8: Tool result returned to model
  ↳ Result appears as tool_result block in conversation history
  ↳ Model incorporates it into next response
```

**Behavioral interference (not contradiction).** Pre-approving `grep` at Step 4 does not suppress the tool description's guidance ("use Grep instead of grep") at Step 1. Both coexist in context. The behavioral effect: the model receives the "don't use grep" instruction, but never encounters the friction signal (approval dialog) that would reinforce it. Over many sessions, the guidance loses influence because it produces no enforcement feedback. Removing the `Bash(grep:*)` pre-approval from `settings.json` restores the approval dialog as a correction signal.

### 4.2 Permission Rule Syntax

Rules use format `Tool` or `Tool(specifier)`. Deny rules take precedence over allow/ask. First matching rule wins.

```json
{
  "permissions": {
    "deny": [
      "Bash(rm *)",
      "Bash(git push --force *)",
      "Read(./.env)",
      "Read(./secrets/**)"
    ],
    "allow": [
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(pytest *)",
      "Read(src/**)"
    ],
    "ask": [
      "Bash(git commit *)",
      "Bash(pip install *)"
    ],
    "defaultMode": "default"
  }
}
```

Tool-specific specifier formats:
- `Bash(command prefix)` — matches commands starting with prefix; `*` is a glob wildcard
- `Read(path)` — matches file reads; supports `**` globs
- `Edit(path)` — matches file edits
- `WebFetch(domain:example.com)` — matches fetches to a specific domain
- `MCP(server:toolname)` — matches MCP tool calls
- `Agent(subagent_type)` — matches subagent launches of that type

### 4.3 Hook Configuration

Hooks are configured in `settings.json` under the `hooks` key. Event types:

| Hook Event | When It Fires |
|---|---|
| `PreToolUse` | Before any tool executes (after permission check) |
| `PostToolUse` | After any tool executes |
| `Notification` | On notification events (background task complete, etc.) |
| `Stop` | When session ends, compact, clear, resume, logout |
| `SubagentStop` | When a subagent completes |
| `PreCompact` | Before compaction; exit code 2 blocks compaction |

Example hook that blocks `git push` unless confirmed:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/pre-bash.sh"
          }
        ]
      }
    ]
  }
}
```

Hook output format (stdout JSON):
```json
{
  "continue": true,
  "stopReason": null,
  "suppressOutput": false,
  "permissionDecision": "allow",
  "additionalContext": "Context string injected into conversation"
}
```

Setting `permissionDecision: "deny"` in a PreToolUse hook overrides a Step 4 allow — hooks have absolute authority to block any tool call regardless of permission settings.

When hooks are configured, `system-prompt-hooks-configuration.md` (1482 tokens) is loaded into the system prompt, describing the hook protocol to the model. If no hooks are configured in any settings scope, this fragment does not fire.

---

## 5. CLAUDE.md and MEMORY.md — Authority and Mechanics

### 5.1 Injection Order and Authority

Two project files are injected into the system prompt (positions 5 and 8a) with different authority levels:

| File | Injection Point | Override Declaration? | Survives Compaction? | Authority |
|---|---|---|---|---|
| `CLAUDE.md` | System prompt position 5 | Yes ("OVERRIDE any default behavior") | Yes (re-read from disk every session) | Highest: overrides all built-in fragments |
| `MEMORY.md` | System prompt position 8a | No | Yes (re-read from disk every session) | State only: behavioral guidance here is advisory |

AGENTS.md has been removed from the Spank project. Its content (voice, design, markdown, security, git conventions, prompt separators) is now consolidated into CLAUDE.md §1–§12, gaining OVERRIDE authority and compaction persistence. The authority gap — AGENTS.md content entered context as a tool result in conversation history with no override declaration and aged out during compaction — is resolved.

### 5.2 CLAUDE.md Fallback Names (binary mechanism)

Claude Code's native `claudemdReader()` checks for alternative names only in the `else` branch — i.e., only when CLAUDE.md is absent. The tweakcc `agentsMd.ts` patch extends this to enable AGENTS.md, GEMINI.md, and QWEN.md as alternative names. Without tweakcc, fallback search order: `AGENTS.md` → `GEMINI.md` → `QWEN.md`. Not applicable to projects that have CLAUDE.md.

### 5.3 MEMORY.md Invocation Sequence

MEMORY.md is injected at position 8a — the first cached section, after all fixed fragments (positions 1–7). The injected block is built by `memoryPromptBuilder()`:

```
[auto-memory header from autoMemoryHeaderText()]
  "# auto memory"
  "You have a persistent auto memory directory at <path>."
  "Its contents persist across conversations."
  ""
  "As you work, consult your memory files to build on previous experience."
  ""
  "## How to save memories:"
  "- Organize memory semantically by topic, not chronologically"
  "- Use the Write and Edit tools to update your memory files"
  "- `MEMORY.md` is always loaded into your conversation context"
  "  — lines after <MEMORY_LINE_LIMIT=200> will be truncated, so keep it concise"
  "- Create separate topic files (e.g., `debugging.md`, `patterns.md`)"

[MEMORY.md file content]
  Lines 1–200 of ~/.claude/projects/<encoded-path>/memory/MEMORY.md
  Lines 201+ are silently discarded
```

The `autoMemoryHeaderText()` header (~12 lines) plus MEMORY.md content together form the memory block at position 8a. The header is functionally equivalent to a condensed version of `system-prompt-agent-memory-instructions.md` (337 tokens) — both are in context simultaneously when memory is enabled. This is genuine duplication.

**MEMORY directory path resolution** (`memoryPathResolver()`):
```javascript
function memoryPathResolver() {
  let override = memoryDirEnvVar() ?? memoryDirConfig();
  if (override) return override;
  let projectsDir = path.join(homeDir(), "projects");
  return (path.join(projectsDir, encodeProjectPath(projectRootGetter()), "memory") + path.sep).normalize("NFC");
  // → ~/.claude/projects/-Users-walter-Work-Spank-spank-py/memory/
}
```

Environment variable: `CLAUDE_MEMORY_DIR` (checked by `memoryDirEnvVar()`). Config key: `autoMemoryDirectory` in `settings.json` (not accepted in project settings to prevent shared repos from redirecting memory writes).

**Two memory systems coexist** in the memory directory:
1. **MEMORY.md** — manually maintained project state file (bootstrap rules, session state, risks). Managed by developer and model per bootstrap rules.
2. **summary.md** — auto-managed session notes written by the `session-memory-update` subagent (invoked by `agent-prompt-session-memory-update-instructions.md`, 756 tks). Updated at lifecycle points; content follows `data-session-memory-template.md` format.

Both files load at position 8a. The memory-selector subagent (`agent-prompt-determine-memory-files-to-attach.md`) chooses which files to include from the memory directory.

### 5.4 Post-Compaction Context Structure

After compaction, the session context is:

```
[Position 1–7: fixed system prompt fragments] (same as fresh session)
[Position 8a: MEMORY.md block]               (re-read from disk — includes updates the model made before compaction)
[Position 8b–8j: other cached sections]      (same as fresh session)
--- system prompt boundary ---
[Compaction summary]                          (first block in conversation history)
[New turns since compaction]
```

MEMORY.md and the compaction summary provide complementary continuity: MEMORY.md carries persistent cross-session state (project facts, current risks, file paths); the compaction summary carries the immediate prior conversation (what was asked, what was attempted, what is pending).

---

## 6. Duplication Analysis

The project files inject instructions that may restate, extend, or contradict what built-in prompts already establish. Where they extend (add a more restrictive rule or project-specific procedure), both are necessary. Where they restate, the project file entry is a pure token cost with no behavioral benefit.

Files analyzed: `/Users/walter/Work/Spank/spank-py/CLAUDE.md`, `/Users/walter/.claude/projects/-Users-walter-Work-Spank-spank-py/memory/MEMORY.md`.

| Project File Section | Built-in Coverage | Relationship | Recommendation |
|---|---|---|---|
| CLAUDE.md §1: "no undue praise" | None in built-ins | Extension | Resolved — in CLAUDE.md |
| CLAUDE.md §1: "no time estimates" | `doing-tasks-no-time-estimates.md` (47 tks) | Restatement | Low priority; built-in covers it |
| CLAUDE.md §1: narrative depth vs. concise | `tone-and-style-concise-output-detailed.md` (89 tks): concise, no filler | Intentional override — CLAUDE.md wins | Resolved — CLAUDE.md §1 explicit |
| CLAUDE.md §8 git: "do NOT commit unless instructed" | `bash-git-commit-and-pr-creation-instructions.md` (1558 tks) | CLAUDE.md is more restrictive | Remove built-in via `includeGitInstructions: false` |
| CLAUDE.md §8: git commit/push/rebase approval | Same git fragments | CLAUDE.md is strictly more restrictive | Same; `includeGitInstructions: false` preferred |
| CLAUDE.md §8: rm, mv, cp, dd approval | `executing-actions-with-care.md` (541 tks) | CLAUDE.md more specific; built-in provides rationale | Keep both; different levels of specificity |
| CLAUDE.md §8: pip install, sudo approval | `doing-tasks-security.md` (67 tks) | Different scope; no overlap | Keep both |
| MEMORY bootstrap: "stop, do not execute" | `system-prompt-auto-mode.md` (188 tks) | Potential contradiction — MEMORY bootstrap wins due to override priority | MEMORY bootstrap is correct; auto-mode only fires in auto mode |
| MEMORY bootstrap: update trigger rules | `agent-memory-instructions.md` (337 tks) + `autoMemoryHeaderText()` header | **Restatement** | Remove update trigger paragraphs from MEMORY bootstrap |
| MEMORY bootstrap: MEMORY versioning/snapshot rule | None in built-ins | Extension (project-specific) | Keep |
| MEMORY bootstrap: permission block stopping rule | `executing-actions-with-care.md` (541 tks) | MEMORY is more specific (one retry max) | Keep in MEMORY; it's more restrictive |
| `settings.json`: pre-approved `cat`, `grep`, `find`, `sed`, `head`, `tail` | `bash-alternative-*.md` (6 files, ~154 tks) instruct to use dedicated tools instead | **Direct behavioral contradiction** | Revoke pre-approvals from `settings.json`; remove bash-alternative fragments |
| `settings.json`: `Bash(pip:*)`, `Bash(npm:*)`, `Bash(mv:*)`, `Bash(cp:*)`, `Bash(curl:*)` | CLAUDE.md §8 requires approval for these | **Conflict**: settings auto-approve, CLAUDE.md requires gate | Remove from `settings.json` allow list; restore intended approval gate |
| `settings.json`: `Bash(git:*)` wildcard | CLAUDE.md §8 requires approval for `git commit`, `git push`, `git rebase`, `git reset --hard`, `git branch -D`, `git clean -fd`, `git checkout .`, `git stash drop` | **Critical conflict**: wildcard pre-approves ALL git subcommands including destructive; `git push --force` runs without any dialog | Replace with safe subcommand allow list: `Bash(git diff *)`, `Bash(git log *)`, `Bash(git status *)`, `Bash(git fetch *)`, `Bash(git show *)` |
| `settings.json`: `Bash(wget:*)` | CLAUDE.md §8 requires approval for non-GET HTTP requests | **Conflict**: wildcard includes `wget -O existingfile` (overwrite), `wget --post-data` | Move to `ask` or remove |
| `settings.json`: `Bash(echo:*)` + `Write` | CLAUDE.md §8 requires approval for output redirection onto existing files and modifications to shell init files | **Conflict**: `echo "x" > ~/.bashrc` runs without approval (echo pre-approved); `Write` pre-approves any path including `~/.zshrc` | Add explicit `deny` entries for shell init dotfiles; note that `Write` cannot be scoped by path in built-in permission syntax |

---

## 7. Reduction Strategy

All reductions are bounded and reversible. tweakcc patches are reverted by reinstalling Claude Code or re-patching with original content. File changes to CLAUDE.md/MEMORY.md are protected by CLAUDE.md §6 preservation rules.

### 7.1 settings.json — Actual Changes Required

All changes are to `~/.claude/settings.json`. The canonical target configuration is `CContext/settings.json_sample`. This section documents the rationale; apply by replacing the current file with the sample contents.

**Background on accumulated allow rules:** The `permissions.allow` entries were added incrementally — each via an "always allow this" response to an approval dialog. They represent commands the model actually ran in past sessions. Removing them restores the approval dialog for those commands. For safe, frequent operations already in CC's built-in safe list (`cd`, `ls`, `echo`, `pwd`, `date`, `whoami`, `wc`, `which`, `sort`, `uniq`), no explicit `allow` entry is needed at all — they are pre-approved at the binary level. For destructive operations (`git push`, `pip install`), restoration of the approval dialog is the point.

**Remove from `permissions.allow`** — these wildcards nullify CLAUDE.md §8 approval gates:

| Rule to remove | Why | Replacement |
|---|---|---|
| `Bash(git:*)` | Covers `git push --force`, `git reset --hard`, `git clean -fd`, `git stash drop` without dialog | Explicit safe subcommands only (see allow list below); `git fetch` excluded — run manually |
| `Bash(pip:*)` | Covers `pip install`, `pip uninstall` without dialog | None; will prompt as intended |
| `Bash(npm:*)` | Covers `npm install`, `npm uninstall` without dialog | None; will prompt as intended |
| `Bash(mv:*)` | Silent overwrite without dialog | None; will prompt as intended |
| `Bash(cp:*)` | Silent overwrite without dialog | None; will prompt as intended |
| `Bash(curl:*)` | Covers `curl -X POST/-X DELETE` without dialog | None; will prompt as intended |
| `Bash(wget:*)` | Silent overwrite without dialog | None; will prompt as intended |
| `Bash(chmod:*)` | Permission broadening without dialog | None; will prompt as intended |
| `Bash(bash:*)` | `bash -c "..."` wrapping bypasses all other command-specific rules | None; run scripts directly via `./script.sh` |
| `Bash(less:*)`, `Bash(top:*)`, `Bash(ping:*)` | Interactive or run-forever; hang the Bash tool indefinitely | None; excluded intentionally |

**Remove from `permissions.allow`** — redundant with CC's built-in safe list (pre-approved at binary level; explicit entries add noise):

`Bash(cd:*)`, `Bash(ls:*)`, `Bash(echo:*)`, `Bash(pwd:*)`, `Bash(date:*)`, `Bash(whoami:*)`, `Bash(wc:*)`, `Bash(which:*)`, `Bash(sort:*)`, `Bash(uniq:*)`, `Bash(env:*)`

**Remove from `permissions.allow`** — native tools; `Read`/`WebFetch`/`WebSearch` are auto-approved in default mode without explicit entries. `Edit` and `Write` retain their entries to suppress per-invocation confirmation dialogs:

`Read`, `WebFetch`, `WebSearch`

**Remove from `permissions.allow`** — contradict tool description guidance (model should use Grep/Glob/Read instead of Bash wrappers):

`Bash(cat:*)`, `Bash(grep:*)`, `Bash(find:*)`, `Bash(sed:*)`, `Bash(head:*)`, `Bash(tail:*)`

**Add top-level keys:**

- `"includeGitInstructions": false` — removes 1558-token git workflow fragment; CLAUDE.md §8 is more restrictive and project-specific
- `"outputStyle": { "keepCodingInstructions": false }` — removes ~700-token `codingDisciplineSection` (13 `doing-tasks-*.md` fragments). **Review before applying**: several fragments (`security`, `blocked-approach`, `no-unnecessary-error-handling`, `read-before-modifying`) have no CLAUDE.md equivalent. Disable only if CLAUDE.md explicitly covers these areas.
- `"env"` block: `ENABLE_CLAUDEAI_MCP_SERVERS`, `CLAUDE_ENV_FILE`, `BASH_DEFAULT_TIMEOUT_MS` (see below)

**Resulting allow list** — git read-only/staging only; `git fetch` excluded (run manually); non-trivial utilities not in the built-in safe list:

```
Bash(git diff *)    Bash(git log *)     Bash(git status *)  Bash(git show *)
Bash(git add *)     Bash(git stash list *)  Bash(git stash show *)
Bash(df:*)   Bash(du:*)   Bash(gh:*)   Bash(gtimeout:*)  Bash(jq:*)
Bash(mkdir:*)  Bash(nvm:*)  Bash(ps:*)   Bash(python:*)   Bash(python3:*)
Bash(rg:*)   Bash(touch:*)  Bash(tree:*)  Bash(uv:*)
Edit  Write
```

**`env` block decisions:**

| Key | Value | Effect |
|---|---|---|
| `ENABLE_CLAUDEAI_MCP_SERVERS` | `"false"` | Disables `claudeai-mcp` plugin; prevents Gmail/Calendar auto-fetch, OAuth failures, and startup errors |
| `CLAUDE_ENV_FILE` | `"/Users/walter/.claude/ccenv.bash"` | Script sourced before each Bash tool call; handles pyenv PATH, `.venv` activation, `PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED` |
| `BASH_DEFAULT_TIMEOUT_MS` | `"300000"` | 5-minute hard ceiling per Bash call; backstop only — python3 child processes survive unless wrapped with `gtimeout --kill-after=5 N python3 ...` |

**`CLAUDE_ENV_FILE` scope:** exports only; aliases set in this script do not propagate to Bash tool subshells (non-interactive bash parses the command before sourcing). `settings.json` `env` block is for static key-value pairs that must reach the CC process itself at startup; `ccenv.bash` is for dynamic setup (conditional PATH, venv activation).

**CLI invocation for Spank sessions** — confirmed from session log analysis (5,279 total tool calls across 23 sessions):

```bash
claude --tools "Bash,Edit,Glob,Grep,Read,WebFetch,WebSearch,Write,Agent,Task"
```

Tool selection rationale (session log counts):

| Tool | Calls | Decision |
|---|---|---|
| `Read` | 1826 | Always required |
| `Edit` | 1736 | Always required |
| `Bash` | 999 | Always required |
| `Grep` | 363 | Always required |
| `Write` | 70 | Always required |
| `Glob` | 77 | Always required |
| `Agent` | 46 | Confirmed active; spawns fresh subagents |
| `ToolSearch` | 44 | Always available; cannot be excluded via `--tools` |
| `Task`+`TaskOutput`+`TaskUpdate`+`TaskCreate`+`TaskList` | 94 total | CC background task system; all excluded if `Task` absent |
| `WebSearch` | 15 | Keep; documentation lookup |
| `WebFetch` | 2 | Keep; small cost |
| `TodoWrite` | **0** | **Excluded**; saves 2161 tokens; zero sessions used it |
| `AskUserQuestion` | 4 | Not in `--tools`; available by default regardless |

`Task` in the logs is CC's built-in background task coordination system — not Tasks.csv. They coexist independently; Tasks.csv is updated by the model via Bash/Edit.

**Session wrapper functions** (`~/.alias`):
- `sc` — standard Spank session: `claude --tools "Bash,Edit,Glob,Grep,Read,WebFetch,WebSearch,Write,Agent,Task"` + pass-through args
- `scr` — read-only/analysis session: restricted tool set + `--append-system-prompt` constraint
- `scc` — continue most recent session: `sc --continue`

**Parallel task control** — the model autonomously decides to launch multiple background processes. No single setting controls the count. Enforce via two layers:
1. CLAUDE.md §7 directive: 3-process ceiling + mandatory `gtimeout --kill-after=5 N python3 ...` wrapping
2. PreToolUse hook: `CContext/limit-python3.sh` — blocks Bash calls that launch python3 when ≥3 already running. Deploy to `spank-py/.claude/hooks/`; register in `spank-py/.claude/settings.json`. See `spank-py/TasksKill.md §Hook enforcement` for hook JSON registration.

### 7.2 tweakcc Targets

Targets requiring tweakcc patches (not available via `settings.json`):

| Target Fragment | Tokens | Justification | Method |
|---|---|---|---|
| `bash-alternative-communication.md` + 5 others | ~154 | Contradicted by `settings.json` pre-approvals; or remove pre-approvals to use the guidance instead | `tweakcc` patch to empty string, or revoke settings.json entries |
| `bash-prefer-dedicated-tools.md` | 71 | Covered by `system-prompt-tool-usage-reserve-bash.md` (75 tks) if bash-alternative removed | `tweakcc` patch to empty |
| `system-prompt-subagent-delegation-examples.md` | 588 | Domain-irrelevant examples (ship audit, migration review); model writes adequate prompts without them | `tweakcc` patch to empty |
| `system-prompt-writing-subagent-prompts.md` | 365 | Agent tool is used (41 invocations observed); review whether this guidance improves subagent quality before removing | `tweakcc` patch to empty (conditional) |
| `tool-description-todowrite.md` | 2161 | Confirmed unused in Spank sessions (0 calls across all session logs); excluded via `--tools` CLI flag — no tweakcc needed | `--tools` flag exclusion |
| **Total (confirmed, settings + tweakcc)** | **~1,178** | Excludes TodoWrite since `--tools` handles it | |
| **Total (confirmed + `--tools` TodoWrite exclusion)** | **~3,339** | | |
| **Total potential (all conditional)** | **~3,704** | | |

Note: `tool-description-bash-git-commit-and-pr-creation-instructions.md` is now handled by `includeGitInstructions: false` — no tweakcc needed.

**How to apply a tweakcc patch (replace a fragment with empty string):**

```bash
cd /Users/walter/Work/Claude/tweakcc

# Edit the fragment file to empty its content
echo "" > /Users/walter/Work/Claude/claude-code-system-prompts/system-prompts/system-prompt-subagent-delegation-examples.md

# Apply patches
pnpm apply

# Verify
claude --version  # should report same version; check for patch application summary
```

**To target a specific prompt only** (without editing the file):
```bash
# Use --patch-filter to apply only one prompt
pnpm apply --patch-filter system-prompt-subagent-delegation-examples
```

### 7.3 MEMORY.md Bootstrap Tightening

Current bootstrap consumes approximately 36 lines, leaving ~155 usable lines for session state out of the 200-line `MEMORY_LINE_LIMIT`.

**Remove from bootstrap** (covered by `autoMemoryHeaderText()` and `agent-memory-instructions.md`):
- The "Update triggers" paragraph (~6 lines)
- The "What belongs here" paragraph (~5 lines)
- The "Lifecycle of an entry" paragraph (~4 lines)

**Replace with** (one line):
```
MEMORY carries current session state and transition risks only. Remove entries when the guarded risk has passed.
```

Saves approximately 14 lines, expanding usable session-state budget to ~169 lines.

### 7.4 Projected Token Savings

| Change | Tokens Saved | Status |
|---|---|---|
| `includeGitInstructions: false` | 1558 | Ready to apply |
| `outputStyle.keepCodingInstructions: false` | ~700 | **Conditional** — review 13 doing-tasks fragments first; `security`, `blocked-approach`, `no-unnecessary-error-handling`, `read-before-modifying` have no CLAUDE.md equivalent |
| Revoke `Bash(git:*)` wildcard → safe subcommands only | 0 tokens, restores git approval gate | Ready to apply — critical |
| Revoke 6 bash file-tool pre-approvals (cat/grep/find/sed/head/tail) | 0 tokens, restores behavioral coherence | Ready to apply |
| Revoke 8 CLAUDE.md §8 conflicting entries (pip/npm/mv/cp/curl/wget/chmod + remove git:*) | 0 tokens, restores approval gates | Ready to apply |
| bash-alternative-*.md fragments (6 files) | ~154 | Ready (after pre-approvals removed) |
| `bash-prefer-dedicated-tools.md` | 71 | Ready (after bash-alternative removed) |
| MEMORY.md bootstrap tightening | ~100 tokens | Ready to apply |
| `tool-description-todowrite.md` via `--tools` | 2161 | **Confirmed**: 0 TodoWrite calls across all Spank session logs |
| **Total confirmed** | **~4,744 tokens** | |
| `subagent-delegation-examples.md` | 588 | Conditional: Agent tool IS used (45 calls); evaluate whether examples improve output quality |
| `writing-subagent-prompts.md` | 365 | Same condition; 41 Task calls observed |
| **Total potential** | **~5,697 tokens** | |

The 0-token items (approval gate restorations) produce no context window savings but are the highest-priority correctness issues: `Bash(git:*)` means `git push --force` runs without approval despite CLAUDE.md §8.

**Session log tool inventory** (all Spank sessions, aggregated):

| Tool | Calls | Notes |
|---|---|---|
| Read | 1826 | Always needed |
| Edit | 1736 | Always needed |
| Bash | 999 | Always needed |
| Grep | 363 | Always needed |
| Glob | 77 | Always needed |
| Write | 70 | Always needed |
| Agent | 46 | Subagent launches; keep in `--tools` |
| ToolSearch | 44 | Built-in; always available regardless of `--tools` |
| Task | 41 | CC built-in background task coordination (not Tasks.csv); keep in `--tools` |
| TaskOutput | 32 | Included when `Task` is in `--tools` |
| TaskUpdate | 10 | Included when `Task` is in `--tools` |
| WebSearch | 15 | Keep in `--tools` |
| TaskCreate | 9 | Included when `Task` is in `--tools` |
| AskUserQuestion | 4 | Available by default; not needed in `--tools` |
| Skill | 3 | Available via slash commands; not in `--tools` |
| WebFetch | 2 | Keep in `--tools` |
| TaskList | 2 | Included when `Task` is in `--tools` |
| **TodoWrite** | **0** | **Confirmed unused; excluded from `--tools`; saves 2161 tokens** |

---

## 8. Extension Mechanism Priority and Configuration Propagation

### 8.1 Authority Hierarchy (Complete)

| Mechanism | Authority Level | Prompt Source | Propagates to Fresh Subagents? | Can Be Blocked By |
|---|---|---|---|---|
| Managed settings | Absolute (cannot be overridden) | MDM / registry / `managed-settings.json` | Yes, global | Nothing |
| Runtime `deny` rules | Absolute (execution gate) | `settings.json` `permissions.deny` | Yes, global | Nothing; deny evaluated first |
| Hooks (PreToolUse) | Enforcement (post-permission) | `settings.json` `hooks` section | Yes, global | Nothing; root authority for interception |
| `disallowedTools` (agent prompts) | API-level filter | `agent-prompt-*.md` frontmatter | No (per-subagent invocation) | Nothing once set |
| Runtime `ask/allow` rules | Permission gate | `settings.json` `permissions.allow/ask` | Yes, global | Hook `permissionDecision: deny` |
| CLAUDE.md injection | Override authority | Project file, position 5, OVERRIDE header | Yes (same working dir) | Nothing in model-space |
| System prompt fragments | Behavioral baseline | Binary (tweakcc-patchable) | Yes (re-assembled per session) | CLAUDE.md OVERRIDE |
| MEMORY.md injection | State guidance, no override | Project file, position 8a | Yes (re-read per session) | Runtime enforcement; compaction replaces conversation |
| Tool descriptions | Advisory only | Binary (tweakcc-patchable) | Per-tool, if tool is active | Ignored by model; CLAUDE.md takes precedence |
| Skills | Session-scoped instructions | `skill-*.md` files | No (not inherited by subagents) | System prompt or CLAUDE.md |
| MCP tools | Advisory only (same as tool descriptions) | MCP server config | Per-tool, same as built-in | Hooks; permissions; `disallowedTools` |

### 8.2 Toolsets (tweakcc Feature)

Toolsets are named subsets of tools. The `writeToolFetchingUseMemo()` function in tweakcc patches the tool aggregation hook so that only tools in the active toolset are included in the API request. Tools outside the toolset have no tool description token cost — they are filtered before the API call.

To define a toolset in `tweakcc/config.json`:
```json
{
  "toolsets": [
    {
      "name": "core",
      "allowedTools": ["Bash", "Read", "Edit", "Write", "Grep", "Glob", "TodoWrite"]
    },
    {
      "name": "full",
      "allowedTools": "*"
    }
  ],
  "defaultToolset": "core",
  "planModeToolset": "full"
}
```

Switching toolsets: `/toolset core` or `/toolset full` in the Claude Code REPL. The `planModeToolset` setting automatically switches toolsets when entering/exiting plan mode (if both `planModeToolset` and `defaultToolset` are set).

### 8.3 MCP Tools vs. Built-in Tools

MCP tools appear in the same `tools` array as built-in tools. From the model's perspective, they are indistinguishable — no separate system prompt fragment marks them as MCP. Hooks intercept MCP tool calls by tool name identically to built-in calls. `disallowedTools` in agent-prompt frontmatter filters MCP tools identically to built-in tools.

MCP resources (read-only data provided by MCP servers) appear as content blocks in context, not as callable tools, and have no hook interception mechanism. The `system-reminder-mcp-resource-no-content.md` and `system-reminder-mcp-resource-no-displayable-content.md` reminders fire on MCP resource load failures.

### 8.4 Subagent Configuration Propagation

| Config Item | Fresh Subagent (subagent_type) | Context-Inheriting Subagent |
|---|---|---|
| `settings.json` permissions | Inherited (global) | Inherited (global) |
| `settings.json` hooks | Inherited (global) | Inherited (global) |
| CLAUDE.md | Re-read from project dir (same CLAUDE.md if same cwd) | Inherited from parent system prompt |
| MEMORY.md | Re-loaded at position 8a (re-reads from disk) | Inherited from parent system prompt |
| Tool descriptions | Per active tool list for this subagent | Inherited from parent tool list |
| `disallowedTools` | Set in this subagent's agent-prompt frontmatter | Can be set (overrides parent) |
| Conversation history | Empty (fresh start) | Full parent history |
| Skills | Not active (must be re-invoked) | Inherited if skill was active in parent |
| Toolsets (tweakcc) | Active toolset from app state | Active toolset from app state |

---

## 9. Practical Examples and Copy-Paste Reference

### 9.1 Verify Current Tool List in a Session

```bash
# In Claude Code REPL: ask the model to report active tools
# This causes it to read the tools array and enumerate them
# Alternatively, launch with --debug to see API request details:
claude --debug 2>&1 | grep '"name"' | head -30
```

### 9.2 Check Which System Prompt Fragments Are Active

```bash
# Use simple mode to strip all fragments and observe baseline:
CLAUDE_CODE_SIMPLE=1 claude --debug 2>&1 | grep "system"

# Full debug with all fragments (pipe to file — output is large):
claude --debug 2>&1 > /tmp/cc-debug.txt
grep "system_prompt" /tmp/cc-debug.txt | head -5
```

### 9.3 Profile Tool Utilization from Session Logs

```bash
# List recent session directories
ls -lt ~/.claude/projects/-Users-walter-Work-Spank-spank-py/*.jsonl 2>/dev/null | head -10
# or
ls -lt ~/.claude/projects/ | head -10

# Count tool invocations in a session
SESSION=/path/to/session.jsonl
jq -r 'select(.type=="tool_use") | .name' "$SESSION" | sort | uniq -c | sort -rn

# Get all tool names across recent sessions
find ~/.claude/projects/-Users-walter-Work-Spank-spank-py/ -name "*.jsonl" \
  | head -5 \
  | xargs -I{} jq -r 'select(.type=="tool_use") | .name' {} \
  | sort | uniq -c | sort -rn

# Show tool invocations with timestamps from one session
jq -r 'select(.type=="tool_use") | [.timestamp, .name] | @tsv' "$SESSION"
```

### 9.4 Context Size Verification — Pre-Session Checklist

Run before starting a session to confirm the baseline is within expected range. Each category corresponds to a potential context consumer.

**1. System prompt fragments** — determined by settings.json keys and binary state:
```bash
# Confirm git fragment is suppressed:
jq '.includeGitInstructions // "not set (default: true)"' ~/.claude/settings.json

# Confirm coding discipline is suppressed:
jq '.outputStyle.keepCodingInstructions // "not set (default: true)"' ~/.claude/settings.json

# Confirm no language section:
jq '.language // empty' ~/.claude/settings.json .claude/settings.json 2>/dev/null

# Confirm no sandbox fragments:
jq '.sandbox.enabled // false' ~/.claude/settings.json .claude/settings.json 2>/dev/null

# Confirm no hooks-configuration fragment (hooks fragment fires if any hook is set):
jq '.hooks // empty' ~/.claude/settings.json .claude/settings.json 2>/dev/null
```

**2. Tool descriptions** — controlled by `--tools` CLI flag:
```bash
# Check which tools will be loaded for a session:
claude --tools "Bash,Edit,Glob,Grep,Read,WebFetch,WebSearch,Write,Agent" --debug 2>&1 \
  | grep -E '"name".*"Bash|Edit|Glob|Grep|Read|Web|Write|Agent|TodoWrite"' | head -20
# TodoWrite should NOT appear if excluded from --tools
```

**3. Cached sections (CLAUDE.md, MEMORY.md, environment info)**:
```bash
# CLAUDE.md token estimate (rough: chars / 4):
wc -c /Users/walter/Work/Spank/spank-py/CLAUDE.md | awk '{print $1/4, "tokens approx"}'

# MEMORY.md line count vs MEMORY_LINE_LIMIT (200):
wc -l ~/.claude/projects/-Users-walter-Work-Spank-spank-py/memory/MEMORY.md
# If > 200, content is truncated; check what falls past line 200
```

**4. MCP servers** — confirm none are active (no mcpInstructionsSection injection):
```bash
# Verify ENABLE_CLAUDEAI_MCP_SERVERS is disabled in settings:
jq '.env.ENABLE_CLAUDEAI_MCP_SERVERS // "not set (default: enabled)"' ~/.claude/settings.json
# Expected: "false"

# Verify no MCP servers configured:
claude mcp list
# Expected: empty or "No MCP servers configured"
```

**5. Skills** — confirm no skills are pre-loaded:
```bash
# Skills are not in baseline context; they load on first invocation (/debug, /loop, etc.)
# Verify: start session without invoking any skills; check that no skill system-reminders fire
ls ~/.claude/skills/ 2>/dev/null  # project-level skill directory if it exists
```

**6. Session continuation** — if using --continue or --resume, prior history adds to context:
```bash
# Check token usage of a prior session before continuing it:
LATEST=$(ls -t ~/.claude/projects/-Users-walter-Work-Spank-spank-py/*.jsonl | head -1)
jq -r 'select(.type=="assistant") | .message.usage.input_tokens // empty' "$LATEST" | tail -1
# If input_tokens is near 184000 (92% of 200K), start fresh rather than continuing
```

**7. Subagent context** — when spawning subagents via Agent tool:
- Fresh subagent (`subagent_type` set): re-runs systemPromptAssembler(), inherits CLAUDE.md + MEMORY.md from same cwd, tool set from agent-prompt frontmatter `disallowedTools`. Does NOT inherit parent conversation history.
- Tool descriptions: each subagent loads the tool schemas for its assigned tool set; use `disallowedTools` in agent-prompt frontmatter to restrict.
- All 45 Agent invocations observed in Spank sessions were fresh subagents.

**8. Compaction threshold** — user-configurable via env var; **verify on each CC upgrade**.

**Binary analysis (v2.1.74):** The binary computes a default threshold as `effectiveWindow - 13000`. With `effectiveWindow` = 200K (see `CLAUDE_CODE_AUTO_COMPACT_WINDOW` in [env-vars](https://code.claude.com/docs/en/env-vars)):

```
defaultThreshold = 200,000 − 13,000 = 187,000 tokens = 93.5% of 200K
```

**Official Claude Code documentation** states autocompact fires at **approximately 95%** capacity by default and that `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` can use **lower** percentages to compact earlier; values **above** the default threshold have **no effect** ([env-vars](https://code.claude.com/docs/en/env-vars)). **GitHub [#31806](https://github.com/anthropics/claude-code/issues/31806)** (closed duplicate; deobfuscated snippet in issue body) claims `defaultThreshold = effectiveWindow - 13000` and a `Math.min` clamp — for **200K** `effectiveWindow` that is **187K** (**93.5%**). The issue **title** says “~83%”; treat the **embedded pseudocode** as the reporter’s claim, not Anthropic’s. **This document does not assert** which figure matches your installed binary without re-verification.

**UI:** `/context` may display a fixed **Autocompact buffer** (e.g. 33K tokens) independent of the above; see `CCLaunch.md` Appendix — Observed.

**Messages API (different product surface):** Default compaction **trigger** **150,000 input tokens** for `compact_20260112` ([Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)).

`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` lowers the client threshold when set below the effective default. Historical analysis used `Math.min(userThreshold, defaultThreshold)` for an upper clamp at the binary default. Set in `settings.json` `env` block (so the CC **process** inherits the vars):

```json
"env": {
  "ENABLE_CLAUDEAI_MCP_SERVERS": "false",
  "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "90"
}
```

The statsig dynamic config `4189951994: {enabled: true, tokenThreshold: 0.92}` is cached locally but applies to a different feature (likely auto-mode continuation), not the compaction threshold formula.

```bash
# Verify current settings.json env block:
jq '.env // empty' ~/.claude/settings.json
# Expected: {"ENABLE_CLAUDEAI_MCP_SERVERS": "false"}
# Optionally add: "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "85"  (or "90") — see §9.4 item 8

# Manual compact at any point:
# In the Claude Code REPL: /compact
# This triggers compaction immediately regardless of threshold,
# letting you control exactly when the summary is created.
```

### 9.5 Validate Team Settings / Swarm State

```bash
# Check if team mode is enabled in current settings:
jq '.teammateMode // "not set"' ~/.claude/settings.json

# Check ~/.claude.json for team context keys (base64-encoded project states):
jq 'keys' ~/.claude.json

# In Claude Code REPL: ask model to report team state
# The model will check its context for team-related injections
# If teamToolsEnabled() returned false, no team fragments are present

# Verify team tools are NOT in session (expected for solo work):
claude --debug 2>&1 | grep -E "TeammateTool|SendMessageTool|TeamDeleteTool"
# Expected output: empty (tools not in tool list)
```

### 9.5 Minimal MEMORY.md Bootstrap Template

The `---` line separates the protected bootstrap rules from the working session state below it. The bootstrap block instructs the model on MEMORY discipline; the session state block is updated freely each session.

The `<!-- SESSION STATE BELOW THIS LINE -->` comment is an optional secondary marker. The model receives the raw file text — HTML comments are NOT invisible to the model, they are visible as `<!-- ... -->`. The comment is ~5 tokens and adds no confusion in practice, but the `---` separator alone is sufficient and simpler. The actual Spank MEMORY.md uses detailed prose bootstrap rules and `---` dividers, not this minimal template form.

```markdown
# PROJECT MEMORY — Spank / spank-py

**POST-COMPACTION RULE:** After compaction, read this file FIRST. Do not continue or execute any pending tasks without explicit instruction from the developer.

**MEMORY VERSIONING:** When adding any entry, append a snapshot reference: `[snap: <YYYY-MM-DD HH:MM>]`. Do not rewrite history.

**PERMISSION BLOCK RULE:** If an approval-required operation is blocked, attempt at most once more with a different approach. If still blocked, stop and report.

**Key paths:**
- Project: `/Users/walter/Work/Spank/spank-py/`
- Memory: `~/.claude/projects/-Users-walter-Work-Spank-spank-py/memory/`
- Docs: (add project-specific paths here)

MEMORY carries current session state and transition risks only. Remove entries when the guarded risk has passed.

---

(session state entries go here)
```

### 9.6 PreToolUse Hook to Block Git Push Without Confirmation

```bash
#!/bin/bash
# ~/.claude/hooks/pre-bash.sh
# Invoked by PreToolUse hook on all Bash calls

TOOL_INPUT=$(cat)  # JSON from stdin: {"tool_name": "Bash", "tool_input": {"command": "..."}}
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.tool_input.command // ""')

# Block git push unless --debug flag is present in the command
if echo "$COMMAND" | grep -qE '^git push'; then
  echo '{"continue": false, "stopReason": "git push requires explicit developer approval. Please confirm before proceeding."}'
  exit 0
fi

# Allow everything else
echo '{"continue": true}'
exit 0
```

Configure in `settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "~/.claude/hooks/pre-bash.sh"}]
      }
    ]
  }
}
```

### 9.7 tweakcc: Empty a Fragment Without Editing the Source Repo

tweakcc locates each fragment in the binary by matching the `.md` file content against the embedded string. The replacement content must be non-empty — an empty file produces a match-all failure. The convention is a short HTML comment as a minimal placeholder: the model receives `<!-- removed: ... -->` (~5 tokens) instead of the original fragment. The replacement comment IS visible to the model.

```bash
cd /Users/walter/Work/Claude/tweakcc

echo "<!-- removed: subagent delegation examples are project-irrelevant -->" \
  > /Users/walter/Work/Claude/claude-code-system-prompts/system-prompts/system-prompt-subagent-delegation-examples.md

pnpm apply

# Verify:
pnpm apply --dry-run
```

### 9.8 Check Installed Claude Code Version and Installation Paths

```bash
claude --version
# e.g.: claude-code 2.1.74

# Find native binary:
ls -la $(which claude)
readlink $(which claude)

# tweakcc: check installation detection:
cd /Users/walter/Work/Claude/tweakcc && pnpm run detect
```

---

## 10. Open Questions and Next Steps

### 10.1 Confirmed Open Questions

**TodoWrite usage and invocation.** TodoWrite is Claude Code's built-in sidebar task panel — a structured checklist rendered in the Claude Code UI, distinct from any project task files (Tasks.csv, GitHub Issues, etc.). It is not a slash command and has no CLI invocation. The model uses it spontaneously when it judges a task complex enough to warrant a checklist (3+ steps, non-trivial planning). A developer can request it explicitly by saying "create a todo list for this" or "track progress with todos" — but this relies on the model's judgment about when to call the tool; it cannot be forced from the command line. Its description is 2161 tokens because it covers task lifecycle, state transitions (`pending` / `in_progress` / `completed` / `cancelled`), when to create vs. skip a todo list, task granularity guidance, and extensive examples. The description is loaded unconditionally for every session. For a project that tracks work in its own files and never uses the sidebar panel, all 2161 tokens are unread overhead. **Action:** audit recent session `.jsonl` files for `TodoWrite` tool_use blocks: `jq -r 'select(.type=="tool_use") | .name' *.jsonl | sort | uniq -c`. If `TodoWrite` is absent across 5+ sessions, patch out via tweakcc.

**hooks-configuration.md load condition.** `system-prompt-hooks-configuration.md` (1482 tks) fires when any hooks are configured across any settings scope. The global `settings.json` currently has no `hooks` key — but project-level `.claude/settings.json` or local `.claude/settings.local.json` may. **Action:** `cat ~/.claude/settings.json .claude/settings.json 2>/dev/null | jq '.hooks'` — if null/empty, this fragment is not in context.


**Subagent delegation examples.** Audit Task tool invocations: `jq -r 'select(.type=="tool_use" and .name=="Agent") | .name' *.jsonl | wc -l`. If the count is low or zero across 5+ sessions, `system-prompt-subagent-delegation-examples.md` (588 tks) is safe to patch out.

### 10.2 Possible Gaps

- **ToolSearch on-demand description loading.** `tool-description-toolsearch-second-part.md` (202 tks) enables the model to call the ToolSearch tool to retrieve tool descriptions on demand. Critically: if a tool's description is removed via tweakcc, the tool still exists and can be accessed by the model through ToolSearch — the description is fetched at invocation time rather than loaded at session start. This makes it safe to remove infrequently-used tool descriptions without losing access to the tools. The tradeoff: the model does not know the tool exists until it performs a ToolSearch, so discovery must be prompted or anticipated.

- **Model setting and context headroom.** Included because model choice determines the context window ceiling, which is the denominator for all token budget math in §3. `claude-sonnet-4-6` provides 200K tokens — the assumption underlying every calculation in this document. If the model is changed (e.g., to a smaller-window variant), all baseline estimates require revision. The immediate risk is not baseline cost but conversation history accumulation: a session reading 10 files × 500 lines each adds ~50K tokens to history alone. A previous session reached context limits mid-task (CCMisses.md Miss 2). Baseline reduction from §7 extends the buffer before compaction fires but does not reduce the rate at which history grows — it buys more tool operations before the threshold, not an unlimited budget.

- **`companyAnnouncements` injection.** If `companyAnnouncements` is set in any settings scope, the announcement text is injected at session startup alongside the system prompt. The token cost and exact injection point are not documented in any extracted fragment. Included as a gap because it could inflate the baseline without appearing in the §3 catalog — if announcements are configured in a managed or enterprise settings scope, the actual baseline is higher than the §3 estimate.

- **Output style effects.** The `outputStyle` setting controls two independent things:
  1. *Tone variant selection.* Three variants exist: the default (`tone-and-style-concise-output-detailed.md` at 89 tks + `output-efficiency.md` at 177 tks = ~266 tks), a short variant (`tone-and-style-concise-output-short.md` at 16 tks), and a third undocumented variant. Switching to the short variant saves ~250 tokens.
  2. *`codingDisciplineSection()` suppression.* When `outputStyle.keepCodingInstructions === false`, `codingDisciplineSection()` is skipped. This section assembles approximately 700 tokens from 13 `doing-tasks-*.md` fragments: `no-unnecessary-additions` (78), `avoid-over-engineering` (30), `no-premature-abstractions` (60), `minimize-file-creation` (47), `no-compatibility-hacks` (52), `no-time-estimates` (47), `no-unnecessary-error-handling` (64), `read-before-modifying` (46), `security` (67), `software-engineering-focus` (104), `help-and-feedback` (24), `ambitious-tasks` (47), `blocked-approach` (90). If `keepCodingInstructions` is false, subtract ~700 tokens from the baseline. These fragments are all behavioral — suppressing them removes coding discipline guidance without patching the binary.

- **MCP server tool descriptions and `claudeai-mcp` auto-fetch.** MCP tool descriptions are provided by the MCP server at connection time, not embedded in the binary. Each MCP server sends its own tool schemas with descriptions of arbitrary length. Token cost is entirely server-dependent. Additionally, `mcpInstructionsSection()` (position 8, conditional: `hasMcpServers()`) adds a cached section explaining MCP protocol to the model — size unquantified. **Check whether MCP is configured:**
  ```bash
  jq '.mcpServers // empty' ~/.claude/settings.json 2>/dev/null
  jq '.mcpServers // empty' .claude/settings.json 2>/dev/null
  claude mcp list
  ```

  **`claudeai-mcp` background auto-fetch (separate from config file):** At startup, a built-in background process named `claudeai-mcp` checks a statsig feature gate and, if `true`, fetches server definitions from `https://api.anthropic.com/v1/mcp_servers`. These servers (currently: Gmail and Google Calendar) are injected at runtime and appear in `claude mcp list` but are NOT stored in any `settings.json` `mcpServers` key. `claude mcp remove` cannot remove them — they are re-fetched on every restart.

  The two auto-fetched servers fail authentication (no OAuth token configured), so they add connection overhead and may trigger `mcpInstructionsSection()` without providing usable tools.

  **Suppression — official approach:**
  Add to `settings.json` under the `env` key:
  ```json
  "env": { "ENABLE_CLAUDEAI_MCP_SERVERS": "false" }
  ```
  This is the documented env var (`ENABLE_CLAUDEAI_MCP_SERVERS`: "Set to `false` to disable claude.ai MCP servers in Claude Code. Enabled by default for logged-in users"). It suppresses the `claudeai-mcp` plugin gate check entirely and does not require `--strict-mcp-config` or any shell alias. No `~/.zshrc` changes needed.

- **Language setting.** `languageSection()` adds a `language` cached section at position 8 when `settings.json` sets `language` to a non-default value. Token cost unknown (the section content is dynamic). To keep this absent from baseline: do not set `language` in any settings scope. The default (no `language` key) skips this section entirely. **Check whether language is set:**
  ```bash
  jq '.language // empty' ~/.claude/settings.json .claude/settings.json 2>/dev/null
  ```
  Empty output means the language section is absent.

- **Sandbox fragments.** The 12 `bash-sandbox-*.md` fragments (~500 tokens total) are included in the Bash tool description only when `sandbox.enabled: true` in settings. To keep these absent: do not set `sandbox.enabled` (it defaults to `false`). Current installation has sandbox not enabled; these are absent from the §3 baseline. If sandbox is ever enabled, add ~500 tokens to the Bash tool description cost. **Check whether sandbox is enabled:**
  ```bash
  jq '.sandbox.enabled // false' ~/.claude/settings.json .claude/settings.json 2>/dev/null
  ```
  `false` on all lines means sandbox fragments are absent.

---

## Notes

### Deferred Tool Loading — Status (D2)

**Finding source:** CDecode.md binary analysis of `hX()` and the `tengu_defer_all_bn4` feature flag. Cross-verified against Anthropic API documentation and GitHub issues (2025–2026).

**CDecode.md finding:** The `hX()` function controls which tool schemas are deferred. The feature flag `Jq("tengu_defer_all_bn4", true)` has a default of `true`, which CDecode.md interpreted as meaning nearly all built-in tools are deferred at session start (withheld from the API tools array), with the model using ToolSearch to fetch them on demand.

**API documentation finding:** The Anthropic `tool_search_tool` documentation and the `defer_loading: true` API feature describe a mechanism that applies to MCP tools and any tool explicitly marked `defer_loading: true`. Built-in Claude Code tools do not carry this flag in their definitions. GitHub issue [#31623](https://github.com/anthropics/claude-code/issues/31623) requests extending deferred loading to built-in subagents, confirming it does not currently apply to them.

**Reconciliation:** The `hX()` logic handles four cases in order:
1. `tool.isMcp === true` → always defer (MCP tools are always deferred)
2. `tool.name === Eh` (ToolSearch) → never defer
3. `mn6` Brief tool → never defer
4. `tengu_defer_all_bn4 === true` → defer (the feature flag)

The `E8R` always-included set (Bash, Glob, Grep, Read, WebFetch, WebSearch) appears to be assembled and sent to the API outside the `hX()` path entirely — their schemas are added to the API request before the deferral filter runs. The `tengu_defer_all_bn4=true` default affects the remaining built-in tools (Edit, Write, Agent, TodoWrite, etc.), but this conflicts with observable session behavior where these tools operate without a ToolSearch call.

**Current understanding:** The deferred loading behavior for built-in tools is unverified for v2.1.74. The `§3.2` "Always loaded — standard solo session" list reflects observed tool availability (tools that work without a preceding ToolSearch call), not a confirmed API tools-array inspection. The distinction between tools present in the API request versus tools present in context-via-description is not established with certainty.

**Action to verify:** Capture a live API request during a Claude Code session and inspect the `tools` array. If Edit, Write, Agent, TodoWrite are present in the tools array at session start, deferral is not active for those tools. If they are absent and only `E8R` + ToolSearch appear, deferral is active and the §3.2 "always loaded" list requires significant revision downward.

```bash
# Intercept API traffic (requires mitmproxy or similar):
claude --debug 2>&1 | grep -E "tools.*schema|deferred|available_tools" | head -20
```

---

## Terms and Definitions

Binary internals terms (obfuscated IDs, tweakcc, telemetry) are in `CInternals.md §Terms`.

| Term | Definition |
|---|---|
| **Fragment** | A single `.md` file embedded as a string literal in the Claude Code binary |
| **System prompt** | Stream 1 of the API request; assembled by `systemPromptAssembler()` from ordered fragments |
| **Tool description** | Stream 2; the `description` field of a tool schema, loaded separately from the system prompt |
| **Override declaration** | The "IMPORTANT: These instructions OVERRIDE..." header prepended to every injected user file |
| **Injection point** | The position in the assembled system prompt where a fragment is inserted (positions 1–8j) |
| **Cached section** | A section computed once and keyed for cache reuse across turns; created by `cachedSection()` |
| **Compaction** | The process of replacing conversation history with a structured summary when context approaches capacity |
| **Compaction prompt** | The variant of the system prompt used during compaction; includes `compactInstructionsInjectionPoint` |
| **Context window** | Total token budget per API call; 200K tokens for Claude 3/4 Sonnet/Opus |
| **Permission mode** | The approval behavior for tool calls; set by `defaultMode` in `settings.json` |
| **Toolset** | A tweakcc-defined named subset of allowed tools; filters the API tools array before the request |
| **Skill** | An on-demand `.md` instruction file invocable via slash command; not in baseline context |
| **Data file** | A `data-*.md` file referenced by a skill; loaded on demand by the skill |
| **System reminder** | An ephemeral `<system_reminder>` block injected mid-conversation on runtime events |
| **Fresh subagent** | A subagent spawned with `subagent_type`; runs its own `systemPromptAssembler()`, empty history |
| **Context-inheriting subagent** | A subagent without `subagent_type`; inherits parent system prompt verbatim |
| **MEMORY_LINE_LIMIT** | The hard 200-line truncation applied to MEMORY.md content before injection (constant = 200) |
| **disallowedTools** | An array in agent-prompt frontmatter that filters tools from the API request for that subagent |
| **OVERRIDE** | The authority level of CLAUDE.md injection; built-in fragments cannot override it in model-space |

---

## References

Binary and tooling references (tweakcc, extraction repo, JSON Schema) are in `CInternals.md §References`.

- [Claude Code Documentation](https://code.claude.com/docs/en/settings) — settings.json schema, permissions, hooks
- [Claude Code Permissions](https://code.claude.com/docs/en/permissions) — rule syntax, permission modes, managed settings
- [Claude Code Memory](https://code.claude.com/docs/en/memory) — auto memory, MEMORY.md, storage location
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks) — hook events, output format, examples
- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) — team mode, swarm, worker roles

