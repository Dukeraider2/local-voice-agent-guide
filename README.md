# Local voice agent: video reproduction guide

A practical reproduction of the local voice-agent setup described in [“How I Built an End-to-End Local Voice Agent, and Made It Fast”](https://www.youtube.com/watch?v=xbedfuqYQYA).

This guide uses the creator's open-source [Pithagoras](https://github.com/thecodacus/Pithagoras) project for the agent UI and orchestration, with its local Voice service for speech recognition and synthesis. It is a reproducible deployment guide, not a fork or replacement of Pithagoras.

## What you will run

```text
Microphone → VAD / turn detection → Whisper STT → local LLM → streamed response text
                                                        ↓
Speaker ← queued audio playback ← Breeze TTS 2 (Q8)
```

The video reports a single-GPU target around 12 GB VRAM, Whisper, Breeze TTS 2, and a Qwen mixture-of-experts model. The speech stack runs locally; the LLM is configured as a local provider such as Ollama or llama.cpp. Model sizing and speed depend on GPU, system RAM, quantization, and context length. The upstream Pithagoras project and current voice service versions can differ from the video build.

## Requirements

- Windows 11, Linux, or macOS with Docker Engine / Docker Desktop and Docker Compose.
- NVIDIA GPU + NVIDIA Container Toolkit for Breeze TTS 2 on GPU. Without supported NVIDIA GPU access, use the documented CPU-compatible Kokoro speech option; Whisper recognition can run without GPU.
- A local LLM server (for example Ollama or llama.cpp) with a model that fits your machine. Keep a smaller context initially; increase it only after measuring memory use.
- Microphone and speakers/headphones.

On Windows, Docker Desktop runs Pithagoras in a Linux container. The project's Compose configuration uses host networking, so follow its current Windows/Docker networking notes if the container cannot reach the local model server.

## Install Pithagoras

Follow the current upstream deployment instructions. Basic Linux/macOS path:

```bash
git clone https://github.com/thecodacus/Pithagoras.git
cd Pithagoras
cp .env.example .env
```

Edit `.env` and set `WORKSPACES_DIR` to a folder for your projects. Set `PORTAL_SECRET` to a newly generated random secret (for example, `openssl rand -hex 32`); do not commit `.env`. Then start the portal:

```bash
docker compose up -d --build
```

Open `http://localhost:4100` immediately and set the portal password before allowing other devices onto the network. In **Settings → Providers**, connect the local LLM endpoint. Do not expose this single-user portal directly to the public internet; use localhost or a private VPN.

For Windows, use PowerShell for the clone and `Copy-Item .env.example .env`; edit `.env` with a text editor and use an absolute Windows path for `WORKSPACES_DIR`. Docker Desktop must be configured for Linux containers. Check the upstream deploy guide for current host networking and GPU passthrough steps.

## Install the local voice service

1. In Pithagoras, open **Settings → Extensions** and install/enable **Docker** and **Voice** if they are not already enabled.
2. Open the **Voice** extension and install the voice service with **Whisper** (start with `base` for lower resource use or `small` for better accuracy) and **Breeze TTS 2**.
3. Let the setup finish. The service builds the pinned audio runtime and downloads models into its persistent Docker volume. The first setup can take a while.
4. In the service's **Your voice** settings, choose an input language and voice. Start with **Fast speech generation** and **Lazy load** enabled.
5. Open a chat, start voice mode, grant microphone access to localhost, and speak. Verify one short turn before tuning performance.

For a host without NVIDIA GPU access, choose Kokoro and CPU recognition where offered. Breeze requires supported NVIDIA GPU access in the current managed service. Engine choices and hardware estimates are shown by the installer; check them before rebuilding an existing service.

## Tune for responsiveness

Apply one change at a time and record the delay from end-of-speech to first audible reply.

1. **Stream in phrases.** Keep response streaming and phrase-sized TTS enabled so speech can begin before the model finishes its full answer.
2. **Keep audio queued.** Use the service's streaming Breeze path. Playback should consume each completed audio chunk while the next chunk is synthesized.
3. **Avoid GPU contention.** Breeze can dominate VRAM. Start with Fast mode and Lazy load; reduce LLM context or model size if audio generation fails or stalls.
4. **Choose context deliberately.** Begin at a modest context setting, then increase it while watching VRAM headroom. The video describes reducing context to fit both speech and LLM workloads.
5. **Keep spoken turns short.** Configure the agent to answer briefly in voice mode and announce tool work before doing it. This keeps tool latency from feeling like an audio failure.
6. **Use turn detection.** Tune VAD/end-of-turn silence conservatively so a short pause does not cut off the speaker. The video describes roughly one second of trailing silence.
7. **Measure the real pipeline.** Use Pithagoras' voice latency profiling and pipeline switches to locate time spent in endpointing, STT, LLM first token, TTS first chunk, and playback. Compare warm and cold starts separately.

The video reports improving subsequent-turn latency from about eight seconds to roughly two–three seconds on its own hardware and software revision. Treat that as a reference result, not a guarantee.

## Verify each stage

- **LLM:** send a text chat and confirm it responds from the configured local provider.
- **STT:** use voice dictation and confirm the recognized text appears before testing speech replies.
- **TTS:** generate a short spoken reply and confirm audio plays.
- **Full loop:** speak a short question, wait for end-of-turn, and confirm the transcript and spoken answer match.
- **Locality:** run without internet after models are downloaded and confirm the already-configured models still respond. Initial image/container/model downloads require internet.

## Troubleshooting

- **Voice says service is unavailable:** check Docker is running and inspect `docker logs --tail 100 pithagoras-voice`.
- **GPU not detected:** verify NVIDIA drivers, NVIDIA Container Toolkit, and `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` per the upstream guide.
- **Model allocation failure:** close duplicate voice services, reduce LLM context or model size, use Fast speech generation, and leave VRAM reserve.
- **Long initial pause:** separate cold model load from warm-turn timing; then inspect voice latency profiling before changing model settings.
- **Microphone permission missing:** use `http://localhost:4100` and allow the browser's microphone prompt.

## Sources

- Video: [How I Built an End-to-End Local Voice Agent, and Made It Fast](https://www.youtube.com/watch?v=xbedfuqYQYA)
- Upstream code: [thecodacus/Pithagoras](https://github.com/thecodacus/Pithagoras)
- [Pithagoras deployment guide](https://thecodacus.github.io/pithagoras/guide/deploying)
- [Pithagoras voice service guide](https://thecodacus.github.io/pithagoras/guide/voice-service)
- [Pithagoras voice latency profiling](https://thecodacus.github.io/pithagoras/guide/voice-latency-profiling)
- [Video research notes](VIDEO_NOTES.md)
- [Transcript retrieval status](TRANSCRIPT_STATUS.md)

## License and attribution

This guide is original documentation based on the linked video and public Pithagoras documentation. Pithagoras is a separate upstream project; see its repository for its code and license. Video transcript text is not included.

