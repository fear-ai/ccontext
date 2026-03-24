# CInternals.md — Claude Code Binary Internals

*Session: 2026-03-14. Discussion of where system, tools, skills, MEMORY, and compact prompts are formed in the Claude Code binary.*

*Canonical URL list for env vars, shortcuts, API compaction:* `CCLaunch.md` → **External reference index**. Published env names: [code.claude.com/docs/en/env-vars](https://code.claude.com/docs/en/env-vars). Appendix **A.3.1** here reconciles binary strings to that table.*

---

## Starting Point

**Developer prompt:** "in the code base of https://github.com/anthropics/claude-code, where are system, tools, skills, MEMORY, compact prompts formed?"

The public GitHub repository at that URL is the examples/plugins repo, not the runtime source. The runtime is distributed as a compiled native binary.

**Developer clarification:** "code in /Users/walter/Work/Claude/claude-code"

That directory contains the public examples/plugins repository (sample agents, hooks, slash command examples) — not the CLI runtime source.

---

## Binary Discovery

```
/opt/homebrew/bin/claude → /opt/homebrew/Caskroom/claude-code/2.1.74/claude
```

- File type: Mach-O 64-bit executable arm64
- Size: 182MB
- Format: Bun-compiled executable — JS source embedded in native binary, function names mangled to short identifiers, string literals preserved

The binary is not a Node.js script or pkg bundle — it is compiled by Bun, which embeds the JS engine and source into a native arm64 executable.

---

## How a TS/JS CLI Gets Compiled to a Native Binary

Three common approaches:

**pkg** (Vercel) — bundles Node.js runtime + JS/TS into a self-contained executable. JS embedded as a virtual filesystem. Strings and some code are extractable.

**esbuild + pkg** — esbuild first bundles all TS/JS into a single minified JS file, then pkg wraps it with the Node runtime.

**Bun compile** — bundles and compiles to a native binary (Mach-O/ELF) with the JS engine and code embedded. This is what Claude Code uses. The 182MB size includes the Bun runtime (~80MB) plus bundled, minified source.

**ncc** (Vercel) — compiles to a single JS file, not a native binary. Ruled out here — the binary is Mach-O.

---

## Extraction Method

Used `strings` on the binary to extract readable text, then `python3` to locate specific function definitions by searching for known byte sequences and extracting surrounding context.

Key finding: function names are mangled (short identifiers like `r2`, `kaK`, `MaK`) but **string literals are fully preserved** in the binary, making the prompt text directly readable.

---

## System Prompt Construction — `r2()` Function

`r2()` is the async system prompt assembler. It builds an ordered array of prompt sections:

```javascript
async function r2(_, T, q, R) {
  // Simple mode fallback
  if (s_(process.env.CLAUDE_CODE_SIMPLE))
    return [`You are Claude Code, Anthropic's official CLI for Claude.\nCWD: ${MT()}\nDate: ${Nw_()}`];

  // Gather environment and settings
  let K = MT();  // CWD
  let [$, O, A] = await Promise.all([uV(K), QH9(), dH9(T, q)]);
  // $ = CLAUDE.md content, O = output style, A = env info

  // Cached prompt sections (keyed, deduplicated across turns)
  let j = [
    kp("memory",              () => qPT()),
    kp("ant_model_override",  () => PaK()),
    kp("env_info_simple",     () => dH9(T, q)),
    kp("language",            () => JaK(H.language)),
    kp("output_style",        () => WaK(O)),
    t97("mcp_instructions",   () => gJ_() ? null : XaK(R), "MCP servers connect/disconnect between turns"),
    kp("scratchpad",          () => CaK()),
    kp("frc",                 () => SaK(T)),
    kp("summarize_tool_results", () => EaK),
    kp("brief",               () => baK()),
  ];
  let f = await e97(j);

  // Assemble final prompt array
  return [
    kaK(O),          // identity + core instructions
    MaK(z),          // tool usage and permission rules
    O === null || O.keepCodingInstructions === true ? GaK() : null,  // coding discipline
    haK(),           // "Executing actions with care"
    ZaK(z, $),       // tool preference rules + CLAUDE.md injection
    vaK(),           // git rules
    yaK(),           // PR rules
    ...f             // cached sections (memory, env, output style, MCP, etc.)
  ].filter(w => w !== null);
}
```

### Section Functions

**`kaK(outputStyle)`** — identity and core instructions:
```
You are an interactive agent that helps users [with software engineering tasks / according to your "Output Style"].
Use the instructions below and the tools available to you to assist the user.
[security rules, URL policy]
```

**`MaK(toolNames)`** — system rules:
```
All text you output outside of tool use is displayed to the user...
Tools are executed in a user-selected permission mode. When you attempt to call a tool that is not automatically allowed...
[AskUserQuestion reference if tool is available]
```

**`GaK()`** — coding discipline (skipped if output style overrides):
```
Don't add features, refactor code, or make "improvements" beyond what was asked...
Don't add error handling, fallbacks, or validation for scenarios that can't happen...
[anti-over-engineering rules]
```

**`haK()`** — action safety:
```
# Executing actions with care
Carefully consider the reversibility and blast radius of actions...
```

**`ZaK(toolNames, claudeMdFiles)`** — tool preference rules:
```
To read files use Read instead of cat, head, tail, or sed
To edit files use Edit instead of sed or awk
To create files use Write instead of cat with heredoc or echo redirection
To search for files use Glob instead of find or ls
To search the content of files, use Grep instead of grep or rg
Reserve using the Bash exclusively for system commands...
```

---

## CLAUDE.md and Project Instructions — `AWq()` / `S3()`

Assembled separately from the section functions above. The injection header:

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.
```

