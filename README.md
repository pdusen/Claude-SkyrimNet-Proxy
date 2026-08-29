# Claude SkyrimNet Proxy

An OpenAI-compatible API proxy that routes requests through a Claude Max subscription via the Claude CLI. Designed for use with [SkyrimNet](https://github.com/MinLL/SkyrimNet-GamePlugin) to power AI-driven NPC conversations in Skyrim.

## How It Works

```
SkyrimNet (game) --> POST /v1/chat/completions --> Proxy (port 8000) --> Anthropic API
                                                                    \-> OpenRouter API
```

1. **Startup**: The proxy spawns a single `claude --print` command to capture authenticated headers and a minimal request template from the Claude CLI.
2. **Per-request**: Uses the cached auth to make direct API calls to `api.anthropic.com` via a persistent connection. No subprocess is spawned per request.
3. **Response**: Translates Anthropic's streaming SSE format into OpenAI-compatible SSE chunks (`chat.completion.chunk`) that SkyrimNet expects.

### Optimizations

- **Direct API calls** with cached auth (~2s vs ~9s with subprocess per request)
- **Persistent TCP/TLS session** reused across all requests
- **Tool definitions stripped** from template (saves ~60KB per request)
- **Extended thinking disabled** to minimize latency (Claude 5 models use `effort: low` instead — see below)
- **Clean temp directory capture** minimizes template bloat (~1KB vs ~16KB)

## Requirements

- **Python 3.10+**
- **Claude CLI** installed and authenticated with a [Claude Max subscription](https://claude.ai/pricing) (`claude` must be on your PATH)
- **Python packages**:

```
pip install fastapi uvicorn aiohttp pydantic
```

## Usage

### Start the proxy

```bash
python proxy.py
```

Or use the included batch file on Windows:

```bash
start-proxy.bat
```

The proxy will:
1. Start an interceptor on port 9999 (internal, used only during startup)
2. Capture auth from the Claude CLI (~5 seconds)
3. Start the API server on `http://127.0.0.1:8000`

### Dashboard

Open `http://127.0.0.1:8000` in a browser to see the status dashboard with a quick test form.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | OpenAI-compatible chat completions (streaming + non-streaming) |
| `/v1/models` | GET | List available models |
| `/config/openrouter-key` | POST | Set OpenRouter API key (JSON body: `{"key": "sk-or-..."}`) |
| `/health` | GET | Health check |
| `/` | GET | Web dashboard |

### Configure SkyrimNet

In your SkyrimNet configuration, set:
- **API Endpoint**: `http://localhost:8000/v1/chat/completions`
- **API Key**: (leave empty or set any value — not required)
- **Model**: `claude-sonnet-4-5-20250929` (recommended), `claude-sonnet-5`, `claude-opus-5`, or any OpenRouter model

### Supported Models

| Model ID | Name | Notes |
|----------|------|-------|
| `claude-fable-5` | Fable 5 | Most capable; highest latency and cost |
| `claude-opus-5` | Opus 5 | Very capable, 1M context |
| `claude-sonnet-5` | Sonnet 5 | Claude 5 balance pick |
| `claude-opus-4-6` | Opus 4.6 | Most capable Claude 4 model |
| `claude-sonnet-4-5-20250929` | Sonnet 4.5 | Best balance (default) |
| `claude-haiku-4-5-20251001` | Haiku 4.5 | Fastest, least capable |
| `provider/model` | OpenRouter | Any model via [OpenRouter](https://openrouter.ai) (requires API key) |

### Claude 5 Models

The Claude 5 family (`claude-fable-5`, `claude-opus-5`, `claude-sonnet-5`) changed the
request format, so the proxy adjusts the captured request template before sending:

- **Sampling parameters are stripped.** `temperature`, `top_p`, and `top_k` were removed
  on Claude 5 and now return a 400. The template captured from the Claude CLI can still
  carry them, so they are dropped for these models.
- **Effort is set to `low`** instead of disabling thinking. Thinking is adaptive-on by
  default on Claude 5; `low` effort keeps NPC latency down without turning it off, which
  can leak `<thinking>` tags into the text an NPC speaks. Change `CLAUDE_5_EFFORT` in
  `proxy.py` to `medium`/`high` for more deliberate dialogue, or `None` to send no
  `output_config`.
- **A trailing assistant turn gets a `Continue.` user turn appended.** Assistant prefills
  were removed on Claude 5. This mirrors the existing fix-up for conversations that do
  not start with a user turn.

Claude 4 models are unaffected — their requests are built exactly as before.

Note that these models are slower and more expensive than Sonnet 4.5. For ambient NPC
chatter, `claude-sonnet-4-5-20250929` or `claude-haiku-4-5-20251001` is still the better
latency trade; Claude 5 is worth it for major characters or long-running conversations.

### OpenRouter Support

You can use any model available on [OpenRouter](https://openrouter.ai) by setting an API key:

1. Get an API key from [openrouter.ai/keys](https://openrouter.ai/keys)
2. Open the dashboard at `http://127.0.0.1:8000` and paste the key in the OpenRouter section
3. Use `provider/model` format in the Model field (e.g. `openai/gpt-4o`, `google/gemini-2.0-flash-001`)

### Model Rotation

You can comma-separate multiple models to rotate between them round-robin:

```
claude-sonnet-4-5-20250929, openai/gpt-4o
```

Each request cycles to the next model in the list. Models containing `/` route through OpenRouter; all others route through Anthropic.

## Important Legal Disclaimer

> **This project operates in a gray area with respect to Anthropic's Terms of Service.**
>
> This proxy uses the Claude CLI's authenticated session to make direct API calls, bypassing the standard Claude Code interface. While it uses a legitimately paid Claude Max subscription, the method of accessing the API may not be explicitly authorized by Anthropic's [Terms of Service](https://www.anthropic.com/legal/consumer-terms) or [Acceptable Use Policy](https://www.anthropic.com/legal/aup).
>
> **Before using this proxy, you should:**
> - Read Anthropic's current Terms of Service in full
> - Understand that this access method could be restricted or blocked at any time
> - Be aware that your account could potentially be affected
> - Accept all responsibility for your use of this software
>
> **This software is provided as-is, with no warranty or guarantee of continued functionality.** The authors are not responsible for any consequences of using this proxy, including but not limited to account suspension or service disruption.

## Troubleshooting

### "Auth not ready" / Startup fails
- Ensure `claude` CLI is on your PATH: `claude --version`
- Ensure you're logged in: `claude` (should start without auth errors)
- Check that port 9999 is not in use

### SkyrimNet NPC not responding
- Check the proxy console for errors
- Verify SkyrimNet is configured with the correct endpoint (`http://localhost:8000/v1/chat/completions`)
- Ensure `stream: true` is being sent (SkyrimNet uses streaming by default)

### "credential only authorized for Claude Code" error
- The cached auth has expired. Restart the proxy to re-capture.

## Changelog

### Claude 5 support
- Added `claude-fable-5`, `claude-opus-5`, and `claude-sonnet-5`
- Strip `temperature` / `top_p` / `top_k` for Claude 5 models (removed upstream; now a 400)
- Set `output_config.effort = "low"` on Claude 5 requests to hold latency down
- Append a `Continue.` user turn when a Claude 5 conversation ends on an assistant turn
  (assistant prefills were removed)
- Model list is now driven by a single `ANTHROPIC_MODELS` catalog in `proxy.py`, shared by
  `/v1/models` and the dashboard

### 2025-02-17
- Added OpenRouter support — use any `provider/model` from [OpenRouter](https://openrouter.ai) (e.g. `openai/gpt-4o`, `google/gemini-2.0-flash-001`)
- Added round-robin model rotation — comma-separate models to cycle between them per request
- Added dashboard UI for configuring OpenRouter API key
- Added `POST /config/openrouter-key` endpoint
- Removed Sonnet 3.7 — not available via Claude Code auth

### Initial Release
- OpenAI-compatible proxy with Claude Max subscription auth
- Direct API calls with persistent session (no subprocess per request)
- Streaming + non-streaming support
- Web dashboard with quick test form
- Models: Opus 4.6, Sonnet 4.5 (default), Haiku 4.5

## License

MIT
