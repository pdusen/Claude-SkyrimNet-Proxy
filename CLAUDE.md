# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

An OpenAI-compatible API proxy that serves `/v1/chat/completions` using the credentials
of a locally installed, logged-in **Claude Code CLI**. It exists so
[SkyrimNet](https://github.com/MinLL/SkyrimNet-GamePlugin) can drive Skyrim NPC dialogue
from a Claude Max subscription instead of a metered API key. Models containing a `/`
are routed to OpenRouter instead.

The entire service is one file: **`proxy.py`** (~919 lines). There is no package and no
test suite.

## Read this first

**[`docs/ARCHITECTURE_AUDIT.md`](docs/ARCHITECTURE_AUDIT.md)** is the authoritative
description of how this code works. Read it before making non-trivial changes. It
contains:

- §2 a component map (line ranges → responsibility)
- §3 the startup MITM auth-capture sequence
- §4 the request path, including the template-splicing logic
- §5 the auth lifecycle and single-flight refresh
- §6 the design decisions worth preserving
- Appendix: an end-to-end request trace

Line references in that document are against `main` @ `2ea1a1e`. If they have drifted,
trust the code and update the document.

## Running it

```bash
pip install -r requirements.txt
python proxy.py            # or start-proxy.bat on Windows
```

Requires Python 3.10+ and `claude` on `PATH`, already authenticated. Startup takes ~5 s
because it spawns one real `claude --print` to capture auth.

- Proxy API: `http://127.0.0.1:8000`
- Interceptor: `127.0.0.1:9999` (internal; must be free at startup and stays open for
  the process lifetime, because runtime re-capture depends on it)

**Do not start the proxy casually.** Every startup spawns the Claude CLI and every test
request spends the user's real subscription quota. Prefer static reasoning; ask before
running it.

## The load-bearing invariants

These are non-obvious and easy to break by accident. See architecture doc §4.2.

1. **`system[0]` of the captured template must be preserved.** It is the Claude Code
   identity/billing block. Dropping it produces
   `"credential only authorized for Claude Code"`. `_build_api_body` keeps it and
   appends the caller's system prompt as `system[1]`.
2. **`<system-reminder>` blocks from the template's first user message must be
   re-attached** ahead of the caller's first user message.
3. **The capture must run from a clean temp dir.** A real `cwd` pulls in `CLAUDE.md`
   and project skills, inflating the per-request payload from ~1 KB to ~16 KB.
4. **`tools`, `thinking`, and `context_management` are stripped at capture time** and
   must stay stripped — ~60 KB and significant latency.
5. **The interceptor only captures a body that has both `system` and `messages`.**
   That guard keeps a preflight from overwriting the template (added in `4135a61`).
6. **A stream may only retry before the first chunk is yielded.** Once bytes are on the
   wire the status cannot be retracted.
7. **The template is deep-copied per request.** Do not mutate `auth.body_template` in
   place; concurrent requests share it.

## Conventions

- Single file, standard library plus `fastapi` / `uvicorn` / `aiohttp` / `pydantic`.
  Do not add dependencies or split the module without being asked.
- Section comments use `# --- Name ---`. Keep new code inside the existing sections.
- Logging is `logger.info(f"[{request_id}] -> ...")` / `<- ...` for request/response
  pairs.
- The dashboard is an f-string of HTML in `dashboard()`. All literal CSS/JS braces are
  doubled (`{{`/`}}`) — preserve that when editing.
- Errors on the streaming path are surfaced as assistant *content* (`[API Error 500]`),
  which an NPC will speak aloud in-game. That is intentional.
- `GLOBAL_OPENROUTER_API_KEY` is loaded from `config.json` at import; `config.json` is
  written by `POST /config/openrouter-key`.

## Editing notes

- `proxy.py` uses **CRLF** line endings. Match them.
- `start-proxy.bat` hardcodes a path to the author's install directory.
- `.gitignore` excludes `test_*.py` and `capture_*.py`, so scratch scripts stay
  untracked by default.

## Scope caution

This project uses the Claude CLI session to make direct API calls, which the README
flags as a ToS gray area. That disclosure is the maintainer's decision and is already
documented — do not add to it, remove it, or relitigate it in code comments.

Make the change that was asked for. This codebase works; do not refactor it, re-rate its
design, or append unrequested findings to these docs.
