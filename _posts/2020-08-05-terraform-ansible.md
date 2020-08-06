---
title: "Infrastructure as a Code: Terraform or Ansible?"
tags: 
  - ansible
  - redhat
  - hashicorp
  - terraform
---

This post serves as introductory post to Infrastructure as a Code and its benefits. In this post, I will share some high level benefits of using tools like Terraform and Ansible to leverage on the benefits of Infrastructure as a Code. The following content are not meant to be too technical, and are based on my personal experiences after these two tools. 

### What is Infrastructure as a Code?

With the advent of technology in the 21st century, more and more organizations are beginning to embrace technology in their environment. This rising trend creates an ever increasing demand for new IT infrastructure, which also gives rise to many different options in the market. With so many options, many companies tend to end up adopting multiple cloud setups to run their workloads. Managing all these different setups can be complex and often repetitive, as system administrators had manually manage and configure the hardware and software that are required to run applications. These tasks usually entail creating virtual machines/instances, installing OS, configuring storage and network, etc. 

The way of application development has become more robust as well. Adoption of modern infrastructures such as Kubernetes facilliates the use of Agile methdology as compared to the traditional Waterfall. This means that testing get vigorous in different stages of app development and, means more development environment needs to be created and destroyed on demand for testing purposes. This pushes a futher need for a way to automate the provisioning of these resources.

In the recent years, several solutions are available in the market in order to help organizations manage all these infrastructure in a automated and repeated manner. The cricital component of such solutions revolve around using "Infrastructure as a Code (IaC)". Here is how [Wikipedia][wikilink] defines IaC:

[wikilink]: https://en.wikipedia.org/wiki/Infrastructure_as_code

> "Infrastructure as code is the process of managing and provisioning computer data centers through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools" 

The idea of "Infrastructure as a Code" allows the definition of the desired infrastructure setup to be condensed in a code and then be consumed on demand. Defining the infrastructure in a machine readable format is the first step of automation. Once that is done, we can store these code somewhere, and use to automate the provisioning or maintaince of the infrastructure, in the way which we want it to be. In this way, we can achieve speed in deployment, consistency in rolling out and full tracibility of configurations made to the infrastructure portion. All these benefits also mean saving time and money. 


## IaC options 

There are currently many options in the market which allows users to reap on the benefits of IaC. Ideally, the tool used should preferrably be platform agnostic, so that the same tool can be used to manage multiple cloud providers. In this post, I will share on mainly the two tools which I have experienced using, which are Terraform and Ansible. Both are Open Source options and you can check their upstream GitHub for more information. Also, you can checkout both HashiCorp and Red Hat official documentation as well.


* Ansible GitHub: [https://github.com/ansible/ansible](https://github.com/ansible/ansible)
* Official Ansible documentation: [https://docs.ansible.com/](https://docs.ansible.com/)
* Terraform GitHub: [https://github.com/hashicorp/terraform](https://github.com/hashicorp/terraform)
* Official Terraform intro: [https://www.terraform.io/intro/index.html](https://www.terraform.io/intro/index.html)


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

* Terraform: [https://www.terraform.io/docs/configuration/index.html](https://www.terraform.io/docs/configuration/index.html)
* Ansible: [https://docs.ansible.com/ansible/latest/modules/modules_by_category.html](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

### Readily available resources

From a user's perspective, the ease of developing these codes will help if there is a library of "templates" or "ready made code" available for reference, or even serve as a self service for users who may not be administrating the infrastructure layer. In Terraform, the piece of code is called `module` (not to be confused with the Ansible module), while in Ansible, it is refered as `playbook`. The snippet which I have shared earlier can be considered as a `module` or `playbook` respectively.

Both tools do boast the availability of a library of codes available for users to use. 

For Terraform, `modules` for common use cases (most of it are for cloud providers) are stored in [Terraform Registry] (https://registry.terraform.io/browse/providers). There are some commnuity ones as well. These `modules` are verified and are official, supported by HashiCorp.

For Ansible, the `playbooks` repository can be retrieved from [Ansible Galaxy](galaxy.ansible.com). The `playbooks` here are contributed mainly by the community.   

### Stateful vs Stateless

Terraform in this case is stateful, as it keeps track of the changes applied before and after in the working directory in the form of a `terraform.tfstate` file. This helps users to visualize potential changes prior to running the `module`, often with `terraform plan`, and reverting back to previous state if required - `terraform destroy`.

Ansible on the other hand is stateless, meaning to say, it does not know the current state of your infrastructure. Whenever the `playbook` runs, it does the check, and changes it to the desired state if required. 

### UI version


Both Terraform and Ansible comes with the UI version, in the form of Terraform Enterprise and Ansible Tower. Both offers REST API endpoint, which allows users to interact via API calls. 

<center>
<img align="center" src="/assets/images/terraform_cloud.png" alt=""> 
  <figcaption>How Terraform Cloud works</figcaption>
</center>

Terraform Enterprise builds primarily on Terraform Cloud, which offers remote execution and state management (imagine the `terraform.tfstate` file is no longer required, as it is taken care on Terraform Cloud), which faciliates a collaboration of `module` developement via Version Control System (VCS), like Github. The purpose is to ensure that Terraform runs in a consistent and reliable environment, and the only source of truth will be whatever that is on the VCS repository.

<center>
<img align="center" src="/assets/images/ansible_tower_sample.png" alt=""> 
  <figcaption>Sample Ansible Tower reference architecture</figcaption>
</center>


Ansible Tower is meant better organization control with RBAC features. In addition, it also supports clustering, and HA failover. This allows users to deploy multiple Ansible Tower instances across environment, and recover from disasters in case of a failure. 

> There are many other features with regards to the enterprise versions of both tools which I did not include, but both versions are definitely targeted at different use cases.

You can refer to the the official documentation for further features of both tools:

* Terraform Cloud: https://www.terraform.io/docs/cloud/index.html
* Ansible Tower: https://docs.ansible.com/ansible/latest/reference_appendices/tower.html
* Ansible Tower HA and Disaster Recovery blog: https://servicesblog.redhat.com/2019/04/08/ansible-tower-high-availability-and-disaster-recovery/

## Summary and Conclusion

Overall, I do feel that both tools are good tools to use. They are easy to install and code. Both tools are agentless as well. In my opinion, Terraform is a better tool to create infrastructure from scratch, especially on cloud providers, given the readily available `modules`, which are on the official supported `Terraform Modules Registry`. Ansible will be more suited for configuration for post infrastructure provisioning, as the basis of its operations lies in the `inventory`, which defines the target machines to run the `playbooks` on. This design implicitly implies that the machines/instances have to be available beforehand (although we can also execute on localhost or bastion host first, at pre provisioning phase, which works as well). 

While Terraform keeps track of the state with `terraform.tfstate`, it allows user to easily know what is the current state. This comes as a double edge sword as well. This also means that the `terraform.tfstate` file has to be managed by the users. You will not have that problem with Ansible, as Ansible only goes one way - to reach the desired state defined in the `playbook`.

The UI versions for both tools have more enterprise features for each of them but they are meant for different use cases, so it is up to the users to decide which is more applicable for them. 

<center>
<img align="center" src="/assets/images/ansible_vs_terraform.png" alt=""> 
  <figcaption>Quick comparison between Terraform and Ansible</figcaption>
</center>

The above table shows a summary of what I have discussed. There are definitely more in depth features of each product that I did not cover in this post, but both are definitely great tools to use for Infrastructure as a Code in my opinion.
