# Claude SkyrimNet Proxy — Architecture Reference

**Written:** 2026-08-29
**Subject:** `proxy.py` (919 lines, single module)
**Repo state:** `main` @ `2ea1a1e`

A description of how the proxy is put together and why it works the way it does.

---

## 1. Overview

`proxy.py` is a single-file FastAPI service that exposes an **OpenAI-compatible
`/v1/chat/completions` endpoint** and satisfies it using the credentials belonging to a
locally-installed, logged-in **Claude Code CLI**. It exists so that SkyrimNet — a Skyrim
mod that drives NPC dialogue from an LLM — can talk to a Claude Max subscription instead
of a metered API key. Models containing a `/` are routed to OpenRouter instead.

The central mechanism is a **startup-time man-in-the-middle capture**. The proxy points
the Claude CLI at a local HTTP listener via `ANTHROPIC_BASE_URL`, runs one throwaway
`claude --print` invocation, and records both the OAuth headers and the full request
body that the CLI would have sent to `api.anthropic.com`. Every subsequent game request
is then served by **replaying that captured envelope** with the message array swapped
out — no subprocess, no CLI, just a direct HTTPS call over a persistent `aiohttp`
session.

The result is roughly a 2 s round trip in place of the ~9 s a per-request subprocess
would cost, which is what makes it viable for real-time NPC dialogue.

---

## 2. Component Map

| Lines | Component | Role |
|---|---|---|
| 39–66 | Module config & `config.json` helpers | `DEFAULT_MODEL`, `INTERCEPTOR_PORT`, CLI discovery, disk config |
| 69–118 | `AuthCache` | Holds headers + body template + shared session; single-flight refresh |
| 121–145 | OpenRouter / round-robin state | Key global, rotation counter, model-routing predicate |
| 149–201 | `interceptor_handler` | The MITM listener — captures and forwards |
| 204–255 | `capture_auth` / `start_interceptor` | Drives the CLI through the interceptor |
| 258–289 | `_build_api_body` | Splices caller messages into the captured template |
| 293–354 | `call_api_direct` | Anthropic non-streaming (collects the SSE) |
| 357–489 | `call_api_streaming_with_retry` | Anthropic streaming + 401 retry + SSE translation |
| 492–586 | `call_openrouter_*` | OpenRouter passthrough (direct + streaming) |
| 592–626 | Models & `lifespan` | Pydantic schemas, startup/shutdown wiring |
| 629–732 | `chat_completions` | The main endpoint: normalize → route → respond |
| 734–773 | `/config/openrouter-key`, `/v1/models`, `/health` | Control-plane endpoints |
| 776–916 | `dashboard` | Server-rendered HTML+JS status page |

---

## 3. Startup Sequence

`lifespan` (`proxy.py:607`) runs three steps before uvicorn accepts traffic:

```
1. start_interceptor()          -> aiohttp web.Application on 127.0.0.1:9999, catch-all route
2. auth.session = ClientSession -> one shared outbound connection pool for the process
3. auth.ensure_ready(force=True)-> triggers capture_auth()
```

### 3.1 The capture

`capture_auth` (`proxy.py:204`) creates a **fresh `tempfile.TemporaryDirectory()`** and
spawns:

```
claude --print --output-format text --model <DEFAULT_MODEL>
       --no-session-persistence --system-prompt "Say ok" "ok"
```

with `cwd=tmpdir` and `ANTHROPIC_BASE_URL=http://127.0.0.1:9999`.

The clean `cwd` is load-bearing, not cosmetic: it means the CLI finds no `CLAUDE.md`, no
project skills, and no repo context, so the `<system-reminder>` payload the CLI attaches
shrinks from roughly 16 KB to roughly 1 KB. Since that payload is copied into **every**
subsequent request, this one decision removes ~15 KB per NPC line.

### 3.2 What the interceptor keeps

`interceptor_handler` (`proxy.py:149`) triages incoming CLI traffic:

- **Haiku warmup / `count_tokens`** (`proxy.py:166`): forwarded straight through, never
  captured. These are CLI housekeeping calls with no useful body template.
- **The real `/v1/messages` call** (`proxy.py:177`): captured, but only if the parsed
  JSON has **both** `system` and `messages`. That guard came from commit `4135a61` and
  keeps a preflight from being stored as the template.

On capture it strips three fields before storing:

| Stripped | Why |
|---|---|
| `tools` | ~60 KB of Claude Code tool schemas; useless for NPC dialogue |
| `thinking` | Extended thinking adds latency the game cannot absorb |
| `context_management` | Rejected by some accounts; removed in commit `8c5f289` |