Each file is labeled by source type:
- `"(project instructions, checked into the codebase)"` — CLAUDE.md in repo
- `"(user's private project instructions, not checked in)"` — local CLAUDE.md
- `"(user's auto-memory, persists across conversations)"` — MEMORY.md
- `"(user's private global instructions for all projects)"` — `~/.claude/CLAUDE.md`
- `"(shared team memory, synced across the organization)"` — team memory

TeamMem entries are wrapped in `<team-memory-content source="shared">` tags. All others are injected as plain text under their path label.

---

## MEMORY — `qPT()` → `aD()`

**Path resolution — `aD()`:**
```javascript
function aD() {
  let _ = u_6() ?? Md4();  // check env var override
  if (_) return _;
  let T = sG.join(ar(), "projects");
  return (sG.join(T, jP(Gd4()), Xd4) + sG.sep).normalize("NFC");
  // resolves to: ~/.claude/projects/<encoded-project-path>/memory/
}
```

Constants: `Xd4 = "memory"`, `kd4 = "MEMORY.md"`.

**`qPT()`** — builds the memory prompt block:
- If team memory enabled: loads both user memory dir and team memory dir, calls `buildCombinedMemoryPrompt()`
- If auto memory enabled: loads user memory dir (`aD()`), calls `KvR()`
- `TPT()` ensures the memory directory exists (mkdir)
- `Dp_()` reads the directory and emits a telemetry event (`tengu_memdir_loaded`)

**`KvR()`** — auto memory prompt text:
```
# auto memory

You have a persistent auto memory directory at `<path>`. [description] Its contents persist across conversations.

As you work, consult your memory files to build on previous experience.

## How to save memories:
- Organize memory semantically by topic, not chronologically
- Use the Write and Edit tools to update your memory files
- `MEMORY.md` is always loaded into your conversation context — lines after <zY> will be truncated, so keep it concise
- Create separate topic files (e.g., `debugging.md`, `patterns.md`) for detailed notes
```

The `zY` constant is the 200-line truncation limit visible in the MEMORY bootstrap rules.

---

## Compact Prompt — Hardcoded Template

The compact/summarization prompt is a hardcoded template with `${aJT}` as the injection point for custom `/compact [instructions]`:

```
Your task is to create a detailed summary of the conversation so far, paying close attention
to the user's explicit requests and your previous actions.
This summary should be thorough in capturing technical details, code patterns, and
architectural decisions that would be essential for continuing development work without losing context.

${aJT}

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests and intents in detail
2. Key Technical Concepts: List all important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created.
   Pay special attention to the most recent messages and include full code snippets where applicable
   and include a summary of why this file read or edit is important.
4. Errors and fixes: List all errors that you ran into, and how you fixed them. Pay special attention
   to specific user feedback that you received, especially if the user told you to do something differently.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results. These are critical for
   understanding the users' feedback and changing intent.
7. Pending Tasks: Outline any pending tasks that you have explicitly been asked to work on.
8. Current Work: Describe in detail precisely what was being worked on immediately before this
   summary request, paying special attention to the most recent messages from both user and assistant.
   Include file names and code snippets where applicable.
9. Optional Next Step: List the next step that you will take that is related to the most recent
   work you were doing. IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
   recent explicit requests, and the task you were working on immediately before this summary request.
```

**The `/compact [instructions]` argument** is interpolated at `${aJT}`. When no argument is given, `aJT` is empty or a default string. The string found verbatim in the binary — `"When you are using compact - please focus on test output and code changes. Include file reads verbatim."` — appears to be an example or prior default, not the current active default.

**Hook integration:** A `pre_compact` hook can return exit code 2 to block compaction, or exit code 0 with stdout to append custom compact instructions. The `Stop` hook fires on compact, clear, resume, and logout.

---

## Skills — YAML + `getPromptForCommand()`

Skills are defined in YAML/Markdown files with a `user-invocable` field. The runtime parses them into skill objects:

```javascript
{
  skillName: f,
  displayName: z.name,
  description: D,
  markdownContent: j,
  allowedTools: Y,
  argumentHint: z["argument-hint"],
  userInvocable: Oa(z["user-invocable"]),  // defaults true
  // ...
}
```

