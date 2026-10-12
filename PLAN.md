# Kaya — plan v3

Always-on voice assistant. Wake word "Hey Kaya", local-first, swappable voice, Home Assistant
over MCP, Skills, browser console. Runs as a **smart speaker on a Pi Zero 2 W + PiSugar
Whisplay HAT**, and the same daemon serves a **browser client + admin console**.

Runtime: **Bun 1.4.3**.

---

## 0. What changed from v2, and why it's bigger than it looks

Three of your requirements each break an assumption from v2:

**"Use Edge TTS, not the Realtime API for voice."** The Realtime API is *speech-to-speech* —
STT, reasoning, and TTS are one model behind one socket. You can't swap TTS out of it. So
dropping Edge TTS means **dropping Realtime as the brain**, and the whole loop becomes a
cascade:

```
wake → VAD → STT → LLM (tools) → TTS → playback
```

That's not a downgrade, it's what a local-first assistant needs anyway. It buys you:
independent STT/LLM/TTS swappability, streaming TTS that starts speaking on the first
sentence rather than the first token, and full control over cost (Edge TTS is free).

**"I'll likely use a local model."** Now the LLM is a pluggable endpoint, not a vendor SDK.
Ollama, llama.cpp, LM Studio, vLLM, OpenAI, or Anthropic — same interface. The critical
constraint: **tool calling must work**, or none of the tools exist. I verified Ollama's
OpenAI-compat layer returns `tool_calls`. That's the one hard requirement on your local model.

**"Use HA MCP."** Good call — it means we *don't* hand-roll entity control. Home Assistant's
built-in MCP Server integration already exposes the Assist pipeline. But it's OAuth 2.0 with
PKCE, and HA uses a draft client-ID-metadata-document flow, **not** RFC 7591 dynamic
registration. Some MCP clients can't auto-register. I verified the SDK ships a complete
client-side OAuth/PKCE implementation (`OAuthClientProvider` with `saveCodeVerifier`,
`saveTokens`, discovery helpers), so we implement the provider and drive the flow ourselves.

**One thing I could not verify, stated plainly:** the sandbox proxy **502s all outbound
WebSocket upgrades** — I confirmed this against a public echo server, in both Node and Bun,
so it's the environment and not the code. That means I could not execute a live Edge TTS or
ElevenLabs synthesis here. I verified everything up to the socket and the failure is
consistent across three separate TTS clients. **First real-hardware test must confirm Edge
TTS actually speaks.** `kaya doctor` will have a `tts.test` command that does exactly this
and prints a loud failure if it can't.

---

## 1. Verification log

