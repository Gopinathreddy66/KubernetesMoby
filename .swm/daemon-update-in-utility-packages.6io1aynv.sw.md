---
title: Daemon Update in Utility Packages
---
# Overview of Daemon Update in Utility Packages

Daemon Update in Utility Packages refers to the capability within the Docker daemon to modify the configuration of containers it manages, whether they are running or stopped. This functionality enables dynamic adjustment of container settings such as resource limits, restart policies, and command arguments without the need to recreate containers.

# Purpose and Benefits

The primary purpose of Daemon Update is to allow containers to adapt to changing requirements efficiently. By enabling live updates, it avoids the downtime and overhead associated with stopping and recreating containers. This dynamic reconfiguration is essential for effective resource management and maintaining desired container behavior in production environments.

# Update Process Workflow

The update process starts with the daemon verifying the validity of the new container settings. Once validated, the daemon applies the updated host configuration to the container. For running containers, resource changes such as CPU shares, memory limits, and block IO weights are applied immediately through interaction with the container runtime interface, specifically containerd. Other changes that cannot be applied live are deferred until the container restarts. Additionally, monitoring parameters like restart policies are updated to reflect the new configuration.

# Safeguards and Error Handling

The update mechanism includes safeguards to maintain container stability and integrity. For example, updates are prevented on containers marked for removal or those with kernel memory changes while running. If any part of the update process fails, the daemon rolls back to the container's previous configuration, ensuring consistency and preventing partial or unstable updates.

# Implementation Details

Daemon Update is implemented primarily within the daemon package, with key methods such as `ContainerUpdate`, `ContainerUpdateCmdOnBuild`, and `update` located in <SwmPath>[daemon/update.go](daemon/update.go)</SwmPath>. The live application of resource changes leverages containerd integration, particularly in <SwmPath>[daemon/update_linux.go](daemon/update_linux.go)</SwmPath>. These components work together to validate, apply, and manage container configuration updates.

# Example Usage

An example of the update process is the `ContainerUpdate` method, which accepts a container identifier and a new host configuration. It first verifies the new settings, then invokes the `update` method. If the container is running, live resource updates are applied through containerd. Should the update encounter any errors, the method restores the container's original configuration to maintain operational correctness.

# Summary

In summary, Daemon Update in Utility Packages provides a robust mechanism for dynamically adjusting container configurations. It balances the need for live updates with safeguards to ensure container stability, making it a vital feature for managing containerized applications efficiently.

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBS3ViZXJuZXRlc01vYnklM0ElM0FHb3BpbmF0aHJlZGR5NjY=" repo-name="KubernetesMoby"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