`getPromptForCommand()` returns the expanded prompt for a skill invocation. Built-in skills (e.g., `claude-api`) have their prompt inline in the binary, including the full TRIGGER/DO NOT TRIGGER logic visible in the system prompt.

The `slash_commands` field in session records lists which slash commands are active. The `skills` field in the session schema lists active skill names.

---

## Environment Info — `dH9()`

Built per-turn, injected as the `env_info_simple` cached section:

```
# Environment
You have been invoked in the following environment:
 - Primary working directory: <CWD>
 - Is a git repository: true/false
 [- This is a git worktree — an isolated copy...]
 [- Additional working directories: ...]
 - Platform: darwin
 - Shell: zsh
 - OS Version: Darwin 25.x.x
 - You are powered by the model named <name>. The exact model ID is <id>.

Assistant knowledge cutoff is <date>.
The most recent Claude model family is Claude 4.5/4.6. Model IDs — Opus 4.6: '...', Sonnet 4.6: '...', Haiku 4.5: '...'.
When building AI applications, default to the latest and most capable Claude models.

<fast_mode_info>
Fast mode for Claude Code uses the same <model> model with faster output. It does NOT switch to a different model. It can be toggled with /fast.
</fast_mode_info>
```

---

## Identity Strings

Three identity strings are defined as constants, selected by `kHT()` based on context:

```javascript
vAq = "You are Claude Code, Anthropic's official CLI for Claude."
EK6 = "You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK."
bK6 = "You are a Claude agent, built on Anthropic's Claude Agent SDK."
```

Selection logic:
- Vertex AI deployment → always `vAq`
- Non-interactive with `--append-system-prompt` → `EK6`
- Non-interactive without → `bK6`
- Interactive → `vAq`

---

## CLI Flags — Context and Prompt Control

Flags verified against v2.1.74 (`claude --help`). Only flags relevant to prompt assembly, tool control, model selection, and session configuration are listed. CDecode.md documented `--system-prompt-file` and `--append-system-prompt-file`; these do not appear in `--help` and may be undocumented or removed.

### System Prompt

| Flag | Effect |
|---|---|
| `--system-prompt <prompt>` | Replace the assembled system prompt with `<prompt>`. Bypasses all fragment assembly; CLAUDE.md, MEMORY.md, and skills are not injected. |
| `--append-system-prompt <prompt>` | Append `<prompt>` after the assembled system prompt. Triggers the `IDENTITY_SDK` identity string variant (see §5 identity branch). |

### Tool Control

| Flag | Effect |
|---|---|
| `--tools <tools...>` | Specify the built-in tool subset. `""` disables all tools; `"default"` uses all; named list e.g. `"Bash,Edit,Read"`. Filters the API tools array before the request. |
| `--allowedTools <tools...>` | Comma- or space-separated allow-list. Supports permission specifiers: `"Bash(git:*) Edit"`. |
| `--disallowedTools <tools...>` | Comma- or space-separated deny-list. Same specifier syntax. |
| `--mcp-config <configs...>` | Load MCP servers from JSON files or inline JSON strings (space-separated). |
| `--strict-mcp-config` | Use only MCP servers from `--mcp-config`; ignore all other MCP configuration (global, project, local settings). |

### Model and Session

| Flag | Effect |
|---|---|
| `--model <model>` | Override the model for this session. Aliases: `sonnet`, `opus`. Full name: `claude-sonnet-4-6`. Affects context window ceiling (200K for Sonnet/Opus 3/4). |
| `--effort <level>` | Set thinking budget: `low`, `medium`, `high`, `max`. |
| `--permission-mode <mode>` | Set permission mode: `acceptEdits`, `bypassPermissions`, `default`, `dontAsk`, `plan`, `auto`. Overrides `defaultMode` in settings.json. |
| `--settings <file-or-json>` | Load additional settings from a JSON file path or inline JSON string. Merged into the settings hierarchy below Managed scope. |
| `--betas <betas...>` | Beta feature headers to pass in API requests. API key users only. |
| `--agent <agent>` | Use a named agent definition for this session. Overrides the `agent` setting. |
| `--agents <json>` | Provide custom agent definitions inline as JSON: `'{"reviewer": {"description": "...", "prompt": "..."}}'`. |

### Non-interactive Output (with `--print`)

| Flag | Effect |
|---|---|
| `-p, --print` | Non-interactive mode: print response and exit. Workspace trust dialog is skipped. |
| `--output-format <format>` | `text` (default), `json`, `stream-json`. Only with `--print`. |
| `--max-budget-usd <amount>` | Maximum spend cap in USD. Only with `--print`. |
| `--input-format <format>` | `text` (default) or `stream-json` for streaming input. Only with `--print`. |

### Session Continuation