| Claim | Result |
|---|---|
| Bun + `openai@7.32.0`, `@picovoice/porcupine-node`, `node-edge-tts`, `@modelcontextprotocol/sdk@1.32.1` | ✅ all install clean |
| Bun native `WebSocket` sends custom `Authorization` headers | ✅ verified vs local Bun WS server |
| **All outbound WSS from this sandbox** | ❌ **502, even `wss://echo.websocket.org`, in Node *and* Bun** — env limit, not code |
| `node-edge-tts` under Bun: WS handshake | ⚠️ reaches Edge, gets 502 from the proxy — same failure as the echo server |
| Edge TTS voice list API | ✅ reachable over HTTPS, returns full catalogue (verified `en-US-AriaNeural` present) |
| Edge TTS DRM handshake (`Sec-MS-GEC` + `Sec-MS-GEC-Version` token) | ✅ `node-edge-tts` generates a correctly-shaped token; a `ws` client builds a spec-correct URL |
| `msedge-tts@2.0.9` | ⚠️ pins `Sec-MS-GEC-Version=1-143.0.3650.96`; `node-edge-tts@1.2.11` uses `.75`. Version drift is a live breakage risk → **pin one and abstract it** |
| `node-edge-tts` API shape | ✅ `streamAudio()` is an `AsyncGenerator<Buffer \| wordBoundary>` — exactly the streaming shape we want |
| **MCP `streamable-http` transport at a `/api/mcp` path, from Bun** | ✅ full round trip: `HassTurnOn, HassGetState` listed, `HassTurnOn` called, args passed |
| **MCP client-side OAuth + PKCE** | ✅ SDK ships `OAuthClientProvider` (`saveCodeVerifier`/`codeVerifier`/`saveTokens`), `auth()`, `discoverOAuthProtectedResourceMetadata`, `UnauthorizedError` |
| Ollama OpenAI-compat returns `tool_calls` | ✅ documented in `docs/api.md` |
| whisper.cpp server | ✅ `/inference` (multipart), `/load` — for local STT |
| `npx skills find` non-interactive | ✅ ranked results + install counts |
| Porcupine native binding under Bun | ✅ loads addon + `.pv` + `.ppn`; fails only at AccessKey parse |
| `"Hey Kaya"` is a built-in Porcupine keyword | ❌ **no** — one-time training required |
| Porcupine **WASM** under Bun | ✅ works, but **only with an absolute http(s) URL** for the `.pv` (relative paths fail) — fine in a real browser |
| Whisplay card / codec | ✅ `whisplaysound` → `plughw:whisplaysound,0`; **ES8389** (WM8960 discontinued), I2C ctrl + I2S data, 8 Ω 1 W |
| ffmpeg 48k stereo → 24k mono → 16k mono | ✅ exact byte counts at each stage |
| JS 24k→16k resampler over 20 chunks | ✅ 32 000 samples, no drift |

---

## 2. Architecture

```
  ┌── SPEAKER (Pi Zero 2 W + Whisplay) ─────────────────────────────┐
  │                                                                 │
  │  ffmpeg ─┬─ 16k mono ─► Porcupine ──"Hey Kaya"──┐                │
  │          │                                       ▼                │
  │          │              Silero VAD / energy gate → end-of-speech  │
  │          └─ 16k mono ─► STT ────────────────┐    │               │
  │                                            ▼    │               │
  │                                     ┌─────────────┴──────┐        │
  │                                     │  LLMClient (pluggable)      │
  │                                     │  Ollama / llama.cpp /        │
  │                                     │  LM Studio / OpenAI / Claude│
  │                                     └─────────────┬──────┘        │
  │                                        tool_calls │               │
  │                          ┌──────────────┬─────────┴──────┐        │
  │                          ▼              ▼                ▼        │
  │                   ToolRegistry    HA MCP server    other MCP      │
  │                    web-search      (OAuth/PKCE)    servers        │
  │                    timers·skills                                       │
  │                          │                                              │
  │              text tokens stream down ──► sentence boundary             │
  │                          ▼                                              │
  │                   TTSProvider (pluggable)                              │
  │              Edge (default) · OpenAI · ElevenLabs                     │
  │                          │  PCM/MP3 stream                             │
  │                          ▼                                              │
  │              ffmpeg decode ─► whisplaysound speaker                   │
  │              buttons/LED via gpiod (no display work)                 │
  └──────────────────────────────────────────────────────────────────────┘
                    ▲ HTTP + WS (same origin, LAN)
      ┌─────────────┴──────────────┐
      │ WEB: voice client + console │
      │ voice: browser mic → STT → │
      │       Kaya server          │
      │ console: skills · MCP ·     │
      │   voices · keys · timers    │
      └────────────────────────────┘
```

**Why a cascade, not Realtime.** It's the only shape that survives "local model + swappable
TTS." The cost is more moving parts and slightly higher latency than speech-to-speech — you
pay for it in STT time + first-token time. Mitigations: VAD to cut dead air, streaming TTS
to start speaking on sentence one, and a local Whisper so STT is ~300 ms not ~800 ms.

**Why the browser posts audio to the server rather than running the whole cascade in-page.**
It keeps STT, LLM, TTS, and tools *one* implementation. The browser sends PCM, the server
runs the identical pipeline the speaker uses. Fix tool bugs once. The tradeoff: mic audio
crosses the LAN — acceptable since it's your own device on your own network, and it's
documented rather than hidden.

