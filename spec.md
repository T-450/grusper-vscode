# opencode-handoff Plugin Specification

**Version:** 0.5.0  
**Source:** https://github.com/joshuadavidthomas/opencode-handoff  
**License:** MIT  
**Minimum OpenCode version:** 1.2.15

---

## Overview

`opencode-handoff` is an [OpenCode](https://opencode.ai/) plugin that generates focused continuation prompts for transferring work across AI sessions. When starting a new session, the AI spends considerable time exploring the codebase before it can do productive work. This plugin frontloads that exploration by capturing what matters from the current session and injecting it as context into a new one.

The plugin is inspired by [Amp's handoff command](https://ampcode.com/manual#handoff).

---

## Architecture

The plugin is structured as a TypeScript module with four source files:

```
src/
  plugin.ts   # Plugin entry point — registers command, tools, and event handlers
  tools.ts    # Tool factory functions: HandoffSession, ReadSession
  files.ts    # @file reference parsing and synthetic part injection
  vendor.ts   # Code extracted from OpenCode for binary detection and file formatting
```

### plugin.ts

The main export is `HandoffPlugin`, an async function conforming to the `Plugin` type from `@opencode-ai/plugin`. It returns an object with four lifecycle hooks:

| Hook | Purpose |
|------|---------|
| `config` | Registers the `/handoff` slash command |
| `tool` | Registers the `handoff_session` and `read_session` tools |
| `chat.message` | Detects handoff prompts in new sessions and auto-injects referenced files |
| `event` | Cleans up internal state when a session is deleted |

A `Set<string>` called `processedSessions` tracks sessions that have already had files injected, preventing duplicate injection.

---

## Installation & Configuration

Add to `~/.config/opencode/opencode.json`:

```json
{
  "plugin": ["opencode-handoff"]
}
```

To pin to a specific version:

```json
{
  "plugin": ["opencode-handoff@0.5.0"]
}
```

Unpinned plugins are fetched from npm on each OpenCode startup. Pinned versions are cached and require a manual version bump to update.

---

## Commands

### `/handoff <goal>`

**Description:** Analyzes the current conversation and generates a continuation prompt for a new session.

**Arguments:** Free-form text describing what the next session should focus on. If omitted, the command captures a natural continuation of the current conversation.

**Behavior:**

1. OpenCode substitutes the command template with the user's argument in `$ARGUMENTS`.
2. The AI analyzes the conversation and identifies:
   - Relevant files to load (target 8–15 files; up to 20 for complex work)
   - Key decisions, constraints, user preferences, and technical patterns
3. The AI immediately calls `handoff_session` with the generated prompt and file list.

**Prompt template structure:**

```
GOAL: You are creating a handoff message to continue work in a new session.

<context>
  Explanation of why frontloading context matters
</context>

<instructions>
  1. Identify all relevant files
  2. Draft the context and goal description
</instructions>

<user_input>
  USER: $ARGUMENTS
</user_input>

---

After generating the handoff message, IMMEDIATELY call handoff_session with your prompt and files.
```

---

## Tools

### `handoff_session`

**Description:** Creates a new OpenCode session with the handoff prompt pre-loaded as an editable draft.

**Arguments:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `prompt` | `string` | Yes | The generated handoff prompt |
| `files` | `string[]` | No | Array of file paths to load into the new session |

**Execution steps:**

1. Prepends a session reference line: `Continuing work from session <sessionID>. When you lack specific information you can use read_session to get it.`
2. If `files` is non-empty, prepends `@file` references (e.g., `@src/plugin.ts @src/tools.ts`).
3. Calls `client.tui.executeCommand({ command: "session_new" })` to open a new session.
4. Waits 150 ms for the TUI to mount the new session's prompt input (race condition mitigation).
5. Calls `client.tui.appendPrompt` with the full prompt text.
6. Shows a success toast: _"Handoff Ready — Review and edit the draft, then send"_ (4 s duration).

**Full prompt format:**

```
Continuing work from session sess_<id>. When you lack specific information you can use read_session to get it.

@src/foo.ts @src/bar.ts

<generated handoff content>
```

---

### `read_session`

**Description:** Reads the full conversation transcript from a previous session. Intended for use by the AI in the new session when the handoff summary lacks specific details.

**Arguments:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `sessionID` | `string` | Yes | — | Full session ID (e.g., `sess_01jxyz...`) |
| `limit` | `number` | No | 100 | Max messages to read (capped at 500) |

**Output format:**

```markdown
## User
<user message text>
[Attached: filename]

## Assistant
<assistant text>
[Tool: tool_name] <tool title>

(End of session - N messages)
```

Truncated results append: `(Showing N most recent messages. Use a higher 'limit' to see more.)`

On error, returns: `Could not read session <id>: <error message>`

---

## File Reference Processing

### Parsing (`files.ts: parseFileReferences`)

Scans text for `@file` references using the regex:

```
/(?<![\w`])@(\.?[^\s`,.]*(?:\.[^\s`,.]+)*)/g
```

Returns a `Set<string>` of matched file paths (without the `@` prefix).

### Injection (`files.ts: buildSyntheticFileParts`)

For each file reference:

1. Resolves the path relative to the project directory.
2. Skips non-files, binary files (checked by extension and byte-frequency heuristic), and unreadable files.
3. Creates two synthetic `TextPartInput` objects that mirror OpenCode's `Read` tool output:
   - **Header part:** `Called the Read tool with the following input: {"filePath": "/abs/path/to/file"}`
   - **Content part:** File content wrapped in `<file>...</file>` tags, with 5-digit line numbers, line-length truncation at 2000 chars, and a footer showing total lines or a continuation hint.

### Auto-injection (`plugin.ts: chat.message` hook)

When a new message arrives:

1. Filters out synthetic text parts; extracts plain text only.
2. Checks if the text contains `"Continuing work from session"` (the handoff marker).
3. If the session has not been processed before, parses `@file` references and injects synthetic file parts via a `noReply` prompt (preserving the current model and agent).
4. Records the session ID to prevent re-processing.

---

## Session Lifecycle

```
User types: /handoff <goal>
         │
         ▼
