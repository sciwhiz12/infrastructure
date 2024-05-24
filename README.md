
# infrastructure

The infrastructure definitions for my personal/testing Kubernetes cluster, using Argo CD.

## Bootstrapping

**TODO.** Rough outline: install cluster (k3s), install argo-cd via Helm, add repo (w/ deploy key for private), install `core` application, install `apps` application

## Core and Services

The `core` directory contains the core services that is required for normal operation of the cluster, such as certificate issuance and renewal, DNS records management, sealed secrets, network ingress, and including Argo CD itself.

The `apps` directory contains the public applications and services running on the cluster.

Each directory is defined as a Helm chart, where each template within corresponds to a single Argo CD `Application`. As certain applications may require extra configuration, the `extras` subdirectory may contain subdirectories corresponding to each application, as an additional source.
