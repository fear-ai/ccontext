# CCLaunch — Claude Code Deployment Guide

Configuration, environment setup, and operational reference for a Claude Code project. Covers the reasoning behind every decision, options not deployed, and directions for future expansion.

---

## Document Map

**Central document.** `CCLaunch.md` (this file) is the crafted, maintained guide: deployment, environment, authority and token-budget framing, external reference index, shortcuts, hooks, and appendices tied to **current** practice. It is the right place for conclusions that shift with new Claude Code versions, project policy, or operational priorities.

**Technical references (stable research outputs).** The other Markdown files in this corpus are written as **collations or decode logs**. Their structure and emphasis reflect the work they document; they are **not** expected to cross-reference this file. Use the table below as the canonical map. Specific findings inside them may be revised only when the underlying work is redone (re-extraction, re-decode, or major version re-analysis)—not on every edit to `CCLaunch.md`.

| Document | Purpose | Provenance and scope |
|---|---|---|
| **`CCLaunch.md`** | Operational and narrative hub: how to run Claude Code safely and efficiently for this project; where to find official docs; architecture summaries that support those decisions. | Authored here; updated for **presentation and present needs**. |
| **`CPrompts.md`** | **Technical reference (companion to `CInternals.md`):** architecture terminology, **fragment catalog** (conditional load, injection order), token-cost tables, reduction and tweakcc-oriented strategy, session-log-derived tool choices. | **Reviewed and collated** from the external extracted-fragments repository **`claude-code-system-prompts`** (and the binary version pinned at investigation time). Fragment lists and counts are **version-coupled** until re-collated. |
| **`CInternals.md`** | **Technical reference (companion to `CPrompts.md`):** **binary/runtime internals** — how the assembled system prompt, MEMORY paths, compact prompt, skills, env block, and CLI flags are realized in the shipped **`claude`** build; obfuscated symbol table and reconstruction notes. | Tied to the same shipped build as the fragment work behind `CPrompts.md`, but organized for **assembly-level** and **symbol-level** lookup after upgrades or when debugging mismatches with docs. |
| **`CDecode.md`** | Shorter **decode/decompile-oriented** pass on the **installed** `claude` native binary (e.g. `strings` / byte-context extraction): tool lists, deferral logic, CLI flags, compact template sketch. | **Parallel track** to `CInternals.md` (installation-focused decode), not a second pass over `claude-code-system-prompts`. Handy for orientation; **`CInternals.md` carries the fuller symbol-level record** where they overlap. |

**Cross-reference convention.** `CCLaunch.md` may cite `CPrompts.md`, `CInternals.md`, or `CDecode.md` for detail. The reverse is **not** required; those files stand alone as archival technical records. **§9** is the hub for **config drift**, **non-authoritative names**, and **how to verify** behavior against Anthropic’s live docs.

**Supporting artifacts:** This map lists **filenames only** (no path prefixes). The same names may appear at the repository root or grouped in one checkout folder depending on how CContext is laid out; only the basename matters for identification.

