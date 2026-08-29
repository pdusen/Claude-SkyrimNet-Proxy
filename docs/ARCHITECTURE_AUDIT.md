# Claude SkyrimNet Proxy — Architecture & Code Audit

**Audit date:** 2026-08-29
**Subject:** `proxy.py` (919 lines, single module), `README.md`, `requirements.txt`, `start-proxy.bat`, `.gitignore`
**Repo state:** `main` @ `2ea1a1e` (clean tree)

---

## 1. Executive Summary

`proxy.py` is a single-file FastAPI service that exposes an **OpenAI-compatible
`/v1/chat/completions` endpoint** and satisfies it using the credentials belonging to
a locally-installed, logged-in **Claude Code CLI**. It exists so that SkyrimNet — a
Skyrim mod that drives NPC dialogue from an LLM — can talk to a Claude Max
subscription instead of a metered API key.

The central trick is a **startup-time man-in-the-middle capture**. The proxy points the
Claude CLI at a local HTTP listener via `ANTHROPIC_BASE_URL`, runs one throwaway
`claude --print` invocation, and records both the OAuth headers and the full request
body that the CLI would have sent to `api.anthropic.com`. Every subsequent game
request is then served by **replaying that captured envelope** with the message array
swapped out — no subprocess, no CLI, just a direct HTTPS call over a persistent
`aiohttp` session.

**Overall assessment.** The core design is clever and the hot path is genuinely well
optimized (persistent TCP/TLS, template caching, tool-definition stripping, correct
single-flight refresh). The weak points are concentrated elsewhere: a **broken global
variable that makes three of the five endpoints raise `NameError`**, an OpenRouter key
that is **written to disk but never read back into the routing path**, several
**silently dropped sampling parameters**, and a **wide-open localhost surface** (no
auth, `Access-Control-Allow-Origin: *`) guarding a paid subscription.

| Area | Verdict |
|---|---|
| Core auth-capture design | Sound and effective |
| Hot-path performance | Well optimized, few complaints |
| Anthropic streaming translation | Correct |
| Concurrency / single-flight refresh | Mostly correct; one shared-cancellation bug |
| OpenRouter integration | **Broken** — split-brain state (see 6.1, 6.2) |
| Request-parameter fidelity | **Lossy** — `max_tokens`, `temperature` dropped (6.4) |
| Local security posture | **Weak** — unauthenticated, CORS `*`, token logged (§7) |
| Packaging / portability | Poor — hardcoded paths, no config surface (§8) |

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
| 629–732 | `chat_completions` | The one real endpoint: normalize → route → respond |
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

The clean `cwd` is load-bearing, not cosmetic: it means the CLI finds no `CLAUDE.md`,
no project skills, and no repo context, so the `<system-reminder>` payload the CLI
attaches shrinks from roughly 16 KB to roughly 1 KB. Since that payload is copied into
**every** subsequent request, this one decision removes ~15 KB per NPC line.

### 3.2 What the interceptor keeps

`interceptor_handler` (`proxy.py:149`) triages incoming CLI traffic:

- **Haiku warmup / `count_tokens`** (`proxy.py:166`): forwarded straight through, never
  captured. These are CLI housekeeping calls with no useful body template.
- **The real `/v1/messages` call** (`proxy.py:177`): captured, but only if the parsed
  JSON has **both** `system` and `messages`. That guard came from commit `4135a61`;
  without it a preflight with an empty body could be stored as the template and blow up
  later in `_build_api_body` with a `KeyError`.

On capture it strips three fields before storing:

| Stripped | Why |
|---|---|
| `tools` | ~60 KB of Claude Code tool schemas; useless for NPC dialogue |
| `thinking` | Extended thinking adds latency the game cannot absorb |
| `context_management` | Rejected by some accounts; removed in commit `8c5f289` |

Both the headers and the template are assigned back-to-back (`proxy.py:194-195`) so a
concurrent reader never observes a half-populated cache. Everything is then forwarded
upstream so the CLI's own invocation still completes normally.

Note the interceptor **is not shut down after capture** — its runner lives until
`lifespan` exits (`proxy.py:623`). That is deliberate: runtime re-capture (§5) depends
on port 9999 still listening.

