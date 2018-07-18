Balena: Moby-based container engine for IoT
===========================================

<img src="docs/static_files/balena-logo-black.svg" alt="Balena" width="50%" />

## Overview

Balena is a new container engine purpose-built for embedded and IoT use cases
and compatible with Docker containers. Based on Docker’s Moby Project, balena
supports container deltas for 10-70x more efficient bandwidth usage, has 3x
smaller binaries, uses RAM and storage more conservatively, and focuses on
atomicity and durability of container pulling.

## Features

- __Small footprint__
	- 3.5x smaller than Docker CE, packaged as a single binary
- __Multi-arch support__
	- Available for a wide variety of chipset architectures, supporting everything from tiny IoT devices to large industrial gateways
- __True container deltas__
	- Bandwidth-efficient updates with binary diffs, 10-70x smaller than pulling layers
- __Minimal wear-and-tear__
	- Extract layers as they arrive to prevent excessive writing to disk, protecting your storage from eventual corruption
- __Failure-resistant pulls__
	- Atomic and durable image pulls defend against partial container pulls in the event of power failure
- __Conservative memory use__
	- Prevents page cache thrashing during image pull, so your application runs undisturbed in low-memory situations

## Transitioning from Docker CE

We left out Docker features that we saw as most needed in cloud deployments and
therefore not warranting inclusion in a lightweight IoT-focused container
engine. Specifically, we’ve excluded:

- Docker Swarm
- Cloud logging drivers
- Plugin support
- Overlay networking drivers
- Non-boltdb discovery backends (consul, zookeeper, etcd, etc.)

Unless you depend on one of the features in Docker that balena omits, using
balena should be a drop-in replacement

Licensing
=========
Balena is licensed under the Apache License, Version 2.0. See
[LICENSE](https://github.com/resin-os/balena/blob/master/LICENSE) for the full
license text.


## Investigating the `write init-p: broken pipe` error

**Behaviour:** When running Balena exec with less than a huge amount of available memory, the exec fails with 'rpc error: code = Unknown desc = oci runtime error: exec failed: container_linux.go:263: starting container process caused "process_linux.go:95: writing config to pipe caused "write init-p: broken pipe"'

Something closes (?) setnsProcess' parentPipe before it gets to write its config during start()

Final reported error: [process_linux.go#L95](https://github.com/resin-os/balena/blob/17.06-resin/vendor/github.com/opencontainers/runc/libcontainer/process_linux.go#L95)

which is called by [container_linux.go#L263](https://github.com/resin-os/balena/blob/17.06-resin/vendor/github.com/opencontainers/runc/libcontainer/container_linux.go#L263)

The flow to get to those files starts as:
- [/libcontainerd/client_linux.go](https://github.com/resin-os/balena/blob/6785696ce1e6dea3559aae73ebfeb23909a0d61a/libcontainerd/client_linux.go#L124) does remote.apiClient.AddProcess(request)

- [containerd/api/gprc/server/server_linux.go](https://github.com/resin-os/balena/blob/328a1d189eca816ca045bd467cd0476f73ea570d/vendor/github.com/containerd/containerd/api/grpc/server/server_linux.go#L62) does SendTask(addProcess)

- [containerd/containerd/supervisor/supervisor.go](https://github.com/resin-os/balena/blob/328a1d189eca816ca045bd467cd0476f73ea570d/vendor/github.com/containerd/containerd/supervisor/supervisor.go#L385) routes the task to add_process.go

- [containerd/containerd/supervisor/add_process.go](https://github.com/resin-os/balena/blob/328a1d189eca816ca045bd467cd0476f73ea570d/vendor/github.com/containerd/containerd/supervisor/add_process.go#L32) calls container.Exec

- [containerd/containerd/runtime/container.go](https://github.com/resin-os/balena/blob/328a1d189eca816ca045bd467cd0476f73ea570d/vendor/github.com/containerd/containerd/runtime/container.go#L483)'s Exec runs normally until createCmd, where err becomes the SystemErrorWithCauses

- ...
