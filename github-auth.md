# Set up cluster authentication with GitHub

## Introduction

There are two categories of users in Kubernetes: service accounts and human users. The Kubernetes
API manages service accounts, but not human users. The creation and management of human users and
managing their authentication is outside the scope of Kubernetes.

Kubernetes supports several authentication strategies. A common strategy is to use external identity
providers via [OpenID
Connect](https://kubernetes.io/docs/reference/access-authn-authz/authentication//#openid-connect-tokens).
These external identity providers can range from standard directory services (e.g. LDAP) to OAuth platforms
such as GitHub and OpenID providers such as Google Accounts, Salesforce and Azure AD v2.

Organizations can authenticate users to Kubernetes clusters using their identity provider.

[Dex](https://github.com/dexidp/dex) acts as an intermediary, providing an essential abstraction
layer between the Kubernetes API server and external identity providers.

[Gangway](https://github.com/heptiolabs/gangway) is a web application that allows obtaining OIDC
tokens from identity providers and automatically generating kubeconfigs to be used by Kubernetes
users.

## Learning Objectives

This guide assumes familiarity with Kubernetes authorization and authentication mechanisms.

Upon completion of this guide, you will be able to leverage the centralized authentication and authorization of
your organization for your Lokomotive cluster.

## Prerequisites

To create a fully functioning OIDC authentication infrastructure, we need the following:

* A Lokomotive cluster accessible via `kubectl` deployed on a supported provider.

* Permissions to create an [Authorized OAuth App](https://github.com/settings/applications) under GitHub organization settings.

* [cert-manager](https://cert-manager.io/docs/) deployed on the cluster.

  Installation instructions for [cert-manager](https://github.com/kinvolk/lokomotive/blob/master/docs/components/cert-manager.md) Lokomotive component.

* [MetalLB](https://metallb.universe.tf/) deployed on the cluster.

  **NOTE**: Required only for the bare metal and Packet providers.

   Installation instructions for [MetalLB](https://github.com/kinvolk/lokomotive/blob/master/docs/components/metallb.md) component.

* [Contour](https://projectcontour.io/) deployed on the cluster.

  Installation instructions for [Contour](https://github.com/kinvolk/lokomotive/blob/master/docs/components/contour.md) component.

* [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) deployed on the cluster.

   Installation instructions for [ExternalDNS](configuration-guides/externaldns) component.

## Estimated Time

This how-to guide is expected to take about 45 minutes.

## Steps

### Step 1: Configure Dex and Gangway

Dex and Gangway are available as a Lokomotive component. A configuration file is needed to install Dex and Gangway.

Create a file named `auth.lokocfg` with the below contents.

```hcl
variable "github_client_id" {
  type = "string"
}

variable "github_client_secret" {
  type = "string"
}

variable "dex_static_client_gangway_id" {
  type = "string"
}

variable "dex_static_client_gangway_secret" {
  type = "string"
}

variable "gangway_redirect_url" {
  type = "string"
}

variable "gangway_session_key" {
  type = "string"
}

# Dex component configuration.
component "dex" {
  # NOTE: This name should match with the contour component configuration
  `ingress_hosts`
  ingress_host = "dex.test-cluster.lokomotive.org"

  issuer_host = "https://dex.test-cluster.lokomotive.org"

  # GitHub connector configuration.
  connector "github" {
    id = "github"
    name = "Github"

    config {
      client_id = var.github_client_id

      client_secret = var.github_client_secret

      # The authorization callback URL as configured with GitHub.
      redirect_uri = "https://dex.test-cluster.lokomotive.org/callback"

      # Can be 'name', 'slug' or 'both'.
      # See https://github.com/dexidp/dex/blob/master/Documentation/connectors/github.md
      team_name_field = "slug"

      # GitHub organization details.
      org {
        name = "your-github-org-change-me"
        teams = [
          "github-team-name-change-me",
        ]
      }
    }
  }

  # Static client details - Gangway.
  static_client {
    name   = "gangway"
    id     = var.dex_static_client_gangway_id
    secret = var.dex_static_client_gangway_secret

    redirect_uris = [var.gangway_redirect_url]
  }
}

# Gangway component configuration.
component "gangway" {
  cluster_name = "test-cluster"

  ingress_host = "gangway.test-cluster.lokomotive.org"

  session_key = var.gangway_session_key

  api_server_url = "https://test-cluster.lokomotive.org:6443"

  # Dex 'auth' endpoint.
  authorize_url = "https://dex.test-cluster.lokomotive.org/auth"

  # Dex 'token' endpoint.
  token_url = "https://dex.test-cluster.lokomotive.org/token"

  # The static client id and secret.
  client_id     = var.dex_static_client_gangway_id
  client_secret = var.dex_static_client_gangway_secret

  # Gangway's redirect URL.
  redirect_url = var.gangway_redirect_url
}
```

### Step 2: Register a New OAuth Application

Go to the GitHub organization Settings
page (`https://github.com/organizations/<your-org-name/settings/applications>`) and register a new
OAuth application.

**Set Homepage URL** to the value of the `ingress_host` field in the Dex component configuration.

HomePage URL must match the `issuer_host` in Dex configuration.

Set **Authorization Callback URL** to the value of the `redirect_uri` in the Dex configuration.

![Registering OAuth application ](github-oauth-app-register.png?raw=true "Register a new OAuth
applicaion")

After registering the application, take note of the ClientID and ClientSecret.

### Step 3: Create Variables File

Create another file `lokocfg.vars` for variables and secrets that should be referenced in the cluster configuration.

```
# A random secret key (create one with `openssl rand -base64 32`)
dex_static_client_gangway_secret="vJ09ouDw1BXEz6onT2+xW8PdofWIG8cN8+f0bv1zKZI="
dex_static_client_gangway_id="gangway"

# A random secret key (create one with `openssl rand -base64 32`)
gangway_session_key="PMXEGiQ7fScPxuKS/DAimsCHueeWxT7HBL6I16sZzHE="
gangway_redirect_url = "https://gangway.test-cluster.lokomotive.org>/callback"

# GitHub OAuth application client ID and secret.
github_client_id = "87a2e79c21e7ed32re51"
github_client_secret = "1708gg95433178e6cb63ae2f86b42b78g3810978"
```

### Step 4: Install Dex and Gangway

To install, execute:

```console
lokoctl component install
```

In few minutes cert-manager component issues the TLS certificates for the Dex and Gangway Ingress hosts.

Issuing an HTTPS request to the discovery endpoint verifies the successful installation
of Dex.

```console
$ curl https://dex.test-cluster.lokomotive.org/.well-known/openid-configuration

{
  "issuer": "https://dex.test-cluster.lokomotive.org",
  .
  .
  .
}
```

To verify the Gangway installation, open the URL `https://gangway.test-cluster.lokomotive.org` on your browser.

### Step 5: Configure An API Server to Use Dex as an OIDC Authenticator

Configuring an API server to use the OpenID Connect authentication plugin requires:

* Deploying an API server with specific flags.

* Dex is running on HTTPS.

* Dex is accessible to both your browser and the Kubernetes API server.

To reconfigure the API server with specific flags, edit the `kube-apiserver` DaemonSet as follows:

```console
kubectl -n kube-system edit daemonset kube-apiserver
```

Add the following arguments:

```console
--oidc-issuer-url=https://dex.test-cluster.lokomotive.org
--oidc-client-id=gangway
--oidc-username-claim=email
--oidc-groups-claim=groups
```

Set the argument values according to the following table

| Argument | Value |
| ------------- | -------- |
| `--oidc-issuer-url`| Value of the `issuer_host` in the Dex configuration. |
| `--oidc-client-id` | The client ID obtained from GitHub in step 2. |

Example:

```console
     containers:
      - command:
        - /hyperkube
        - kube-apiserver
        - --advertise-address=$(POD_IP)
        - --allow-privileged=true
        - --anonymous-auth=false
        - --authorization-mode=RBAC
        .
        .
        .
        .
        --oidc-issuer-url=https://dex.test-cluster.lokomotive.org
        --oidc-client-id=gangway
        --oidc-username-claim=email
        --oidc-groups-claim=groups
```

It may take a few moments for the API server pods to restart. You can check the status by running the following command:

```console
kubectl get pods -n kube-system
```

## Step 6: Authenticate With Gangway (For Users)

Sign in to Gangway using the URL `https://gangway.test-cluster.lokomotive.org`.
ou should be able to authenticate via GitHub. Upon successful authentication, you should be redirected to https://gangway.test-cluster.lokomotive.org/commandline.

Gangway provides further instructions for configuring `kubectl` to gain access to the cluster.

## Step 7: Authorize Users (For Cluster Administrators)

By default, newly authenticated users/groups don't have any permissions on the cluster.

Create [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) bindings for
users/groups with permissions as necessary.

Example: Provide view access to the user `jane@example.com`.

```console
kubectl create clusterrolebinding view-only --clusterrole view --user='jane@example.com'
```

## Summary

In this guide you've learned how use Dex and Gangway to leverage existing identity providers for authentication and authorization on Lokomotive clusters."
