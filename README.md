# Local ROCm Voice Inference Gateway

A Podman-first local speech inference project targeting AMD GPUs, beginning with the Framework Desktop AI Max 395 / Radeon 8060S (`gfx1151`).

## Current project standard

**The project's primary scope is now a ROCm/HIP-accelerated local speech inference service.** This replaces the earlier primary focus on converting the Pithagoras deployment guide from Docker to Podman.

The implementation standard is:

1. Verify the target host's ROCm/HIP stack and GPU architecture.
2. Prove native HIP inference with a pinned audio.cpp revision.
3. Qualify ASR and TTS models with correctness and performance measurements.
4. Expose a stable local HTTP inference API.
5. Package the validated service with Podman.
6. Integrate Pithagoras, Hermes, and other agents as API clients.

The canonical scope, architecture, phases, engineering rules, and acceptance criteria are maintained in [PROJECT_SCOPE.md](PROJECT_SCOPE.md). Changes to implementation should follow that document.

## Architecture at a glance

```text
Pithagoras / Hermes / other clients
                 |
              HTTP API
                 |
       Local Speech Gateway
                 |
             audio.cpp
              /     \
         HIP/ROCm   CPU fallback
             |
      Radeon 8060S (gfx1151)
                 |
        Podman deployment
```

## Why audio.cpp

The current [audio.cpp project](https://github.com/0xShug0/audio.cpp) documents a native HIP/ROCm backend, an HTTP server, and a broad set of speech/audio model families. Its HIP guide explicitly lists `gfx1151` as a supported architecture. That is promising architecture-level evidence, not a guarantee that every model or the host's currently installed ROCm version will work. Local build, runtime, model, and performance validation are mandatory.

See the upstream [HIP/ROCm build guide](https://github.com/0xShug0/audio.cpp/blob/main/docs/build/HIP.md).

## Mon Capitan: memorial voice and teaching initiative

The project also defines **Mon Capitan**, an original local AI voice persona and teaching framework created in tribute to Sir Patrick Stewart and the example many viewers found in his portrayal of Captain Jean-Luc Picard. It is an homage to the influence of the performances, not a recreation or impersonation of Stewart or Picard. The persona uses an original voice design, with no unauthorized voice cloning, and teaches through integrity, compassion, reason, curiosity, diplomacy, courage, accountability, and respect for human dignity.

See the [Mon Capitan Charter](MON_CAPITAN_CHARTER.md) for its voice specification, teaching principles, behavioral requirements, evaluation criteria, and attribution rules. This is a design standard; implementation and voice qualification remain pending.

## Current status

- **Architecture selected:** audio.cpp HIP/ROCm as the first inference-engine candidate.
- **GPU target:** `gfx1151` (Radeon 8060S).
- **Deployment target:** Podman.
- **Host validation:** pending diagnostic output from the target machine.
- **Native HIP build:** not yet performed.
- **ASR/TTS qualification:** not yet performed.
- **Podman GPU runtime validation:** not yet performed.
- **Agent integration:** not yet performed.

Do not interpret this README or upstream architecture support as a claim that the system is already working.

## First implementation gate

Before changing the host, inspect and record:

- Ubuntu and kernel versions.
- ROCm and HIP versions.
- `rocminfo` GPU agent and `gfx1151` target.
- HIP compiler version.
- hipBLAS, rocBLAS, and hipBLASLt library availability.

Then compare those facts against the pinned audio.cpp revision's build requirements. Do not upgrade ROCm or install packages until that comparison is complete.

## Previous Podman/Pithagoras work

The earlier Docker-to-Podman compatibility work is retained as a secondary, optional portal deployment experiment:

- [Podman compatibility review](PODMAN_COMPATIBILITY_REVIEW.md)
- [Podman Compose override](compose.podman.yaml)

That socket-compatibility route is **not** the AMD speech acceleration architecture. Pithagoras' current managed Voice extension uses Docker Engine API operations and NVIDIA-specific `DeviceRequests`; it should not be treated as a native ROCm path. The new design runs the speech engine as its own service and connects Pithagoras to it over HTTP.

## Project rules

- Podman-first. Do not install Docker as a workaround.
- Verify official documentation and actual host state before commands.
- One copy-and-paste command at a time; inspect output before proceeding.
- Pin upstream revisions and model versions.
- Measure CPU/HIP correctness, latency, throughput, and resource use.
- Preserve CPU fallback when GPU execution is unsupported or slower.
- Do not claim GPU acceleration until runtime diagnostics prove HIP is active.
- Keep endpoints local or private-tailnet by default.

See [PROJECT_SCOPE.md](PROJECT_SCOPE.md) for the full engineering standard and acceptance gates.
