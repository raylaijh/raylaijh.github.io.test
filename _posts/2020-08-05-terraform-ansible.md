---
title: "Terraform or Ansible?"
tags: 
  - ansible
  - redhat
  - hashicorp
  - terraform
---

### What is Infrastructure as a Code?

With the advent of technology in the 21st century, more and more organizations are beginning to embrace technology in their environment. This rising trend creates an ever increasing demand for new IT infrastructure, which also gives rise to many different options in the market. With so many options, many companies tend to end up adopting multiple cloud setups, running in their workloads. Managing all these different setups can be complex and often mundane, as system administrators had manually manage and configure the hardware and software that are required to run applications.

However, in the recent years, several solutions are available in the market in order to help organizations manage all these infrastructure in a automated and repeated manner. The cricital component of such solutions revolve around using "Infrastructure as a Code (IaC)". Here is how [Wikipedia][wikilink] defines IaC:

[wikilink]: https://en.wikipedia.org/wiki/Infrastructure_as_code

> "Infrastructure as code is the process of managing and provisioning computer data centers through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools" 

The idea of "Infrastructure as a Code" allows the definition of the desired infrastructure setup to be condensed in a code and then be consumed on demand. Defining the infrastructure in a machine readable format is the first step of automation. Once that is done, we can store these code somewhere, and use to automate the provisioning or maintaince of the infrastructure, in the way which we want it to be. In this way, we can achieve speed in deployment, consistency in rolling out and full tracibility of configuration made to the infrastructure. All these benefits also mean saving time and money. 


## IaC options 

There are currently many options in the market which allows users to reap on the benefits of IaC. Ideally, the tool used should preferrable be platform agnostic, so that the same tool can be used to manage multiple cloud providers. In this post, I will share on mainly the two tools which I have experienced using, which are Terraform and Ansible. Both are Open Source options and you can check their upstream GitHub for more information. Also, you can checkout both HashiCorp and Red Hat official documentation as well.


Ansible GitHub: [https://github.com/ansible/ansible](https://github.com/ansible/ansible)
Official Ansible documentation: [https://docs.ansible.com/](https://docs.ansible.com/)
Terraform GitHub: [https://github.com/hashicorp/terraform](https://github.com/hashicorp/terraform)
Official Terraform intro: [https://www.terraform.io/intro/index.html](https://www.terraform.io/intro/index.html)


> Disclaimer: The following information are based on my personal experience in using them. There might be other features which I have not touched on which could make one better than the other in some cases. The following content are meant to illustrate a high level comparison between the two tools.

### Ease of Coding

As a tools which levearage on Infrastructure as a Code, ease of developing these codes is vital for users. In Terraform, the code is written in HCL (HashiCorp Configuration Language), while in Ansible, it is YAML (YAML Ain't Markup Language). Below are some snippets of the code used to provision a simple instance on AWS.


```css
# example main.tf in Terraform
provider "aws" {
  profile = "default"
  region  = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-2757f631"
  instance_type = "t2.micro"
}
```

```yaml
# example Ansible playbook in Ansible
- ec2:
    key_name: mykey
    instance_type: t2.micro
    image: ami-123456
    wait: yes
    group: webserver
    count: 3
    vpc_subnet_id: subnet-29e63245
    assign_public_ip: yes
```

Both codes are human readable and easy for administrators who may not have coding background to develop. As long as you have defined the "key words", which we refer to as `provider` and `resource` in Terraform, `modules` in Ansible, both Terraform and Ansible will be able to recognize them and perform the intended tasks. The entire list of usable "key words" are documented the official website of both tools. 

Terraform: [https://www.terraform.io/docs/configuration/index.html](https://www.terraform.io/docs/configuration/index.html)
Ansible: [https://docs.ansible.com/ansible/latest/modules/modules_by_category.html](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

### Readily available resources

From a user's perspective, the ease of developing these codes will help if there is a library of "templates" or "ready made code" available for reference, or even serve as a self service for users who may not be administrating the infrastructure. In Terraform, the piece of code is called `module` (not to be confused with the Ansible module), while in Ansible, it is refered as `playbook`. The snippet which I have shared earlier can be considered as a `module` or `playbook` respectively.

Both tools do boast the availability of a library of codes available for users to use. 

For Terraform, `modules` for common use cases (most of it are for cloud providers) are stored in [Terraform Registry] (https://registry.terraform.io/browse/providers). There are some commnuity ones as well. These `modules` are verified and are official, supported by HashiCorp.

For Ansible, the `playbooks` repository can be retrieved from [Ansible Galaxy](galaxy.ansible.com). The `playbooks` here are contributed mainly by the community.   

### Stateful vs Stateless

Terraform in this case is stateful, as it keeps track of the changes applied before and after in the working directory in the form of a `terraform.tfstate` file. 

Ansible on the other hand is idempotent

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