| Flag | Effect |
|---|---|
| `-c, --continue` | Continue the most recent conversation in the current directory. |
| `-r, --resume [value]` | Resume a specific session by UUID, or open an interactive picker. |
| `--fork-session` | When resuming, create a new session ID instead of reusing the original. |
| `--no-session-persistence` | Disable session file writes. Sessions cannot be resumed. Only with `--print`. |

---

## Terms

| Term | Definition |
|---|---|
| **Bun compile** | The compilation method used by Claude Code; bundles and compiles TS/JS to a native Mach-O/ELF binary with the JS engine embedded; function names are mangled, string literals are preserved |
| **obfuscated identifier** | A short mangled function or variable name (e.g., `r2`, `kaK`) produced by Bun's minifier; maps to a human-readable name in Appendix A |
| **string literal extraction** | The method of recovering prompt text from the binary by running `strings` and searching for known fragments; works because Bun preserves string literals verbatim |
| **tweakcc** | Binary patching tool by Piebald AI (`/Users/walter/Work/Claude/tweakcc`); modifies string literals in the Bun-compiled binary by regex-matching prompt content in `cli.js` and replacing with `.md` file content; requires `pnpm apply` after editing fragments |
| **CLAUDE.md fallback** | The mechanism in `claudemdReader()` that searches for `AGENTS.md`, `GEMINI.md`, `QWEN.md` in the same directory when `CLAUDE.md` is absent; only fires in the `else` branch — has no effect when CLAUDE.md exists |
| **team tools** | TeammateTool, SendMessageTool, TeamDeleteTool; registered only when `teamToolsEnabled()` returns true; absent from the tool list in standard solo sessions |
| **tengu_** | Internal telemetry event name prefix used throughout the binary (e.g., `tengu_memdir_loaded`, `tengu_claude_md_permission_error`, `tengu_ext_at_mentioned`); events are emitted via an internal analytics function |
| **JSONL session file** | A `.jsonl` file under `~/.claude/projects/<encoded-path>/` containing one structured JSON event per line for each session; used for tool utilization profiling (see CPrompts.md §9.3) |
| **cli.js** | The JavaScript bundle extracted from the Bun native binary; tweakcc patches this file and then rebuilds the binary |
| **BUILD_TIME** | An ISO 8601 timestamp embedded in `cli.js` (format: `BUILD_TIME:"2025-12-09T19:43:43Z"`); extracted by tweakcc for version-aware patching |

---

## References