Both the headers and the template are assigned back-to-back (`proxy.py:194-195`) so a
concurrent reader never observes a half-populated cache. Everything is then forwarded
upstream so the CLI's own invocation still completes normally.

The interceptor **is not shut down after capture** — its runner lives until `lifespan`
exits (`proxy.py:623`). That is deliberate: runtime re-capture (§5) depends on port 9999
still listening.

---

## 4. The Request Path

### 4.1 Normalization (`chat_completions`, `proxy.py:630`)

1. **Model selection.** `req.model` is split on commas (`proxy.py:129`). With more than
   one entry, `pick_model_round_robin` (`proxy.py:134`) advances a module-level counter,
   so rotation is *global across all callers* rather than per-conversation.
2. **Provider routing.** `is_openrouter_model` (`proxy.py:142`) is a bare `"/" in model`
   test. Slash means OpenRouter; anything else means Anthropic.
3. **Role split.** `system` messages are pulled out into `system_prompt`;
   `user`/`assistant` go into the conversation list.
4. **Shape fixes.** If the first message is not `user`, a synthetic
   `{"role": "user", "content": "Continue."}` is prepended — Anthropic requires a
   user-first alternation. Consecutive same-role messages are then merged with `\n\n`.

### 4.2 Body construction (`_build_api_body`, `proxy.py:258`)

This is the heart of the proxy and worth reading closely:

```python
body = copy.deepcopy(auth.body_template)

billing = body["system"][0]          # Claude Code identity block — PRESERVED
body["system"] = [billing]
if system_prompt:
    body["system"].append({"type": "text", "text": system_prompt})
```

Two pieces of the captured template are deliberately kept:

- **`system[0]`** — the Claude Code identity/billing block. Anthropic's Max-subscription
  endpoint keys off it; removing it produces the
  `"credential only authorized for Claude Code"` error the README documents.
- **`<system-reminder>` text blocks** from the template's first user message
  (`proxy.py:267-274`). These are re-attached ahead of the caller's first user message.

The NPC persona is appended as `system[1]`, behind the Claude Code identity block. The
template is deep-copied per request, so concurrent requests never share mutable state.

Finally `body["model"]` is overwritten with the requested model and `body["stream"]` is
forced to `True` **on both paths** — the non-streaming endpoint streams internally and
reassembles the text before returning.

**Claude 5 compatibility.** Because the template is captured from whatever model
`DEFAULT_MODEL` names, it can carry fields the Claude 5 family rejects. Step 4 of
`_build_api_body` handles this for models matching `is_claude_5_model()`:

- `temperature` / `top_p` / `top_k` are dropped (removed upstream; each is a 400).
- `output_config.effort` is set from `CLAUDE_5_EFFORT` (default `"low"`). Thinking is
  adaptive-on by default on Claude 5, so effort — rather than disabling thinking — is
  what holds latency down. Disabling thinking outright risks `<thinking>` tags leaking
  into the text an NPC speaks.
- A trailing assistant turn gets a `Continue.` user turn appended, because assistant
  prefills were removed on Claude 5.

The predicate is a regex (`_CLAUDE_5_RE`) matching `claude-<name>-5` plus dated
snapshots. It deliberately does not match `claude-haiku-4-5`, where the `-5` is a minor
version. Claude 4 request bodies are built exactly as before.

### 4.3 Streaming translation (`call_api_streaming_with_retry`, `proxy.py:357`)

The generator bridges Anthropic SSE to OpenAI SSE:

- Emits a leading OpenAI **role chunk** (`delta: {role: "assistant", content: ""}`),
  which several clients require before any content.
- Reads with `resp.content.iter_any()` into a string buffer and splits on `\n`, so a
  network chunk that lands mid-line is reassembled rather than dropped.
- Maps `content_block_delta` → `text_delta` into OpenAI `chat.completion.chunk` frames.
  Other event types — `message_start`, `ping`, `content_block_start/stop`,
  `message_delta` — are skipped.
- Terminates with a `finish_reason: "stop"` chunk followed by `data: [DONE]`.

**Retry design.** The `for attempt in range(2)` loop retries a 401/403 *only before the
first chunk has been yielded* (`proxy.py:386-392`). Once bytes are on the wire an HTTP
status cannot be retracted, so a later failure degrades instead of retrying.

**Error surfacing.** A non-retryable failure is emitted as assistant *content* —
`"[API Error 500]"` (`proxy.py:409`). SkyrimNet will have an NPC speak the error string
in-game, which surfaces problems without needing the console open.

### 4.4 OpenRouter path (`proxy.py:492-586`)

