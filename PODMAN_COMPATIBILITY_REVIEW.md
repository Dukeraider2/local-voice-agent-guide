# Legacy Podman / Pithagoras Compatibility Review

**Review date:** 2026-10-31  
**Status:** Secondary integration note; not the primary project architecture  
**Superseded as project direction by:** [PROJECT_SCOPE.md](PROJECT_SCOPE.md)

## Why this document remains

The original project request focused on adapting the Pithagoras portal and its managed Voice extension from Docker to Podman. Source review found that the managed Voice extension depends on `pithagoras-docker`, calls Docker Engine API endpoints directly, shares the portal's network namespace, and requests NVIDIA GPUs through Docker-specific `HostConfig.DeviceRequests`.

This remains useful background for an optional Pithagoras portal deployment. It is no longer the main implementation target.

## Current primary direction

The project now builds a standalone local speech inference service using audio.cpp as the first engine candidate, with ROCm/HIP acceleration on the Framework Desktop Radeon 8060S (`gfx1151`). The service will expose ASR and TTS over HTTP. Pithagoras, Hermes, and other agents will consume that API.

The implementation sequence and acceptance criteria are defined in [PROJECT_SCOPE.md](PROJECT_SCOPE.md).

## What the earlier Podman work establishes

- A Podman Compose override was drafted to expose the rootless Podman socket to the Pithagoras portal through its Docker-compatible API client.
- The override sets `DOCKER_SOCKET=/var/run/docker.sock` and disables SELinux labeling for that socket mount.
- Static source review identified compatibility points: API ping, image pull, container lifecycle, volume lifecycle, log/exec operations, and shared network namespace behavior.
- The upstream managed Voice GPU path is NVIDIA-specific. It does not provide a native AMD ROCm configuration.

## Limitations

No end-to-end Podman compatibility test was completed in this environment. The previous override is experimental and must not be described as a validated Podman port. No GPU acceleration is claimed through the Pithagoras managed Voice extension.

The Compose override is retained only for the optional portal experiment. It is not required for the standalone speech gateway and is not the way the gateway should access ROCm.

## Decision

**Do not expand the Docker-compatible API shim into the core speech implementation.** First validate audio.cpp natively with HIP on the target host. After ASR/TTS works and measurements are recorded, package the same validated runtime with Podman and expose its HTTP API to clients.

The project remains experimental until the acceptance criteria in [PROJECT_SCOPE.md](PROJECT_SCOPE.md) pass on the target machine.
