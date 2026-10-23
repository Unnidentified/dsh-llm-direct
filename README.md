# dsh-llm-direct

Native DSH LLM adapters for OpenAI- and Anthropic-compatible endpoints — **no pi-ai**.

DSH's Models page writes hand-declared endpoints into the `llm-pi-ai` plugin, so every custom
provider is served by pi-ai. This plugin registers its own `LlmAdapter` through
`ctx.llm.registerAdapter()` — the same seam DSH's built-in DeepSeek adapter uses — and speaks
the provider HTTP APIs itself.

| endpoint key         | wire protocol             | used for                    |
|----------------------|---------------------------|-----------------------------|
| `openai-chat`        | OpenAI Chat Completions   | Concentrate AI, RunInfra, … |
| `anthropic-messages` | Anthropic Messages        | Merge Gateway, …            |

## Install

This is a DSH **bundle**. Install it with the plugin manager rather than editing profile
files by hand:

```
plugin_manager  install_bundle  target: github:Unnidentified/dsh-llm-direct
```

That adds the package to the profile, registers the bundle, and mounts the `llm-direct` row.
No `cordis.patch.yml` edit and no `node_modules` copy is needed.

pi-ai is left completely untouched, so both families of routes coexist.

## Configure providers

The bundle's patch mounts the row **without** `config`: endpoints, model ids and cache
behaviour are deployment data, not package data. Supply them from your profile's own
`cordis.patch.yml` with a patch that targets the row by id:

```yaml
- id: llm-direct
  config:
    providers:
      my-openai-route:
        displayName: My OpenAI-compatible route
        apiKeyEnv: MY_API_KEY
        baseURL: https://api.example.com/v1
        endpoint: openai-chat
        models:
          - id: some-model
            name: Some Model
            contextWindow: 200000
            maxTokens: 32768
            inputModalities: [text, image]
```

Two rules the loader enforces:

- A targeting patch replaces `config` **wholesale** - it is never deep-merged, so restate
  every field the row needs.
- Do **not** also `- insert:` a row with id `llm-direct`. Two entries sharing one id make the
  loader reject the whole plugin tree, which surfaces as *every* provider disappearing from
  the model picker, not just these.

**Restart DSH** after installing or updating - plugin code is loaded once at startup.

## Images

Images are resolved through the harness attachment plane (`ctx.attachments.readImageRequest`)
and sent natively:

* `openai-chat` → message content parts `{type:"image_url", image_url:{url:"data:<mime>;base64,…"}}`
* `anthropic-messages` → content blocks `{type:"image", source:{type:"base64", media_type, data}}`

Behaviour:

* The model must advertise image input (`inputModalities: [text, image]`), or images degrade to
  the harness `textOnlyImageText` placeholder.
* A model that accepts images but receives no attachment service fails with
  `UNSUPPORTED_CONTENT` rather than silently dropping them.
* Alt text for vision is unaffected: images are sent to the model, not described by a proxy.
* Oversized sets are trimmed oldest-first to the route budget (`imageBudget.maxImages` /
  `.maxBytes`, default 20 / 40 MB) and each dropped occurrence becomes the harness
  `offloadedImageText` placeholder — so a huge prompt degrades visibly instead of failing.
* `imageTarget` (`width`/`height`/`maxBytes`, default 1568×1568 / 5 MB) selects the request
  version the attachment provider derives.

## Verified behaviour

* **Merge Gateway (Anthropic branch), live** — complete turns, multi-tool calls, and correct
  cache accounting: `step 1 inputTokens 345 / cacheRead 4608 / output 109`, with
  `uncached + cacheRead + output == totalTokens`.
* **Merge Gateway, image input, live** — a 900×300 PNG containing `CACHE42 ZEBRA` was read
  back correctly by the model through this adapter.
* **Concentrate (OpenAI branch)** — request is field-valid (`reasoning_effort`, tools, no
  `prompt_cache_key`/`prompt_cache_retention`) and the image path was confirmed on the wire as
  a `data:image/png;base64,…` part in a tool-result turn. The account currently answers `424`
  ("no providers allowed"), which is an access issue, not a protocol one.

Wire shaping was derived by probing, not from pi-ai:

* OpenAI: `stream_options.include_usage`; `prompt_tokens_details.cached_tokens` →
  `cacheReadTokens`, with `inputTokens = prompt_tokens − cached_tokens`.
* Merge: the authoritative usage arrives in `message_delta` (its `message_start` frame is
  zeroed), so both frames are read.

## Limitations

* **Anthropic `thinking` is not requested**, and reasoning blocks are not replayed on
  Anthropic routes; Anthropic thinking needs signature round-tripping that Merge's DeepSeek
  vendor does not provide. Reasoning effort is carried on the OpenAI branch as
  `reasoning_effort`, and `reasoning-delta` frames are surfaced if a vendor sends them.
* **Files** are not uploaded; a `file` block becomes the harness `fileHandleText` handle.
* **`cacheControl` defaults to `none`** — both vendors cache automatically, which the probes
  confirmed. Set `cacheControl: short|long` to add `cache_control` markers on Anthropic routes.
* **`cacheRetention` defaults to `none`** and applies to OpenAI routes only. `short` sends
  `prompt_cache_retention: "in_memory"`, `long` sends `"24h"`; those are the only two values
  Concentrate accepts (`5m`/`1h`/`30m` are rejected with 400).
  On Concentrate this is the **routing** control as much as a TTL one. One model id is served
  by nine providers and each keeps its **own** prompt cache, so an unpinned session starts
  cold on whichever provider it lands on, and a later session can land somewhere else again.
  Only `deepseek` declares `prompt_cache_retention` support, so sending the field pins the
  request to `deepseek` — one cache namespace that stays warm across sessions. Measured on
  the first call of three consecutive fresh sessions:

  | config | first-call hit | provider |
  |---|---|---|
  | unpinned, no retention | 75.5% / 96.7% / 75.5% | novita (varying) |
  | pinned + `cacheRetention: long` | 93.6% / 93.6% / 93.6% | deepseek (stable) |

  Trade-off: pinning removes automatic fallback to the other eight providers if `deepseek`
  is unhealthy, in exchange for a deterministic, long-lived cache.
* Streaming only; `stop` maps to `stop_sequences` (Anthropic) / `stop` (OpenAI).