---

## 4. The Request Path

### 4.1 Normalization (`chat_completions`, `proxy.py:630`)

1. **Model selection.** `req.model` is split on commas (`proxy.py:129`). With more than
   one entry, `pick_model_round_robin` (`proxy.py:134`) advances a module-level counter —
   rotation is *global across all callers*, not per-conversation.
2. **Provider routing.** `is_openrouter_model` (`proxy.py:142`) is a bare `"/" in model`
   test. Slash means OpenRouter; anything else means Anthropic.
3. **Role split.** `system` messages are pulled out into `system_prompt`;
   `user`/`assistant` go into the conversation list. Any other role is dropped silently.
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
  endpoint appears to key off it; removing it produces the
  `"credential only authorized for Claude Code"` error the README documents.
- **`<system-reminder>` text blocks** from the template's first user message
  (`proxy.py:267-274`). These are re-attached ahead of the caller's first user message.

The NPC persona is therefore appended as `system[1]`, *behind* the Claude Code identity
block. That is an inherent, structural limitation: the model is told it is Claude Code
first and Lydia second. It largely works, but it explains any tendency to break
character or answer meta-questions as an assistant.

Finally `body["model"]` is overwritten with the requested model and `body["stream"]` is
forced to `True` **on both paths** — the non-streaming endpoint streams internally and
reassembles.

### 4.3 Streaming translation (`call_api_streaming_with_retry`, `proxy.py:357`)

The generator does a genuinely correct SSE bridge:

- Emits a leading OpenAI **role chunk** (`delta: {role: "assistant", content: ""}`),
  which several clients require before any content.
- Reads with `resp.content.iter_any()` into a string buffer and splits on `\n`, so a
  network chunk that lands mid-line is handled correctly rather than dropping the line.
- Maps only `content_block_delta` → `text_delta` into OpenAI
  `chat.completion.chunk` frames. Everything else — `message_start`, `ping`,
  `content_block_start/stop`, `message_delta`, `thinking_delta` — is ignored.
- Terminates with a `finish_reason: "stop"` chunk followed by `data: [DONE]`.

**Retry design.** The `for attempt in range(2)` loop retries a 401/403 *only before the
first chunk has been yielded* (`proxy.py:386-392`). This is the right call: once bytes
are on the wire, an HTTP status cannot be retracted, so a mid-stream failure has to
degrade rather than retry.

**Error surfacing.** A non-retryable failure is emitted as assistant *content* —
`"[API Error 500]"` (`proxy.py:409`). SkyrimNet will therefore have an NPC **speak the
error string aloud in-game**. That is a defensible choice for a mod (it is visible
without opening a console) but it is worth knowing.

### 4.4 OpenRouter path (`proxy.py:492-586`)

Much simpler, because OpenRouter is already OpenAI-shaped. The system prompt becomes a
normal `{"role": "system"}` message and the payload is posted with a `Bearer` key. The
streaming variant is a **raw passthrough**: it re-frames on `\n\n` and forwards each SSE
event verbatim, including OpenRouter's own `[DONE]`. No translation, and no `model`
field rewriting.

---

## 5. Auth Lifecycle & Concurrency

`AuthCache.ensure_ready` (`proxy.py:87`) implements a correct **single-flight** pattern,
which is the most carefully-written part of the file:

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

Expiry handling is **reactive, not proactive**. There is no TTL and no background
refresh timer; the cache is invalidated only when the API answers 401/403 or returns a
body containing `"credential"` (`proxy.py:322-325`, `proxy.py:402-405`). The first
request after token expiry therefore pays the full ~5 s recapture cost, and
non-streaming callers pay it inside their own request (`proxy.py:694-702`).

### 5.1 Bug — a timed-out waiter cancels the refresh for everyone

`asyncio.wait_for` **cancels** the future it is waiting on when the timeout fires. Since
every waiter shares the *same* `_refresh_task`, the first waiter to hit 60 s cancels the
capture that all the other waiters are also depending on, and they all receive
`CancelledError` rather than a result.

