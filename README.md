# chat-ui (working name)

Chat UI that runs entirely in your browser, for vLLM and other OpenAI-compatible servers. No backend, no install, nothing private stored.

Everything is in `index.html` (marked, DOMPurify and highlight.js are loaded from cdnjs, pinned with SRI hashes; system fonts, no web fonts). No build step.

## Features

- Streaming chat (`/chat/completions`, SSE); Stop button or Esc
- Model list from `GET /models`
- Markdown + code highlighting; remote images in model output are shown as links, never loaded
- Reasoning streams live in a short auto-scrolling box, then collapses to "Thought for Ns" plus its last line. Comes from vLLM's `reasoning`/`reasoning_content` deltas (`--reasoning-parser`) or inline `<think>` tags
- Output tokens split into thinking and answer (`usage.completion_tokens_details.reasoning_tokens`)
- TTFT and output tok/s per response (from the server's `usage` when available, otherwise ≈ chunk count)
- Prefix-cache hit rate and speculative-decoding acceptance per response, from vLLM's `/metrics` counters diffed around the request (only accurate when no other requests run concurrently); `usage.prompt_tokens_details.cached_tokens` is used for the cache rate when the server sends it (`--enable-prompt-tokens-details`)
- Edit a user message / regenerate a response
- Image input for vision models: file picker, paste or drag & drop; click any image to view it full size
- Optional system prompt, temperature, top-p, max tokens — left blank, nothing is sent and the server/chat-template defaults apply
- Nothing private is stored: chats and settings live in memory and a reload resets them. The only thing saved is the light/dark choice (localStorage `theme`); no cookies or IndexedDB
- Thinking-effort slider (`reasoning_effort`). Stops are probed per model via vLLM's `/tokenize` (values the chat template rejects are dropped; the default is detected); other servers get none/low/medium/high
- Settings in a side panel; changes apply immediately. The system prompt is optional: when empty, no system message is sent
- Model + thinking chip in the composer; context ring with a usage tooltip
- Optional tools (off by default; Tools toggle in the composer, per-tool switches and exact descriptions in Settings):
  - `web_search`: DuckDuckGo results via [Jina Reader](https://jina.ai/reader/) (no key, ~20 req/min shared, Jina sees queries and URLs)
  - `read_webpage`: page or PDF as plain text via Jina Reader; Markdown with links on request (~4× larger)
  - `run_js`: JavaScript in a Worker inside a sandboxed iframe: opaque origin (no access to the page, settings or cookies), CSP blocks all network, 5 s timeout
  - `current_time`
  - Built for small contexts: ≤6K characters per result, earlier turns' tool results dropped, tools withheld once 3/4 of the context is used; up to 6 model calls per answer
- Light / dark toggle (follows the system setting until used, then remembered)

## Running vLLM for it

```bash
vllm serve <model> --api-key <key> --allowed-origins '["https://<you>.github.io"]'
```

- `--allowed-origins` is required: recent vLLM allows no cross-origin access by default (older versions defaulted to `*`, which lets **any** website you visit call your server). List exactly the origins you open the page from — `http://localhost:8080` and `http://127.0.0.1:8080` are different origins — and set `--api-key`.
- For tools add `--enable-auto-tool-choice --tool-call-parser <parser>` (Qwen3.x: `qwen3_xml`).
- For reasoning models add the matching `--reasoning-parser` (e.g. `qwen3`, `deepseek_r1`) so thinking arrives separately from the answer.
- Base URL `http://localhost:8000` and `http://localhost:8000/v1` both work (`/v1` is added when the URL has no path).

## Connection gotchas (HTTPS page → local server)

- **Same machine:** `http://localhost:8000` works from an HTTPS page (localhost counts as secure).
- **Another machine:** `http://<LAN IP>` is blocked as mixed content. Put HTTPS in front of vLLM (e.g. `tailscale serve 8000`) and use the `https://` URL. (The page also passes Chrome's `targetAddressSpace: 'local'` hint, which may let recent Chrome reach http LAN hosts after a permission prompt. This is untested, so don't rely on it.)
- **Chrome Local Network Access:** Chrome asks for permission when a public site connects to localhost/LAN. If you blocked it, re-allow it in site settings.

## Local development

```bash
python3 -m http.server 8080
```

Then open http://localhost:8080 (`--allowed-origins` must include `http://localhost:8080`).

## Notes

- The API key is kept in memory only and sent only to the configured server.
