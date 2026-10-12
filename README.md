<div align="center">

# Kaya

**An always-on voice assistant that runs on your own hardware.**

Say *"Hey Kaya"* and it answers questions, searches the web, sets timers, and controls
your home — through Home Assistant's own MCP server.

</div>

---

> **Status: pre-build.** This repo currently holds the design and the verified technical
> findings. The implementation follows the build order in [`PLAN.md`](./PLAN.md). Nothing
> here is a working assistant yet — see [Status](#status) for exactly what exists.

---

## What it is

A smart speaker for a Raspberry Pi with a mic and a speaker, plus a browser client and
web console served from the same process.

```
  "Hey Kaya"  →  wake  →  STT  →  LLM (+ tools)  →  TTS  →  speaker
                ↑                                            │
                └──────────── Porcupine, on-device ───────────┘
```

- **Wake word** — Porcupine, running entirely on-device. Audio never leaves the machine
  while idle.
- **Voice in** — whisper.cpp on a LAN box, or OpenAI/ElevenLabs STT.
- **Brain** — any OpenAI-compatible endpoint: Ollama, llama.cpp, LM Studio, vLLM, OpenAI,
  or MiniMax. Tested first against **MiniMax M3.1-Flash-Preview**.
- **Voice out** — **Edge TTS by default** (free, no API key, no signup), with OpenAI,
  ElevenLabs, and MiniMax selectable from the dashboard.
- **Home** — Home Assistant's native MCP server over OAuth 2.0 + PKCE. No hand-rolled
  entity control.
- **Skills** — the [find-skills](https://github.com/vercel-labs/skills) ecosystem,
  browsable and installable from voice or the console.

Built with [Bun](https://bun.sh). Targets a **Raspberry Pi 3** with a
[PiSugar Whisplay HAT](https://www.pisugar.com/products/whisplay-hat-for-pi-zero-2w-audio-display)
(dual mics, 1 W speaker, buttons, RGB LED).

---

## Status

| | |
|---|---|
| ✅ | Architecture designed and documented — [`PLAN.md`](./PLAN.md) |
| ✅ | Runtime, wake-word engine, TTS clients, MCP SDK verified working on this platform |
| ✅ | Whisplay audio path verified: capture fan-out, sample rates, device names |
| ⬜ | Implementation — steps 1–11, not started |
| ⬜ | `"Hey Kaya"` custom keyword — needs a one-time training run, see below |

### Known prerequisites

Two things you'll hit on first run, both documented rather than discovered the hard way:

1. **Bun has no 32-bit ARM build.** A Pi 3 must run **64-bit Raspberry Pi OS** or Bun
   won't start. `bun run doctor` checks this first and tells you the fix.
2. **`"Hey Kaya"` isn't a built-in Porcupine keyword.** Train it once (~2 min, free) at
   [Picovoice Console](https://console.picovoice.ai/) and download the `.ppn`. Until then
   the bundled `hey barista` keyword stands in so the pipeline is testable end to end.

---

## Quickstart

```bash
bun install
cp .env.example .env      # fill in MINIMAX_API_KEY
bun run doctor            # checks arch, audio, wake model, TTS, LLM
bun run keyword           # fetch the wake-word model
bun run dev               # start the assistant + web console
```

Then open `http://<pi-ip>:8080` for the console.

Full setup, hardware notes, and troubleshooting land in `README.md` as the build
progresses.

---

## Plan

[`PLAN.md`](./PLAN.md) is the real document — architecture, the verification log, module
layout, tool contracts, build order, and acceptance bar.

A few decisions worth knowing before you read it:

- **A cascade, not the Realtime API.** OpenAI's Realtime is speech-to-speech, so its TTS
  can't be swapped. Choosing Edge TTS means composing STT → LLM → TTS instead, which is
  also what makes a local model possible.
- **Tool calling is a hard requirement on the LLM.** `capabilities.ts` probes the endpoint
  at boot and refuses to start without it — a voice assistant that can't call tools is
  just a chatbot.
- **Home Assistant via MCP, not REST.** HA already understands "make it cozier" through
  its Assist pipeline. Reimplementing that would be strictly worse.
- **STT belongs on a LAN box.** A Pi 3 is fine for wake word, audio, and playback. It's
  too slow to transcribe without starving the wake word of CPU.

---

## License

MIT — see [`LICENSE`](./LICENSE).

Wake word detection uses [Porcupine](https://picovoice.ai/products/voice/wake-word/)
(Apache 2.0) with a free-tier AccessKey. Edge TTS is an unofficial client for a public
Microsoft endpoint; swap in another provider if that stops being acceptable to you.