This is made likely rather than theoretical by the timeout values: `capture_auth`'s own
subprocess timeout is 60 s (`proxy.py:226`) and `ensure_ready`'s default is also 60 s
(`proxy.py:87`). A genuinely slow CLI start means the outer timer, armed slightly
earlier by whichever request arrived first, wins.

**Fix:** wrap the shared task in `asyncio.shield(...)`, or await
`asyncio.wait({task}, timeout=...)` and inspect the result instead of letting `wait_for`
cancel it. The inner timeout should also be strictly shorter than the outer one.

---

## 6. Correctness Findings

### 6.1 `openrouter_api_key` is never initialized — three endpoints raise `NameError`

**Severity: High.** The module defines `GLOBAL_OPENROUTER_API_KEY` (`proxy.py:123`) but
several call sites read a *different*, never-assigned name:

| Line | Site |
|---|---|
| `proxy.py:760` | `/v1/models` — `if openrouter_api_key:` |
| `proxy.py:771` | `/health` — `"openrouter_configured": openrouter_api_key is not None` |
| `proxy.py:781-782` | `/` dashboard — `or_status` / `or_color` |

`openrouter_api_key` only comes into existence when `POST /config/openrouter-key`
(`proxy.py:735`) executes `global openrouter_api_key` and assigns it. Verified by AST
scan: the only module-level assignments are `CLAUDE_PATH`, `CONFIG_FILE`,
`DEFAULT_MODEL`, `GLOBAL_OPENROUTER_API_KEY`, `INTERCEPTOR_PORT`, `_cfg`,
`_round_robin_counter`, `app`, `auth`, `logger`.

**Consequence:** on a fresh start, `GET /`, `GET /health`, and `GET /v1/models` all
return **500 Internal Server Error** (`NameError: name 'openrouter_api_key' is not
defined`). The dashboard the README tells users to open is unreachable until they POST
a key — which they cannot do, because the form that posts it lives on the dashboard.

**Fix:** delete the second name entirely and use `GLOBAL_OPENROUTER_API_KEY` everywhere,
with `global GLOBAL_OPENROUTER_API_KEY` in the setter.

### 6.2 Saving an OpenRouter key has no effect until restart

**Severity: High.** `chat_completions` reads `GLOBAL_OPENROUTER_API_KEY`
(`proxy.py:666`), which is populated **only once, at import time**, from `config.json`
(`proxy.py:123`). The setter writes `config.json` and assigns the *other* variable.

So after saving a key through the dashboard:

- the dashboard reports **"Configured (saved)"** (because it reads `openrouter_api_key`),
- `/health` reports `openrouter_configured: true`,
- but routing still sees `GLOBAL_OPENROUTER_API_KEY is None` and falls through to the
  request's `Authorization` header. With no header present, `.removeprefix` is called on
  `None`, raising `AttributeError`, which is caught (`proxy.py:668`) and reported as
  **401 "OpenRouter API key not configured"**.

The user is told the key is saved and simultaneously told it is not configured. The
same split affects **clearing**: `cfg.pop(...)` plus `openrouter_api_key = None` leaves
`GLOBAL_OPENROUTER_API_KEY` still holding the old key for the life of the process, so a
"cleared" key keeps being used.

### 6.3 `except AttributeError` is too broad

At `proxy.py:664-669` the `try` wraps the entire key-resolution expression. It is
intended to catch "no `Authorization` header, so `.removeprefix` on `None`", but it will
convert *any* `AttributeError` raised in that expression into a misleading 401. Prefer
an explicit `request.headers.get("authorization") or ""` and a truthiness check.

### 6.4 `max_tokens` and `temperature` are silently discarded

**Severity: Medium.**

- **`max_tokens` on the Anthropic path.** It is threaded through `chat_completions`
  (`proxy.py:659`) into `call_api_direct` (`proxy.py:293`) and
  `call_api_streaming_with_retry` (`proxy.py:357`) — and then never used.
  `_build_api_body` (`proxy.py:258`) does not set it, so the request always uses whatever
  value the *captured CLI template* carried. SkyrimNet's configured response length is
  ignored. (It *is* honored on the OpenRouter path, `proxy.py:501` / `proxy.py:540`.)
