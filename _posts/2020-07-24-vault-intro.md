---
title: "Vault + Kubernetes"
tags: 
  - vault
  - kuberbetes
  - hashicorp
---

## Vault overview

<figure>
	<a href="https://www.vaultproject.io/docs/what-is-vault"><img src="https://github.com/raylaijh/raylaijh.github.io/blob/master/assets/images/vault_logo.png"></a>
	
</figure>

Vault is a open source tool developed by HashiCorp for securely accessing secrets. Secrets are essentially anything that you tight control over, such as API keys, passwords, or certificates. These information are usually sensitve information and most organzations have a designated team to manage them. 

A modern system typically requires access to a multitude of secrets: database credentials, API keys for external services, credentials for service-oriented architecture communication, etc. Understanding who is accessing what secrets is already very difficult and platform-specific, hence Vault is one of the solutions in the market which provides a one stop management tool for secrets management, irregardless of platforms.

One of the platforms which are commonly adopted by enterprises is Kubernetes. Kubernetes allows monolithic applications to be broken down to many microservices, which encourages better DevOps flow of application development. However, with this architecture, microservices often have to talk to each other - which means secrets have to be configured with service accounts across the entire Kubernetes cluster. The number of secrets increases exponentially as the cluster size grow. This can lead to a potential secrets sprawl if the secrets are not managed well. Vault could well be a solution to aid organizations who are having issues in managing these secrets.

## Vault architecture


