# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork Status

This is a fork of the unmaintained upstream repo `pasky/claude.vim`. Active development is ongoing. The plan for current work is at `/home/dynohub/.claude/plans/binary-mixing-fairy.md` — read it before making changes. **Nothing in the plan has been implemented yet as of the last session.**

The owner's design philosophy: Claude should only access what the user explicitly provides; all code changes must go through vimdiff review; Claude should never write directly to the filesystem. When in doubt, defer to the plan.

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
- `elinks`/`felinks` (optional, for the `open_web` tool)

## Architecture

All core logic lives in `plugin/claude.vim` (~1200 lines of VimScript).

**API layer** (`s:ClaudeQueryInternal`): issues `curl` requests with streaming SSE output to the Anthropic API or AWS Bedrock. The streaming parser (lines ~178–226) manually handles SSE events (`content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`) and accumulates tool-use JSON before decoding.

**Implement mode** (`s:ClaudeImplement`): sends the visually selected code plus an instruction, then calls `s:ApplyCodeChangesDiff` to open a vimdiff review.

**Chat mode** (`s:OpenClaudeChat`): manages a persistent `__claude__` buffer. Every request includes all open Vim buffers (`:buffers`) — token usage grows with open buffers.

**Tool execution** (`g:claude_tools`): inline tools — `python`, `shell`, `open`, `new`, `open_web` — run during a response and feed results back into the conversation automatically.

**Bedrock support**: `plugin/claude_bedrock_helper.py` translates between the Anthropic API format and the AWS Bedrock format; invoked as a subprocess when `g:claude_use_bedrock = 1`.

Supporting files:
- `plugin/claude_system_prompt.md` — system prompt for chat mode
- `plugin/claude_implement_prompt.md` — system prompt for implement mode

## Key Patterns

**Vim/Neovim compat**: `has('nvim')` guards throughout; async jobs use `jobstart`/`job_start` and their respective callback APIs.

**Code change formats**: Claude produces edits using vim buffer locators (`/pattern/<CR>V][c`) for simple replacements, or `vimexec` blocks for multi-step command sequences. `s:ApplyCodeChangesDiff` interprets both.

**Token cost tracking**: input costs $3/1M tokens, output $15/1M tokens; displayed after each response.
