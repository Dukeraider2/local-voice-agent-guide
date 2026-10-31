# Project Scope and Implementation Standard

**Status:** New project direction adopted for implementation planning  
**Effective date:** 2026-10-31  
**Primary target:** Framework Desktop AI Max 395, Radeon 8060S, AMD GPU target `gfx1151`, Ubuntu 26.04.1  
**Deployment preference:** Podman-first; no Docker installation as a workaround

## 1. Scope change

The project is no longer primarily a Docker-to-Podman documentation conversion.

The new primary objective is to build and validate a **local, ROCm/HIP-accelerated speech inference service** for AMD hardware, then make that service reusable by Pithagoras and other local agents. Podman remains the preferred packaging and deployment layer. Pithagoras becomes an integration client, not the foundation of the speech runtime.

This is a material scope change. The previous Docker-compatible Podman socket approach is retained only as an optional compatibility experiment for running the upstream Pithagoras portal or its existing managed Voice extension. It is not the target architecture for AMD speech acceleration.

## 2. Product objective

Deliver a local speech service that can provide:

- Automatic speech recognition (ASR / speech-to-text).
- Text-to-speech (TTS).
- A stable local HTTP API suitable for Pithagoras, Hermes, and other agents.
- AMD GPU acceleration through ROCm/HIP where supported and measurably beneficial.
- CPU fallback for unsupported models, operations, or runtime conditions.
- Reproducible Podman deployment without requiring Docker Engine.

The service must remain independently useful even if Pithagoras changes, is unavailable, or is replaced.

## 3. Standard architecture

```text
Microphone / agent client
          |
          v
  Agent or Pithagoras
          |
          | HTTP API
          v
+----------------------------------+
| Local Speech Inference Gateway   |
|                                  |
|  ASR endpoint   TTS endpoint     |
|       |              |           |
|       +------ audio.cpp ---------+
|                      |
|             Backend selection
|               /          \
|          HIP/ROCm         CPU
|              |
|       Radeon 8060S
|         gfx1151
+----------------------------------+
          |
          v
   Podman deployment
   (service isolation,
    persistence, lifecycle)
```

### Component responsibilities

- **audio.cpp:** First-choice inference engine to evaluate because its current upstream project documents a HIP/ROCm backend and a server API. Model support, endpoint behavior, and performance must be verified per model.
- **ROCm/HIP:** Host GPU compute stack. The host's installed runtime, compiler, libraries, and architecture support must be inspected before selecting build flags or container images.
- **Speech Inference Gateway:** Stable service boundary. It should expose documented HTTP endpoints and avoid coupling clients to backend-specific details.
- **Podman:** Preferred container runtime and lifecycle manager. The deployment must expose only the required GPU devices and runtime libraries, use persistent model storage, and avoid privileged mode unless a demonstrated requirement is documented.
- **Pithagoras / Hermes / other agents:** Clients of the gateway. Integrations should use the gateway API rather than embedding GPU runtime logic in each agent.

## 4. Implementation sequence

### Phase 0 — Host and toolchain verification

Before installing or changing anything:

1. Record the exact Ubuntu release and kernel.
2. Inspect the installed ROCm/HIP version and compiler.
3. Confirm the GPU reports `gfx1151`.
4. Verify required HIP, hipBLAS, rocBLAS, and hipBLASLt libraries are present and discoverable.
5. Record the current driver/runtime package state.
6. Compare the installed stack against the exact audio.cpp revision's documented and tested requirements.

No ROCm upgrade, package replacement, environment override, or system configuration change is authorized by this project scope alone.

### Phase 1 — Native HIP proof of capability

Build audio.cpp natively on the host before containerizing it.

- Pin an explicit upstream commit or release.
- Configure the HIP backend for `gfx1151`.
- Keep CUDA disabled.
- Use conservative integrated-GPU settings. Do not enable CUDA graphs or HIP VMM unless the exact upstream guidance and local measurements justify doing so.
- Build the CLI and server targets.
- Confirm the runtime actually selects HIP and the Radeon 8060S, rather than silently falling back to CPU.
- Run a minimal supported ASR and TTS model end to end.

The upstream documentation lists `gfx1151` as a supported target, but this does not guarantee every model, kernel, or installed ROCm version will work. Treat the host proof as mandatory.

### Phase 2 — Model qualification and performance baseline

Qualify candidate ASR and TTS models individually.

For each model, record:

- Exact model family, revision, file format, and checksum.
- Whether the model is supported by the pinned audio.cpp revision.
- Whether model execution uses HIP, CPU, or a mixture.
- Cold-start and warm inference latency.
- Real-time factor or audio generation speed.
- Peak system memory and GPU/shared-memory use.
- CPU utilization and GPU utilization.
- Audio quality / transcription quality checks.
- Known limitations and repeatability.

Compare CPU and HIP where both are supported. GPU acceleration is accepted only when correctness is maintained and measurements show a meaningful benefit. Do not assume that moving a model to the GPU is automatically faster on a shared-memory integrated GPU.

### Phase 3 — Stable inference gateway

Expose the qualified ASR and TTS functions through a stable local HTTP API.

- Prefer audio.cpp's server API if it meets the gateway requirements.
- Document request/response formats, health checks, model discovery, errors, and concurrency behavior.
- Keep model/backend configuration server-side.
- Provide a stable hostname/port for local clients.
- Keep the service private to localhost or the private tailnet; do not expose it publicly by default.
- Test simultaneous and queued requests before claiming multi-agent readiness.

If the upstream server API cannot meet the requirements, add a small adapter service rather than forking or modifying the inference engine unnecessarily.

### Phase 4 — Podman packaging

Containerize only after native HIP inference works.