AI analyzes conversation
         │
         ▼
AI calls handoff_session(prompt, files)
         │
         ▼
New session created (session_new TUI command)
         │  (150ms delay)
         ▼
Handoff prompt appended to new session draft
         │
         ▼
User reviews draft in new session
         │
         ▼
User sends draft → chat.message hook fires
         │
         ▼
parseFileReferences scans prompt text
         │
         ▼
buildSyntheticFileParts reads files from disk
         │
         ▼
Files injected as noReply synthetic parts
         │
         ▼
AI in new session has full context and can start work immediately
```

---

## Binary File Detection (`vendor.ts`)

Files are considered binary if:

- Their extension is in a known binary set (`.zip`, `.exe`, `.dll`, `.class`, `.jar`, `.pyc`, etc.)
- OR more than 30% of the first 4096 bytes are non-printable characters (bytes outside 0x09–0x0D and 0x20–0xFF)
- OR the file contains a null byte (`0x00`)

---

## File Content Formatting (`vendor.ts`)

Constants:

| Name | Value | Description |
|------|-------|-------------|
| `DEFAULT_READ_LIMIT` | 2000 | Max lines returned per file |
| `MAX_LINE_LENGTH` | 2000 | Max characters per line before truncation |

Output format:

```
<file>
00001| first line
00002| second line
...
(End of file - N lines)
</file>
```

Or, when truncated:

```
(File has more lines. Use 'offset' parameter to read beyond line N)
```

---

## Event Handling

The `event` hook listens for `session.deleted` events and removes the corresponding session ID from `processedSessions`, freeing memory and allowing reprocessing if the session is somehow recreated.

---

## Dependencies

| Package | Version | Role |
|---------|---------|------|
| `@opencode-ai/plugin` | ^1.2.15 | Plugin API types, `tool()` helper |
| `@opencode-ai/sdk` | ^1.2.15 | `TextPartInput` type, client API |
| `zod` | ^4.1.13 | Schema validation (via `tool.schema`) |

**Dev dependencies:** `@types/bun`, `@types/node`, `typescript ~5.9.3`  
**Runtime:** Bun ≥ 1.0.0

---

## Development Setup

```sh
git clone https://github.com/joshuadavidthomas/opencode-handoff
cd opencode-handoff
bun install
```

Symlink the plugin for local development:

```sh
mkdir -p ~/.config/opencode/plugin
ln -sf "$(pwd)/src/plugin.ts" ~/.config/opencode/plugin/handoff.ts
```

Type-check:

```sh
bun run typecheck
```
