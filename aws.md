## Requirements

* AWS account and IAM credentials.
* AWS Route53 DNS Zone (registered Domain Name or delegated subdomain).
* Terraform v0.12.x and [terraform-provider-ct](https://github.com/poseidon/terraform-provider-ct)
  installed locally.random
* An SSH key pair for management access.
* `kubectl` installed locally to access the Kubernetes cluster.

## Limitations

In this alpha release, AWS clusters don't run the Cloud-Controller-Manager so LoadBalancer services don't work yet.

## Install simplest cluster

### Step 1. Set up a working directory

For example:

```
mkdir -p ~/lokomotive-infra/myawscluster
cd ~/lokomotive-infra/myawscluster
```

### Step 2. Define cluster configuration

Create a file named `myawscluster.lokocfg` with the following contents:

```hcl
cluster "aws" {
  # Change the location where lokoctl stores the cluster assets.
  asset_dir = "${pathexpand("~/lokoctl-assets/myawscluster")}"

  # Change the cluster name.
  cluster_name = "myawscluster"

  controller_count = 3
  controller_type = "t3.medium"

  # Define a Flatcar Linux channel ('stable', 'beta', 'alpha' or 'edge')
  os_channel = "stable"

  # Change DNS Zone.
  dns_zone = "example.com"

  # Change DNS zone ID
  dns_zone_id = "PWXH07GXXXX"

  # Change SSH Public keys
  ssh_pubkeys = [
    "ssh-rsa AAAAB3Nz...",
  ]

  # CNI plugin ("calico", "flannel")
  networking = "calico"

  # Number of worker nodes
  worker_count = 3
  worker_type = "t3.medium"
}
```

For a full configuration reference see the [AWS guide](https://github.com/kinvolk/lokomotive/blob/master/docs/installer/aws.md).

### Step 3. Set up credentials

Make sure your AWS credentials are configured, for example:

```
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

You also need to have your SSH public key in the `ssh-agent`.
Add your SSH private key to `ssh-agent`

```console
ssh-add ~/.ssh/id_rsa
ssh-add -L
```

### Step 4. Create cluster

```
lokoctl-0.0.1 cluster install
```

## Verify

The `kubeconfig` is generated under the assets directory in the following path:

```
cluster-assets/auth/kubeconfig
```

So you can access your cluster like this:

```
export KUBECONFIG=$HOME/lokoctl-assets/myawscluster/cluster-assets/auth/kubeconfig
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
