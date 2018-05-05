---
title: "Manage Multiple Environments With Terraform Workspaces"
date: 2018-05-05T16:09:09+02:00
tags: [
    "terraform",
    "workspaces",
]
categories: [
    "Terraform",
    "DevOps",
]
draft: true
---

The beginning
-----

When you want to manage (create, modify and remove) your infrastructure, getting started with Terraform is easy. Just create files ending with .tf containing the description of the resources you want to have.

For example, if we want to create a small infrastructure in AWS cloud provider:

* a S3 bucket (for terraform)
* a S3 bucket (for our website)
* a CloudFront distribution

![AWS infra](/images/aws_infra_basic.png)

We just need to create some .tf files like this:

```
aws_s3.tf
aws_cloudfront.tf
root.tf
variables.tf
output.tf
```

aws_s3.tf :
```
resource "aws_s3_bucket" "com-scraly-terraform" {
    bucket = "${var.aws_s3_bucket_terraform}"
    acl    = "private"

    versioning {
        enabled = true
    }

    tags {
        Tool    = "${var.tags-tool}"
        Contact = "${var.tags-contact}"
    }
}


```

aws_cloudfront.tf:
```

```

root.tf:
```
provider "aws" {
  region = "eu-central-1"
}
```

variables:
```
variable "default-aws-region" {
    default = "eu-central-1"
}

variable "tags-tool" {
    default = "Terraform"
}

variable "tags-contact" {
    default = "Aurelie Vache"
}

variable "aws_s3_bucket_terraform" {
    default = "com.scraly.terraform"
}
```

output.tf:
```

```

Now we need to initialize terraform (only the first time), generate a plan and apply it.

```
$ terraform init
...
```

Init command will initialize your working directory which contains .tf configuration files.

It's the first command to execute for a new configuration, or after doing a checkout of an existing configuration in a given git repository for example.

Init command will:

* download and install terraform providers/plugins
* initialize backend (if defined)
* download and install modules (if defined)

Since terraform v0.11+, instead of doing plan and then apply it, if you are in interractive use, now you just need to execute terraform apply. The command will create the plan, prompt the user, if Yes, apply the plan and make all the changes.
```
$ terraform apply
...
yes
...

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

cloudfront_distribution = 123456789
```

Our infra is created, great. But we work alone, in only one environment.

Tfstate should be store in remote
-----

In a company or in an OSS project, you don't work alone so you need to stop to store the tfstate locally and start to store it remotely, in the "cloud", to share it.

For information or reminder, a tf state is a snapshot of your infrastructure from when you last ran terraform apply. By default, the tfstate is stored locally in *terraform.tfstate* file.
But when we work in team, we must store the tfstate remotely:

![Terraform state](/images/tf_state.png)

We now create a backend resource in order to store the tfstate in a bucket s3 and encrypt it.

backend.tf:
```
# Backend configuration is loaded early so we can't use variables
terraform {
  backend "s3" {
    region = "eu-central-1"
    bucket = "com.scraly.terraform"
    key = "state.tfstate"
    encrypt = true
  }
}
```

When we created the s3 bucket resources in which we put our tfstate, we activate the versioning. It's not an error or a copy paste. It's recommended to enable versioning for state files. AWS S3 buckets has that capability, which you can leverage since terraform has a backend for it. Imagine, suddently, your state file got corrupted. Thanks to the state files versioning, you can use an older state and breathe ;-).

We created before our tfstate file so we need to convert local state to remote state (and store our state in the s3 bucket we created):

`$ terraform state push`

Workspaces
-----

Before terraform workspaces feature, in order to handle with multiple environments, the solution was to create one folder per environment/cloud provider account and put it .tf files. The solution was not convenient, easily maintanable with duplicate .tf files.

Since Terraform v0.10+, to manage multiple distinct sets of infrastructure resources/environments, we can use terraform workspace.

Create workspaces:
```
$ terraform workspace new dev
Created and switched to workspace 'dev'
$ terraform workspace new preprod
Created and switched to workspace 'preprod'
$ terraform workspace new prod
Created and switched to workspace 'prod'
```

Select the dev workspace:
`$ terraform workspaces select dev`

List workspaces:
```
$ terraform workspace list
default
* dev
preprod
prod
```

After an apply will be successfully executed, now, your tfstate will be stored in the good environment folder in the s3 bucket:

```
com.scraly.terraform
env:/
    dev/
       state.tfstate    
    preprod/
        state.tfstate    
    prod/
        state.tfstate
```

Perfect, because it's a best practice to separate tfstate per environment.
Like you saw, with terraform workspaces we manage easily several/multiple environments without headaches.