### Hardware targets

| Board | Verdict | Notes |
|---|---|---|
| **Pi 3 / 3B+** | ✅ **works — and it's the better board** | Cortex-A53 @ 1 GHz, **1 GB RAM**. Same wake-word binary as the Zero 2 W. The extra 512 MB is the whole story |
| Pi Zero 2 W | ✅ works | Cortex-A53, 512 MB. Fine for wake + audio + remote LLM; too tight for local STT |
| Pi 4 / 5 | ✅ comfortable | Room for local whisper.cpp |

**On a Pi 3 specifically — what I verified, and the one real risk.**

Porcupine's npm tarball ships `cortex-a53` and `cortex-a53-aarch64` native binaries, and the
Pi 3 is a Cortex-A53. Wake word is a direct hit, in both 32- and 64-bit form. Picovoice's own
benchmarks are quoted on Pi 3 hardware (11× more accurate than Snowboy, 6.5× faster), so it's
a configuration they actually test.

**Bun needs 64-bit.** Bun ships `bun-linux-aarch64.zip` and **no armv7 build**. The Pi 3 is
ARMv8-A so it *can* run a 64-bit OS — Raspberry Pi OS 64-bit is available for Pi 3 — but the
default `uname -m` on a stock Pi 3 image is `armv7l`, and **Bun will not run there**. This is
the single thing that will waste an afternoon if you hit it cold. → `kaya doctor` checks
`process.arch` first and fails with the exact remedy, not a segfault.

**The real risk is local STT, and the answer is: don't run it on the Pi.** PiSugar's driver
claims "all models with a 40-pin header" so the HAT is fine, but the Pi 3's 1 GHz Cortex-A53
makes whisper.cpp `base` roughly **1–2 s per short utterance and `tiny` around 1 s** — fine in
isolation, but it's the longest pole in the cascade and it's fighting the wake word for the
same cores. Two options, and I'd take the second:

- **Local STT** — works, needs a **USB SSD not an SD card** (constant logging will eat the
  card), and you should expect to tune model quantisation.
- **Remote STT on a LAN box** ⭐ — a Pi 4/5 or any x86 machine on the network runs
  `whisper-server`, Kaya points at it over HTTP. The Pi stays a thin, always-reliable
  voice terminal, and a reboot of the STT box can't take your assistant down.

Either way the Pi 3's job is: wake word, audio I/O, VAD, streaming TTS playback, and the
console. All of which is what it's actually good at.

---

## 3. Module layout