- **`temperature` on both paths.** It is a declared field on `ChatRequest`
  (`proxy.py:601`), which in Pydantic v2 means it is excluded from `model_extra`.
  `extra_params` (`proxy.py:660`) is built from `model_extra` only, so `temperature`
  reaches neither provider. A user tuning temperature for NPC variety gets no effect
  anywhere.

Note the asymmetry this creates: undeclared extras like `top_p` *do* reach OpenRouter
(because they land in `model_extra`), while the declared `temperature` does not.

### 6.5 Only the last `system` message survives

The loop at `proxy.py:637-641` assigns rather than accumulates, so a request with
multiple `system` messages keeps only the final one. OpenAI clients that split
instructions across several system turns will lose content silently. Joining with
`\n\n` would match the merge behavior already applied to user/assistant turns.

### 6.6 Empty completions become HTTP 500

`if not response: raise HTTPException(500, "Empty response")` (`proxy.py:679`). A model
that legitimately returns an empty string — refusal, immediate stop, zero-length turn —
is reported as a server error. `is None` is the correct test.

### 6.7 Interceptor forwards every method as POST

The route is registered as `"*"` (`proxy.py:244`) but the handler unconditionally calls
`session.post` (`proxy.py:198`). A `GET` or `OPTIONS` reaching the interceptor is
replayed upstream as a POST. In practice the CLI only issues POSTs, so this is latent
rather than active.

### 6.8 Capture never fires if `DEFAULT_MODEL` is set to Haiku

The warmup skip is `if "haiku" in model or "count_tokens" in request.path`
(`proxy.py:166`). Since `capture_auth` spawns the CLI with `--model DEFAULT_MODEL`
(`proxy.py:211-212`), setting `DEFAULT_MODEL = "claude-haiku-4-5-20251001"` makes the
capture request match the skip condition, so nothing is ever captured and startup fails
on the 60 s timeout with a confusing error. The skip should key on a warmup marker, not
on the model name.

### 6.9 Template shape is assumed, not validated

`_build_api_body` indexes `body["system"][0]` (`proxy.py:262`) and `body["messages"][0]`
(`proxy.py:269`). The preflight guard in the interceptor ensures both keys *exist*, but
not that either list is non-empty. An upstream change to the CLI's request shape
surfaces as an `IndexError`/`KeyError` on every request rather than a clear diagnostic.

### 6.10 Minor observations

- **Dead branch.** `call_api_direct`'s non-SSE JSON parse (`proxy.py:330-336`) is
  unreachable — `_build_api_body` always forces `stream: True`.
- **Token usage is fabricated.** `len(text) // 4` (`proxy.py:682-684`) is a rough
  characters-per-token guess, not real accounting. Fine for a mod; do not build billing
  on it.
- **Round-robin is process-global.** Two concurrent NPCs interleave on the counter, so
  neither gets a stable model. Acceptable for the stated use case; surprising if not
  expected.
- **No multimodal support.** `ChatMessage.content` is typed `str` (`proxy.py:594`), so
  OpenAI's content-block array form is rejected with a 422 at validation.
- **`_save_config` is not atomic** (`proxy.py:61`) — a crash mid-write truncates
  `config.json`. `_load_config` recovers (it catches `JSONDecodeError`) but the key is
  lost.
- **Redundant `ensure_ready`.** The streaming generator calls it (`proxy.py:368`) after
  `chat_completions` already did (`proxy.py:683`); harmless, since a ready cache returns
  immediately.

---

## 7. Security Findings

> The project's own README already flags the Anthropic ToS question, and that
> disclosure is not re-litigated here. This section covers only the *implementation*
> risks, which are separate from the licensing question.

### 7.1 The full OAuth token is written to the log at INFO

**Severity: High.**

```python
logger.info(f"New Auth code: {auth.headers.get('Authorization', 'Error - No Auth Found')}")
```
— `proxy.py:197`.

Every capture and every renewal prints the complete `Authorization` header — a live
bearer token for the user's Claude account — to stdout. That console is routinely
screenshotted or pasted into issue reports when a mod misbehaves, and `.gitignore`
already anticipates `*.log` files existing. Log a fingerprint at most (length, or the
last four characters), or drop the line.

