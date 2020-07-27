---
title: "Using Vault as a Certificate Manager on Kubernetes"
tags: 
  - vault
  - openshift
  - hashicorp
  - helm
  - certificate
  - pki
---

In this post, we will proceed to try on some of the features that Vault can offer as a secrets manager. One of the feature is that Vault is able to be configured as a certifcate manager. This enables your services to establish their identity and communicate securely over the network with other services or clients internal or external to the cluster.

Jetstack's cert-manager enables Vault's PKI secrets engine to dynamically generate X.509 certificates within Kubernetes through an Issuer interface.
In this guide, we will configure the PKI secrets engine and Kubernetes authentication. Then install Jetstack's cert-manager, configure it to use Vault, and request a certificate.

## Prerequisites

The following are required prior to the setup

  * Running OpenShift cluster
  * Helm CLI installed (Refer [here][helm-ocp-install] to learn how to install Helm CLI on OpenShift 4)
  * Vault running on OpenShift (refer to previous blog entry [here][vault-ocp])
  
  [vault-ocp]: https://raylaijh.github.io/vault-ocp/
  
  
