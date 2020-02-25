# Set up HTTP load balancing on Packet

## Introduction

Kubernetes passes on the responsibility of creating a load balancer for services of type `LoadBalancer`
to the underlying cloud provider. Bare metal providers such as Packet, however, typically don't have an implementation
of network load-balancers. Therefore, these services always remain in the `Pending` state forever.

MetalLB aims to address this problem by offering a load balancer implementation for bare metal Kubernetes
clusters using standard routing protocols.

Contour, on the other hand, addresses the need for an efficient and smooth ingress traffic management at
the [Layer 7](https://en.wikipedia.org/wiki/OSI_model#Layer_7:_Application_Layer) level.

Contour is an Ingress controller for Kubernetes that works by deploying the Envoy proxy as a reverse proxy and load balancer.

This guide provides installation steps to configure MetalLB and Contour to help you set up HTTP load balancing
on a Lokomotive cluster with Packet Provider.

## Learning Objectives

This guide assumes familiarity with Kubernetes and has a basic understanding of Ingress and Load Balancers.

Upon completion of this guide, you will be able to use Service type `LoadBalancer` in your Lokomotive cluster on Packet.

## Prerequisites

To set up HTTP load balancing, we need the following:

* A Lokomotive cluster accessible via `kubectl` deployed on Packet.

* IPv4 address pools for MetalLB to allocate — one address per LoadBalancer Service. You'll need to request a Public IPv4 from Packet in your intented Location and Packet Project in the Packet UI.


## Estimated Time

This how-to guide is expected to take about 15 minutes.

## Steps

### Step 1: Configure MetalLB and Contour

MetalLB and Contour are available as a Lokomotive components. A configuration file is needed to install them.

MetalLB operates in two modes: BGP and Layer 2. Lokomotive supports MetalLB in BGP mode.

Create a file named `ingress.lokocfg` with the below contents.

```hcl
# MetalLB component configuration.
component "metallb" {
  address_pools = {
    default = ["a.b.c.d/X"]
  }
}

# Contour component configuration.
component "contour" {}
```

### Step 2: Install MetalLB and Contour

To install, execute:

```console
lokoctl component install
```

MetalLB installs in `metallb-system` namespace, whereas Contour installs in `projectcontour` namespace.

In few minutes pods from MetalLB and Contour are in `Running` state.

To verify that the BGP sessions are established, check the logs of the MetalLB speaker pods:

```console
kubectl -n metallb-system logs speaker-89764
...
{"caller":"bgp.go:63","event":"sessionUp","localASN":65000,"msg":"BGP session established","peer":"10.88.72.128:179","peerASN":65530,"ts":"2019-09-17T13:10:43.194650355Z"}
```

Contour service has an external IP address if it is properly set up with MetalLB.

```console
kubectl get svc envoy -n projectcontour
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
envoy     LoadBalancer   10.3.101.86   1XX.7X.XX9.XXX   80:30511/TCP,443:32317/TCP   5m
