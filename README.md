# Lokomotive Kubernetes

## Introduction

This is an alpha release of Lokomotive Kubernetes.

### Components

Components are applications that are core to a Kubernetes cluster.
Some components included in Lokomotive Kubernetes are:

* Dex/Gangway
* Metrics server
* Prometheus operator
* Calico
* etc.

### Versioning

So you can test updates, this [alpha release](https://github.com/kinvolk/lokomotive-kubernetes-alpha/releases/tag/v0.0.1) contains two fictitious versions of lokoctl (the Lokomotive Kubernetes CLI): 0.0.1 and 0.0.2.

In the first iteration, the Lokomotive Kubernetes version is fully contained in the lokoctl binary.
A Lokomotive Kubernetes version includes a particular Kubernetes version and a set of component versions.

Here's the changelog for the fictitious Lokomotive Kubernetes 0.0.2:

#### Version 0.0.2

- Update Kubernetes from v1.17.1 to v1.17.3
- Update Dex from v0.20.0 to v0.21.0

### Updates

Lokomotive Kubernetes is fully self-hosted, that is, it runs both the Kubernetes Control Plane and the Kubelet as Kubernetes pods.
This means we can do in-place updates.

The lokoctl command to install a cluster is idempotent, so to update Lokomotive Kubernetes you run the same command that installs a cluster with a different version of lokoctl.

**Note**: there's a [constant Terraform diff when re-running `lokoctl cluster install`](https://github.com/kinvolk/lokomotive/issues/24). This is just cosmetic and doesn't have negative effects.

**Note**: there's a bug in the runc version shipped on Flatcar Container Linux that prevents Kubelet updates to work, so that's disabled for now and the Kubelet version will stay the same.
We have tested the updates work fine with a newer runc but this is not yet shipped in a working state on Flatcar Container Linux.

## Create a cluster

* [Packet quickstart](packet.md)
* [AWS quickstart](aws.md)
* [Bare-metal quickstart](bare-metal.md)

## Install components

To list available components:

```console
lokoctl-0.0.1 component list
```

To install a component add the relevant section to your `.lokocfg` file, for example:

```hcl
component metrics-server {}
```

You can find more detailed information about Lokomotive Kubernetes components here: https://github.com/kinvolk/lokomotive/tree/master/docs/configuration-guides/components

Note that these references are not final and might have inconsistencies.

## Component workflows

This section includes howtos for some common component workflows.

* [Set up HTTP load balancing on Packet](load-balancing.md)
* [Set up cluster authentication with GitHub](github-auth.md)
