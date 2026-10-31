# Local voice agent: Podman-first reproduction guide

A practical reproduction guide for the local voice-agent workflow described in [“How I Built an End-to-End Local Voice Agent, and Made It Fast”](https://www.youtube.com/watch?v=xbedfuqYQYA), using the upstream [Pithagoras](https://github.com/thecodacus/pithagoras) project.

> **Compatibility status:** The portal can be launched with Podman Compose, and Podman's API offers a Docker-compatible endpoint. However, Pithagoras' managed Voice extension is explicitly written against the Docker Engine API. This repository documents an **experimental Podman compatibility path**, not a fully upstream-supported or end-to-end validated Podman port. The reliable target for this first pass is CPU speech (Kokoro + Whisper). NVIDIA GPU speech through the managed Voice extension is not claimed to work.

## What you will run

```text
Microphone → VAD / turn detection → Whisper or Qwen3-ASR → local LLM → streamed response text
                                                                  ↓
Speaker ← queued audio playback ← Kokoro / Breeze / Chatterbox
```

The video describes a local Qwen-family LLM, Whisper, Breeze TTS 2, Silero VAD, and phrase-streamed speech. Current Pithagoras has evolved: its managed Voice extension supports Breeze, Chatterbox or Kokoro for speech synthesis and Whisper or Qwen3-ASR for recognition. The current extension builds its speech runtime and downloads models inside a separate container.

## Requirements

- Linux host with Podman and a Compose provider supported by `podman compose`.
- A local LLM server such as Ollama or llama.cpp, with a model sized for the machine.
- Microphone and speakers/headphones.
- For the experimental managed-Voice path: a running Podman API socket with Docker API compatibility. Rootless Podman is preferred where the required host paths and permissions work.
- Internet access for the initial image, source, package and model downloads.

**Important:** Do not install Docker as a workaround. If the Podman compatibility checks fail, stop and record the failing API operation rather than switching runtimes silently.

## Install the Pithagoras portal with Podman

Follow [Pithagoras' deployment guide](https://thecodacus.github.io/pithagoras/guide/deploying) for the current upstream release. This repository includes a Compose override at [compose.podman.yaml](compose.podman.yaml) for the rootless Podman socket.

1. Start the rootless Podman socket for your user:

   ```bash
   systemctl --user enable --now podman.socket
   ```

2. Confirm the socket exists and answers:

   ```bash
   test -S "$XDG_RUNTIME_DIR/podman/podman.sock" && podman --remote info
   ```

3. Create the upstream `.env` file. Set `WORKSPACES_DIR` to a real host directory and generate a unique portal secret:

   ```bash
   mkdir -p "$HOME/pithagoras-workspaces"
   printf "WORKSPACES_DIR='%s'\nPORTAL_SECRET=%s\nPODMAN_SOCKET_PATH=%s\n" "$HOME/pithagoras-workspaces" "$(openssl rand -hex 32)" "$XDG_RUNTIME_DIR/podman/podman.sock" > .env
   chmod 600 .env
   ```

   Run this from the Pithagoras checkout directory. Review the generated `.env`; never commit it.

4. Launch with the Podman override:

   ```bash
   podman compose -f docker-compose.yml -f compose.podman.yaml up -d --build
   ```

5. Check the portal:

   ```bash
   podman ps
   podman logs --tail 100 pithagoras
   ```

Open `http://localhost:4100`, set the portal password, and connect the local LLM provider in **Settings → Providers**. Keep the portal on localhost or a private VPN. Do not expose it publicly.

The override replaces the upstream Docker socket bind with the rootless Podman socket, sets `DOCKER_SOCKET`, and disables SELinux labeling for the socket mount as required by Podman's documentation. The socket grants broad control over the user's Podman containers and files. Treat access to it as privileged.

## Managed Voice extension: experimental Podman compatibility

The upstream Voice extension depends on `pithagoras-docker`. Its source constructs Docker Engine API requests directly, including container create/inspect/start/stop, image pulls, volume operations, and a container network namespace shared with the portal. Podman's API exposes a Docker-compatible API, but that does not guarantee every Docker API field or behavior used by Pithagoras is implemented identically.

For the first compatibility test:

1. In Pithagoras, open **Settings → Extensions → Discover** and install **Docker** and **Voice**.
2. Verify the Docker extension reports the Podman API as available. A socket file existing is not enough; the API ping must succeed.
3. Choose CPU-compatible speech engines first: **Kokoro (CPU)** for synthesis and **Whisper (CPU)** for recognition. Do not select Breeze or Chatterbox GPU mode for this Podman test.
4. Install Voice and inspect its setup log.
5. Verify that the voice container starts, its health endpoints respond, and a short speech round-trip works.

If the extension reports Docker unavailable, fails to create the voice container, cannot share the portal network namespace, or fails to mount its named volume, stop there. Capture the exact API error and Podman version. Do not delete existing volumes or containers as a troubleshooting shortcut.

### GPU limitation

The current Pithagoras Voice source requests NVIDIA GPUs using Docker's `HostConfig.DeviceRequests` field and probes GPU access by creating a temporary container with the same Docker-specific request. Podman supports GPU access through its own mechanisms, including CDI, but the upstream extension does not currently expose a Podman-specific CDI configuration. Therefore GPU acceleration through the managed Voice extension is **not validated** by this guide. A real GPU port requires an upstream adapter or a tested compatibility implementation, plus a complete rebuild and GPU validation.

For the Framework Desktop AI Max 395 / Radeon 8060S environment, do not assume the NVIDIA/CUDA voice path is applicable. A separate ROCm-compatible speech stack or CPU Kokoro route needs its own validation.

## Verify each stage

- **Portal:** Pithagoras is running and the UI loads.
- **Container API:** Docker extension reports available through the Podman socket.
- **LLM:** text chat returns from the configured local provider.
- **STT:** a short spoken phrase produces a transcript.
- **TTS:** a short response produces audible output.
- **Full loop:** ask one short question and verify transcript and spoken response.
- **Persistence:** restart the portal and confirm settings and workspace mounts remain.
- **Locality:** after initial downloads, disconnect internet and confirm already-installed models still work.

Record Podman version, Compose provider/version, Pithagoras commit, selected engines, CPU/RAM/GPU use, setup errors, and measured end-of-speech-to-first-audio latency.

## Troubleshooting

- **Socket missing:** check `systemctl --user status podman.socket` and `$XDG_RUNTIME_DIR/podman/podman.sock`.
- **API unavailable:** verify the socket is mounted at `/var/run/docker.sock` inside the portal and `DOCKER_SOCKET=/var/run/docker.sock` is set.
- **Permission/SELinux error:** confirm the socket mount and `security_opt: label=disable` override. Do not make the socket world-writable.
- **Voice container creation fails:** capture the exact Podman API response. Likely compatibility points include Docker API fields, container namespace sharing, volume operations, and GPU DeviceRequests.
- **No GPU detected:** this is expected for the current unported NVIDIA DeviceRequests path. Use the CPU-compatible engine selection; do not infer that the host GPU itself is unavailable.
- **Microphone permission missing:** use localhost or HTTPS and allow microphone access in the browser.

## Sources

- Video: [How I Built an End-to-End Local Voice Agent, and Made It Fast](https://www.youtube.com/watch?v=xbedfuqYQYA)
- Upstream project: [thecodacus/Pithagoras](https://github.com/thecodacus/pithagoras)
- [Pithagoras deployment guide](https://thecodacus.github.io/pithagoras/guide/deploying)
- [Pithagoras voice service guide](https://thecodacus.github.io/pithagoras/guide/voice-service)
- [Pithagoras Docker extension guide](https://thecodacus.github.io/pithagoras/guide/docker)
- [Podman system service and Docker API compatibility](https://docs.podman.io/en/latest/markdown/podman-system-service.1.html)
- [Podman Compose](https://docs.podman.io/en/latest/markdown/podman-compose.1.html)
- [Video research notes](VIDEO_NOTES.md)
- [Transcript retrieval status](TRANSCRIPT_STATUS.md)

## License and attribution

This repository contains original documentation and a Compose override for experimenting with Podman's Docker-compatible API. Pithagoras is a separate upstream project; consult its repository for its code and license. This guide does not modify or redistribute Pithagoras source code.
