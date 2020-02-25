## Requirements

* Terraform v0.12.x, [terraform-provider-matchbox](https://github.com/poseidon/terraform-provider-matchbox)
and [terraform-provider-ct](https://github.com/poseidon/terraform-provider-ct) installed locally.
* Machines with 2GB RAM, 30GB disk, PXE-enabled NIC, IPMI.
* PXE-enabled [network boot](https://coreos.com/matchbox/docs/latest/network-setup.html) environment.
* Matchbox v0.6+ deployment with API enabled
* Matchbox credentials `client.crt`, `client.key`, `ca.crt`
* An SSH key pair for management access.
* `kubectl` installed locally to access the Kubernetes cluster.

Note that the machines should only be powered on after starting the installation, see below.

## Install simplest cluster

### Step 1. Set up a working directory

For example:

```
mkdir -p ~/lokomotive-infra/mybaremetalcluster
cd ~/lokomotive-infra/mybaremetalcluster
```

### Step 2. Machines, DNS and Matchbox Set Up

#### Machines

Mac addresses collected from each machine.

For machines with multiple PXE-enabled NICs, pick one of the MAC addresses. MAC addresses will be
used to match machines to profiles during network boot.

Example:

```console
52:54:00:a1:9c:ae (node1)
52:54:00:b2:2f:86 (node2)
52:54:00:c3:61:77 (node3)
```

#### DNS

Create DNS A (or AAAA) record for each node's default interface.

Cluster nodes will be configured to refer to the control plane and themselves by these fully
qualified names and they will be used in generated TLS certificates.

Example:

```console
node1.example.com (node1)
node2.example.com (node2)
node3.example.com (node3)
```

#### Matchbox

One of the requirements is to have Matchbox with TLS enabled deployed.

Verify the matchbox read-only HTTP endpoints are accessible.

```console
curl http://matchbox.example.com:8080
matchbox
```

Verify your TLS client certificate and key can be used to access the Matchbox API.

```console
openssl s_client -connect matchbox.example.com:8081 \
  -CAfile /path/to/matchbox/ca.crt \
  -cert /path/to/matchbox/client.crt \
  -key /path/to/matchbox/client.key
```

## Step 3. Set up credentials

You need to have your SSH public key in the `ssh-agent`.
Add your SSH private key to `ssh-agent`

```console
ssh-add ~/.ssh/id_rsa
ssh-add -L
```

### Step 4. Define cluster configuration

Create a file named `mybaremetalcluster.lokocfg` with the following contents:

 ```hcl
cluster "bare-metal" {
  # Change the location where lokoctl stores the cluster assets.
  asset_dir = "${pathexpand("~/lokoctl-assets/mybaremetalcluster")}"

  # Cluster name.
  cluster_name = mybaremetalcluster

  # SSH Public keys.
  ssh_pubkeys = [
    "ssh-rsa AAAAB3Nz...",
  ]

  # Whether the operating system should PXE boot and install from matchbox /assets cache.
  cached_install = "true"

  # Matchbox CA crt path.
  matchbox_ca_path = pathexpand("/path/to/matchbox/ca.crt")
  # Matchbox client crt path.
  matchbox_client_cert_path = pathexpand("/path/to/matchbox/client.crt")
  # Matchbox client key path.
  matchbox_client_key_path = pathexpand("/path/to/matchbox/client.key")
  # Matchbox https endpoint.
  matchbox_endpoint = "matchbox.example.com:8081"
  # Matchbox HTTP read-only endpoint.
  matchbox_http_endpoint = "http://matchbox.example.com:8080"

  # Domain name.
  k8s_domain_name = "node1.example.com"

  # FQDN of controller nodes.
  controller_domains = [
    "node1.example.com",
  ]

  # MAC addresses of controllers.
  controller_macs = [
    "52:54:00:a1:9c:ae",
  ]

  # Names of the controller nodes.
  controller_names = [
    "node1",
  ]

  # FQDN of worker nodes.
  worker_domains = [
    "node2.example.com",
    "node3.example.com",
  ]

  # Mac addresses of worker nodes.
  worker_macs = [
    "52:54:00:b2:2f:86",
    "52:54:00:c3:61:77",
  ]

  # Names of the worker nodes.
  worker_names = [
    "node2",
    "node3",
  ]
}
```

### Step 4. Create cluster

```
lokoctl-0.0.1 cluster install
```

**Proceed to Power on the PXE machines while this loops.**

## Verify

The `kubeconfig` is generated under the assets directory in the following path:

```
cluster-assets/auth/kubeconfig
```

So you can access your cluster like this:

```
export KUBECONFIG=$HOME/lokoctl-assets/mybaremetalcluster/cluster-assets/auth/kubeconfig
kubectl get nodes
```

## Destroy cluster

```
lokoctl-0.0.1 cluster destroy
```

## Upgrade Lokomotive

```
lokoctl-0.0.2 cluster install
```

## Downgrade Lokomotive after upgrading

```
lokoctl-0.0.1 cluster install
```
