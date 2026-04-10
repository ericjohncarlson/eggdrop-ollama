# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file Eggdrop IRC bot script (`ollama.tcl`) that integrates with Ollama AI models. Written in Tcl, it connects to an Ollama instance (default: over WireGuard at `10.66.66.5:11434`) and exposes IRC commands (`!gpt`, `!gpt-status`, `!gpt-models`, `!gpt-model`, `!gpt-system`, `!gpt-clear`).

## Architecture

All logic lives in `ollama.tcl` — there is no build system, no tests, and no dependencies beyond Eggdrop's Tcl environment. The script is loaded via `source scripts/ollama.tcl` in an Eggdrop config.

Key runtime dependencies (Tcl packages): `http`, `json`, `tls`.

### Core flow

1. `bind pub` registers IRC command handlers
2. `gpt_query` is the main handler: rate-limits per user/channel, builds a context-aware prompt from `conversation_history`, constructs JSON manually (string interpolation, not a library), POSTs to Ollama's `/api/generate` endpoint
3. `send_response` splits long replies across multiple IRC messages respecting `max_response_length`

### State (all in-memory, lost on restart)

- `query_tracker` — dict keyed by `chan:nick`, holds list of timestamps for rate limiting
- `conversation_history` — dict keyed by channel, holds last N `{user_msg assistant_msg}` pairs
- `ollama_model` / `ollama_system_prompt` — mutable globals changed via IRC commands

## Development Notes

- No test suite exists. Test manually by loading the script in an Eggdrop bot (`.rehash` or restart).
- JSON payloads are built via string interpolation — inputs are sanitized with `string map` for `\`, `"`, `\n`, `\r`, `\t`.
- The Ollama API is called synchronously with `::http::geturl`; the bot blocks during requests (mitigated by timeout + progress timer).
- Configuration variables are at the top of the file (lines 6–20).
