---
title: "Vault + Kubernetes"
tags: 
  - vault
  - kuberbetes
  - hashicorp
---

## Vault - Secrets as a service

<center>
<img align="center" src="/assets/images/vault_logo_1.png" alt="">
</center>


Vault is a open source tool developed by HashiCorp for securely accessing secrets. Secrets are essentially anything that you tight control over, such as API keys, passwords, or certificates. These information are usually sensitve information and most organzations have a designated team to manage them. 

A modern system typically requires access to a multitude of secrets: database credentials, API keys for external services, credentials for service-oriented architecture communication, etc. Understanding who is accessing what secrets is already very difficult and platform-specific, hence Vault is one of the solutions in the market which provides a one stop management tool for secrets management, irregardless of platforms. The image below shows how secrets are being consumed across a typical developer workflow, and how Vault can help to manage all these secrets.

<center>
<img align="center" src="/assets/images/vault_before_after.png" alt="">
</center>

Here are some features of Vault, quoted from Vault [documentation][vault-doc] :

[vault-doc]: https://www.vaultproject.io/docs/what-is-vault

* Secure Secret Storage: Arbitrary key/value secrets can be stored in Vault. Vault encrypts these secrets prior to writing them to persistent storage, so gaining access to the raw storage isn't enough to access your secrets. Vault can write to disk, Consul, and more.

* Dynamic Secrets: Vault can generate secrets on-demand for some systems, such as AWS or SQL databases. For example, when an application needs to access an S3 bucket, it asks Vault for credentials, and Vault will generate an AWS keypair with valid permissions on demand. After creating these dynamic secrets, Vault will also automatically revoke them after the lease is up.

* Data Encryption: Vault can encrypt and decrypt data without storing it. This allows security teams to define encryption parameters and developers to store encrypted data in a location such as SQL without having to design their own encryption methods.

* Leasing and Renewal: All secrets in Vault have a lease associated with them. At the end of the lease, Vault will automatically revoke that secret. Clients are able to renew leases via built-in renew APIs.

* Revocation: Vault has built-in support for secret revocation. Vault can revoke not only single secrets, but a tree of secrets, for example all secrets read by a specific user, or all secrets of a particular type. Revocation assists in key rolling as well as locking down systems in the case of an intrusion.



## Vault on Kubernetes

<center>
<img align="center" src="/assets/images/vault_kube.png" alt="">
</center>


One of the platforms which is commonly adopted by enterprises is Kubernetes. Kubernetes allows monolithic applications to be broken down to many microservices, which encourages better DevOps flow of application development. However, with this architecture, microservices often have to talk to each other - which means secrets have to be configured with service accounts across the entire Kubernetes cluster. The number of secrets increases exponentially as the cluster size grow. This can lead to a potential secrets sprawl if the secrets are not managed well. Vault presents itself as a solution as it is a centralized secrets manager, which can help to control the way services acquire the secrets they need in order to communicate with other services.

## Vault authentication workflow on Kubernetes

In a nutshell, how Kubernetes talk to Vault via the Kubernetes authentication method. Prior to that, Kubernetes administrator has to create a service account on their cluster as it will be used to talk to Vault. When agents/pods try to communicate with Vault, it will present the [Kubernetes JWT token][kube-token-doc] of this particular service account in order to authenticate to Vault API, before extracting any information from Vault. Post authentication, the service account can also be configured to bind to a certain policy predefined on Vault, so as to control which path the client can access in Vault. To understand specially on the detailed steps to perform the above, please refer to [HashiCorp Vault Learn page][vault-learn-page]. The diagram below illustrates the whole process described earlier.

<center>
<img align="center" src="/assets/images/Vault_arch.png" alt="">
</center>

[vault-learn-page]: https://learn.hashicorp.com/vault/identity-access-management/vault-agent-k8s
[kube-token-doc]: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens

## Vault Agent Side-car injection

We can leverage the Vault Agent Side-car Injector to pull secrets from Vault upon pod creation. This is useful for applications running in containers which depends on secrets to be available before it can begin to run. The Vault Side-car injector can pre-extract the required secrets and run as a init container and mount it as a shared volume, prior to running your applications. More examples can be found in the documentation below.

* Documentation: [https://www.vaultproject.io/docs/platform/k8s/injector/][link1]
* More Examples: [https://www.vaultproject.io/docs/platform/k8s/injector/examples/][link2]

[link1]: https://www.vaultproject.io/docs/platform/k8s/injector/
[link2]: https://www.vaultproject.io/docs/platform/k8s/injector/examples/

This diagram further illustrates the workflow with Vault Agent Side-car injection when running on Kubernetes
<center>
<img align="center" src="/assets/images/vault_arch_sidecar.png" alt="">
</center>

## References
* [HashiCorp Vault: Delivering Secrets with Kubernetes][link3]
* [Getting Started with HashiCorp Vault][link4]

[link3]: https://medium.com/hashicorp-engineering/hashicorp-vault-delivering-secrets-with-kubernetes-1b358c03b2a3
[link4]: https://medium.com/rafay-systems/getting-started-with-hashicorp-vault-ac9f67d7e519
