# Local Network Voice Tutor Architecture (Thin Mobile Client + Home Inference Server)

This guide describes a practical architecture for a conversational language-learning app where:

- a **phone app** handles UI + microphone + audio playback
- a **local machine** (desktop/server on home LAN) runs STT, LLM, and TTS
- a **streaming bridge** keeps latency low enough to feel conversational

## 1) High-level architecture

```text
Phone (Flutter/React Native)
  ├─ Audio capture (16 kHz PCM chunks)
  ├─ WebSocket client
  ├─ Chat UI + correction highlights
  └─ Audio playback (streamed TTS)
            │
            ▼
Local Inference Gateway (WebSocket + REST)
  ├─ Session manager / auth
  ├─ Voice activity detection (VAD)
  ├─ STT service (faster-whisper)
  ├─ LLM service (Ollama or vLLM)
  ├─ Feedback formatter (corrections + explanations)
  └─ TTS service (XTTSv2 / StyleTTS2)
            │
            ▼
Optional data stores (SQLite/Postgres + vector DB)
```

## 2) Why WebSockets over plain request/response

WebSockets support full-duplex streaming so the app and server can exchange partial results continuously:

- phone streams microphone frames while user speaks
- server emits partial transcripts
- server emits incremental assistant text tokens
- server emits audio chunks as TTS becomes available

This removes "turn-taking dead time" and improves perceived responsiveness.

## 3) Suggested real-time pipeline

1. **Audio ingest**  
   Phone sends ~20–40 ms PCM frames over WebSocket.
2. **VAD + chunking**  
   Server detects speech segments and silence boundaries.
3. **Streaming STT**  
   STT returns partial transcript and a final transcript for each segment.
4. **LLM response generation**  
   LLM receives final user utterance + conversation context + language-learning prompt.
5. **Correction layer**  
   Post-process model output into:
   - natural reply
   - correction objects (`original`, `corrected`, `reason`, `severity`)
6. **Streaming TTS**  
   Convert reply text into audio chunks and stream to phone for immediate playback.

## 4) WebSocket event schema (minimal)

Use a typed envelope:

```json
{
  "type": "stt.partial",
  "session_id": "abc123",
  "seq": 42,
  "ts": 1731000000,
  "payload": {}
}
```

Core message types:

- `audio.append` (client -> server): raw/base64 PCM chunk
- `audio.commit` (client -> server): end of utterance (or VAD timeout)
- `stt.partial` (server -> client): incremental transcript
- `stt.final` (server -> client): stable transcript segment
- `llm.delta` (server -> client): incremental text tokens
- `feedback.corrections` (server -> client): structured grammar/pronunciation guidance
- `tts.chunk` (server -> client): binary/audio chunk (opus/aac/pcm)
- `turn.done` (server -> client): turn complete with usage + latency metrics
- `error` (bidirectional): recoverable/non-recoverable failure

## 5) Latency targets for natural conversation

Target p95 values for each stage:

- VAD boundary detect: **< 150 ms**
- STT first partial: **< 300 ms** from speech start
- LLM first token: **< 500 ms** from finalized transcript
- TTS first audio byte: **< 400 ms** from first text chunk
- End-to-end first audible response: **~1.0–1.5 s**

## 6) Hardware sizing quick guide

These are rough starting points for local deployment:

- **Good baseline**: 12–16 GB VRAM GPU, 32 GB RAM  
  Suitable for 7B–14B-class chat + fast Whisper variants.
- **Strong setup**: 24 GB VRAM GPU, 64 GB RAM  
  Better concurrency and higher-quality TTS with lower latency.
- **High-end**: 48 GB+ aggregate VRAM (or Apple high-memory unified architecture)  
  Enables larger models and multi-user sessions with fewer compromises.

For multilingual language tutoring, prioritizing **low latency + stable throughput** usually beats running the absolute largest model.

## 7) Reliability checklist

- keep session state server-side with short reconnect window
- include sequence IDs for reordering and dedupe
- implement heartbeats/ping-pong for dead connection detection
- degrade gracefully:
  - if TTS stalls, fall back to text-only
  - if STT confidence is low, request repetition
- persist conversation and correction artifacts for progress tracking

## 8) Secure remote access (outside home Wi-Fi)

Prefer overlay networking over opening router ports directly:

- Tailscale (private mesh VPN)
- Cloudflare Tunnel (authenticated outbound tunnel)

Also add:

- per-device auth token
- TLS termination
- request rate limits
- audit logging for session events

## 9) Recommended implementation order

1. single-user local LAN prototype with one WebSocket endpoint
2. add streaming STT partials + finals
3. add LLM token streaming
4. add chunked TTS playback
5. add correction metadata and UI highlights
6. add auth + remote access (Tailscale/Cloudflare)

This sequence gets you to a "real-time feeling" quickly, then layers quality and security.