```
kaya/
├─ src/
│  ├─ index.ts            # subcommands: speak | serve | doctor | skills | tts | keyword
│  ├─ config.ts           # layered: defaults < file < .env < dashboard (runtime-mutable)
│  ├─ pipeline/
│  │  ├─ conversation.ts  # the orchestrator: wake → VAD → STT → LLM → TTS
│  │  ├─ vad.ts           # Silero-via-onnx OR energy-gated fallback
│  │  └─ turn.ts          # end-of-turn, barge-in, follow-up window
│  ├─ stt/
│  │  ├─ provider.ts      # interface: transcribe(Int16Array, sampleRate) → text
│  │  ├─ whisper-cpp.ts   # local, HTTP /inference
│  │  ├─ openai.ts        # gpt-4o-transcribe
│  │  └─ elevenlabs.ts    # optional
│  ├─ llm/
│  │  ├─ provider.ts      # interface: complete(messages, tools) → AsyncIterable<events>
│  │  ├─ openai-compat.ts # Ollama, llama.cpp, LM Studio, vLLM, OpenAI
│  │  ├─ anthropic.ts     # Claude, via a small adapter
│  │  ├─ tool-loop.ts     # the tool-call ↔ result cycle, provider-agnostic
│  │  └─ capabilities.ts  # probe once: tools? json mode? streaming? → adapt behavior
│  ├─ tts/
│  │  ├─ provider.ts      # interface: speak(text, voice) → AsyncIterable<AudioChunk>
│  │  ├─ edge.ts          # DEFAULT — node-edge-tts, MIT, no key, no signup
│  │  ├─ openai.ts        # optional
│  │  ├─ elevenlabs.ts    # optional, WS streaming w/ eleven_flash_v2_5
│  │  ├─ voices.ts        # per-provider voice catalog for the dashboard
│  │  └─ decode.ts        # MP3/AAC → PCM via one long-lived ffmpeg
│  ├─ wake/
│  │  ├─ porcupine.ts · keyword-setup.ts · pierror.ts
│  ├─ audio/  capture.ts · playback.ts · resample.ts · device.ts
│  ├─ hw/     buttons.ts · led.ts
│  ├─ tools/
│  │  ├─ registry.ts      # ← the spine
│  │  ├─ web-search.ts · timers.ts · skills.ts · research.ts
│  │  └─ spotify.ts       # STUB, feature-flagged
│  ├─ mcp/
│  │  ├─ client.ts        # stdio + streamable-http
│  │  ├─ oauth.ts         # OAuthClientProvider impl: PKCE, discovery, token store
│  │  └─ bridge.ts        # mcp__server__tool → session tool list
│  ├─ ha/     # NOT a REST client — config only. Tools come from the MCP server.
│  ├─ server/ http.ts · ws.ts · routes/{tools,config,skills,mcp,tts,timers}.ts
│  └─ web/    index.html · app.js · style.css   (no build step)
├─ assets/   porcupine_params.pv · hey_barista_linux.ppn (bootstrap)
└─ kaya.config.ts · package.json · .env.example · README.md
```

**`tools/registry.ts` is still the spine.** One array of `{name, description, parameters, handler}`.
Session config, the `/api/tools/:name` route, the MCP merge, and the system prompt all read
from it. HA is not a special case — it arrives as MCP tools and merges like everything else.

---

## 4. TTS — the new requirement

```ts
interface TTSProvider {
  id: string;
  listVoices(): Promise<Voice[]>;
  speak(text: string, voiceId: string, opts?: SpeakOptions): AsyncIterable<AudioChunk>;
}
```

### Edge (default) — `node-edge-tts@1.2.11`, MIT
No key, no signup, no cost. ~300 neural voices. `streamAudio()` yields MP3 chunks plus
**word-boundary timings**, which is a genuine bonus: we can flash the LED or the browser UI
on the spoken word rather than on the whole-sentence boundary.

Two gotchas I'm handling:
- **DRM version drift.** `msedge-tts` pins `Sec-MS-GEC-Version=1-143.0.3650.96`;
  `node-edge-tts` uses `.75`. Microsoft rotates these. If Edge breaks, it breaks in *both*
  packages. → All Edge access goes through one adapter file with the version as a config
  constant, so a rotation is a one-line fix, not a refactor. `doctor` prints the pinned
  version so you can see drift.
- **MP3 output.** Edge returns MP3, not PCM. → One long-lived `ffmpeg` decode process per
  provider, fed chunks, emitting PCM to the playback pipe. Not a subprocess per sentence.

### ElevenLabs (optional)
WS streaming at `wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input`,
`eleven_flash_v2_5` (~75 ms). Sends text as the LLM emits it. Note: the `stream-input` WS
does **not** support `eleven_v3`/`eleven_v4` — the dashboard must enforce that pairing or the
model picker must switch you to HTTP streaming. I'll gate that in the UI.

### OpenAI (optional)
`gpt-4o-mini-tts` or `tts-1-hd`, PCM out. Easiest of the three, no decode step needed.

### MiniMax (optional) — free rider if you're already on a MiniMax key
`speech-2.8-turbo` / `speech-02-turbo` over the **WebSocket** T2A endpoint, 300+ system
voices plus cloned ones, `pcm`/`mp3`/`flac`/`wav` out. Since your test model is MiniMax, this
needs no second key and no second account. Voice list via `POST /v1/get_voice`.
Worth it for **voice cloning** — ten seconds of your own recording gives Kaya a voice that's
actually *yours*, which is a much better answer to "whose voice is this thing" than picking
`en-US-AriaNeural` off a list. Note cloning needs account verification.

