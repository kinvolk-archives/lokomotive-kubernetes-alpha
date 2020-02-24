## Requirements

* Packet account, Project ID and auth token (sometimes also referred to as "API key").
* Local BGP enabled on your Packet project.
* AWS account and IAM credentials (Route53 DNS configuration).
* AWS Route53 DNS Zone (registered Domain Name or delegated subdomain).
* Terraform v0.12.x and [terraform-provider-ct](https://github.com/poseidon/terraform-provider-ct)
  installed locally.
* An SSH key pair for management access.
* `kubectl` installed locally to access the Kubernetes cluster.

## Install simplest cluster

### Step 1. Set up a working directory

For example:

```
mkdir -p ~/lokomotive-infra/mypacketcluster
cd ~/lokomotive-infra/mypacketcluster
```

### Step 2. Define cluster configuration

Create a file named `mypacketcluster.lokocfg` with the following contents:

```hcl
cluster "packet" {
  # Change the location where lokoctl stores the cluster assets.
  asset_dir = "${pathexpand("~/lokoctl-assets/mypacketcluster")}"

  # Change the cluster name.
  cluster_name = "mypacketcluster"

  controller_count = 3

  # Change Packet server location
  facility = "sjc1"

  # Define a Flatcar Linux channel ('stable', 'beta', 'alpha' or 'edge')
  os_channel = "stable"

  # Change Packet project ID
  project_id = "0281e5d1-8f67-45ba-9ce3-changeme"

  dns {
    # Change DNS Zone.
    zone = "example.net"

    provider {
      route53 {
        # Change AWS zone ID
        zone_id = "PWXH07GXXXX"
      }
    }
  }

  # Change SSH Public keys
  ssh_pubkeys = [
    "ssh-rsa AAAAB3Nz...",
  ]

  # Change to your external IP address to allow for management access.
  management_cidrs = ["0.0.0.0/0"]

  # Change to internal Packet IPs to allow cluster communication.
  #
  # You can find it in the "IP & Networks" configuration of your project for your location
  # If you don't see an IP block for your location you have to create a new
  # server on that location and the IP block will be assigned.
  node_private_cidr = "10.0.0.0/8"

  # CNI plugin ("calico", "flannel")
  networking = "calico"

  worker_pool "worker-pool-1" {
    count = 3
  }
}
```

For a full configuration reference see the [Packet guide](https://github.com/kinvolk/lokomotive/blob/master/docs/installer/packet.md).

### Step 3. Set up credentials

Make sure your Packet auth token is exported:

```
export PACKET_AUTH_TOKEN=<PACKET_AUTH_TOKEN>
```

Make sure your AWS credentials are configured, for example:

```
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Step 4. Create cluster

```
lokoctl-0.0.1 cluster install
```

*Tip*: sometimes servers fail to provision on Packet, if that happens re-run the installation command and cluster installation will continue.

## Verify

The `kubeconfig` is generated under the assets directory in the following path:

```
cluster-assets/auth/kubeconfig
```

So you can access your cluster like this:

```
export KUBECONFIG=$HOME/lokoctl-assets/mypacketcluster/cluster-assets/auth/kubeconfig
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
