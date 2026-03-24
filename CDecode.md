# CDecode.md — Claude Code Binary Internals

Investigation of the Claude Code binary at `/Users/walter/Work/Claude/claude-code/` to determine where system prompt, tools, skills, MEMORY, and compact prompts are assembled.

---

## Binary Format

Claude Code is compiled with **Bun** (not pkg/Node), producing a Mach-O arm64 executable (~182MB). The JS source is embedded in the binary with mangled function names but preserved string literals. Inspection method: `strings` output filtered with python3 byte-search using known needle strings (e.g., `"You are Claude Code"`) to locate function boundaries, then extracting surrounding context windows.

---

## System Prompt Assembly — `r2()`

The function `r2()` builds the system prompt as an ordered array of sections. Reconstructed from extracted context:

```js
function r2() {
  return [
    identity_section(),         // "You are Claude Code, Anthropic's official CLI..."
    tool_use_rules(),           // tool call formatting, parallel calls, etc.
    coding_discipline(),        // task approach, security, over-engineering prohibitions
    action_safety(),            // reversibility, approval-required operations
    claude_md_injection(),      // CLAUDE.md content (user + project)
    memory_injection(),         // MEMORY.md content if present
    cached_sections(),          // environment info, skills, etc.
  ]
}
```

CLAUDE.md files are read from two locations and injected in order: `~/.claude/CLAUDE.md` (user-level) then the nearest `CLAUDE.md` walking up from the working directory (project-level). Content appears verbatim under a labeled section header.

---

## MEMORY Path Resolution — `aD()`

The function `aD()` resolves the memory directory path:

```
~/.claude/projects/<encoded-working-directory>/memory/
```

The working directory path is encoded by replacing `/` with `-` and prefixing with `-`. Example: working directory `/Users/walter/Work/Spank/spank-py` → encoded key `-Users-walter-Work-Spank-spank-py` → memory path `~/.claude/projects/-Users-walter-Work-Spank-spank-py/memory/`.

`MEMORY.md` in that directory is always loaded into context. Individual memory files are read on demand or loaded alongside `MEMORY.md`.

---

## Tool Name Constants

Extracted from binary string literals:

```js
U6  = "Bash"
eO  = "Glob"
G$  = "Grep"
f7  = "Read"
wz  = "WebFetch"
Oh  = "WebSearch"
e7  = "Edit"
o4  = "Write"
rw  = "NotebookEdit"
Eh  = "ToolSearch"    // never deferred — see hX() below
```

---

## Tool Lists

```js
Xc  = [U6, C8R].filter(Boolean)   // C8R = null in standard builds → Xc = ["Bash"]
E8R = [...Xc, eO, G$, f7, wz, Oh] // always-included: Bash, Glob, Grep, Read, WebFetch, WebSearch
b8R = [e7, o4, rw]                 // write tools: Edit, Write, NotebookEdit
```

`E8R` tools are always present in the tool schema list sent to the model. They are not subject to deferral.

---

## Deferred Tool Logic — `hX()`

Most tools other than `E8R` are deferred: their full JSON schemas are withheld from context and only their names appear in `<available-deferred-tools>` messages. The deferral decision function:

```js
function hX(tool) {
  if (tool.isMcp === true)        return true;   // all MCP tools deferred
  if (tool.name === Eh)           return false;  // ToolSearch never deferred
  if (mn6 && tool.name === mn6)   return false;  // Brief tool never deferred
  if (Jq("tengu_defer_all_bn4", true)) return true;  // feature flag: defer all (default: true)
  return tool.shouldDefer === true;              // per-tool flag fallback
}
```

`tengu_defer_all_bn4` defaults to `true`, meaning in normal operation nearly every tool except the always-included `E8R` set and ToolSearch itself is deferred. The model must call ToolSearch to retrieve schemas before invoking deferred tools.

---

## ToolSearch — Schema Fetcher

ToolSearch (constant `Eh = "ToolSearch"`) is the mechanism for fetching deferred tool schemas on demand. It is never deferred itself. Extracted description strings:

```
LuR  — short description used in tool listing
CuR() — returns full parameter schema
SuR  — usage guidance injected into system prompt
```

Query forms supported (from description text):
- `"select:Read,Edit,Grep"` — fetch exact tools by name
- `"notebook jupyter"` — keyword search, up to max_results best matches
- `"+slack send"` — require "slack" in name, rank by remaining terms

`max_results` defaults to 5.

---

## System Prompt Override — CLI Flags

These flags are defined in the CLI argument parser:

```
--system-prompt <prompt>           Replace the default system prompt with <prompt>
--system-prompt-file <file>        Replace from file (flag is hidden in --help)
--append-system-prompt <prompt>    Append <prompt> after the default system prompt
--append-system-prompt-file <file> Append from file (hidden)
--tools <tools...>                 Tool allow-list: "" disables all, "default" = all,
                                   "Bash,Edit,Read" = named subset
--disallowedTools <tools...>       Deny list; supports patterns e.g. "Bash(git:*)"
```

`--system-prompt` and `--system-prompt-file` are mutually exclusive replacements. `--append-system-prompt` and `--append-system-prompt-file` extend the assembled prompt without discarding it. The `--tools` empty-string form `""` is the documented way to disable all tool access for a session.

---

## Compact Prompt

The compact prompt is a hardcoded 9-section template. The injection point `${aJT}` carries any instructions passed via `/compact [instructions]`. When no instructions are given, `aJT` is an empty string and the template's default guidance applies. The template instructs the model to produce a structured summary that will replace prior context at the next turn.

---

## Skills Loading

Skills are loaded from the `~/.claude/skills/` directory (or equivalent project-local path). Each skill file is read and its content is available for injection when invoked via the Skill tool or a `/skill-name` slash command. The loaded skill list appears in the system prompt under a skills section. The binary scans for `.md` files in the skills directory and exposes them as callable entries.

---

## Environment Info — `dH9()`

`dH9()` assembles the environment block injected into the system prompt:

- Platform (`darwin`, `linux`, `win32`)
- Shell (from `$SHELL`)
- OS version (from `uname -r` or equivalent)
- Working directory
- Model ID (resolved at session start from the API response or config)
- Git status snapshot (branch, clean/dirty, recent commits)

The git status block visible at session start (`gitStatus:` in the system prompt) is produced here and is a one-time snapshot — it does not update during the session.

---

## Identity String Constants

The opening identity line (located by searching for `"You are Claude Code"`):

```
"You are Claude Code, Anthropic's official CLI for Claude."
```

This is the first string in the assembled system prompt. The remainder of the identity section follows immediately in `r2()`.