Simpler, because OpenRouter is already OpenAI-shaped. The system prompt becomes a normal
`{"role": "system"}` message and the payload is posted with a `Bearer` key. `max_tokens`
and any undeclared extra parameters from the request are forwarded here
(`proxy.py:501`, `proxy.py:540`). The streaming variant is a **raw passthrough**: it
re-frames on `\n\n` and forwards each SSE event verbatim, including OpenRouter's own
`[DONE]`.

### 4.5 Response assembly

Non-streaming replies are wrapped in an OpenAI `chat.completion` envelope
(`proxy.py:686`). `usage` counts are character-based estimates (`len(text) // 4`,
`proxy.py:682-684`) rather than real token accounting.

---

## 5. Auth Lifecycle & Concurrency

`AuthCache.ensure_ready` (`proxy.py:87`) implements a **single-flight** pattern:

```python
async with self._refresh_lock:
    if self.is_ready and not force: return
    if self._refresh_task and not self._refresh_task.done():
        task = self._refresh_task          # join the in-flight refresh
    else:
        self.invalidate()
        self._refresh_task = asyncio.create_task(capture_auth())
        task = self._refresh_task
# await OUTSIDE the lock
await asyncio.wait_for(task, timeout=timeout)
```

Awaiting outside the lock is the key detail: N simultaneous requests arriving on an
expired token produce **exactly one** `claude --print` subprocess, and the other N−1
queue behind the same task. Without this, a busy scene in-game could spawn a dozen CLI
processes at once.

Expiry handling is **reactive**. There is no TTL and no background refresh timer; the
cache is invalidated when the API answers 401/403 or returns a body containing
`"credential"` (`proxy.py:322-325`, `proxy.py:402-405`). The first request after token
expiry pays the recapture cost, and subsequent requests reuse the refreshed headers.

Because the interceptor is still listening (§3.2), that mid-run refresh works exactly
like the startup capture.

---

## 6. Design Decisions Worth Preserving

These are the non-obvious choices that make the approach work:

1. **Single-flight refresh** (`proxy.py:87`) — including the subtle part, awaiting
   outside the lock so waiters coalesce instead of serializing.
2. **Clean-tempdir capture** (`proxy.py:205`) — cuts ~15 KB from every request.
3. **Retry-before-first-chunk** (`proxy.py:386`) — confines the retry to the only window
   where it is safe.
4. **Buffered line splitting** in the SSE reader (`proxy.py:429-433`) — handles chunk
   boundaries correctly, which is the usual failure mode in hand-rolled SSE parsers.
5. **Preserving `system[0]` and the `<system-reminder>` blocks** (`proxy.py:262`,
   `proxy.py:267-274`) — the insight the whole approach depends on.
6. **Tool / thinking / context_management stripping** (`proxy.py:185-187`) — the right
   latency trade for a real-time game.
7. **Per-request `deepcopy` of the template** (`proxy.py:259`) — keeps concurrent
   requests isolated from each other.

---

## Appendix — End-to-End Request Trace

A streaming Anthropic request, `POST /v1/chat/completions`:

```
SkyrimNet
  |
  |  {"model":"claude-sonnet-4-5-20250929","stream":true,
  |   "messages":[{"role":"system",...},{"role":"user",...}]}
  v
chat_completions (630)
  |-- parse_model_list / round-robin        (129,134)
  |-- is_openrouter_model -> False          (142)
  |-- split system vs conversation          (637)
  |-- prepend "Continue." if not user-first (645)
  |-- merge consecutive same-role turns     (649)
  |-- auth.ensure_ready()                   (683)  <- single-flight; may spawn claude --print
  v
call_api_streaming_with_retry (357)
  |-- _build_api_body (258)
  |     deepcopy(template)
  |     system   = [ CLAUDE_CODE_BLOCK, npc_persona ]
  |     messages[0].content = [ <system-reminder> blocks..., user_text ]
  |     model    = requested ;  stream = True
  |-- POST https://api.anthropic.com/v1/messages?beta=true
  |     headers = captured CLI headers (Content-Length rewritten)
  |     session = auth.session (persistent TCP/TLS)
  |
  |-- 401/403 and attempt == 0 ?  -> invalidate, force refresh, retry once (386)
  |-- non-200                     -> emit "[API Error N]" as content, DONE  (395)
  |
  v  200 OK, text/event-stream
  |-- yield role chunk                                          (415)
  |-- for each content_block_delta/text_delta -> OpenAI chunk    (443)
  |-- yield finish_reason "stop"                                (467)
  |-- yield [DONE]                                              (476)
  v
SkyrimNet renders the NPC line
```