**Dashboard:** pick provider, pick voice, hear a 5-second sample before saving. Voice lists
are fetched live from the provider (Edge voices list, `POST /v1/get_voice`, `GET /v1/voices`,
OpenAI's voice enum), so the picker is never stale or hand-maintained. Config change → next
utterance uses it, no restart.

---

## 5. LLM — pluggable, local-first

```ts
interface LLMProvider {
  id: string;
  complete(req: ChatRequest): AsyncIterable<LLMEvent>;   // text_delta | tool_call | done
}
```

`openai-compat.ts` is the workhorse: one `fetch`-based client against
`{baseUrl, apiKey, model}` covers **Ollama, llama.cpp, LM Studio, vLLM, LM Studio, OpenAI**
because they all speak the same dialect. `anthropic.ts` adapts Claude's different tool shape.

**`capabilities.ts` is the part I'd insist on.** Local models vary wildly, so at boot Kaya
probes the endpoint once and adapts:

| Probe | If missing |
|---|---|
| `tool_calls` supported | Hard fail at boot with a clear message — **no tools, no assistant** |
| Streaming deltas | Buffer the whole response, play it at once. Slower, still works |
| Parallel tool calls | Force one tool per turn |
| System role | Merge instructions into the first user message |

Without this, a weak local model produces a subtly broken assistant and you debug it for an
afternoon. With it, you get a warning at boot: *"qwen2.5:3b — no tool calling, Kaya will
only chat."*

**STT**, same treatment: `whisper.cpp` `/inference` (local, ~300 ms on tiny/base for short
utterances), OpenAI `gpt-4o-transcribe`, or ElevenLabs STT.

### MiniMax M3.1-Flash-Preview — the current test target

Good news first: it **does** support `tools` / `tool_calls` over the OpenAI-compatible route,
and it's explicitly built for tool use. So the hard requirement in the table above is met and
no adapter is needed — it's just `openai-compat.ts` with different coordinates:

```ts
{ provider: "openai-compat", baseUrl: "https://api.minimax.io/v1", model: "MiniMax-M3.1-Flash-Preview" }
```

**But there are four gotchas, and the first one will bite immediately in a voice app.**

**1. Thinking cannot be turned off.** M3.1-Flash always thinks; sending
`thinking: {type: "disabled"}` returns a **400**. And `reasoning_effort` defaults to **`max`**.
For a voice assistant, `max` reasoning depth before the first spoken word is exactly wrong —
you'd get a five-second gap after "what's the weather." → **Kaya sets `reasoning_effort: "low"`
by default** for the conversational path, and exposes it in the dashboard (low / medium / high
/ xhigh / max) so you can crank it for hard questions like `research`. This is a per-request
field, not a provider setting, so it's a dashboard slider rather than a config boolean.

**2. Thinking must be preserved across tool rounds, or the loop silently degrades.** M3 does
"interleaved thinking" — it reasons *between* tool calls, and the reasoning state has to round-
trip. With `reasoning_split: true` thinking comes back in a `reasoning_details` field, and the
**entire** assistant message must be appended to history verbatim. Rebuilding the message from
just `content` + `tool_calls` — which is what most OpenAI-compatible code does, because that's
all the standard SDK types carry — **drops the reasoning state and quietly breaks multi-step
tool use.** This is exactly the kind of bug that looks like "the model got dumber after the
first tool call." → `tool-loop.ts` stores the **raw assistant object** and re-appends it
untouched. The `capabilities.ts` probe detects `reasoning_split` support and turns it on.

**3. Model id is case-sensitive.** `MiniMax-M3.1-Flash-Preview` — both Ms capitalised, hyphens
exact. A wrong-case id is rejected. The dashboard preset hardcodes it.

**4. `developer` role is rejected** on the OpenAI-compatible route — only
`system` / `user` / `assistant`. Our prompt builder only ever emits those, but the capability
probe asserts it so a future change doesn't quietly 400.

Also: `max_completion_tokens`, not `max_tokens`. And the deprecated `function_call` field is
unsupported — `tools` only.

**Result: the test path needs no vendor adapter and no API key from a third party.** One
`MINIMAX_API_KEY` drives both the brain *and* — if you want — the voice, since MiniMax also
ships TTS (§4, `minimax.ts`).

---

## 6. Home Assistant — MCP only

Configured as one entry in `mcpServers`:

```ts
mcpServers: {
  homeassistant: { url: "http://homeassistant.local:8123/api/mcp", transport: "streamable-http" }
}
```

**OAuth/PKCE** is the real work. `mcp/oauth.ts` implements the SDK's `OAuthClientProvider`:
discovery via `.well-known/oauth-protected-resource` then `oauth-authorization-server`,
PKCE verifier persistence, token + refresh storage, and a local callback listener so
`kaya doctor` can complete the flow on first run. On 401 the SDK throws `UnauthorizedError`
and we hand back a URL to open once. Tokens persist in `~/.kaya/tokens.json` at 0600.

**If your local model is the bottleneck, the model is irrelevant here** — HA's MCP tools are
discovered once and become a static tool list. The model just has to emit valid
`tool_calls`. HA already understands "make it cozier" via its Assist pipeline, so we don't
translate natural language ourselves. That's the whole point of using their MCP server
instead of writing a REST client.

---

## 7. Other tools

- **`web_search`** — no hosted search anywhere in this stack now, so: OpenAI Responses
  `web_search` → Parallel hosted MCP (free, no key) → Tavily. `mode: fast|deep`.
- **`timer`/`timers`** — create/list/cancel/pause/resume, persisted to disk, **speaks when it
  fires** through whatever TTS is configured. Fires even if the pipeline is mid-conversation.
- **`find_skill`/`install_skill`** — `npx skills find` / `npx skills add`, install gated
  behind a spoken confirmation. `find-skills/SKILL.md` vendored verbatim.
- **`mcp`** — any stdio or streamable-http server. Tools merge as `mcp__<server>__<tool>`.
- **`spotify_*`** — deliberate stub, `features.spotify: false`.

---

## 8. Web console

Same origin as the API, no build step.

- **Voice** — mic → live transcript → audio out, push-to-talk, wake toggle
- **Voices** ⭐ — provider picker, live voice list, sample playback, rate/pitch/volume
- **LLM** — provider presets (Ollama / llama.cpp / LM Studio / OpenAI / Claude), base URL,
  model, API key (write-only), capability probe results shown explicitly
- **STT** — same shape as LLM
- **Skills** — browse, read, install, remove
- **MCP** — add/edit/remove, probe, discover tools, **HA OAuth connect button**
- **Keys** — write-only fields; server never echoes a stored secret
- **Timers**, **System** (audio device, wake sensitivity, log tail), **Doctor** (runs the
  §9 checks, clickable)

---

## 9. Build order

1. **Scaffold + audio I/O** — capture fan-out, playback, device detect, level meter. No keys.
2. **Wake word** — Porcupine, bootstrap keyword, `kaya:keyword`. No keys.
3. **TTS spike** ⭐ — Edge adapter + MP3→PCM decode + `kaya tts test`. **No keys. Proves the
   riskiest assumption first**, before anything is built on top of it.
4. **STT** — whisper.cpp, then OpenAI fallback. No keys if local.
5. **LLM provider layer** — openai-compat, `capabilities.ts` probe, system prompt.
6. **Tool registry + `web_search`** — proves the tool loop end to end.
7. **`timers`**
8. **MCP bridge** — stdio + streamable-http + OAuth/PKCE. **HA lands here.**
9. **Skills**
10. **Web console** — voice, then panels.
11. **Hardware polish** — buttons, LED, gain, systemd.

Steps 1–4 need no API keys at all. That's the point: **the entire local-first path is
buildable and testable before you spend anything**, and step 3 de-risks the one thing I
couldn't verify from here.

**Step 0, before any of it: `kaya doctor`.** It checks, in order and with plain-language
remedies: `process.arch` is `x64`/`arm64` (**not `armv7l`** — Bun has no armv7 build),
the `whisplaysound` card exists, the wake model + keyword load, ffmpeg is present, and the
configured LLM answers. If you run only one command after cloning, run that one. It costs
thirty seconds and it's the difference between a five-minute setup and a five-hour one.

---

## 10. Acceptance bar

- [ ] `kaya doctor` → arch, mic, speaker, wake model, **TTS speaks a real sentence**, LLM reachable
- [ ] **Pi 3 / 64-bit OS: wake word fires from the shipped `cortex-a53` binary**, no build step
- [ ] 24 h soak on the target board — no SD-card corruption, no memory creep, wake stays reliable
- [ ] Wake word → full turn → spoken answer, latency stated
- [ ] "set a timer for 3 minutes" → fires, speaks, survives reboot
- [ ] "turn off the den lamp" via **HA MCP** → lamp off, confirmed audibly
- [ ] "make it cozier" → HA's Assist pipeline resolves it
- [ ] Switch Edge → ElevenLabs / OpenAI / MiniMax in the dashboard → next reply uses the new voice, no restart
- [ ] **M3.1-Flash: `reasoning_effort: "low"` → audible latency acceptable** (with `max` it should be noticeably worse)
- [ ] **M3.1-Flash: two tool calls in a row (e.g. "timer, then web search")** → both succeed, proving `reasoning_details` round-trips
- [ ] Point LLM at Ollama with a tool-calling model → tools work
- [ ] Point at a model *without* tool calling → boot warning, not silent breakage
- [ ] `find_skill("home assistant")` → ranked results
- [ ] Kill network mid-answer → reconnect, no zombie session
- [ ] Audio never leaves the machine while idle

---

## 11. Open questions

1. **Which STT?** Local whisper.cpp on a Pi 3 is ~1–2 s per short utterance and fights the
   wake word for cores. I'd run `whisper-server` on a Pi 4/5 or any x86 box on your LAN and
   point Kaya at it, keeping the Pi as a thin voice terminal. Confirm remote-over-local?
2. ~~Which local model?~~ **Answered: MiniMax M3.1-Flash-Preview.** It does `tool_calls` over
   the OpenAI-compatible route, so no adapter needed — just `baseUrl` + `model`. See §5 for
   the four protocol gotchas, especially **`reasoning_effort` defaults to `max` and thinking
   cannot be disabled**, which for a voice assistant would mean a multi-second gap before the
   first word. Kaya will default to `low` and expose it as a dashboard slider. Confirm that's
   the behaviour you want, or whether you'd rather it default to `medium`.
3. **Which TTS for the non-default slots?** Adapters for all four (Edge default, OpenAI,
   ElevenLabs, MiniMax) get built regardless. Do you want MiniMax TTS wired up too, given the
   key's already there — specifically for **voice cloning** so Kaya can speak in your own
   cloned voice? That's a much better answer to "whose voice is this thing" than picking
   `en-US-AriaNeural` off a list.
4. **Which board, Pi 3 or Zero 2 W?** Both work. Pi 3 is the better assistant (1 GB vs 512 MB)
   and it's what the rest of the plan assumes. Just want to confirm the target before I tune
   defaults.

Defaults if I don't hear otherwise: remote STT on LAN, `reasoning_effort: "low"`, all four TTS
adapters built with Edge active, targeting a Pi 3 on 64-bit Raspberry Pi OS.

**Next step: `kaya doctor` + step 1 and step 3, together** — audio I/O and the TTS spike,
because if Edge TTS doesn't speak on your hardware, that changes the plan and I'd rather find
out in the first hour than the fifth day. None of it needs an API key.