### 7.2 The API is unauthenticated and CORS is fully open

**Severity: High.**

```python
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])
```
— `proxy.py:626`.

`/v1/chat/completions` requires no credential of any kind (the README says so
explicitly: *"API Key: leave empty"*). Combined with `allow_origins=["*"]`, **any web
page open in the user's browser** can `fetch("http://127.0.0.1:8000/v1/chat/completions")`
and both *spend the user's Claude Max quota* and *read the response*. The wildcard is
what makes the response readable; without it this would be a blind write.

The proxy binds to `127.0.0.1` (`proxy.py:919`), which is the right default and limits
this to local origins — but "local" includes every browser tab and every other process
on the machine.

**Mitigations, roughly in order of effort:** restrict `allow_origins` to
`["http://127.0.0.1:8000", "http://localhost:8000"]`; require a shared secret in
`Authorization` for `/v1/chat/completions` (SkyrimNet can send one); at minimum protect
`POST /config/openrouter-key`, which is currently a same-machine, no-credential write.

### 7.3 The interceptor is an open relay to `api.anthropic.com`

**Severity: Medium.** `interceptor_handler` (`proxy.py:149`) accepts any path on
`127.0.0.1:9999`, copies the caller's headers verbatim (minus `Host`), and replays the
request to `https://api.anthropic.com{request.path_qs}`, returning the response body.
Any local process can use it to reach Anthropic with arbitrary paths and headers.
Because the caller must supply its own credentials, this is an SSRF-shaped egress
channel rather than direct credential theft — but the listener has no reason to remain
open to arbitrary clients, and it stays up for the whole process lifetime (§3.2).

### 7.4 `config.json` holds a plaintext key and is not gitignored

**Severity: Medium.** `_save_config` (`proxy.py:61`) writes the OpenRouter API key in
cleartext to `config.json`, in the **repo directory**
(`os.path.dirname(os.path.abspath(__file__))`, `proxy.py:49`). `.gitignore` lists
`.env`, `captured_body.json`, `*.log` — but **not `config.json`**. A contributor who
runs the proxy from a clone and then commits will publish their OpenRouter key.

**Fix:** add `config.json` to `.gitignore` (one line, do it now), and prefer a
user-config location such as `%APPDATA%` / `~/.config` over the source tree.

### 7.5 Upstream error bodies are echoed to the caller

`HTTPException(status_code=resp.status, detail=error_text[:200])` (`proxy.py:327`,
`proxy.py:521`) forwards the provider's raw error text. Upstream errors can quote
request metadata; truncation to 200 characters limits but does not eliminate the
exposure.

---

## 8. Packaging & Operability

- **`start-proxy.bat` hardcodes someone else's path.**
  ```bat
  cd /d "E:\Tools\claude-skyrimnet-proxy"
  ```
  This fails for every user who did not clone to that exact drive and directory.
  `cd /d "%~dp0"` is the portable form and is a one-line fix.
- **No configuration surface.** `DEFAULT_MODEL` (`proxy.py:41`), `INTERCEPTOR_PORT`
  (`proxy.py:42`), the bind host and port 8000 (`proxy.py:919`) are all module
  constants. Changing any of them means editing source. Environment variables or
  `argparse` would cost little.
- **Port conflicts fail opaquely.** If 9999 is occupied, `site.start()` raises inside
  `lifespan` and uvicorn reports a startup traceback with no hint that a port is the
  cause — even though the README's troubleshooting section correctly names it as a
  likely culprit.
- **`requirements.txt` is accurate** — `fastapi`, `uvicorn`, `aiohttp`, `pydantic` are
  all imported and all used. No unpinned surprises beyond the `>=` floors.
- **No tests.** `.gitignore` excludes `test_*.py` and `capture_*.py`, which suggests
  testing has been done ad hoc and deliberately kept out of the repo. There is no CI and
  no automated regression coverage for the SSE translation — the single most
  format-sensitive part of the codebase.