- Use Podman, not Docker, as the supported runtime.
- Pin the base image and userspace ROCm dependencies to a tested version.
- Map only the required AMD device nodes and groups.
- Avoid `--privileged` unless a documented test proves it is required.
- Keep model data in a persistent volume or bind mount.
- Provide health checks, restart behavior, resource limits where appropriate, and clear logs.
- Validate that the container is using the AMD GPU and not silently falling back to CPU.
- Document host-driver versus container-userspace compatibility.

The exact Podman GPU configuration is a test result, not something to guess in advance. Do not publish a supposedly working Compose file until it passes on the target machine.

### Phase 5 — Agent integration

Integrate clients only after the gateway API is stable.

1. Test with a simple HTTP client.
2. Integrate Pithagoras as one client.
3. Integrate Hermes and other agents through the same API.
4. Confirm clients can select ASR/TTS models without managing ROCm or GPU devices.
5. Test service restart, unavailable model, timeout, and concurrent request behavior.

The existing Pithagoras Docker extension may remain useful for its portal lifecycle, but its NVIDIA-specific `DeviceRequests` path is not the AMD implementation.

## 5. Engineering standards

These standards apply to all project changes:

- **Verify first:** consult current official project documentation and inspect the actual host state before commands or package changes.
- **One command at a time:** provide one copy-and-paste command, inspect its output, then choose the next step.
- **No guessed dependencies:** verify exact package names, versions, OS compatibility, and architecture support.
- **Pin sources:** record upstream repository, commit SHA or release, build options, and model revisions.
- **Measure, do not assume:** record backend selection, latency, throughput, resource use, and correctness.
- **CPU remains a supported fallback:** use it where GPU support is absent or slower.
- **Podman-first:** do not add Docker as a hidden dependency or fallback.
- **Least privilege:** no privileged containers or broad device access without a demonstrated and documented requirement.
- **No unverified claims:** distinguish documented upstream support, locally built, locally tested, benchmarked, and production-ready states.
- **Reproducibility:** preserve exact build commands, config files, logs, and test results in the repository.
- **Privacy:** keep audio and model services local/private by default; document any external network access.

## 6. Out of scope

Unless explicitly approved later, the project does not aim to:

- Port the entire Pithagoras Docker extension to Podman.
- Reproduce NVIDIA CUDA behavior on AMD.
- Guarantee that every audio.cpp model family supports HIP.
- Upgrade or replace the host ROCm stack without compatibility evidence and owner approval.
- Run a public internet-facing speech API.
- Claim real-time or multi-agent performance without measurements.
- Replace the user's existing LLM inference stack.

## 7. Acceptance criteria

The project is not considered complete until all applicable criteria pass on the target Framework Desktop:

1. Host ROCm/HIP toolchain and `gfx1151` are identified and recorded.
2. A pinned audio.cpp revision builds with HIP enabled.
3. Runtime logs or backend diagnostics prove that HIP is active on the Radeon 8060S.
4. At least one qualified ASR model and one qualified TTS model complete real audio tests.
5. CPU and HIP performance/resource measurements are recorded where comparable.
6. A documented local HTTP endpoint serves ASR and TTS requests.
7. Podman deployment passes the same functional tests and demonstrably retains HIP acceleration.
8. Pithagoras can use the gateway without relying on its NVIDIA-specific managed Voice path.
9. Setup, operation, upgrade, recovery, and troubleshooting are documented.

Until then, label the work **experimental / in validation**. Architecture support in upstream documentation is a reason to test, not proof of project acceptance.

## 8. Upstream technical reference

- [audio.cpp](https://github.com/0xShug0/audio.cpp)
- [audio.cpp HIP/ROCm build guide](https://github.com/0xShug0/audio.cpp/blob/main/docs/build/HIP.md)
- [audio.cpp server documentation](https://github.com/0xShug0/audio.cpp/tree/main/app/server)
- [Podman documentation](https://docs.podman.io/)


## 9. Mon Capitan memorial and teaching initiative

The project includes **Mon Capitan**, an optional voice-and-teaching persona conceived as a memorial to Sir Patrick Stewart and the example many viewers found in his portrayal of Captain Jean-Luc Picard in *Star Trek: The Next Generation*. Stewart died on October 28, 2026. This initiative honors the influence of his work; it does not imply his participation, endorsement, or authorization.

Mon Capitan is not a digital recreation of Stewart, Picard, or any other identifiable person or character. Its voice must be an original, properly licensed or generated voice design using broad qualities only: a mature, resonant British baritone; precise diction; measured pacing; warmth; expressive natural prosody; and quiet, thoughtful authority. Do not train or configure a voice clone from Stewart's recordings. Any future voice-cloning workflow requires explicit permission from the living rights-holder or authorized representative and appropriate rights clearance.

The teaching framework takes inspiration from themes portrayed by Picard, while remaining an independently authored framework:

- Integrity and consistency between stated values and actions.
- Compassion and respect for the dignity of others.
- Reason, evidence, curiosity, and intellectual humility.
- Diplomacy and de-escalation before force where feasible.
- Courage to question authority and accept accountability.
- Respect for life, pluralism, and principled disagreement.
- Lifelong learning and willingness to revise conclusions.

The persona should teach through explanation, questions, ethical dilemmas, and examination of competing perspectives. It must not present itself as Sir Patrick Stewart, claim to speak for him, or suggest that these project principles are his personal doctrine. It should support the user's independent judgment rather than demand loyalty or obedience.

The full charter, voice requirements, behavioral principles, evaluation criteria, and attribution rules are maintained in [MON_CAPITAN_CHARTER.md](MON_CAPITAN_CHARTER.md). This initiative is a project design specification; implementation and model/voice qualification remain separate acceptance tasks.
