# OpenShell and HAMi Integration

This repository documents an end to end test of running HAMi GPU sharing on top of an OpenShell gateway. It records every command, every output, and the gap that the test exposed.

OpenShell is the NVIDIA project for sandboxed AI agent runtimes. It runs a Kubernetes (K3s) cluster inside a container and creates each agent sandbox as a pod in that cluster. HAMi is a CNCF Sandbox project that turns one physical NVIDIA GPU into many isolated slices, with hard limits on memory and compute.

In theory the two should work together. OpenShell already runs on Kubernetes and HAMi is Kubernetes native. The test in this repo confirms that HAMi works correctly when an OpenShell sandbox happens to request a memory slice. It also shows that OpenShell never asks for one, so the first GPU sandbox claims the entire physical card and no second sandbox can schedule. That kills the sharing that HAMi exists to provide.

## Files

The TESTING.md file walks through the full test on a real GPU host. Every command is shown together with its actual output.

The PROPOSAL.md file (added later) will describe the change that OpenShell needs.

## Upstream issue

The findings in this repo were filed as a feature request on the OpenShell side:

https://github.com/NVIDIA/OpenShell/issues/1065

That issue contains the proposed design and links back to this repository for the full reproduction.

## These are real measurements

Nothing in this repository is synthetic, mocked, or paraphrased. Every command was run on a live GPU machine on 2026 April 29. Every output block was copied directly from the terminal of that session. The hostname, the GPU bus ID, the HAMi annotations, the sandbox names, the pod logs, all of it is what the cluster actually returned at the moment of the test.

The software involved is real software from real organisations. OpenShell is the open source platform from NVIDIA at github.com/NVIDIA/OpenShell. HAMi is the CNCF Sandbox project from the Project HAMi community at github.com/Project-HAMi/HAMi. K3s is from SUSE. The container runtime is the public NVIDIA Container Toolkit. The cloud machine was rented from Nebius.

The aim of this writeup is to give a maintainer or another engineer enough information to reproduce the same result on their own hardware. The tooling exists, the limitation is real, and the fix is small.

## Test machine

One Nebius node called `nebius-tarantula`. One NVIDIA L40S with 46068 MB of memory. Ubuntu host running kernel 6.11.0, NVIDIA driver 580.126.09, and nvidia container runtime 1.18.2. K3s v1.34 on the host (used for an isolated baseline earlier in the day), K3s v1.35 inside the OpenShell gateway container. HAMi chart v2.8.1.

## Headline result

HAMi enforces the memory cap exactly as advertised. A pod that asks for `nvidia.com/gpumem: 8000` sees an 8 GB card from inside the container, even though the host card is 46 GB.

The OpenShell sandbox specification has no field for GPU memory. The driver always requests `nvidia.com/gpu: 1` and nothing else. Under HAMi this means the first sandbox is allocated 46068 MB of GPU memory and 100 percent of the cores. The very next pod that asks for any GPU memory fails to schedule with `CardInsufficientMemory`.

The fix is to plumb a memory request through the sandbox spec. The PROPOSAL.md document covers what needs to change and where.
