# Coaching notes: K2 Think (IFM) or Claude

The demo works with no backend — every question carries a written explanation
that renders instantly. This directory is the optional second layer: a short
note about the specific answer the student picked, written by a model.

Two providers behind one response contract (`{ note: string | null }`), chosen
with the `PROVIDER` variable:

| `PROVIDER` | Calls | Credential |
|---|---|---|
| `ifm` (default) | `POST https://api.ifm.ai/v1/chat/completions` | `IFM_API_KEY` (token shaped `IFM-xf…`) |
| `anthropic` | Anthropic Messages API via the official SDK | `ANTHROPIC_API_KEY` |

## Does this need Cloudflare DNS?

No. A Worker is reachable at `https://<name>.<subdomain>.workers.dev` the moment
you deploy it, and that URL is all `COACH.endpoint` needs. You'd add a Cloudflare
DNS record only to serve the same Worker from your own hostname
(`api.yourdomain.com`) — cosmetic here, and unrelated to how the token is sent.

## Can the page call IFM directly, with no Worker?

Technically yes — I checked, and `api.ifm.ai` does send CORS headers. A preflight
from a GitHub Pages origin comes back `204` with `access-control-allow-origin`
echoing the origin and `Authorization` in `access-control-allow-headers`, and
`GET /v1/models` is public (it lists `IFM/K2-Think-v2` and
`IFM/K2-Horizon-375B-A23B`).

You still shouldn't. `index.html` is a static file served to anyone; a token in
it is a public token, and "hidden" in a variable or fetched at runtime makes no
difference. Whoever finds it spends your quota. The Worker exists so the token
stays server-side, and so rate limiting, logging, and spend caps live in one
place. If you want browser-direct anyway, do it only with a token you can rotate
freely and treat as burnable — never a production key.

## Deploy

```bash
cd api
npm init -y && npm i -D wrangler
npm i @anthropic-ai/sdk          # only needed for PROVIDER=anthropic
npx wrangler secret put IFM_API_KEY
npx wrangler deploy
```

`wrangler.toml`:

```toml
name = "paladin-feedback"
main = "feedback.ts"
compatibility_date = "2026-09-01"

[vars]
PROVIDER = "ifm"
IFM_MODEL = "IFM/K2-Think-v2"
ALLOWED_ORIGINS = "https://tfeng12753.github.io"
```

Then point the page at it — in `index.html`:

```js
const COACH = { endpoint: 'https://paladin-feedback.<your-subdomain>.workers.dev', timeoutMs: 9000 };
```

Leave `endpoint: null` and the page behaves exactly as it does today.

Check a token without spending anything:

```bash
curl -s https://api.ifm.ai/v1/models -H "Authorization: Bearer $IFM_API_KEY"
```

## Self-hosted K2

`IFM_BASE_URL` overrides the gateway, so a local vLLM or SGLang server works the
same way: set it to `http://your-host:8000/v1` and keep `PROVIDER=ifm`.

## Design notes

- **The note is additive.** The written explanation renders immediately; the note
  arrives underneath a moment later. If the request fails, times out after 9s,
  is blocked, or returns nothing, the page drops the slot and the explanation
  stands alone. Feedback never waits on the network.
- **IFM path:** plain `fetch` against the OpenAI-compatible endpoint — no SDK.
  Reasoning depth goes through `chat_template_kwargs.reasoning_effort: "low"`,
  since these notes are short and the route is latency-sensitive. K2 Think
  returns its chain of thought separately in `reasoning_content`; the worker
  reads only `content`, and strips stray `<think>` tags in case a self-hosted
  build inlines them.
- **Anthropic path:** `claude-opus-5` at `effort: "low"`, with the system prompt
  marked `cache_control` so it caches across requests, refusal fallbacks on
  (`fallbacks: "default"`), and `stop_reason` checked before reading content.
- **`max_tokens` is 1200 on both.** Reasoning and the visible reply share that
  budget, so a smaller cap can truncate the note mid-sentence.
- **Untrusted input:** question content is wrapped in a `<QUESTION>` block and the
  system prompt says to treat everything inside it as data. The note is rendered
  through the page's normal escaping, so it can't inject markup; `\( … \)` math
  in a note renders through KaTeX like any other question text.
- **Cost:** one short call per answered question, cached per question and choice,
  so re-reading a question you already answered costs nothing.

## Other hosts

The handler is a standard `fetch(request)` export. On Vercel or Netlify, rename
the default export to that framework's handler signature and read credentials
from `process.env`; nothing else changes.