- [tweakcc GitHub](https://github.com/Piebald-AI/tweakcc) — binary patching tool source
- [claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) — extracted prompt fragment repository
- [JSON Schema for settings.json](https://json.schemastore.org/claude-code-settings.json) — enables autocomplete and validation in VS Code/Cursor
- [Claude Code Documentation](https://code.claude.com/docs/en/settings) — settings.json schema, permissions, hooks (authoritative runtime reference)

---

## Notes

### Version Update Protocol

When Claude Code updates (e.g., to v2.2.x), perform these steps in order.

**Prompt maintenance (CPrompts.md scope):**
1. Check the extraction repo for fragment changes: `cd claude-code-system-prompts && git pull && git log --oneline -10`
2. Re-run the duplication analysis for any changed fragments against CLAUDE.md/MEMORY.md
3. Re-apply tweakcc patches (`pnpm apply`) — tweakcc re-detects new binary locations automatically
4. Verify that `includeGitInstructions` still exists in the new version schema

**Symbol table maintenance (CInternals.md scope):**

5. Run `strings` on the new binary and locate updated function names (re-identification procedure below)
6. Update Appendix A entries with new obfuscated IDs; note the version in each changed entry
7. Update pseudocode blocks in §3–§5 with new identifiers where obfuscated IDs appear
8. Check tweakcc's `agentsMd.ts` patch comments — they document `claudemdReader()` name changes across versions

**Re-identification procedure:**
```bash
strings /opt/homebrew/Caskroom/claude-code/<new-version>/claude > /tmp/new-strings.txt

grep -n "CLAUDE_CODE_SIMPLE" /tmp/new-strings.txt          # → systemPromptAssembler()
grep -n "tengu_claude_md_permission_error" /tmp/new-strings.txt  # → claudemdReader()
grep -n "auto memory" /tmp/new-strings.txt                 # → memoryPromptBuilder()
grep -n "teamName\|workerId" /tmp/new-strings.txt          # → teamToolsEnabled()
```

### Unexplored Internal Modules

The binary contains additional modules not yet analyzed: the LSP integration module, the worktree management module, the cron/task scheduler module, and the MCP server management module. These may expose additional prompt injection points or conditional load conditions not documented in the current catalog.

---

## Appendix A. Obfuscated Binary Symbol Table

The Claude Code binary is compiled with Bun, which minifies and obfuscates all function and variable names. The following table maps each obfuscated identifier (as it appears in the binary via `strings` extraction or tweakcc patch source) to its descriptive name, with its version, source of discovery, and functional description.

All identifiers are specific to **v2.1.74** unless otherwise noted. After a version update, re-identify by searching for characteristic string literals in the new binary (see §Notes → Version Update Protocol above).

### A.1 System Prompt Assembly Functions

| Descriptive Name | Obfuscated ID | Discovered In | Description |
|---|---|---|---|
| `systemPromptAssembler()` | `r2` | `strings` + CInternals.md | Main async function; assembles ordered array of system prompt sections at session start and after compaction |
| `identitySection()` | `kaK` | CInternals.md extraction | Builds position 1: identity string + core security rules; takes outputStyle parameter |
| `toolRulesSection()` | `MaK` | CInternals.md extraction | Builds position 2: system rules about tool output, permission mode behavior, AskUserQuestion reference |
| `codingDisciplineSection()` | `GaK` | CInternals.md extraction | Builds position 3: anti-over-engineering rules; skipped if `outputStyle.keepCodingInstructions === false` |
| `actionSafetySection()` | `haK` | CInternals.md extraction | Builds position 4: "Executing actions with care" — reversibility, blast radius guidance |
| `claudemdInjector()` | `ZaK` | CInternals.md extraction | Builds position 5: tool preference rules + CLAUDE.md injection; signature `ZaK(toolNames, claudemdContent)` |
| `gitRulesSection()` | `vaK` | CInternals.md extraction | Builds position 6: git operation rules |
| `prRulesSection()` | `yaK` | CInternals.md extraction | Builds position 7: PR creation and review rules |
| `cachedSection()` | `kp` | CInternals.md extraction | Creates a keyed cached section; signature `kp(key, fn)` — fn called once, result cached by key |
| `conditionalCachedSection()` | `t97` | CInternals.md extraction | Creates a cached section with a note for conditional loading; signature `t97(key, fn, note)` |
| `cachedSectionsEvaluator()` | `e97` | CInternals.md extraction | Evaluates array of cached sections, resolving promises and filtering nulls |
| `simpleModeCheck()` | `s_` | CInternals.md extraction | Returns true if `CLAUDE_CODE_SIMPLE` env var is set; triggers minimal 3-line system prompt |
| `modelOverrideSection()` | `PaK` | CInternals.md extraction | Builds `ant_model_override` cached section |
| `languageSection()` | `JaK` | CInternals.md extraction | Builds `language` cached section from `appState.language` |
| `outputStyleSection()` | `WaK` | CInternals.md extraction | Builds `output_style` cached section from output style config |
| `mcpInstructionsSection()` | `XaK` | CInternals.md extraction | Builds `mcp_instructions` cached section; gated by `hasMcpServers()` |
| `scratchpadSection()` | `CaK` | CInternals.md extraction | Builds `scratchpad` cached section |
| `featureFlagsSection()` | `SaK` | CInternals.md extraction | Builds `frc` (feature flags/rollout control) cached section |
| `summarizeToolResultsText` | `EaK` | CInternals.md extraction | String constant for the `summarize_tool_results` cached section |
| `briefModeSection()` | `baK` | CInternals.md extraction | Builds `brief` mode indicator cached section |

### A.2 CLAUDE.md and Memory Functions

| Descriptive Name | Obfuscated ID | Discovered In | Description |
|---|---|---|---|
| `claudemdReader()` | `_t7` (v2.1.29) / `q8B` (v2.0.0) / `gE2` (v1.0.24) | tweakcc `agentsMd.ts` | Reads CLAUDE.md from a path; contains AGENTS.md fallback `else` branch; signature `_t7(path, type)` |
| `projectInstructionsAssembler()` | `AWq` / `S3` | binary `strings` | Assembles all project-level injected files into the OVERRIDE-header-prefixed block |
| `memoryPromptBuilder()` | `qPT` | CInternals.md extraction | Builds the full MEMORY injection block at position 8a |
| `autoMemoryHeaderText()` | `KvR` | CInternals.md extraction | Returns static header block ("# auto memory…") plus how-to-save instructions |
| `memoryPathResolver()` | `aD` | CInternals.md extraction | Resolves the memory directory path: checks env var override, then config, then `~/.claude/projects/<encoded>/memory/` |
| `memoryDirEnvVar()` | `u_6` | CInternals.md extraction | Reads `CLAUDE_MEMORY_DIR` environment variable |
| `memoryDirConfig()` | `Md4` | CInternals.md extraction | Reads `autoMemoryDirectory` from settings config |
| `memoryDirEnsure()` | `TPT` | CInternals.md extraction | Ensures the memory directory exists (mkdir -p equivalent) |
| `memoryDirReader()` | `Dp_` | CInternals.md extraction | Reads the memory directory contents; emits `tengu_memdir_loaded` telemetry event |
| `MEMORY_LINE_LIMIT` | `zY` | CInternals.md extraction | Integer constant = 200; hard truncation limit for MEMORY.md content |
| `MEMORY_DIR_NAME` | `Xd4` | CInternals.md extraction | String constant = `"memory"` |
| `MEMORY_FILENAME` | `kd4` | CInternals.md extraction | String constant = `"MEMORY.md"` |
| `compactInstructionsInjectionPoint` | `aJT` | CInternals.md extraction | Template variable `${aJT}` in the compact prompt; receives `/compact [instructions]` argument |

### A.3 Environment and Identity Functions

| Descriptive Name | Obfuscated ID | Discovered In | Description |
|---|---|---|---|
| `envInfoBuilder()` | `dH9` | CInternals.md extraction | Builds the `env_info_simple` cached section: CWD, git status, platform, shell, OS version, model name/ID |
| `cwdGetter()` | `MT` | CInternals.md extraction | Returns current working directory |
| `dateGetter()` | `Nw_` | CInternals.md extraction | Returns current date string |
| `homeDir()` | `ar` | CInternals.md extraction | Returns home directory path |
| `projectRootGetter()` | `Gd4` | CInternals.md extraction | Returns project root directory |
| `encodeProjectPath()` | `jP` | CInternals.md extraction | Encodes project path for use in `~/.claude/projects/` directory name |
| `outputStyleReader()` | `QH9` | CInternals.md extraction | Reads the current output style configuration |
| `identityStringSelector()` | `kHT` | CInternals.md extraction | Selects identity string based on context (interactive, SDK, vertex, agent SDK) |
| `IDENTITY_STANDARD` | `vAq` | CInternals.md extraction | String constant: "You are Claude Code, Anthropic's official CLI for Claude." |
| `IDENTITY_SDK` | `EK6` | CInternals.md extraction | String constant: "You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK." |
| `IDENTITY_AGENT` | `bK6` | CInternals.md extraction | String constant: "You are a Claude agent, built on Anthropic's Claude Agent SDK." |
| `hasMcpServers()` | `gJ_` | CInternals.md extraction | Returns true when MCP servers are configured; gates `mcpInstructionsSection()` |
| `path.join()` | `sG.join` | CInternals.md extraction | Node.js path module reference (`sG` is the path module variable) |

### A.3.1 Environment variables: binary strings vs published reference

Symbols in this appendix (e.g. `CLAUDE_CODE_SIMPLE` → `simpleModeCheck()`, `CLAUDE_MEMORY_DIR` → `memoryDirEnvVar()`) show where the binary **reads** particular names. They are **not** an exhaustive or guaranteed-stable list of supported variables. Before depending on any name for automation or documentation, reconcile it with Anthropic’s maintained table: [Claude Code — Environment variables](https://code.claude.com/docs/en/env-vars). Variables that appear in disassembly or `strings` output but **not** in that table may be internal, deprecated, or version-specific.

For **platform-level** tool scaling (deferred tool loading, tool search, programmatic tool calling), see [Advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use); Claude Code maps part of that story to `ENABLE_TOOL_SEARCH` and MCP docs linked from the same env-var page.

### A.4 Team / Swarm Functions

| Descriptive Name | Obfuscated ID | Discovered In | Description |
|---|---|---|---|
| `teamToolsEnabled()` | `T6` | binary `strings` + team tool analysis | `isEnabled()` predicate for TeammateTool, SendMessageTool, TeamDeleteTool; true only when team name, worker ID, and team context all present |
| `isTeammate()` | `ZO` | binary `strings` (team module) | Returns true if current session is a teammate role (not team lead) |
| `isTeamLead()` | `YX` | binary `strings` (team module) | Returns true if current session is a team lead role |
| `isInProcessTeammate()` | `f2` | binary `strings` (team module) | Returns true if running as in-process (not tmux) teammate |
| `getTeammateContext()` | `fY` | binary `strings` (team module) | Returns full teammate context object |
| `isTeammateCheck()` | `gQ_` | binary `strings` (team module) | Combined predicate: team active AND not lead |
| `getTeamName()` | `QR` | binary `strings` (team module) | Returns team name string from session state |
| `getWorkerId()` | `w2` | binary `strings` (team module) | Returns worker ID for current session |
| `getWorkerName()` | `sK` | binary `strings` (team module) | Returns display name for worker |
| `getWorkerColor()` | `vz` | binary `strings` (team module) | Returns ANSI color for worker's display |
| `isTeamLeadOrSolo()` | `TGK` | binary `strings` (team module) | Returns true if team lead or no worker (solo) |
| `waitForTeammates()` | `WWq` | binary `strings` (team module) | Waits for all in-process teammates to become idle |
| `setDynamicTeamContext()` | `XvR` | binary `strings` (team module) | Sets dynamic team context (team name, roles) at runtime |
| `runWithTeammateContext()` | `wPT` | binary `strings` (team module) | Executes callback with teammate context active |
| `hasActiveTeammates()` | `JWq` | binary `strings` (team module) | Returns true if any in-process teammates are currently working |
| `hasPendingTeammates()` | `YPT` | binary `strings` (team module) | Returns true if any in-process teammates have active tasks |
| `isPlanModeRequired()` | `Gp_` | binary `strings` (team module) | Returns true when plan mode is required for current context |

### A.5 Skills and Tool Parsing

| Descriptive Name | Obfuscated ID | Discovered In | Description |
|---|---|---|---|
| `skillUserInvocableParser()` | `Oa` | CInternals.md extraction | Parses the `user-invocable` YAML field of a skill; defaults to `true` if absent |
| `TODOWRITE_TOOL_NAME` | `km` | binary `strings` | String constant = `"TodoWrite"` |
| `getPromptForCommand()` | (not recovered) | CInternals.md §Skills | Returns expanded prompt for a skill invocation |

### A.6 tweakcc Internal Symbols (TypeScript, not binary)

| tweakcc Function | File | Description |
|---|---|---|
| `applySystemPrompts()` | `patches/systemPrompts.ts` | Applies all system prompt patches to `cli.js` |
| `writeAgentsMd()` | `patches/agentsMd.ts` | Patches `claudemdReader()` to add AGENTS.md/GEMINI.md/QWEN.md fallback |
| `writeToolsets()` | `patches/toolsets.ts` | Applies all 8 sub-patches for toolset support |
| `writeToolFetchingUseMemo()` | `patches/toolsets.ts` | Sub-patch 2: modifies tool aggregation hook to filter by active toolset |
| `writeToolsetFieldToAppState()` | `patches/toolsets.ts` | Sub-patch 1: adds `toolset` field to app state initialization |
| `writeToolsetComponentDefinition()` | `patches/toolsets.ts` | Sub-patch 3: adds `/toolset` slash command UI component |
| `writeSlashCommandDefinition()` | `patches/toolsets.ts` | Sub-patch 4: registers `/toolset` as a slash command |
| `insertShiftTabAppStateVar()` | `patches/toolsets.ts` | Sub-patch 5: injects toolset state getter in statusline component |
| `appendToolsetToModeDisplay()` | `patches/toolsets.ts` | Sub-patch 6: appends `[toolset-name]` to mode display |
| `appendToolsetToShortcutsDisplay()` | `patches/toolsets.ts` | Sub-patch 7: appends toolset to shortcuts display |
| `writeModeChangeUpdateToolset()` | `patches/toolsets.ts` | Sub-patch 8b: injects toolset-swap code on plan mode entry/exit |
| `getAppStateSelectorAndUseState()` | `patches/toolsets.ts` | Helper: finds `appStateUseSelectorFn` and `appStateSetState` variable names in binary |
| `detectUnicodeEscaping()` | `patches/systemPrompts.ts` | Detects if binary uses `\uXXXX` escaping for non-ASCII chars |
| `extractBuildTime()` | `patches/systemPrompts.ts` | Extracts `BUILD_TIME` ISO timestamp for version-aware patching |
| `loadSystemPromptsWithRegex()` | `systemPromptSync.ts` | Loads all prompt `.md` files and generates regex patterns for matching in binary |
| `computeMD5Hash()` | `systemPromptHashIndex.ts` | Computes MD5 of applied prompt content for change detection |
| `setAppliedHash()` | `systemPromptHashIndex.ts` | Stores applied prompt hash in hash index for incremental patching |

### A.7 Settings and Configuration Keys (settings.json)

| Key | Type | Notes |
|---|---|---|
| `permissions.allow` | `string[]` | Permission rules auto-approved without dialog |
| `permissions.deny` | `string[]` | Permission rules blocked; evaluated first |
| `permissions.ask` | `string[]` | Permission rules requiring per-use confirmation |
| `permissions.defaultMode` | `string` | `default`, `acceptEdits`, `plan`, `dontAsk`, `bypassPermissions` |
| `permissions.additionalDirectories` | `string[]` | Extra working directories Claude can access |
| `hooks` | object | Hook event configurations (PreToolUse, PostToolUse, etc.) |
| `model` | `string` | Override default model; e.g., `"claude-sonnet-4-6"` |
| `includeGitInstructions` | `boolean` | Default `true`; set to `false` to suppress git workflow fragments (1558 tokens) |
| `autoMemoryDirectory` | `string` | Override memory directory path; not accepted in project settings |
| `outputStyle` | `string` | Selects tone variant; affects `codingDisciplineSection()` skip logic |
| `language` | `string` | Preferred response language |
| `env` | object | Environment variables applied to all sessions |
| `sandbox.enabled` | `boolean` | Enable bash sandboxing (default: `false`) |
| `teammateMode` | `string` | `auto`, `in-process`, or `tmux`; controls swarm display |
| `cleanupPeriodDays` | `number` | Days before inactive sessions are deleted (default: 30) |
| `effortLevel` | `string` | `low`, `medium`, `high`; persisted thinking budget |
| `disableAllHooks` | `boolean` | Disable all hooks and custom status line |
| `attribution.commit` | `string` | Git commit attribution text (empty to disable) |
| `attribution.pr` | `string` | PR description attribution text (empty to disable) |

### A.8 tweakcc MiscConfig Keys (tweakcc config.json `misc` section)

| Key | Type | Description |
|---|---|---|
| `showTweakccVersion` | `boolean` | Show tweakcc version in Claude Code startup |
| `showPatchesApplied` | `boolean` | Show which patches were applied at startup |
| `expandThinkingBlocks` | `boolean` | Auto-expand thinking blocks in UI |
| `enableConversationTitle` | `boolean` | Enable conversation title generation |
| `hideStartupBanner` | `boolean` | Hide the "Claude Code" startup banner |
| `hideCtrlGToEdit` | `boolean` | Hide the Ctrl+G to edit hint |
| `increaseFileReadLimit` | `boolean` | Increase file read size limit |
| `suppressLineNumbers` | `boolean` | Suppress line numbers in file read output |
| `suppressRateLimitOptions` | `boolean` | Suppress rate limit recovery UI options |
| `enableSwarmMode` | `boolean` | Enable team/swarm mode features |
| `enableSessionMemory` | `boolean` | Enable session memory (summary.md) system |
| `enableRememberSkill` | `boolean` | Enable the `/remember` skill |
| `tokenCountRounding` | `number \| null` | Round token counts to this multiple for privacy |
| `autoAcceptPlanMode` | `boolean` | Auto-accept plan mode entry |
| `allowBypassPermissionsInSudo` | `boolean \| null` | Allow `bypassPermissions` mode when running as sudo |
| `filterScrollEscapeSequences` | `boolean` | Filter ANSI scroll escape sequences from output |
| `enableWorktreeMode` | `boolean` | Enable worktree mode features |
| `statuslineThrottleMs` | `number \| null` | Throttle status line update interval |
| `tableFormat` | `string` | Table display format: `default`, `ascii`, `clean`, `clean-top-bottom` |

---

## Appendix B. Tool Name and List Constants

Source: CDecode.md binary analysis (v2.1.74). These are string and array literals extracted from the binary; they are minified variable names, not function obfuscations.

### B.1 Tool Name String Constants

| Obfuscated ID | Value | Tool |
|---|---|---|
| `U6` | `"Bash"` | Bash shell tool |
| `eO` | `"Glob"` | Glob file search |
| `G$` | `"Grep"` | Grep content search |
| `f7` | `"Read"` | Read file |
| `wz` | `"WebFetch"` | Web fetch |
| `Oh` | `"WebSearch"` | Web search |
| `e7` | `"Edit"` | Edit file |
| `o4` | `"Write"` | Write file |
| `rw` | `"NotebookEdit"` | Jupyter notebook edit |
| `Eh` | `"ToolSearch"` | ToolSearch — never deferred (hardcoded exception in `hX()`) |
| `mn6` | (Brief tool name) | Brief mode agent-to-user communication tool; also exempt from deferral |
| `C8R` | `null` | Null in standard builds; placeholder for an optional core tool |

### B.2 Tool List Array Constants

| Obfuscated ID | Construction | Contents |
|---|---|---|
| `Xc` | `[U6, C8R].filter(Boolean)` | `["Bash"]` in standard builds (C8R is null) |
| `E8R` | `[...Xc, eO, G$, f7, wz, Oh]` | Always-included set: Bash, Glob, Grep, Read, WebFetch, WebSearch |
| `b8R` | `[e7, o4, rw]` | Write-capable set: Edit, Write, NotebookEdit |

`E8R` is the set of tools whose schemas are always sent to the API at session start, regardless of the deferral feature flag. These are never subject to `hX()` deferral logic.

### B.3 Deferral Feature Flag

| Constant | Usage | Default |
|---|---|---|
| `"tengu_defer_all_bn4"` | Feature flag checked in `hX()` via `Jq("tengu_defer_all_bn4", true)` | `true` (second argument to `Jq` is the default) |

The `Jq()` call pattern is `Jq(flagName, defaultValue)` — the second argument is the default if the flag is not explicitly set. `tengu_defer_all_bn4` defaults to `true` per this signature. However, online verification (Anthropic API documentation and GitHub issues) confirms that in practice deferred loading in Claude Code applies only to MCP tools (`tool.isMcp === true`), not to built-in tools. The `E8R` set (Bash, Glob, Grep, Read, WebFetch, WebSearch) is always included unconditionally, preceding the `hX()` check entirely. The interaction between the `tengu_defer_all_bn4=true` default and the `E8R` always-included set is the key to understanding which tools are actually deferred. See CPrompts.md §Notes — Deferred Tool Loading for the current understanding.