- **README drift.** The changelog is dated `2025-02-17` while the git history runs
  through April 2026, and the documented dashboard / `/health` endpoints do not work as
  described because of finding 6.1.

---

## 9. What Is Genuinely Well Built

It is worth being specific about this, because the design decisions below are not
accidental:

1. **Single-flight refresh** (`proxy.py:87`) is implemented correctly, including the
   subtle part — awaiting outside the lock so waiters coalesce instead of serializing.
2. **Clean-tempdir capture** (`proxy.py:205`) cuts ~15 KB from every single request.
   Small change, large recurring payoff.
3. **Retry-before-first-chunk** (`proxy.py:386`) correctly recognizes that a streaming
   response cannot be un-sent, and confines the retry to the only window where it is
   safe.
4. **Buffered line splitting** in the SSE reader (`proxy.py:429-433`) handles chunk
   boundaries properly, which is the standard bug in hand-rolled SSE parsers.
5. **Preserving `system[0]` and the `<system-reminder>` blocks** (`proxy.py:262`,
   `proxy.py:267-274`) is the non-obvious insight that makes the whole approach work at
   all.
6. **Tool / thinking / context_management stripping** (`proxy.py:185-187`) is the right
   latency trade for a real-time game.

---

## 10. Recommendations, Prioritized

| # | Priority | Finding | Fix |
|---|---|---|---|
| 1 | **P0** | 6.1 — `NameError` on `/`, `/health`, `/v1/models` | Use one global name throughout |
| 2 | **P0** | 6.2 — saved OpenRouter key never routes | Assign the same global the router reads |
| 3 | **P0** | 7.1 — bearer token logged at INFO | Delete or fingerprint the log line (`proxy.py:197`) |
| 4 | **P1** | 7.4 — key in un-gitignored `config.json` | Add `config.json` to `.gitignore` |
| 5 | **P1** | 7.2 — no auth + CORS `*` on a paid endpoint | Narrow `allow_origins`; add a shared secret |
| 6 | **P1** | 6.4 — `max_tokens` / `temperature` dropped | Set `body["max_tokens"]`; forward `temperature` |
| 7 | **P1** | 5.1 — one waiter's timeout cancels the shared refresh | `asyncio.shield`; make inner timeout shorter |
| 8 | **P2** | 8 — `start-proxy.bat` hardcoded path | `cd /d "%~dp0"` |
| 9 | **P2** | 6.5 — multiple system messages collapse to the last | Join with `\n\n` |
| 10 | **P2** | 6.6 — empty completion returns 500 | Test `is None` |
| 11 | **P2** | 7.3 — interceptor is an open relay | Bind to a random port; drop it to capture-only paths |
| 12 | **P3** | 6.7, 6.8, 6.9, 6.10 | Method passthrough, Haiku skip, template validation, dead branch |
| 13 | **P3** | 8 — no config surface, no tests | Env vars for host/port/model; unit tests for SSE translation |

---

## Appendix A — End-to-End Request Trace

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

---

## Appendix B — Verification Notes

Claims in this document were checked against the source rather than inferred:

- **6.1** — confirmed by an AST walk of module-level `Assign`/`AnnAssign` targets;
  `openrouter_api_key` is absent from the resulting set.
- **6.4** — confirmed by `grep -n "max_tokens\|temperature"`: `max_tokens` appears in
  every Anthropic call signature but never inside `_build_api_body`
  (`proxy.py:258-289`); `temperature` appears only at its declaration
  (`proxy.py:601`).
- **6.2** — confirmed by tracing `GLOBAL_OPENROUTER_API_KEY` (assigned only at
  `proxy.py:123`, read only at `proxy.py:666`) against `openrouter_api_key` (assigned
  only at `proxy.py:741` / `proxy.py:746`).
- **5.1** — follows from documented `asyncio.wait_for` semantics (it cancels the awaited
  future on timeout) applied to the shared `_refresh_task` at `proxy.py:105`.
- **7.4** — confirmed by reading `.gitignore`; `config.json` is not listed.

Not verified by execution: the proxy was not started during this audit (doing so would
consume the user's subscription and spawn the Claude CLI), so runtime behavior is
derived from static reading plus the AST/grep checks above.
