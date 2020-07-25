---
title: "Running Vault on OpenShift"
tags: 
  - vault
  - openshift
  - hashicorp
  - consul
  - helm
---

Having discussed on Vault on Kubernetes in the [previous post][vault-intro], we will proceed with the detailed steps on running Vault on Kubernetes. In this scenario, we will run Vault on [OpenShift][ocp-docs], which is an enterprise version of Kubernetes developed by Red Hat.

[ocp-docs]: https://docs.openshift.com/
[vault-intro]: https://raylaijh.github.io/vault-intro

The following steps show how to use Helm to install Vault on OpenShift. Most of the steps are taken from the references below:

  * [https://learn.hashicorp.com/vault/kubernetes/minikube][link1]
  * [https://learn.hashicorp.com/vault/new-release/openshift][link2]
  
[link1]: https://learn.hashicorp.com/vault/kubernetes/minikube
[link2]: https://learn.hashicorp.com/vault/new-release/openshift

## Prerequisites

The following are required prior to the setup

  * Running OpenShift cluster
  * Helm CLI installed (Refer [here][helm-ocp-install] to learn how to install Helm CLI on OpenShift 4)
  
Helm is a package manager that installs and configures all the necessary components to run Vault in several different modes. To install Vault via the Helm chart in the next step requires that you are logged in as administrator within a project.
  
In my example, the following versions of OpenShift and Helm installed are as follows:

```css
$ oc version
Client Version: 4.4.8
Server Version: 4.4.8
Kubernetes Version: v1.17.1+3f6f40d
```

```css
$ helm version
version.BuildInfo{Version:"v3.2.3+4.el8", GitCommit:"2160a65177049990d1b76efc67cb1a9fd21909b1", GitTreeState:"clean", GoVersion:"go1.13.4"}
```

[helm-ocp-install]: https://docs.openshift.com/container-platform/4.4/cli_reference/helm_cli/getting-started-with-helm-on-openshift-container-platform.html

## Installing Vault on OpenShift

We will follow the HashiCorp's [official document][vault-minikube] as a guide to install Vault. In this guide, Vault is deployed with Consul as a backend.

[vault-minikube]: https://learn.hashicorp.com/vault/kubernetes/minikube


### Install the Consul Helm chart

Consul is a service mesh solution that launches with a key-value store. Vault requires a storage backend like Consul to manage its configuration and secrets when it is run in high-availability.

The recommended way to run Consul on Kubernetes is via the Helm chart. Helm is a package manager that installs and configures all the necessary components to run Consul in several different modes. A Helm chart includes templates that enable conditional and parameterized execution. These parameters can be set through command-line arguments or defined in YAML.

Create `helm-consul-values.yml` with the following contents:
```yaml
global:
  datacenter: vault-kubernetes-guide

client:
  enabled: true

server:
  replicas: 1
  bootstrapExpect: 1
  disruptionBudget:
    maxUnavailable: 0
```

Add the HashiCorp Helm repository.
```css
$ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" has been added to your repositories
```

Install the latest version of the Consul Helm chart with parameters `helm-consul-values.yml` applied.
```css
$ helm install consul hashicorp/consul --values helm-consul-values.yml
```

Get all the pods within the default namespace. The Consul client and server pods are displayed here prefixed with `consul`.You should see 1x consul-server pods and N consul pods, where N is the number of worker nodes in the cluster.

Wait until the server and client pods report that they are `Running` and ready `(1/1)`.
```css
$ oc get pods -n default
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-2llf7                     1/1     Running   0          20s
consul-consul-6p7bb                     1/1     Running   0          20s
consul-consul-server-0                  1/1     Running   0          20s
```

### Install the Vault Helm chart

The recommended way to run Vault on OpenShift is via the Helm chart. 

Create `helm-vault-values.yml`
```yaml
server:
  affinity: ""
  ha:
    enabled: true
```

Install the latest version of the Vault Helm chart with parameters `helm-vault-values.yml` applied
```css
helm install vault hashicorp/vault --values helm-vault-values.yml
```

The Vault pods and Vault Agent Injector pod are deployed in the default namespace.

Get all pods running in the default namespace
```css
$ oc get pods -n default

NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-2llf7                     1/1     Running   0          1m3s
consul-consul-6p7bb                     1/1     Running   0          1m3s
consul-consul-server-0                  1/1     Running   0          1m3s
vault-0                                 0/1     Running   0          2m5s
vault-1                                 0/1     Running   0          2m5s
vault-2                                 0/1     Running   0          2m5s
vault-agent-injector-7898f4df86-trdjn   1/1     Running   0          2m5s
```

The `vault-0`, `vault-1`, `vault-2`, and `vault-agent-injector` pods are deployed. The Vault servers report that they are `Running` but they are not ready `(0/1)`. That is because Vault in each pod is executes a status check defined in a readinessProbe. The readinessProbe is failing because the Vault pods are sealed.

### Initialize and unseal Vault

Initialize Vault with one key share and one key threshold. The unseal key is extracted and stored in `cluster-keys.json`
```css
$ oc exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```


To display the unseal key:
```css
$ cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
GDibK7TonxYICNjwvsxidNnwfh8YsYkG3kylZ+lV7lE=
```

> Insecure operation: Do not run an unsealed Vault in production with a single key share and a single key threshold. This approach is only used here to simplify the unsealing process for this demonstration.

Create a variable named VAULT_UNSEAL_KEY to capture the Vault unseal key.
```css
$ VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

Unseal all the Vault pods one by one
```css
$ oc exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
$ oc exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
$ oc exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
```

Now all the Vault pods should be `Running` at `(1/1)`
```css
$ oc get pods -n default
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-2llf7                     1/1     Running   0          4h34m
consul-consul-6p7bb                     1/1     Running   0          4h34m
consul-consul-server-0                  1/1     Running   0          4h34m
vault-0                                 1/1     Running   0          4h33m
vault-1                                 1/1     Running   0          4h33m
vault-2                                 1/1     Running   0          4h33m
vault-agent-injector-7898f4df86-trdjn   1/1     Running   0          4h33m
```

Now you should have a Vault instance running on your OpenShift cluster! Next we'll learn how to use Vault to provide secrets to applications running on OpenShift

