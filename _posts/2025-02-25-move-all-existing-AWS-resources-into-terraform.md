---
layout: post
title: How to import existing AWS resources into Terraform
subtitle: Creating all terraform files and structures for AWS. From manual create to IaC. 
excerpt_image: /assets/images/posts/move-resources-into-terraform/terraform-gif.gif
categories: Terraform
tags: [Terraform, AWS, Cloud]
top: 2
---

![banner](/assets/images/posts/move-resources-into-terraform/terraform-gif.gif)

### Setup your aws config file

```bash
~/.aws/config
```

### Install terraformer

You can find out about all the installation methods from [GitHub repo Terraformer](https://github.com/GoogleCloudPlatform/terraformer)

```bash
brew install terraformer
```

### Create version.tf file

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ue-west-1"
}
```

### Init terraform

And check that you providers is correct

```bash
terraform init
terraform providers
```

### Start import

Set specific resources or all resources in the account

```bash
terraformer import aws --resources=ec2_instance,ebs --regions=eu-west-1 # for some resources
terraformer import aws --resources="*" --regions=eu-west-1 # for all resources
```

At this stage you may encounter errors like this. This means that in your AWS account does not have these entities, or Terraformer cannot connect to it with account that was transferred to the local AWS config:

```bash
panic: runtime error: index out of range [0] with length 0
...
...
...
aws error initializing resources in service cloud9, err: operation error Cloud9: ListEnvironments, https response error StatusCode: 400
```

Then you need use `--excludes` parameter, passing resources that cause errors:

```bash
terraformer import aws --resources="*" --excludes="cloud9,identitystore" --regions=eu-west-1
```

Terraformer by default separates each resource into a file, which is put into a current service directory. The default path for resource files is `{generated}/{provider}/{service}/{resource}.tf` and can vary for each provider.

### Documentation (links) that can help

[GitHub repo Terraformer](https://github.com/GoogleCloudPlatform/terraformer)