| File | Role |
|---|---|
| `CLAUDE_SAMPLE.md` | Patch document with rationale; merge into project `CLAUDE.md` manually |
| `MEMORY_SAMPLE.md` | `MEMORY.md` template: bootstrap and lifecycle patterns |
| `settings.json_sample` | Target shape for `~/.claude/settings.json` — apply manually |
| `ccenv.bash` | Shared PATH/venv exports for Bash tool subshells. **Documented** mechanism: set `CLAUDE_ENV_FILE` in `settings.json` → `env` to a path to this script so Claude Code **sources** it before each Bash command ([env-vars](https://code.claude.com/docs/en/env-vars)). **If your client does not actually source it** (only literal `env` key-value injection is visible), duplicate the same setup with a **PreToolUse** hook on `Bash` that sources this file — see §2.3 |

**Files on disk vs. the map.** The table above is the **maintained** artifact set. Other paths may exist in a checkout (scratch notes, one-off audits, local CSVs). **Presence on disk still shows up in search and file trees** — that is what “mention” means in tooling terms even when prose never names the file. Those files are **not** part of this map unless promoted here; treat them as **local workspace noise** until you explicitly adopt them.

**Active installs, not versioned in this repo:** `~/.claude/settings.json`, `~/.claude/ccenv.bash`, `~/.alias` (this corpus documents only the **`sc`** wrapper), and every **`CLAUDE.md` layer** the runtime loads (see §2.7).

---

## External reference index

Authoritative online sources cited across this corpus. Prefer these over second-hand summaries when behavior drifts across Claude Code versions.

| Topic | URL | Use |
|---|---|---|
| **Environment variables (Claude Code)** | [code.claude.com/docs/en/env-vars](https://code.claude.com/docs/en/env-vars) | Canonical list of supported `env` keys and semantics; compare to binary-derived notes in `CPrompts.md` / `CInternals.md` |
| **Settings (Claude Code)** | [code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings) | `settings.json` schema, precedence, managed config — reconcile local files after upgrades |
| **CLI reference** | [code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference) | Flags including `--debug`; use to confirm launch-time behavior |
| **Interactive mode (keyboard shortcuts)** | [code.claude.com/docs/en/interactive-mode](https://code.claude.com/docs/en/interactive-mode) | Full shortcut tables, multiline input, `/btw`, task list; basis for a one-page reference card |
| **Compaction (Messages API, beta)** | [platform.claude.com/docs/en/build-with-claude/compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) | Server-side `compact_20260112` strategy; default **trigger** **150,000 input tokens** (configurable, min 50,000) — **not the same mechanism as Claude Code’s in-app autocompact** |
| **Advanced tool use (platform)** | [anthropic.com/engineering/advanced-tool-use](https://www.anthropic.com/engineering/advanced-tool-use) | Tool Search Tool, programmatic tool calling, `defer_loading`, MCP toolsets — architectural pattern for large tool libraries without front-loading every schema |
| **Documentation index (llms.txt)** | [code.claude.com/docs/llms.txt](https://code.claude.com/docs/llms.txt) | Machine-discoverable map of CC docs pages |
| **Autocompact threshold (community)** | [github.com/anthropics/claude-code/issues/31806](https://github.com/anthropics/claude-code/issues/31806) | Deobfuscated `getAutoCompactThreshold` discussion (`effectiveWindow - 13000`, `Math.min` clamp); **not** official documentation; duplicate-closed issue |

**Technique linkage:** Reducing built-in tool exposure in Claude Code (`--tools`, `CLAUDE_CODE_SIMPLE` / `--bare`) aligns with the same goal as the platform’s deferred tool loading: pay token cost only for tools the session actually uses. MCP at scale is addressed with **Tool Search** and `ENABLE_TOOL_SEARCH` ([env-vars](https://code.claude.com/docs/en/env-vars)); the engineering article explains `defer_loading` and search-first workflows at the API level.

---

## 1. Architecture Overview

### 1.1 The Five Context Streams

Every Claude Code session assembles context from five independent sources:

1. **System prompt** — assembled at session start from ~67 fragments by
   `systemPromptAssembler()`. Conditional on settings, mode, and environment.
   Never changes within a session.

2. **Tool definitions** — injected alongside the system prompt. Each enabled
   tool's JSON schema and description consume tokens. The `--tools` flag
   controls which tools are present. MCP tool schemas are deferred until
   fetched via `ToolSearch`.

3. **Injected user files** — `CLAUDE.md` (position 5), `MEMORY.md` (position 8a
   when invoked), skills (on-demand). These are injected into the system prompt
   region, not the conversation.

4. **System reminders** — small injections mid-conversation (e.g., output style
   active, compaction notice). Not individually controllable.

5. **Conversation history** — the running transcript. The largest consumer.
   Replaced by a summary after compaction.

### 1.2 Authority Hierarchy

From highest to lowest:

```
Managed settings (MDM)           → absolute; cannot be overridden
Runtime deny rules               → absolute execution gate
Hooks (PreToolUse/PostToolUse)   → enforcement layer; fires before/after tool
CLAUDE.md (pos 5, OVERRIDE)      → highest model-space authority; survives compaction
disallowedTools / --tools        → filters tool availability before model sees them
allow rules / defaultMode        → approval dialog control
Tool descriptions                → behavioral guidance in tool schema
System prompt fragments          → general behavioral rules
Model training                   → lowest; overridden by all of the above
```

`CLAUDE.md` carries the `OVERRIDE` declaration and is re-injected after
compaction. It is the correct place for project-specific rules that must
persist across long sessions.

### 1.3 Token Budget and Compaction

Default context window: **200,000 tokens** for standard Sonnet/Opus-class models in Claude Code (unless a different model or extended window is selected).

**Epistemic stance (read this first):** Several numbers appear in docs, UI, reverse-engineered issues, and API reference (**~95%**, **13,000**, **187,000 / 93.5%**, **~33,000 / 16.5%**, **150,000**). **This corpus does not certify that any combination is simultaneously true** for your installed `claude --version`. Treat them as **labels from different layers** (marketing copy, status-line math, client UI meter, deobfuscated client logic, HTTP API defaults). Re-validate after every upgrade.

**Compaction-related surfaces (do not merge the numbers across rows):**

| Layer | What it is | Reported figures (verify locally) |
|---|---|---|
| **Published Claude Code env-var doc** | Describes `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`; states default autocompact at **approximately 95%** of context capacity ([env-vars](https://code.claude.com/docs/en/env-vars)) | **~95%** — documentation claim; **not** proven here to match runtime for your build |
| **Community reverse-engineering (GitHub)** | [Issue #31806](https://github.com/anthropics/claude-code/issues/31806) (closed duplicate) quotes deobfuscated `getAutoCompactThreshold`: `defaultThreshold = effectiveWindow - 13000`, then `Math.min(userThreshold, defaultThreshold)` so overrides **cannot raise** the threshold above that default | **13,000** token subtraction from **`effectiveWindow`**; if `effectiveWindow` = **200,000**, default = **187,000** tokens (**93.5%** of 200K). Denominator for `%` override is **`effectiveWindow`** (same issue). The issue title says “~83%”; the **arithmetic in the quoted snippet** matches **93.5%** for a 200K window — treat the title as unreliable |
| **`/context` UI** | In-session meter | Observed **~33,000** token **Autocompact buffer** slice on a 200K bar (see Appendix — Observed); **not** documented as tied to `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` |
| **Messages API (beta)** | `compact_20260112` | Default **trigger** **150,000 input tokens** ([Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)); **not** Claude Code CLI autocompact |

Informal **~5%** “headroom” talk sometimes appears as **100% − 95%** from the published default; that **does not** imply equality with the **`/context` ~16.5% buffer** or **13,000** token subtraction — those are separate ideas until Anthropic publishes one formula tying them together.

**Changelog / release notes:** No entry in this repository ties the **~95%** doc sentence to a specific Claude Code release. The public **anthropics/claude-code** GitHub repo is primarily examples/plugins, not a versioned runtime changelog; behavior must be confirmed with **`claude --version`**, **`/context`**, and optional `--debug` / telemetry inspection on your machine.

**Applying env to the main process vs. Bash tool:** Login-shell exports apply to **`claude` only if** the parent that launches it has them; IDE shortcuts may not. Prefer `settings.json` → `"env"` for portability ([env-vars](https://code.claude.com/docs/en/env-vars), [Settings](https://code.claude.com/docs/en/settings)). The **`env` object applies key-value pairs** to the Claude Code process; it does not, by itself, execute a shell script. For the **Bash tool** (non-interactive bash, no login init), the **published** variable is **`CLAUDE_ENV_FILE`**: upstream documents it as a **script sourced before each Bash command** (same env-vars page). **Verify** sourcing with a probe in §9; if behavior on your build does not match (PATH/venv never updates), use a **PreToolUse** hook on `Bash` to `source` `ccenv.bash` instead of relying on `CLAUDE_ENV_FILE` alone. See §2.3.

System prompt baseline (before conversation): ~8,000–10,000 tokens depending on tools and fragments. See `CPrompts.md §7.4` for the savings table.

---

## 2. Environment Setup

### 2.1 Installation

Claude Code is distributed as a compiled npm package (`@anthropic-ai/claude-code`)
or as a platform-specific native binary:

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

The native installer (preferred for performance) places the binary at
`/usr/local/bin/claude`. Both forms embed the same minified JavaScript source.

**Version pinning:** CC updates frequently and system prompt content, permission
behavior, and hook schemas can change across versions. The current deployment
is pinned to **v2.1.74**. See `CPrompts.md` introduction for extraction method.

### 2.2 ~/.claude — User-level Configuration

**Location:** `~/.claude/`

**Scope:** Applies to all CC sessions for this user, across all projects.
Lower priority than project-level `<repo>/.claude/` and managed settings.

**Contents:**

| Path | Purpose |
|---|---|
| `settings.json` | Main config: model, permissions, env, hooks. Canonical target: `settings.json_sample` |
| `ccenv.bash` | Intended to run before each Bash tool call (via `CLAUDE_ENV_FILE` per docs, or via PreToolUse if sourcing is not observed). PATH, venv, Python vars. See §2.3 |
| `projects/<slug>/` | Session JSONL files. One directory per project (slug = path-derived). Not config |
| `debug/` | Debug logs when `--debug` is used. Not config |

**Worktree intersection:** The native `--worktree` flag creates worktrees at
`<repo>/.claude/worktrees/<name>/`, not in `~/.claude/`. Only session storage
in `~/.claude/projects/` gains new entries per worktree (each worktree = distinct
slug). No other ~/.claude config is worktree-specific.

**Preference:** Central local configuration (`~/.claude/settings.json`) is preferred now, as a familiar, single place to review. Per-project config in repo `.claude/` is recommended for projects with different CC needs (MCP, hooks, allow rules) but adds another location to maintain.
Exception: deny rules for secrets (`.env`, `~/.ssh`) belong in user-level config, as they apply to all work.

### 2.2a User-level `settings.json`

**Location:** `~/.claude/settings.json`

**Canonical target:** `settings.json_sample`. Apply by replacing the current file. The sample includes inline comments explaining each decision.

**Key decisions and their effects:**

| Key | Value | Effect | Tokens saved |
|---|---|---|---|
| `includeCoAuthoredBy` | `false` | Omits "Co-authored-by: Claude" from git commits | 0 tokens; behavioral |
| `gitAttribution` | `false` | Disables git user attribution in CC commits | 0 tokens; behavioral |
| `alwaysThinkingEnabled` | `false` | Extended thinking disabled by default | Significant per-call; save compute |
| `model` | `"claude-sonnet-4-6"` | Pins model; avoids prompt for selection | 0 tokens; operational |
| `includeGitInstructions` | `false` | Removes 1558-token git workflow guide | **1558 tokens** |
| `outputStyle` / coding discipline | *(see §5.1)* | **Published** knob: `keep-coding-instructions` in custom output-style Markdown frontmatter ([Output styles](https://code.claude.com/docs/en/output-styles)). **Binary-derived:** object form `outputStyle: { "keepCodingInstructions": … }` in `settings.json` (see `CPrompts.md` / `CInternals.md`) — not shown in public settings examples; verify before relying on it | See §5.1 |
| `env.ENABLE_CLAUDEAI_MCP_SERVERS` | `"false"` | Disables claudeai-mcp plugin | Prevents startup OAuth errors |
| `env.CLAUDE_ENV_FILE` | path to `ccenv.bash` | **Per [env-vars](https://code.claude.com/docs/en/env-vars):** script sourced before each Bash command. **If your build does not source it**, use PreToolUse (§2.3) | Operational |
| `env.BASH_DEFAULT_TIMEOUT_MS` | `"300000"` | **Per [env-vars](https://code.claude.com/docs/en/env-vars):** default timeout for long-running bash commands. Per-call limits may still use **`timeout_ms`** on the Bash tool invocation; **verify** this env var actually changes behavior (§9) | Operational safety |

**Permission model — deny rules:** Block access to sensitive file paths regardless
of any allow rule. Evaluated first in the permission chain.

```json
"deny": [
  "Read(./.env)",
  "Read(./.env.*)",
  "Read(./config/credentials.json)",
  "Read(./node_modules/**)",
  "Read(./secrets/**)",
  "Read(~/.aws/**)",
  "Read(~/.ssh/**)"
]
```

**Permission model — allow rules:** Pre-approve specific commands to suppress
the approval dialog. The rule format is `Bash(<prefix>:<glob>)` for shell
commands and `ToolName` for native tools.

Key decisions on the allow list:
- **`git fetch` excluded** — intentionally; run manually when you want it
- **`bash:*` excluded** — `bash -c "..."` wraps bypass all per-command rules;
  run scripts directly as `./script.sh` instead
- **`less`, `top`, `ping` excluded** — interactive or run-forever; hang the
  Bash tool indefinitely
- **`cd`, `ls`, `echo`, `pwd`, `date`, `whoami`, `wc`, `which`, `sort`, `uniq`
  excluded** — already in CC's built-in safe list; no entry needed
- **`Read`, `WebFetch`, `WebSearch` excluded** — auto-approved in default mode
- **`Edit`, `Write` retained** — require explicit allow to suppress
  per-invocation confirmation dialogs in default mode

Full allow list: `settings.json_sample §allow`.

### 2.3 `CLAUDE_ENV_FILE` — Per-Bash-call Environment

**Location:** `~/.claude/ccenv.bash`

**Why it exists:** CC runs each Bash tool call in a non-interactive bash
subshell. Standard shell init files (`.bashrc`) are not sourced in
non-interactive mode. `pyenv` shims and virtual environments therefore do not
appear on PATH automatically. **`CLAUDE_ENV_FILE`** names a script that **Anthropic’s environment-variable reference** describes as **sourced before each Bash command** ([env-vars](https://code.claude.com/docs/en/env-vars)). That same page lists **`BASH_DEFAULT_TIMEOUT_MS`** as the default timeout for long-running bash commands — separate from any **`timeout_ms`** field on an individual Bash tool call.

**Operational caveat:** Some sessions observe only **literal** `env` key-value injection into the CC process and **no** pre-Bash `source` of the file named by `CLAUDE_ENV_FILE`. If `PATH` / venv never reflect `ccenv.bash` inside Bash tool calls, do not assume the variable alone is sufficient: add a **`PreToolUse` hook** on `Bash` that runs `source ~/.claude/ccenv.bash` (or equivalent). Re-check after upgrades.

**Scope limitation:** Environment variable exports only. Aliases defined in
this script do not take effect because bash parses the command string before
the sourced file's aliases are available in non-interactive mode. For hard
blocks, use `settings.json` deny rules. For behavioral rules, use `CLAUDE.md`.

**Process-level env (compaction, MCP, etc.):** Variables must be visible to the
**Claude Code executable’s process**. Practical options: (1) `export` in
`~/.zshrc` and launch via `~/.alias` functions that run `claude` (inherits
environment); (2) `~/.claude/settings.json` → `"env"` (applies whenever CC reads
that config). Putting these **only** in `ccenv.bash` affects **Bash tool**
subshells if `CLAUDE_ENV_FILE` points there — **not** the main CC process unless
you also use (1) or (2). A PreToolUse hook that runs a script in a **child
process** cannot inject env into the parent CC process; hooks are for gating or
side effects, not a replacement for zshrc/`settings.json` inheritance for the app
itself. For **Bash tool** PATH/venv, **`CLAUDE_ENV_FILE` is the documented first choice** when sourcing works; otherwise use **PreToolUse** as above — non-interactive bash still does not source your usual login files.

**Current contents** (stripped to essentials):

```bash
# pyenv: add shims if not already on PATH
if [ -d "$HOME/.pyenv/shims" ] && [[ ":$PATH:" != *":$HOME/.pyenv/shims:"* ]]; then
  export PATH="$HOME/.pyenv/shims:$HOME/.pyenv/bin:$PATH"
fi

# Activate project venv if present (CC sets CWD to project root)
if [ -f ".venv/bin/activate" ]; then source ".venv/bin/activate"; fi

export PYTHONDONTWRITEBYTECODE=1   # suppress __pycache__ noise
export PYTHONUNBUFFERED=1          # pytest output appears immediately
```

**Why pyenv:** `python3 --version` resolves through `/Users/walter/.pyenv/shims/python3`
(version 3.12.12 via `.pyenv/version`). Without the PATH addition, `python3` in
Bash tool calls would resolve to system Python.

**Why `PYTHONUNBUFFERED`:** Without it, pytest output is buffered and arrives as
a single block at process exit — useless for monitoring a running test suite.

**Commented-out options** (available, not deployed):
- `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR=1` — prevents CWD drift after `cd`
  commands in Bash calls
- `BASH_MAX_OUTPUT_LENGTH=20000` — truncates Bash tool output to N chars;
  default is unbounded (context bloat risk for verbose commands)

### 2.4 Shell Wrapper Function

```bash
# Standard session — full tool set minus TodoWrite (0 calls across all sessions)
sc() {
  claude --tools "Bash,Edit,Glob,Grep,Read,WebFetch,WebSearch,Write,Agent,Task" "$@"
}
```

- `sc`: starts a fresh session
- `sc --continue`: resumes the most recent session in the current directory
- `sc --resume <id>`: resumes a specific session by ID (or opens an interactive picker with no ID argument)

**`--tools` effect:** Tool names not in the list are not sent to the model.
The model has no awareness of excluded tools. Tool descriptions for excluded
tools are not injected into the system prompt, saving their token cost.
`ToolSearch` is always available regardless of `--tools`.

**Limited deployment:** The corpus describes a **full** target (narrow allow list in `settings.json_sample`, optional project hooks). A real machine often adopts it **in pieces** — §3.1.1 explains **partial migration**, **merge**, and **posture** in prose. **`~/.alias`** is documented here with **`sc`** only; your shell may still define other helpers. **`~/.claude/ccenv.bash`** vs login-shell exports: see **§9** (mutability and verification). **What actually merges on one real setup** (Spank + home) is in §3.1.2.

Reconcile any numbers in this file against **`claude --version`**, your **`~/.claude/settings.json`**, and **`/context`** on your machine.

**Why `--tools` whitelist, not `--disallowedTools`:** We use an explicit
whitelist (`--tools "Bash,Edit,..."`) rather than `--disallowedTools "TodoWrite"`.
Initial suggestion for `--disallowedTools` assumed it would remove TodoWrite's
schema (~2161 tokens) from the API request in addition to `--tools`. Investigation
showed that `--tools` with a whitelist already excludes non-listed tools from the
request entirely — their schemas are not sent. `--disallowedTools` would add
redundant exclusion with no token benefit. The whitelist also makes the intended
tool set explicit and self-documenting. Reserve `--disallowedTools` for cases
where a single tool must be excluded without specifying a full whitelist.

**`--append-system-prompt`:** Appended after the standard system
prompt, after `CLAUDE.md`. Has OVERRIDE-level authority in the sense that it
appears last in the system prompt assembly and can restate or tighten
constraints for a specific session without modifying `CLAUDE.md`. Use for
ephemeral scope constraints (analysis-only, read-only, topic-limited sessions).

### 2.5 Repo `.claude/` — Project-level Configuration

**Two JSON files:**

| File | Purpose |
|---|---|---|
| `settings.json` | Hooks, project-specific allow rules, MCP. Merged with ~/.claude/settings.json |
| `settings.local.json` | Local overrides. CC writes here when user approves a tool via dialog — accumulates allow entries. Should be gitignored |

**Difference:** `settings.json` is the canonical project config. `settings.local.json`
is created/updated by CC when the user approves a tool that wasn't in the allow
list. It grows over time with entries like `Bash(pytest:*)`, `Bash(rg:*)`. That
accumulation can look like "trash" but is structural — CC uses it to avoid
re-prompting for the same command. Trash (stale paths, duplicate rules) may be
historical from prior cleanup; periodic review of both files is recommended.

**Maintenance burden:** Two locations (～/.claude + repo .claude) means two places
to review. Per-project config is positive when projects need different hooks or
MCP. For single-project use, central local (~/.claude only) is simpler and is our current preference; create repo `.claude` when the project needs **git-tracked** hooks, MCP, or allow rules that differ from `~/.claude`.

**When repo .claude exists:** Add `.claude/worktrees/` to `.gitignore` if using
`--worktree`. Add `.claude/settings.local.json` to `.gitignore`.

When deploying hooks (see §8. Pending Applies), the repo may need:

**`<repo>/.claude/settings.json`** — hook registration (illustrative `PreToolUse` entry; replace the hook path with your own script):

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/your-hook.sh",
        "timeout": 5
      }]
    }]
  }
}
```

**`<repo>/.claude/hooks/`** — place executable hook scripts here and register them as above. PreToolUse hooks receive tool input on stdin (JSON) and can allow or block the call by printing `{"continue": true}` or `{"continue": false, "stopReason": "…"}`. Use for project-specific ceilings, path blocks, or audit logging as documented in Claude Code’s hooks guide.

**`<repo>/.claude/settings.local.json`** — developer-specific allow overrides
accumulated via approval dialogs. Should be gitignored.

### 2.6 Hooks — Reference

From the embedded CC hooks configuration guide
(`system-prompt-hooks-configuration.md`, v2.1.76):

> *Hooks run commands at specific points in Claude Code's lifecycle.*

Hook input arrives on stdin as JSON:
```json
{
  "session_id": "abc123",
  "tool_name": "Bash",
  "tool_input": { "command": "python3 -m pytest tests/" }
}
```

To block, output JSON with `"continue": false`:
```json
{ "continue": false, "stopReason": "explanation shown to model" }
```

Available hook events:

| Event | Matcher | Purpose |
|---|---|---|
| `PreToolUse` | Tool name | Run before tool; can block |
| `PostToolUse` | Tool name | Run after successful tool |
| `PostToolUseFailure` | Tool name | Run after tool fails |
| `PermissionRequest` | Tool name | Run before permission prompt |
| `PreCompact` | `"manual"` / `"auto"` | Before context compaction |
| `PostCompact` | `"manual"` / `"auto"` | After compaction; receives summary |
| `Stop` | — | When Claude stops (includes clear, resume, compact) |
| `SessionStart` | — | Session start |
| `UserPromptSubmit` | — | User message submission |

Hook types: `command` (shell command), `prompt` (LLM condition evaluation),
`agent` (agent with tools). Only `command` is deployed here.

### 2.7 `CLAUDE.md` — Project Instructions

**Location in prompt:** Position 5, with `OVERRIDE` declaration. Higher
authority than all system prompt fragments. Re-injected after compaction.

**Where CC looks (typical; confirm on your `claude --version`):**

1. **`~/.claude/CLAUDE.md`** — user-level instructions. Lowest priority among injected project-style instruction files when both user and repo rules exist.
2. **Walk upward from the current working directory** — nearest **`CLAUDE.md`** (and supported alternate names) on the path to the filesystem root. **Project / repo `CLAUDE.md`** is usually the one in the git root you opened; it **wins** over the user file when both apply. CC’s merge order is “user first, then project” so later content can refine or override earlier.
3. **Additional directories** — if you use `--add-dir` / multi-root workflows, see official **`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`** ([env-vars](https://code.claude.com/docs/en/env-vars)): default is **not** to load `CLAUDE.md` from added dirs unless set to `1`.

**Copies that are *not* auto-loaded (basenames only in this table):**

| File | Role |
|---|---|
| `CLAUDE_SAMPLE.md` | Patch notes for editing a real `CLAUDE.md`; CC does not load it unless its content lives under a discovered path from §2.7. |
| `CLAUDE.md` (template) | May exist as a Cursor-rule seed or sample in a doc repo; CC only loads a file named like this when it is **on the walk-up path from CWD** or under **`~/.claude/`**, not merely because a template exists elsewhere. |
| Project `CLAUDE.md` | The copy at the **git root of the project you opened** is the usual instruction surface for that repo. |
| `~/.claude/CLAUDE.md` | Optional user-wide layer; applies together with project files per CC merge rules. |

**Mechanism:** Matched files are read and injected verbatim, with the OVERRIDE preface described in `CInternals.md` / `CPrompts.md`.

**Proposed replacement:** `CLAUDE_SAMPLE.md`. Changes from current:
- §3 Edit: extended to cover code files via `read-before-modifying` complement
- §7: adds gtimeout requirement, 3-process ceiling, and stop-on-repeated-failure
  directive
- §8 Git: adds `git add -A/.`, `git stash pop/apply`, sole-gate statement, npm
- §8 Persistent env: adds Write tool advisory note
- §9 Design: adds system-boundary validation rule
- §11 Security: adds HEC data handling elaboration

**Relationship to system prompt fragments:** `CLAUDE.md` adds project-specific
context to general rules and should not duplicates them. Four fragments in particular
(`security`, `blocked-approach`, `no-unnecessary-error-handling`,
`read-before-modifying`) provide general rules; `CLAUDE_SAMPLE.md` adds examples of project-specific application of each. See `CCLaunch.md §5.1` for the decision.

### 2.8 `MEMORY.md`

**Location in prompt:** Position 8a, injected only when MEMORY.md exists in
the project root. Preceded by a built-in header. Truncated at 200 lines
(`MEMORY_LINE_LIMIT`).

**Purpose:** Session-to-session continuity. Carries current state, in-progress
decisions, and things that haven't yet landed in their authoritative document.
Not a permanent record — entries are removed when the underlying decision is
captured elsewhere.

**Template:** `MEMORY_SAMPLE.md`. Includes a protected bootstrap
section (session orientation order, update triggers, lifecycle rules, stopping
rule, post-compaction behavior) and the developer preference (no emojis, plain
text).

**Regaining focus on features/requirements/design:** If sessions regress to
leading with "number of tests passed" instead of features/requirements/design:
(1) CLAUDE.md: add rule "Lead with feature/requirement/design context; test
counts are supporting evidence." (2) MEMORY bootstrap: lead with requirement and design intent before implementation detail."
(3) Session start: prompt "What feature/requirement are we working on?" before
"What tests pass?"

---

## 2.9 Postponed — For Later Consideration

**PostCompact hook.** The STOP-on-restart rule has not reliably prevented
automatic execution after compaction. A PostCompact hook would inject a
signal into the model's context at resumption. Captured for later:

```json
"PostCompact": [{
  "matcher": "auto",
  "hooks": [{
    "type": "command",
    "command": "echo 'Context compaction complete. Read MEMORY.md. Present orientation summary. Wait for developer confirmation before executing any task.'"
  }]
}]
```

Add to `~/.claude/settings.json` hooks section when/if deployed.

**Notification hook.** macOS desktop notification when CC needs attention
(permission prompt, idle, auth complete). Test: `osascript -e 'display
notification "Claude Code needs your attention" with title "Claude Code"'`.
Justification: long test runs (30–90s); developer switches windows; no signal
when CC finishes.

---

## 3. Configuration Reference

### 3.0 Keyboard shortcuts (reference-card seed)

For a future **Getting Started** one-pager, pull the full tables from [Interactive mode](https://code.claude.com/docs/en/interactive-mode). Commonly used shortcuts (macOS/Linux; Option-as-Meta may be required on macOS — see that page):

| Shortcut | Action |
|---|---|
| `?` | Show shortcuts for current environment |
| `Ctrl+C` | Cancel input or generation |
| `Ctrl+D` | Exit session |
| `Ctrl+L` | Clear screen (history retained) |
| `Ctrl+O` | Toggle verbose / expand MCP lines |
| `Ctrl+B` | Background bash/agents (tmux: press twice) |
| `Ctrl+T` | Toggle task list (not theme picker) |
| `Ctrl+G` | Open prompt in system editor |
| `Esc` `Esc` | Rewind / summarize |
| `Shift+Tab` / `Alt+M` | Cycle permission modes |
| `Option+P` / `Alt+P` | Switch model |
| `/` | Commands and skills |
| `!` | Bash mode |
| `@` | File path mentions |

Multiline: `\` + Enter, or `Option+Enter` on macOS, or `Shift+Enter` in several terminals; run `/terminal-setup` where needed.

### 3.1 `settings.json` Key Reference

User-level keys commonly used in this setup. The **complete** environment-variable list is maintained upstream: [Environment variables](https://code.claude.com/docs/en/env-vars). Reconcile any name from binary analysis (`CPrompts.md`, `CInternals.md`) against that table before relying on it across upgrades.

Keys not in `settings.json_sample` may be omitted here deliberately (see §5).

| Key | Type | Effect |
|---|---|---|
| `model` | string | Default model. CLI `--model` overrides per-session |
| `includeCoAuthoredBy` | bool | Appends "Co-authored-by: Claude" to git commits |
| `gitAttribution` | bool | CC git attribution in commits |
| `alwaysThinkingEnabled` | bool | Extended thinking on by default |
| `includeGitInstructions` | bool | Injects 1558-token git workflow fragment |
| `outputStyle` | string **or** object | **Published:** string name (e.g. `"Explanatory"`). **Binary-derived:** object with `keepCodingInstructions` / `name` may affect `codingDisciplineSection()` and reminders (`CPrompts.md`, `CInternals.md`) — **undocumented** in public settings JSON examples; for a **documented** switch use custom output style frontmatter **`keep-coding-instructions`** ([Output styles](https://code.claude.com/docs/en/output-styles)) |
| `env` | object | Key-value pairs set in CC's process environment at startup |
| `permissions.allow` | array | Pre-approved tool invocation patterns |
| `permissions.deny` | array | Blocked tool invocation patterns (evaluated first) |
| `permissions.defaultMode` | string | `"default"`, `"acceptEdits"`, `"auto"`, `"bypassPermissions"` |
| `hooks` | object | Hook event registrations (see §2.6) |

### 3.1.1 Partial migration, merge, and posture

**Partial migration** means you adopt **one** part of the target setup (for example project hooks or a `sc` wrapper) **without** replacing the whole **`~/.claude/settings.json`** allow list. That is normal: the sample file is a **target**, not an atomic deploy.

**Merge** is how Claude Code combines scopes. User-level and project-level `settings.json` are **merged**; hooks and `permissions` from both participate. If the home file still carries dozens of `Bash(*:*)` rules and the project file adds **only** `hooks`, the **effective allow set** remains the **home** list unless the project file **also** replaces or tightens `permissions` or adds **`deny`** rules that block patterns the home file granted.

**Posture** is the policy choice behind the list shape. A **wide** home `permissions.allow` trades fewer approval dialogs for higher risk that wildcards pre-approve destructive commands (`git push`, `pip install`, arbitrary `bash -c`). A **tight** list restores dialogs for those prefixes and is what `settings.json_sample` encodes. Relying on **`CLAUDE.md`** alone while the allow list stays wide leaves **binary-level** auto-approval in force for anything the wildcard matches — model instructions are not the same layer.

**Operational takeaway:** “Wide” is the default outcome of interactive “always allow” growth. Tightening is a **deliberate** edit to `permissions.allow`, usually coordinated with `CLAUDE.md` and Bash rules.

### 3.1.2 Observed effective configuration (one developer, Spank + home)

Claude Code **merges** user and project `settings.json`. In the setup reviewed for this paragraph, **`~/.claude/settings.json`** holds **`model`**, feature booleans, **`permissions.allow`** with many broad Bash patterns, **`permissions.deny`** for a few secret paths, and **no** `hooks` block. **`spank-py`** contributes a **project** `settings.json` that contains **only** `hooks`: **`PreToolUse`** on **`Bash`** runs **`~/.claude/ccenv.bash`** so each Bash tool invocation gets pyenv shims, optional `.venv` activation, and shared exports. **Permissions are not narrowed** by that project file — the running allow list is still the **wide** home array. **`sc`** in **`~/.alias`** launches `claude` with an explicit **`--tools`** whitelist; that changes **which tools are registered**, not the string patterns in `permissions.allow` for Bash when Bash is present. **`CLAUDE.md`** loaded for Spank is the file at the **Spank repository root** when CWD is that tree; an optional **`~/.claude/CLAUDE.md`** would still apply per §2.7 if present. **`~/.alias`** and **`ccenv.bash`** may duplicate exports; only names on Anthropic’s [env-vars](https://code.claude.com/docs/en/env-vars) table are authoritative — see **§9**.

**`env` block keys with special meaning to CC:**

| Variable | Effect |
|---|---|
| `CLAUDE_ENV_FILE` | **Documented:** script sourced before each Bash command ([env-vars](https://code.claude.com/docs/en/env-vars)). **If not observed:** PreToolUse `source` fallback (§2.3) |
| `BASH_DEFAULT_TIMEOUT_MS` | **Documented:** default timeout for long-running bash commands ([env-vars](https://code.claude.com/docs/en/env-vars)). Per-call **`timeout_ms`** may still apply; verify effect (§9) |
| `BASH_MAX_TIMEOUT_MS` | Maximum allowed timeout (per-call override ceiling) |
| `BASH_MAX_OUTPUT_LENGTH` | Truncate Bash output to N characters |
| `ENABLE_CLAUDEAI_MCP_SERVERS` | `"false"` disables claudeai-mcp auto-fetch |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Compaction threshold (e.g. `"85"`); can only lower |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Token denominator for that percentage ([env-vars](https://code.claude.com/docs/en/env-vars)); decouples compaction math from status-line `%` |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | `"1"` resets CWD after each Bash call |

### 3.2 CLI Flags — Context and Prompt Related

| Flag | Effect |
|---|---|
| `--tools <list>` | Comma-separated tool names; only listed tools sent to model |
| `--disallowed-tools <list>` | Exclude specific tools from default set |
| `--system-prompt <text>` | Replace the system prompt entirely |
| `--append-system-prompt <text>` | Append to system prompt after CLAUDE.md |
| `--model <name>` | Override model for this session |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `auto`, `bypassPermissions` |
| `--settings <path>` | Use alternate settings.json |
| `--mcp-config <json>` | Inline MCP config (replaces MCP file config) |
| `--strict-mcp-config` | Ignore user-level MCP settings; use only `--mcp-config` |
| `--continue` / `-c` | Resume most recent session in current directory |
| `--resume` / `-r` | Resume specific session by ID or interactive picker |
| `--no-session-persistence` | Don't save session to disk |
| `--betas <list>` | Enable beta features |
| `--effort <level>` | Thinking effort level |
| `--max-budget-usd <n>` | Session cost ceiling |
| `--allowedTools <list>` | Alias for `--tools` |
| `--print` / `-p` | Non-interactive; print response and exit |
| `--output-format` | `text`, `json`, `stream-json` (use with `--print`) |

### 3.3 Permission Rule Syntax

```
ToolName                           # any invocation of this tool
ToolName(pattern)                  # invocation where input matches pattern
Bash(command:*)                    # Bash call where command starts with "command "
Bash(git diff *)                   # Bash call matching this exact prefix
Read(./path/*)                     # Read of paths matching glob
Write(~/.ssh/**)                   # Write to any path under ~/.ssh/
```

Patterns are prefix-matched against the tool's primary input parameter. For
`Bash`, that is the command string. For `Read`/`Write`/`Edit`, it is the
file path.

**Wildcard behavior:** `Bash(git:*)` matches any command beginning with `git`
— including `git push --force` and `git reset --hard`. This is the principal
misconfiguration risk in accumulated allow lists.

---

## 4. Session Analysis

### 4.1 Working from JSONL

Session transcripts live under `~/.claude/projects/<encoded-path>/` as
`*.jsonl` (one JSON object per line). Inspect them with shell tools you already
trust — e.g. `ls -lt` for recency, `wc -c` for size, `jq` for structured
fields, `rg` for substrings. Tool-invocation counts can be derived by
filtering lines for the event shapes your Claude Code version emits (see
`CPrompts.md` for profiling notes). There is no single canonical schema across
all releases; re-sample one file after each upgrade.

Current project corpus: 23 sessions, 165 MB, 5,279 total tool calls.

**Top findings from session analysis:**

| Tool | Calls | Decision driven |
|---|---|---|
| Read | 1826 | Always in `--tools` |
| Edit | 1736 | Always in `--tools` |
| Bash | 999 | Always in `--tools` |
| TodoWrite | **0** | **Excluded from `--tools`; saves 2161 tokens** |
| Agent | 46 | Retained in `--tools` |
| Task | 41 | Retained in `--tools` (all TaskOutput/Update/Create/List included) |
| ToolSearch | 44 | Always available; cannot be excluded |

### 4.2 Use Cases

**Token cost estimation:** Cross-reference session size (bytes), tool call
count, and known per-fragment token costs from `CPrompts.md §7.4` to estimate
actual context consumption per session.

**Compaction detection:** Sessions that hit the compaction threshold will show
a discontinuity in conversation history. A PostCompact hook (not yet deployed)
could log the summary size and session token count at compaction time.

**Permission-related lines:** If your JSONL includes permission or
`tool_result` events, filter with `jq` or `rg` for denied or rejected tool
uses. Exact field names differ by CC version — inspect a fresh file before
relying on a one-liner.

**Tool frequency drift:** Recompute tool-invocation histograms after a batch of
new sessions so `--tools` and allow lists stay aligned with actual usage.

**Single-session forensics:** For any session file, structured review (quoted
evidence, classification, what the developer did next) is a manual pattern;
keep methodology in whatever working notes you use outside this corpus.

**Session pruning — keep latest N:** No built-in CC command. To retain only
the 10 most recent sessions per project:

```bash
cd ~/.claude/projects/-<path>
ls -t *.jsonl | tail -n +11 | xargs rm
```

`ls -t` sorts by mtime (newest first). `tail -n +11` skips the first 10.
Adjust the number for a different retention count. Run periodically or when
the `--resume` picker becomes unwieldy.

### 4.3 Future Enhancements

**Token counting per session:** Parse the `usage` field in JSONL events to sum
`input_tokens` and `output_tokens` per session. This gives actual cost data
rather than size estimates.

**Compaction event detection:** Look for entries where the conversation summary
is injected. Correlates with context depth and session length to tune
`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`.

**Timeline visualization:** Plot tool calls over time within a session to
identify phases (exploration, implementation, testing) and detect anomalies
(repeated retries, long gaps, tool clustering).

**Subagent session linkage:** Agent and Task tool calls produce child sessions.
Linking parent session → child session IDs would give a complete picture of
parallel work and context inheritance.

**Permission event dashboard:** Aggregate approval/denial events across all
sessions to identify which commands are frequently prompted (candidates for
the allow list) vs frequently denied (candidates for the deny list).

**Regression detection:** When CC updates to a new version, re-run session
analysis to detect tool call pattern changes that could indicate prompt
behavior changes warranting updates to `settings.json_sample` or `CLAUDE_SAMPLE.md`.

---

## 5. Options Not Deployed

### 5.1 Coding discipline fragments (`codingDisciplineSection`) — Option A Chosen

**Decision:** Keep all 13 doing-tasks fragments active (Option A).

**Where the knob lives in published docs:** [Output styles](https://code.claude.com/docs/en/output-styles) documents **`keep-coding-instructions`** in the **YAML frontmatter of a custom output style Markdown file** (under `~/.claude/output-styles` or project `.claude/output-styles`), not the top-level `settings.json` schema examples (those show `"outputStyle": "Explanatory"` as a **string** only).

**Binary / extraction path:** The shipped client code paths described in `CPrompts.md` and `CInternals.md` also read an **`outputStyle` object** with a **`keepCodingInstructions`** property; `settings.json_sample` uses that object form for convenience. Treat it as **version-coupled and schema-undocumented** until Anthropic lists it beside other `settings.json` keys — if it has no effect in your build, use the **custom style frontmatter** key above instead.

When suppression works, `keepCodingInstructions === false` (object form) or the equivalent frontmatter choice skips the entire `codingDisciplineSection` (~700 tokens, 13 fragments). Four fragments have no CLAUDE.md equivalent and correct specific failure modes:

| Fragment | System rule | CLAUDE.md complement |
|---|---|---|
| `security` (67t) | OWASP top 10 in code | `CLAUDE_SAMPLE.md §11`: HEC event data attack surface |
| `blocked-approach` (90t) | Don't retry identical failures | `CLAUDE_SAMPLE.md §7`: stop on repeated test failure, report state |
| `no-unnecessary-error-handling` (64t) | Only validate at system boundaries | `CLAUDE_SAMPLE.md §9`: names queue/asyncio/inter-module as non-boundaries |
| `read-before-modifying` (46t) | Read before proposing changes | `CLAUDE_SAMPLE.md §3`: extends to Splunk.md, API.md, HEC.md |

The system fragment provides the rule; `CLAUDE_SAMPLE.md` provides the
project-specific application. Neither duplicates the other.

The 9 remaining fragments are either redundant with CLAUDE.md
(`no-time-estimates`), low-value (`help-and-feedback`, `ambitious-tasks`), or
represent discipline the model applies reliably without instruction
(`avoid-over-engineering`, `no-premature-abstractions`).

**If token pressure becomes critical:** Implement Option B — add concise
equivalents of the 4 critical fragments to `CLAUDE.md` (~80 tokens), then
disable coding-discipline injection via the **documented** custom-style **`keep-coding-instructions`** frontmatter (or the object form in `settings.json` if verified on your build) for a net saving of ~620 tokens.

### 5.2 Autocompact — official controls vs. other numbers

**Controls (authoritative):** [Environment variables](https://code.claude.com/docs/en/env-vars) documents **`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`** (lower % → compact earlier; values above the default threshold have no effect) and **`CLAUDE_CODE_AUTO_COMPACT_WINDOW`** (token denominator for that percentage, and decoupling from status-line `%`). **Layered figures** (~95% doc, ~187k/200k from community deobfuscation, `/context` buffer slice, **150k** Messages API default) stay in **§1.3** — do not merge them across rows. Community issue [#31806](https://github.com/anthropics/claude-code/issues/31806) is **not** a release note; trust its **arithmetic**, not its title, and re-check your binary after upgrades. **Dead env names** and **verification** live in **§9**.

### 5.3 MCP Servers — Disabled

`ENABLE_CLAUDEAI_MCP_SERVERS=false` in the `env` block prevents the
`claudeai-mcp` plugin from auto-fetching Gmail and Google Calendar MCP servers
at startup. Without this, the plugin attempts OAuth, fails (unauthenticated),
logs errors, and adds to startup time.

To enable specific MCP servers, use `--mcp-config` with an explicit JSON
configuration rather than the auto-fetch mechanism:
```bash
claude --mcp-config '{"mcpServers":{"myserver":{"command":"npx","args":["-y","my-mcp"]}}}'
```

### 5.4 tweakcc Patches — None Applied at v2.1.74

**Decision:** No binary patches applied. The `--tools` flag handles the only
high-value target (TodoWrite, 2161 tokens), and the remaining candidates either
provide useful guidance or are too small to justify patching complexity.

See §6 (tweakcc section) for full analysis.

### 5.5 `--permission-mode` — Not Explicitly Set

Default mode is `default`: prompts for write/edit operations not in the allow
list. The explicit allow entries for `Write` and `Edit` in `settings.json`
suppress these prompts in practice.

`acceptEdits` mode: auto-accepts all Edit/Write operations.
`auto` mode: no prompts for any tool; runs fully autonomously.
`bypassPermissions` mode: skips the permission layer entirely.

Not set in `sc()` because the allow list covers the necessary auto-approvals
and retaining the permission dialog for non-allowed writes is the intended behavior.

### 5.6 `alwaysThinkingEnabled: false`

Extended thinking (token budgets for internal reasoning visible as `<thinking>`
blocks) is disabled by default. This reduces per-call token cost significantly
for routine operations.

Enable per-session: `sc --betas thinking-token-budget-2024-06-20` or set
`alwaysThinkingEnabled: true` in `settings.json` for persistent extended thinking.

### 5.7 `BASH_MAX_OUTPUT_LENGTH` — Not Set

Without this, long-running commands (test suites, verbose builds) can produce
output that consumes thousands of context tokens. Set in `ccenv.bash`:

```bash
export BASH_MAX_OUTPUT_LENGTH=20000
```

Commented out currently. Enable if context bloat from verbose Bash output
is observed in session logs.

### 5.8 Toolsets via tweakcc — Not Deployed

tweakcc supports named toolsets (`/toolset` command in CC) that go beyond
`--tools`. Unlike `--tools`, toolsets completely hide the tool schema and
description from the model. The `--tools` flag controls availability but the
tool descriptions still appear in the system prompt.

> *"Built-in tools that are not in the currently active toolset are not even
> sent to the model. As a result, Claude has no idea of tools that are not
> enabled in the current toolset."*
> — tweakcc README §Toolsets

For maximum token reduction, a tweakcc toolset would save the description
tokens for all excluded tools. Currently, `--tools` handles tool exclusion and
the description savings come from excluding tool descriptions by name. The
`TodoWrite` description (2161 tokens) is already excluded via `--tools`.

---

## 6. tweakcc and claude-code-system-prompts

### 6.1 claude-code-system-prompts

**Repository:** `github.com/Piebald-AI/claude-code-system-prompts`
**Local path:** `/Users/walter/Work/Claude/claude-code-system-prompts/`

Maintained by Piebald AI. Contains all system prompt fragments extracted from
the CC npm package's compiled JavaScript, organized as individual markdown files
with YAML frontmatter (`ccVersion`, `description`, `variables`).

Purpose for this project:
- **Fragment reference:** Read any fragment to understand exact token cost
  and content before deciding to suppress or patch it
- **Changelog:** Track what changes across CC versions to identify when
  `settings.json_sample` or `CLAUDE_SAMPLE.md` need updates
- **tweakcc source:** tweakcc downloads from this repo's `data/prompts` branch
  to populate `~/.tweakcc/system-prompts/`

The extracted fragments underpin all token cost estimates in `CPrompts.md §7.4`.
Template variables (e.g., `${BASH_TOOL_NAME}`, `${GREP_TOOL_NAME}`) appear as
literals in the extracted files; they are interpolated at runtime by CC.

### 6.2 tweakcc — Intent and Mechanism

**Repository:** `github.com/Piebald-AI/tweakcc`
**Local path:** `/Users/walter/Work/Claude/tweakcc/`

tweakcc patches Claude Code's minified JavaScript source, reading
customizations from `~/.tweakcc/config.json`. From the README:

> *"tweakcc works by patching Claude Code's minified `cli.js` file... For
> native installations it's extracted from the binary using node-lief, patched,
> and then the binary is repacked."*

> *"When you update your Claude Code installation, your customizations will be
> overwritten, but they're remembered in your configuration file, so they can
> be reapplied by just running `npx tweakcc --apply`."*

**Capabilities beyond system prompt patching:**

| Feature | Relevance |
|---|---|
| Custom toolsets (`/toolset`) | Alternative to `--tools`; hides tools completely |
| Input pattern highlighters | Developer UI; no functional effect |
| Session naming (`/title`, `/rename`) | Session organization |
| Custom themes | Developer UI |
| Thinking verb customization | Developer UI |
| MCP startup optimization | ~50% startup speedup with non-blocking connections |
| Token count rounding | Rounds displayed token counts |
| Opus Plan 1M mode | Combines Opus planning + Sonnet 1M context execution |
| AGENTS.md support | Allows per-project agent configuration via AGENTS.md |
| Auto-accept plan mode | Auto-accepts plan phase outputs |
| Unlock session memory | Enables experimental MEMORY.md session persistence |
| `CLAUDE_CODE_CONTEXT_LIMIT` | Override context limit for custom API endpoints |
| `--restore` | Restores CC to unpatched state |

**System prompt patching mechanism:**

For each fragment, tweakcc maintains a markdown file in
`~/.tweakcc/system-prompts/`. To suppress a fragment, replace its content with
an empty file or a minimal placeholder. From the README:

> *"To customize any part of the system prompt, simply edit the markdown files
> in `~/.tweakcc/system-prompts/` and then run `npx tweakcc --apply`."*

When CC updates and Anthropic modifies a fragment you have also modified,
tweakcc generates an HTML diff showing both changes side by side for conflict
resolution.

### 6.3 Current Deployment Decision — No Patches at v2.1.74

**Frozen at v2.1.74.** No patches applied. Rationale by target:

| Candidate fragment | Tokens | Decision | Reason |
|---|---|---|---|
| `tool-description-todowrite.md` | 2161 | **Handled** | `--tools` exclusion; no binary patch needed |
| `system-prompt-subagent-delegation-examples.md` | 588 | Not patched | Agent (46 calls) and Task (41 calls) active; examples are relevant |
| `system-prompt-writing-subagent-prompts.md` | 365 | Not patched | Same; quality of subagent prompts depends on this guidance |
| `bash-alternative-*.md` (6 files) | ~154 total | Not patched | Small; "communication" variant (don't echo for output) is unique |
| `bash-prefer-dedicated-tools.md` | 71 | Not patched | Small; reinforces Grep/Glob/Read preference |

**When to reconsider:** If Agent and Task tool calls drop to near-zero across
new sessions, the subagent delegation fragments become eligible. If the project
stops using agentic workflows, ~953 tokens become recoverable with no guidance
loss.

**Version stability note:** tweakcc patches are version-specific. At each CC
update, patches must be reapplied (`npx tweakcc --apply`). If a patched
fragment is changed by Anthropic, tweakcc generates a diff for manual
resolution. Deferring patches until necessary reduces maintenance overhead.

### 6.4 Future Opportunities

**Targeted fragment suppression:** The 13 doing-tasks fragments are individually
patchable. If Option B is eventually chosen (disable coding discipline section),
a subset could be suppressed via tweakcc rather than the all-or-nothing
`keepCodingInstructions` setting. Example: suppress only `no-time-estimates`
(47t) and `help-and-feedback` (24t) while retaining the others.

**Custom `blocked-approach` text:** The current fragment references
`${ASK_USER_QUESTION_TOOL_NAME}` as the escalation path. A patched version
could name the project-specific escalation: "report queue state and the exact
failing command before asking."

**Subagent model configuration:** tweakcc supports configuring which model
each subagent type uses (Plan, Explore, general). For cost optimization,
Explorer subagents could be routed to a faster/cheaper model while the main
agent uses Sonnet.

**MCP startup optimization:** tweakcc's non-blocking MCP connection feature
(~50% startup speedup) is relevant if MCP servers are re-enabled in the future.

**Session memory (`/remember`):** tweakcc 4.0.0 added an experimental
`/remember` skill and `MEMORY.md` persistence mechanism. The current
`MEMORY_SAMPLE.md` already implements the same lifecycle rules
manually. The tweakcc version would integrate with CC's native session
persistence — worth evaluating when the feature stabilizes.

---

## 7. Further Expansion

**Structured decision log:** Add a compact repository-local decision table
(what, outcome, status: applied/pending/conditional, artifact). Currently
decisions are embedded in `CPrompts.md §7`, which mixes rationale with
technical reference; a separate table would surface state faster.

**Cross-version tracking:** After CC updates beyond the version pinned in
`CPrompts.md` / `CInternals.md`, re-sample JSONL from a few sessions and
recompute tool-frequency histograms. Update `settings.json_sample`
allow list if new commands appear frequently. Re-evaluate tweakcc targets
against updated fragment sizes.

**Hook expansion:** Beyond any hooks you already deploy, candidates:

| Hook event | Purpose |
|---|---|
| PostCompact | Log summary size and session token count; calibrate compaction threshold |
| PreToolUse on Write | Block writes to shell init files (~/.bashrc etc.) regardless of model behavior |
| PostToolUse on Bash | Log command and exit code to a session audit file |
| SessionStart | Print current token budget and compaction threshold reminder |

**Read-only session pattern:** A non-wrapper workflow uses `claude` with a
smaller `--tools` set plus `--append-system-prompt` (analysis-only constraints)
and optional `--append-system-prompt` that summarizes **project-local**
long-running work before deep reading begins.

**Automatic session archiving:** A shell wrapper that, after each session,
appends one summary line (session id, byte size, rough tool counts from `jq`
or `rg`) to a `session-log.txt`. Provides a running record without opening
full transcripts each time.

**MEMORY.md automation:** A `PostCompact` hook that automatically updates
`MEMORY.md` with the compaction timestamp and summary word count. Provides
visibility into how frequently compaction fires and what context is being
summarized.

**Toolset for documentation sessions:** A tweakcc toolset named `docs` that
includes only `Read`, `Grep`, `Glob`, `WebSearch`, `WebFetch` — for
documentation review sessions where no file modification is intended. More
restrictive than a default **`sc`** session (which still includes Bash, Edit,
Write, etc.) and removes tool description tokens for all editing tools.

---

## 8. Pending Applies

Manual actions required before the setup is fully deployed:

1. **`~/.claude/settings.json`** — Replace with `settings.json_sample`.
   Adds `includeGitInstructions`, `env` block, narrows allow list to 23 entries.

2. **Project `CLAUDE.md`** — Apply diff from `CLAUDE_SAMPLE.md`.
   Key additions (section numbers refer to **that** patch file): §7 gtimeout/limit,
   §8 git/npm/Write advisory, §9 validation scope, §11 HEC data security, §3
   read-before-modify.

3. **`<repo>/.claude/settings.json`** (optional) — If the project uses
   repo-local hooks or MCP, create this file and register hooks as in §2.5.
   Otherwise a user-level `~/.claude/settings.json` may suffice.

---

## 9. Mutability, false leads, and verification

### 9.1 Mutability

Defaults, supported `settings.json` keys, and env semantics **change across Claude Code releases**. After every upgrade, re-check [Settings](https://code.claude.com/docs/en/settings), [Environment variables](https://code.claude.com/docs/en/env-vars), and the [CLI reference](https://code.claude.com/docs/en/cli-reference). This repository’s prose is **not** a substitute for those pages.

### 9.2 False leads and stale shapes

Ignore env names **not** listed on [env-vars](https://code.claude.com/docs/en/env-vars) — in particular the pair **`CLAUDE_COMPACT_THRESHOLD`** and **`CLAUDE_COMPACT_THRESHOLD_TOKENS`** (often copy-pasted into shell or `ccenv` scripts; **no documented effect** in the official table). Other common confusions: **`DISABLE_BUG_COMMAND`** (legacy; use **`DISABLE_FEEDBACK_COMMAND`**, [env-vars](https://code.claude.com/docs/en/env-vars)); **`includeGitInstructions`** vs **`CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS`** (when both are set, the **env** wins — avoid contradicting pairs, [env-vars](https://code.claude.com/docs/en/env-vars)); **Messages API** **150k** compaction vs **CLI** autocompact ([Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) vs [env-vars](https://code.claude.com/docs/en/env-vars)); GitHub **issue titles** vs **quoted code** in [#31806](https://github.com/anthropics/claude-code/issues/31806); **`settings.json` keys** you inherited from an old file but no longer see in [Settings](https://code.claude.com/docs/en/settings) — diff against the doc when upgrading.

### 9.3 Confirming what actually loaded

Use **`claude --version`** first. Then **`claude --debug`** ([CLI reference](https://code.claude.com/docs/en/cli-reference)): short session, capture output to a file, scan for config scope and model — it is **verbose**; turn off after the check. Use **`/context`** after changing `--tools`, `MEMORY.md`, or major settings (§1.3, Appendix). **Probe** `permissions.deny` on a throwaway path if you need proof the rules engine saw your JSON. For **Bash-tool** env, `echo` a variable you define **only** in `ccenv.bash` (not as a flat key under `settings.json` → `env`) inside a tool call — confirms **`CLAUDE_ENV_FILE` sourcing** or your **PreToolUse** `source` ran. Inspect **`~/.claude/debug/`** when `--debug` is on (§2.2). **Habit:** after substantive config or version changes, one **`/context`** snapshot; if static totals move by more than a few hundred tokens, log the date (Appendix pattern).

---

## Appendix — Observed (`/context`)

**Observation date:** 2026-03-20.

**Source:** Claude Code `/context` command immediately after a relatively fresh
session launch. Model shown in the UI: **claude-sonnet-4-6**. Stated window:
**200,000 tokens**.

This appendix records **empirical UI numbers** for comparison with §1.3 and
`CPrompts.md §7.4`. It does not replace those sections; where they disagree with
live `/context`, treat this appendix as **version- and build-specific evidence**
and re-run `/context` after CC upgrades.

### A.1 Raw meter (as displayed)

| Category | Tokens | Share of 200k |
|---|---:|---:|
| **Total used (headline)** | 17,000 | 8.5% |
| System prompt | 5,600 | 2.8% |
| System tools | 8,700 | 4.3% |
| Memory files | 3,000 | 1.5% |
| Skills | 237 | 0.1% |
| Messages | 8 | 0.0% (rounded) |
| **Free space** | 150,000 | 74.8% |
| **Autocompact buffer** | 33,000 | 16.5% |

**Free space — Claude’s `/context` figures:** **150,000** tokens **and**
**74.8%** of the 200k window (both as shown in the UI; treat the percentage as
Claude’s reported share, not a hand-derived value). The token count is the
**exact remainder** of the three-way partition: `200,000 − 17,000 − 33,000 =
150,000`. Pure arithmetic on those rounded heads would give 150,000 ÷ 200,000 =
**75.0%**; the displayed **74.8%** is slightly lower, which is consistent with
the product computing the percentage from a **finer-grained token total** (e.g.
~149,600 free → 74.8%) while still labeling free space as **150k** in the token
column — **do not assume token and percentage rows round the same way**.

**Sum check:** 17,000 + 150,000 + 33,000 = 200,000. The UI therefore treats
**free space** as budget *excluding* both current **used** rows and a fixed
**Autocompact buffer** slab — not as `200,000 - used` alone.

**Fixed overhead before substantive chat:** System prompt + system tools +
memory files + skills = 5,600 + 8,700 + 3,000 + 237 ≈ 17,537 tokens,
consistent with the **~17k** headline (rounding in the UI).

### A.2 Relation to §1.3 (compaction numbers)

This appendix records **one** `/context` snapshot. §1.3 states the general rule:
**do not assume** the **~95%** doc line, **13k/187k** reverse-engineering
([#31806](https://github.com/anthropics/claude-code/issues/31806)), **33k** UI buffer,
and **150k** Messages API default are one consistent story — they are **different layers**
and may not all match your build.

**Messages API:** **150,000** input-token default trigger for `compact_20260112` only
([Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)).

### A.3 Relation to “~8k–10k system baseline” (§1.3, `CPrompts.md §7.4`)

The **~8,000–10,000** token figure in §1.3 refers to a **tuned** baseline
(fragment and tooling choices in that analysis — e.g. narrowed `--tools`,
specific `settings.json` flags) and is **not** the same decomposition as the
`/context` rows. The observed session shows **5.6k** system prompt and **8.7k**
system tools **in addition** to **3k** memory files: the **prompt-only** estimate
and the **prompt + tool schemas + MEMORY + skills** total will diverge by design.
When comparing token plans, always state whether the number is **prompt-only**,
**tools only**, or **full static overhead** (all non-message categories).

### A.4 Guidance — minimizing context use (Claude Code)

The largest levers, ordered roughly by impact for typical workflows,
with pointers to this file:

1. **Conversation history** — Still the dominant consumer in long sessions (§1.1).
   Start a **new session** when the task changes; avoid carrying irrelevant
   transcript. After compaction, verify critical facts are in **MEMORY.md** or
   repo docs so they survive summary loss.

2. **Tool surface (`--tools`)** — Use an explicit whitelist (`sc` in §2.4). Excluding
   unused tools removes their **schema and description** from system tools
   (e.g. TodoWrite exclusion documented in §4.1 / §6.3). Do not assume
   `--disallowedTools` adds savings beyond `--tools` if the whitelist already
   omits the tool (§2.4).

3. **System prompt fragments (`settings.json`)** — `includeGitInstructions: false`
   saves the git workflow fragment (§2.2a). Suppressing **`codingDisciplineSection`** (~700 tokens) is covered in §5.1: **published** path is custom output style **`keep-coding-instructions`**; the sample’s **`outputStyle` object** is binary-derived — verify or prefer frontmatter. Option B there (move critical rules into `CLAUDE.md`, then disable) remains the escape hatch when token pressure is extreme.

4. **MEMORY.md** — Injected when present; subject to line truncation in CC
   (§2.8). Keep it **short and current**; move settled material into canonical
   repo docs. Remove bootstrap duplication if the same rules exist in
   `CLAUDE.md`.

5. **Skills** — Fewer installed skills and smaller `SKILL.md` bodies reduce the
   skills row; use **explicit-only** invocation where the platform supports
   disabling automatic skill selection for rarely used skills (see Agent Skills
   docs in Cursor/Codex ecosystems; CC skill tooling evolves by version).

6. **MCP** — Disabled auto-fetch via `ENABLE_CLAUDEAI_MCP_SERVERS` in this
   deployment (§5.3). Each enabled MCP server adds tools and startup cost; keep
   the set minimal.

7. **Bash output** — Set `BASH_MAX_OUTPUT_LENGTH` in `ccenv.bash` if verbose test
   or build logs bloat the transcript (§5.7). Prefer paginated or filtered
   commands when investigating large output.

8. **Compaction threshold** — `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` and
   `CLAUDE_CODE_AUTO_COMPACT_WINDOW` per [env-vars](https://code.claude.com/docs/en/env-vars)
   (§1.3, §5.2, §9). Do **not** confuse with Messages API `compact_20260112` (**150k** default
   trigger, [Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)).

9. **Subagents (Agent / Task)** — Each subagent has its **own** context (see
   `CPrompts.md` / CC subagent documentation). Parallel runs multiply token
   use; delegate when isolation is worth the cost, not for trivial single-shot
   tasks.

10. **`CLAUDE.md` size** — Project instructions are high-authority and persist
    across compaction (§1.2, §2.7). Keep them **dense and referential** (link to
    `Product.md`, `API.md`, etc.) instead of pasting large specifications.

**Re-measurement:** See **§9** (closing paragraphs).
