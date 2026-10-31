# Podman Compatibility Review

**Repository reviewed:** `thecodacus/pithagoras`  
**Review date:** 2026-10-31  
**Target repository:** `Dukeraider2/local-voice-agent-guide`  
**Review scope:** Current upstream deployment, Docker extension API use, managed Voice container creation, GPU assumptions, and a Podman-first documentation/Compose adaptation.

## Executive result

**Disposition: conditional / experimental. Not production-validated.**

The local repository is a reproduction guide, not a maintained fork of Pithagoras. The safe conversion scope is therefore to convert the guide and provide a Podman Compose override. The upstream Pithagoras managed Voice implementation remains Docker-API-dependent. Podman's Docker-compatible API makes an interoperability experiment plausible, but static source review does not prove full behavioral compatibility.

The first test target is CPU-only speech with Kokoro and Whisper. NVIDIA GPU speech is explicitly out of scope until the upstream Docker `DeviceRequests` path is adapted or demonstrated to work with Podman.

## Findings

| Severity | Finding | Evidence / impact | Disposition |
|---|---|---|---|
| Critical | Managed Voice requires the Pithagoras Docker extension | `extensions/voice/package.json` declares `pithagoras-docker` as a required dependency. Voice does not manage containers independently. | Podman Docker-API compatibility is a prerequisite, not an optional convenience. |
| High | Pithagoras speaks Docker Engine API directly | `extensions/docker/src/engine.ts` sends raw HTTP requests over a Unix socket and checks `GET /_ping`; `DOCKER_SOCKET` selects the socket path. | A Podman socket can be pointed at the client, but the API operations still require compatibility testing. |
| High | Voice shares the portal network namespace | `extensions/voice/src/server/service.ts` calls `sharedNetwork()`; the Docker extension constructs `NetworkMode: container:<portal-id>` for a containerized portal. | Test container namespace sharing and loopback health endpoints before accepting the port. |
| High | NVIDIA GPU selection is Docker-specific | Voice creates probe and service containers using `HostConfig.DeviceRequests` with `Driver: nvidia` and GPU capabilities. The setup script runs `nvidia-smi` and builds CUDA runtimes. | Do not claim GPU acceleration works through the compatibility socket. Podman's CDI path is not exposed by this upstream request path. |
| Medium | Voice requires image, volume, lifecycle, logs, inspect and exec API operations | Docker extension uses image pull, container create/inspect/start/stop/remove, volume create/inspect/remove and log/exec endpoints. | Exercise the whole lifecycle, not only `/_ping`. |
| Medium | Current guide had become stale | It referred to an earlier managed service setup, Breeze as the primary fixed choice, and generic Docker commands. Current upstream supports selectable Breeze/Chatterbox/Kokoro and Whisper/Qwen3-ASR. | README now distinguishes the original video stack from current upstream behavior. |

## Podman adaptation added

- `compose.podman.yaml` overlays the upstream portal Compose file with the rootless Podman socket at `/var/run/docker.sock`.
- Sets `DOCKER_SOCKET=/var/run/docker.sock`, which is the environment variable read by the current upstream Docker API client.
- Adds `security_opt: [label=disable]` for access to the mounted Podman socket, as required by Podman's API service guidance.
- README uses `podman compose` and requires rendering the merged config before launch.
- README explicitly limits the first managed Voice attempt to CPU speech engines and states that the integration is experimental.

## Compatibility matrix

| Capability | Static evidence | Status |
|---|---|---|
| Rootless Podman API socket | Official Podman API exposes a Docker-compatible API and systemd user socket | Documented; not tested in this environment |
| Compose launch of portal | Podman Compose is an external Compose provider wrapper | Documented; not tested in this environment |
| Pithagoras socket ping | Upstream uses Docker `GET /_ping`; Podman offers Docker API compatibility | Plausible; runtime test required |
| Image pull / create / inspect / start / stop | Upstream relies on Docker Engine API endpoints | Unverified against Podman |
| Named volume lifecycle | Upstream creates/adopts/removes a named voice volume | Unverified against Podman |
| `NetworkMode=container:<id>` | Upstream requires portal/voice loopback sharing; Podman compatibility API supports Docker API surface but exact behavior is untested here | Unverified; blocking acceptance |
| NVIDIA `DeviceRequests` | Upstream sends Docker-specific NVIDIA request fields | Unsupported by this guide; requires a Podman-specific upstream adapter or proven API translation |
| End-to-end speech | Requires all above plus engine build/model startup and browser audio | Not tested |

## Required runtime acceptance tests

1. Render the Compose merge and verify the rootless socket is the only socket mounted into the portal.
2. Verify the portal can call Podman's Docker-compatible `/_ping` endpoint from inside the container.
3. Install the Pithagoras Docker extension and confirm its overview reports the engine available.
4. Install Voice with Kokoro on CPU and Whisper on CPU.
5. Confirm the extension can pull images, create the voice container, create/adopt the named volume, and start the container.
6. Confirm the voice container shares the portal network namespace and health checks on the expected loopback ports succeed.
7. Complete microphone → transcript → local LLM → spoken response.
8. Restart the portal and host user session; verify socket activation and service recovery behavior.
9. Record Podman version, Compose provider/version, Pithagoras revision, API errors, logs, and resource/latency measurements.

Any failure in steps 2–7 blocks a claim of Podman support. Do not install Docker as a workaround without explicit project-owner approval.

## Sources

- Pithagoras deployment guide: https://thecodacus.github.io/pithagoras/guide/deploying
- Pithagoras voice service guide: https://thecodacus.github.io/pithagoras/guide/voice-service
- Pithagoras Docker extension guide: https://thecodacus.github.io/pithagoras/guide/docker
- Upstream Voice package: https://github.com/thecodacus/pithagoras/blob/main/extensions/voice/package.json
- Upstream Voice service implementation: https://github.com/thecodacus/pithagoras/blob/main/extensions/voice/src/server/service.ts
- Upstream Docker API implementation: https://github.com/thecodacus/pithagoras/blob/main/extensions/docker/src/engine.ts
- Podman system service: https://docs.podman.io/en/latest/markdown/podman-system-service.1.html
- Podman Compose: https://docs.podman.io/en/latest/markdown/podman-compose.1.html

## Review limitations

This was a source/documentation review against upstream `main` as available on 2026-10-31. No Podman host or target machine was available to run the portal, API compatibility tests, GPU tests, or end-to-end audio tests. The added configuration is an experiment, not a certification of compatibility.
