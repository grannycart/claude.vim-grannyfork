# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork Status

This is a fork of the unmaintained upstream repo `pasky/claude.vim`. **All planned work is complete.** The original plan is at `~/.claude/plans/binary-mixing-fairy.md` for reference, but is fully implemented — do not treat it as pending work.

The owner's design philosophy: Claude should only access what the user explicitly provides; all code changes must go through vimdiff review; Claude should never write directly to the filesystem. When in doubt, ask before acting.

## What was changed from upstream

All changes are in `plugin/claude.vim` unless noted.

**Model IDs** (lines ~14, ~27): Updated defaults from old date-suffixed IDs to `claude-sonnet-4-6` / `us.anthropic.claude-sonnet-4-6-20251001-v1:0`.

**Vimdiff multi-hunk fix** (`s:ApplyChange`, ~line 275): Added `\<Esc>` at the end of the `execute 'normal ...'` command so Vim returns to normal mode between applying multiple code changes. Without this, the second and later changes silently failed when they were multi-line.

**`g:claude_restrict_filesystem`** (default 1): Config var added near top of file. In `s:SendChatMessage`, `g:claude_tools` is filtered to remove `open`/`new`/`shell`/`python` before each API call when set. `open_web` and `web_search` are unaffected. Set to 0 in `.vimrc` to re-enable filesystem tools for a project.

**`g:claude_thinking_budget`** (default 0): Config var added near top of file. When > 0, `s:ClaudeQueryInternal` adds `thinking: {type: enabled, budget_tokens: N}` to the API request and adjusts `max_tokens` accordingly. The `interleaved-thinking-2025-05-14` beta header is intentionally omitted — it causes the API to deliver all thinking as `<thinking>` XML tags in the text stream for `claude-sonnet-4-6` rather than as structured SSE events. Without it, the first thinking block arrives as structured events (`thinking_delta`) and is accumulated into `s:current_thinking_block`; on `content_block_stop` this calls `s:AppendThinkingBlock` which writes a ```` ```thinking ```` / ```` ``` ```` block. Subsequent interleaved thinking blocks arrive as `<thinking>...</thinking>` in the text stream; `s:ConvertInlineThinkingBlocks` (called at end of response in `s:FinalChatResponse` and `s:CancelClaudeResponse`) rewrites those lines to the same ```` ```thinking ```` / ```` ``` ```` form. `signature_delta` events are silently ignored. The fold mechanism (`GetChatFold`) collapses each block as a level-2 fold; it distinguishes openers (```` ```word ````) from closers (bare ```` ``` ````) by line content rather than prev-level, so consecutive thinking blocks fold independently. A syntax rule (`claudeChatThinkingMark`) highlights the opening line distinctively.

**Web search** (`s:ExecuteTool`, `s:ExecuteOpenWebTool`): Switched from Google to `https://lite.duckduckgo.com/lite/` (Google blocked elinks). Added `input()` confirmation prompt at the top of `s:ExecuteOpenWebTool` — shown before every fetch, covering both direct `open_web` calls and `web_search` (which routes through the same function). Fetched content is returned directly as the tool result string (no persistent buffer); truncated to 10,000 characters; triple backticks escaped to `` ` ` ` `` to prevent the chat buffer's fold and code-change parsers from misinterpreting web content.

**`g:claude_user_instructions`** (default ''): Config var added near top of file. In `s:SendChatMessage`, appended to the system prompt as `# User Preferences\n\n<instructions>` when non-empty.

## Project Overview

`claude.vim` is a Vim/Neovim plugin for AI pair programming with Claude. Unlike code-completion tools, it provides a chat/instruction-centric interface where Claude can see and modify code in real-time with human review via vimdiff.

Two main interaction modes:
- **ClaudeImplement**: select code, send an instruction, review changes in a vimdiff split
- **ClaudeChat**: full persistent conversation with access to all open buffers

No build step — the plugin is pure VimScript with an optional Python helper for AWS Bedrock.

## Development

There is no build, test, or lint tooling. Testing is manual: source the plugin in Vim/Neovim and exercise the commands interactively.

Dependencies:
- Vim 7.4.1486+ or Neovim
- `curl` (required for API calls)
- Python 3 + `boto3` (optional, AWS Bedrock only)
- `elinks`/`felinks` (optional, for the `open_web`/`web_search` tools)

## Architecture

All core logic lives in `plugin/claude.vim` (~1250 lines of VimScript).

**API layer** (`s:ClaudeQueryInternal`): issues `curl` requests with streaming SSE output to the Anthropic API or AWS Bedrock. The streaming parser (`s:HandleStreamOutput`, lines ~185–235) handles SSE events: `content_block_start`/`delta`/`stop` for tool-use blocks, thinking blocks, and text; `message_delta` for token usage. Thinking and tool-use JSON are accumulated in script-level variables (`s:current_thinking_block`, `s:current_tool_call`) and flushed on `content_block_stop`.

**Implement mode** (`s:ClaudeImplement`): sends the visually selected code plus an instruction, then calls `s:ApplyCodeChangesDiff` to open a vimdiff review. `s:ApplyChange` applies each hunk via `execute 'normal ...\<Esc>'` — the `<Esc>` is critical to ensure normal mode is restored between hunks.

**Chat mode** (`s:OpenClaudeChat`): manages a persistent `Claude Chat` buffer. Every request includes all open Vim buffers (`:buffers`) — token usage grows with open buffers. `s:SendChatMessage` builds the tool list (filtering filesystem tools if restricted), appends user instructions to the system prompt, then calls `s:ClaudeQueryInternal`.

**Tool execution** (`g:claude_tools`): inline tools — `python`, `shell`, `open`, `new`, `open_web`, `web_search` — run during a response and feed results back into the conversation automatically. `python` and `shell` prompt for confirmation before executing. `open_web` and `web_search` prompt for URL confirmation before fetching.

**Fold system** (`GetChatFold`): lines matching `^You:` or `^System prompt:` open level-1 folds; lines matching `^\s*\`\`\`` toggle level-2 folds (code blocks and thinking blocks). `s:CloseCurrentInteractionCodeBlocks` collapses all level-2 folds at end of response.

**Bedrock support**: `plugin/claude_bedrock_helper.py` translates between the Anthropic API format and the AWS Bedrock format; invoked as a subprocess when `g:claude_use_bedrock = 1`.

Supporting files:
- `plugin/claude_system_prompt.md` — system prompt for chat mode
- `plugin/claude_implement_prompt.md` — system prompt for implement mode

## Key Patterns

**Vim/Neovim compat**: `has('nvim')` guards throughout; async jobs use `jobstart`/`job_start` and their respective callback APIs. `s:HandleStreamOutputNvim` wraps `s:HandleStreamOutput` for Neovim — changes to the parser only need to go in the main function.

**Code change formats**: Claude produces edits using vim buffer locators (`/pattern/<CR>V][c`) for simple replacements, or `vimexec` blocks for multi-step command sequences. `s:ApplyCodeChangesDiff` interprets both.

**Token cost tracking**: input costs $3/1M tokens, output $15/1M tokens; displayed after each response.

**Config vars** (all near top of file, all use `!exists` guards):
- `g:claude_api_key` — required
- `g:claude_model` — default `claude-sonnet-4-6`
- `g:claude_restrict_filesystem` — default 1
- `g:claude_thinking_budget` — default 0
- `g:claude_user_instructions` — default ''
