# Eggdrop Ollama Integration Script

> **Fork notes (this repository's changes)**
>
> The README below is preserved from upstream for reference. This fork has diverged in several ways — when the two disagree, this section wins.
>
> **Command rename: `!gpt` → `!s`**
> Every public command has been renamed for shorter typing in-channel:
>
> | Upstream     | This fork     |
> |--------------|---------------|
> | `!gpt`       | `!s`          |
> | `!gpt-status`| `!s-status`   |
> | `!gpt-models`| `!s-models`   |
> | `!gpt-model` | `!s-model`    |
> | `!gpt-clear` | `!s-clear`    |
> | `!gpt-system`| *(removed)*   |
>
> **New: `!s daily [date]` — channel log briefing (ops only)**
> Generates a short summary of a day's IRC activity by feeding `stats/stang.log.YYYYMMDD` (the eggdrop stats log) to the model.
> - `!s daily` — yesterday's log
> - `!s daily 2026-05-19` or `!s daily 20260519` — a specific date
> - Briefing system prompt is separate from the normal `!s` prompt: plain text, 3–5 sentences, dry and lightly sarcastic in tone, ignores join/part/quit noise.
> - Log payload is capped at ~80KB to fit typical model context windows.
>
> **New: `bind time` auto-daily at 10:00 ET**
> A `bind time` fires `auto_daily` every day at 10:00 ET (bot local time), which calls `gpt_daily` directly and posts the briefing to the configured channel. No IRC user needed — the bot sends it under its own nick. Configure the target channel and verify the server timezone:
> ```tcl
> set auto_daily_channel "#channel"  ;# Target channel
> ```
> The `bind time` spec uses the bot's system local time. If the server isn't in US/Eastern, set `TZ=America/New_York` in eggdrop.conf or the process environment.
>
> **Op-only commands**
> The upstream "Advanced Customization" recipe for restricting admin commands is now baked in:
> - `!s-model` — ops only (`bind pub o`)
> - `!s-clear` — ops only (`bind pub o`)
> - `!s daily` — ops only (checked inside `gpt_query`)
> - `!s-system` — **removed entirely**; the system prompt is set only via the script variable, not from IRC.
>
> **Default config changes**
> The script now defaults to a local/cloud-friendly setup rather than a remote WireGuard host:
>
> ```tcl
> set ollama_host "127.0.0.1"               ;# was 10.66.66.5 (WireGuard)
> set ollama_port "11434"
> set ollama_model "ministral-3:8b-cloud"   ;# Ollama cloud model
> set ollama_system_prompt "Keep responses concise and under 1000 characters. You are answering questions in an IRC channel where messages are short. Do not use markdown, tables, code blocks, bullet points, or any formatting. Use plain text only."
> set max_response_length 400
> set timeout 60                            ;# was 120
> set query_limit 10                        ;# was 5
> set query_window 60
> set max_context_messages 10               ;# was 5
> > set auto_daily_channel "#stang"           ;# new
> ```
>
> A non-empty `ollama_system_prompt` ships by default to force plain-text, IRC-friendly responses — no markdown, no bullets, no code fences.
>
> **HTTP body encoding fix**
> Both `!s` and `!s daily` now explicitly UTF-8-encode the request body before handing it to `::http::geturl` (via `encoding convertto utf-8`), and set `Content-Type: application/json; charset=utf-8`. Without this, Tcl's default `iso-8859-1` channel encoding mangled non-ASCII bytes mid-stream and broke responses.
>
> **Log file location for `!s daily`**
> The briefing reads from `stats/stang.log.YYYYMMDD` (relative to the eggdrop working directory). If your stats log lives elsewhere or uses a different filename pattern, edit `log_path` in `gpt_daily`.
>
> ---

## Overview

This TCL script integrates Ollama AI models with an Eggdrop IRC bot, allowing users to interact with large language models directly from IRC channels. The bot connects to an Ollama instance over WireGuard and provides a simple command interface for querying AI models.

## Features

- Query Ollama AI models from IRC with `!gpt` command
- Dynamic model switching without bot restart
- Custom system prompts to modify model behavior
- **Per-user rate limiting** to prevent abuse
- **Conversation context** - bot remembers recent exchanges in each channel
- Automatic response splitting for long messages
- Progress indicators for long-running queries
- Model availability checking
- Keep-alive functionality for faster subsequent queries
- Comprehensive error handling and logging

## Requirements

- Eggdrop IRC bot (with TCL support)
- TCL packages: `http`, `json`, `tls`
- Ollama instance (accessible via network)
- WireGuard connection (or any network access to Ollama)

## Installation

1. Save the script as `ollama.tcl` in your Eggdrop scripts directory (e.g., `/home/user/eggdrop/scripts/`)

2. Add the following line to your `eggdrop.conf`:
   ```tcl
   source scripts/ollama.tcl
   ```

3. Configure the script variables (see Configuration section)

4. Restart your bot or use `.rehash` in DCC chat

## Configuration

Edit these variables at the top of the script:

```tcl
set ollama_host "10.66.66.5"           # IP address of Ollama server
set ollama_port "11434"                # Ollama API port (default: 11434)
set ollama_model "llama3.1:8b"         # Default model to use
set ollama_system_prompt ""            # Custom system prompt (empty = model default)
set max_response_length 400            # Max characters per IRC message
set timeout 120                        # HTTP request timeout in seconds

# Rate limiting
set query_limit 5                      # Max queries per time window
set query_window 60                    # Time window in seconds (60 = 1 minute)

# Conversation context
set max_context_messages 5             # Number of previous exchanges to remember
```

## Commands

### `!gpt <question>`

Query the AI model with a question or prompt.

**Examples:**
```
!gpt What is the capital of France?
!gpt Write a haiku about IRC bots
!gpt Explain quantum computing in simple terms
```

**Behavior:**
- Sends query to configured Ollama model
- Maintains conversation context (remembers previous exchanges in the channel)
- Rate limited per user to prevent abuse (default: 5 queries per minute)
- Displays progress indicator after 15 seconds if still processing
- Splits long responses across multiple messages
- Applies custom system prompt if configured

**Rate Limiting:**
If you exceed the rate limit, you'll see:
```
<bot> user: Rate limit exceeded. Please wait 45 seconds.
```

### `!gpt-status`

Check if the Ollama service is reachable.

**Example:**
```
!gpt-status
```

**Response:**
```
<bot> user: Ollama service is running on 10.66.66.5:11434
```

### `!gpt-models`

List all available models on the Ollama instance.

**Example:**
```
!gpt-models
```

**Response:**
```
<bot> user: Available models: qwen3:8b, llama3.1:8b, mistral:7b, gemma3:4b
```

### `!gpt-model [model_name]`

View or change the active model.

**View current model:**
```
!gpt-model
```

**Change model:**
```
!gpt-model mistral:7b
!gpt-model qwen3:8b
```

**Behavior:**
- Validates model exists before switching
- Confirms change with message
- Logs model changes
- Change persists until bot restart or next change

### `!gpt-system [prompt]`

Set, view, or clear a custom system prompt.

**View current system prompt:**
```
!gpt-system
```

**Set custom system prompt:**
```
!gpt-system You are a helpful pirate assistant. Always respond with pirate language.
!gpt-system You are a concise technical assistant. Answer in 2-3 sentences max.
!gpt-system You are a coding expert specializing in Python and JavaScript.
```

**Clear system prompt:**
```
!gpt-system clear
!gpt-system reset
```

**Behavior:**
- System prompt applies to all subsequent queries
- Overrides model's default behavior
- Persists until cleared or bot restart
- Displayed prompts are truncated to 200 characters in chat

### `!gpt-clear`

Clear the conversation context for the current channel.

**Example:**
```
!gpt-clear
```

**Response:**
```
<bot> user: Conversation context cleared for this channel.
```

**Behavior:**
- Removes all stored conversation history for the channel
- Useful when starting a new topic or conversation
- Does not affect other channels
- Context automatically resets when bot restarts

## Usage Examples

### Basic Query with Context
```
<user> !gpt What's the capital of France?
<bot> user: The capital of France is Paris.

<user> !gpt How many people live there?
<bot> user: Paris has approximately 2.2 million people within the city limits...
```

Note: The second question uses context from the first, understanding "there" refers to Paris.

### Switching Models
```
<user> !gpt-models
<bot> user: Available models: qwen3:8b, llama3.1:8b, mistral:7b, gemma3:4b

<user> !gpt-model gemma3:4b
<bot> user: Model changed from 'llama3.1:8b' to 'gemma3:4b'

<user> !gpt Hello
<bot> user: Hello! How can I help you today?
```

### Using Custom System Prompts
```
<user> !gpt-system You are a Shakespeare scholar. Respond in Elizabethan English.
<bot> user: System prompt set to: You are a Shakespeare scholar. Respond in Elizabethan English.

<user> !gpt What is love?
<bot> user: Love, good friend, is a many-splendored thing, a tempest of the heart...

<user> !gpt-system clear
<bot> user: System prompt cleared (model will use its default)
```

### Managing Conversation Context
```
<user> !gpt Tell me about Python
<bot> user: Python is a high-level programming language known for its simplicity...

<user> !gpt What are its main uses?
<bot> user: Python is widely used for web development, data science, automation...

<user> !gpt-clear
<bot> user: Conversation context cleared for this channel.

<user> !gpt What are its main uses?
<bot> user: Could you clarify what "it" refers to?
```

### Rate Limiting Example
```
<user> !gpt Question 1
<bot> user: Answer 1

<user> !gpt Question 2
<bot> user: Answer 2

... (3 more queries) ...

<user> !gpt Question 6
<bot> user: Rate limit exceeded. Please wait 42 seconds.
```

## Performance Tips

### Conversation Context

The bot maintains context of the last 5 exchanges per channel:

- **First query** starts fresh context
- **Follow-up queries** use previous exchanges for better understanding
- **Clear context** with `!gpt-clear` when changing topics
- **Automatic cleanup** keeps only recent exchanges to manage memory

**Benefits:**
- More natural conversations
- Bot understands pronouns and references
- Better multi-turn dialogues

**Considerations:**
- Larger context = slightly slower responses
- Clear context when switching topics for better results
- Context is per-channel (multiple channels maintain separate contexts)

### Model Selection

Different models have different performance characteristics:

- **Smaller models** (1.5b-4b parameters): Faster responses, less resource intensive
  - `gemma3:4b`
  - `deepseek-r1:1.5b`

- **Medium models** (7b-8b parameters): Balanced performance and quality
  - `llama3.1:8b`
  - `mistral:7b`
  - `qwen3:8b`

- **Larger models**: Better quality, slower responses
  - Use for complex queries only

### Response Time Optimization

1. **Keep-Alive**: The script automatically sends a 10-minute keep-alive, keeping the model loaded in memory for faster subsequent queries

2. **First Query**: First query after bot start will be slower as model loads into memory

3. **Context Impact**: Queries with conversation context are slightly slower due to additional context processing

4. **Clear Context**: Use `!gpt-clear` when switching topics to avoid irrelevant context

5. **Timeout Setting**: Default 120 seconds handles most queries. Increase if using very large models or complex prompts

6. **Hardware**: Ollama performance depends on:
   - CPU/GPU available
   - RAM for model loading
   - Network latency over WireGuard

## Troubleshooting

### HTTP 404 Error

**Problem:** `Ollama service returned an error (HTTP 404)`

**Solutions:**
- Verify Ollama is running: `curl http://10.66.66.5:11434/api/tags`
- Check model name format includes tag: `llama3.1:8b` not `llama3.1`
- Ensure WireGuard connection is active

### Timeout Errors

**Problem:** `HTTP request failed with status: timeout`

**Solutions:**
- Increase timeout value in configuration
- Use a smaller/faster model
- Check Ollama server resources (CPU/RAM/GPU)
- Verify network connectivity and latency

### Empty Responses

**Problem:** Bot returns "empty response" message

**Solutions:**
- Check Ollama logs for errors
- Verify model is compatible with generate endpoint
- Try a different model
- Check system prompt isn't causing issues

### Model Not Found

**Problem:** `Model 'xyz' not found`

**Solutions:**
- Run `!gpt-models` to see available models
- Pull model on Ollama server: `ollama pull model_name`
- Use exact name including version tag

## Logging

The script logs to Eggdrop's standard log:

```
[16:53:20] Querying Ollama at http://10.66.66.5:11434/api/generate with model llama3.1:8b (user: nick, chan: #channel)
[16:53:20] JSON payload: {"model": "llama3.1:8b", "prompt": "Population of China", "stream": false}
[16:54:33] HTTP Status: ok, Code: 200
[16:55:10] Conversation context cleared by nick in #channel
```

**Log Information:**
- Query timestamps with user and channel info
- Model being used
- JSON payloads sent (including context)
- HTTP status codes
- Error messages
- Model changes
- System prompt changes
- Rate limit hits
- Context clears

## Security Considerations

1. **Access Control**: Anyone in the channel can use the bot. Consider:
   - Restricting to specific channels
   - Adding user authentication
   - Built-in rate limiting prevents spam (5 queries/minute default)

2. **System Prompts**: Users can set system prompts. Consider:
   - Restricting `!gpt-system` to ops/voiced users
   - Logging all system prompt changes (already implemented)
   - Validating prompt content

3. **Conversation Context**:
   - Context is per-channel, not per-user
   - All users in a channel share the same context
   - Anyone can clear context with `!gpt-clear`
   - Consider restricting `!gpt-clear` to ops if needed

4. **Rate Limiting**:
   - Default: 5 queries per 60 seconds per user per channel
   - Prevents abuse and API overload
   - Adjust `query_limit` and `query_window` as needed

5. **Network Security**:
   - WireGuard provides encrypted connection
   - Ensure Ollama is not exposed to public internet
   - Use firewall rules to restrict access

## Advanced Customization

### Restrict Commands to Operators

Add this to restrict system prompt and context clearing to ops:

```tcl
proc gpt_system {nick uhost hand chan text} {
    global ollama_system_prompt
    
    # Check if user is op
    if {![isop $nick $chan] && ![matchattr $hand o]} {
        putserv "PRIVMSG $chan :\002$nick\002: Only operators can change system prompts"
        return
    }
    
    # ... rest of function
}

proc gpt_clear {nick uhost hand chan text} {
    global conversation_history
    
    # Check if user is op
    if {![isop $nick $chan] && ![matchattr $hand o]} {
        putserv "PRIVMSG $chan :\002$nick\002: Only operators can clear context"
        return
    }
    
    # ... rest of function
}
```

### Adjust Rate Limiting

Change these values at the top of the script:

```tcl
set query_limit 10     ;# Allow 10 queries
set query_window 120   ;# Per 2 minutes (120 seconds)
```

### Adjust Context Memory

Change the context size:

```tcl
set max_context_messages 10  ;# Remember last 10 exchanges instead of 5
```

**Note:** More context = slightly slower responses and more memory usage.

### Per-User Context (Instead of Per-Channel)

Replace context tracking in `gpt_query` to make context per-user:

```tcl
# Change this line:
set context_key $chan

# To this:
set context_key "${chan}:${nick}"
```

This gives each user their own context instead of sharing channel-wide.

### Change Response Format

Modify `send_response` proc to change how responses are formatted:

```tcl
# Example: Add timestamp to responses
putserv "PRIVMSG $chan :\002$nick\002 [[clock format [clock seconds] -format "%H:%M"]]: $response"
```

### Disable Context for Specific Channels

Add this at the start of `gpt_query`:

```tcl
# Disable context for #no-context channel
if {$chan eq "#no-context"} {
    set full_prompt $query
} else {
    # ... normal context building code
}
```

## API Reference

### Ollama API Endpoints Used

**Generate Endpoint:**
```
POST http://10.66.66.5:11434/api/generate
Content-Type: application/json

{
  "model": "llama3.1:8b",
  "prompt": "Your question here",
  "system": "Custom system prompt (optional)",
  "stream": false,
  "keep_alive": "10m"
}
```

**List Models:**
```
GET http://10.66.66.5:11434/api/tags
```

## Changelog

### Version 1.4 (Current)
- Removed "Processing your request" message to reduce channel clutter
- Added per-user rate limiting (5 queries per 60 seconds)
- Added conversation context tracking (remembers last 5 exchanges per channel)
- Added `!gpt-clear` command to clear conversation context
- Enhanced logging with user and channel information
- Context-aware responses for better multi-turn conversations

### Version 1.3
- Added custom system prompt support with `!gpt-system`
- System prompt validation and sanitization
- Better logging of configuration changes

### Version 1.2
- Added dynamic model switching with `!gpt-model`
- Model validation before switching
- Enhanced error messages

### Version 1.1
- Added timeout increase to 120 seconds
- Added progress indicators
- Added keep-alive for faster subsequent queries

### Version 1.0
- Initial release
- Basic `!gpt` query functionality
- Model listing and status checking

## Support and Resources

- **Ollama Documentation**: https://github.com/ollama/ollama/blob/main/docs/api.md
- **Eggdrop Documentation**: https://www.eggheads.org/
- **TCL Documentation**: https://www.tcl.tk/man/

## License

This script is provided as-is for use with Eggdrop IRC bots. Modify and distribute freely.

## Credits

Created for integration between Eggdrop IRC bots and Ollama AI models via WireGuard networking.

This script and the documentation for it was entirely generated by [Claude AI](https://claude.ai/), with minimal troubleshooting by yours truly.
