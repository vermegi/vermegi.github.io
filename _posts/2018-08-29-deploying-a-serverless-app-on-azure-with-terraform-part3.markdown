---
layout: post
title:  "Deploying a serverless app to azure with Terraform - Part 3"
date:   2018-08-29 14:15:05
categories: azure Terraform Serverless
background: "/assets/2018-08/dev.png"
---

In the previous posts we build a [Terraform][terraformstart] template for deploying multiple resources to Azure and we created a build agent in a container which we can run on the fly. In this third part I will further automate the deployment by adding the release tasks for creating the solution architecture in Azure with Terraform.

What we need for this is first a task which can run our Terraform tasks. For this I will simply use a Bash Script release task and create a bash script, run.sh, which will contain our different terraform stept (init and apply). 

![resources]({{ "assets/2018-08/bash2.PNG" | absolute_url }}){: .img-fluid}

{% highlight shell_session %}
#!/bin/bash -e

echo "************ Initialize provider"
terraform init

echo "************ Run apply
terraform apply -auto-approve 
{% endhighlight %}

The bash task is configured to run this run.sh script and to use the setup directory as a working directory. 

![resources]({{ "assets/2018-08/bash3.PNG" | absolute_url }}){: .img-fluid}

To be able to run this bash script, we need to be logged into Azure. I tried using a service principal from VSTS for this, but it is quite impossible to use this in tasks which are not made to work with a service principal. Another way of doing this, is by configuring your azurerm provider in the Terraform script. So something like this:

{% highlight terraform %}
#configure the azure provider
provider "azurerm" {
  subscription_id = "xxxx"
  client_id       = "xxxx"
  client_secret   = "xxxx"
  tenant_id       = "xxxx"
}
{% endhighlight %}

Now, all these values are sensitive, so you don't want them to be checked into your source repo together with the rest of your source code. So what we'll do, is use variables for them. 

{% highlight terraform %}
#configure the azure provider
provider "azurerm" {
  subscription_id = "${var.subscription_id}"
  client_id       = "${var.client_id}"
  client_secret   = "${var.client_secret}"
  tenant_id       = "${var.tenant_id}"
}
{% endhighlight %}

We will define these variables in a new file in the setup directory release.tfvars.

{% highlight terraform %}
subscription_id = "__ARM_SUBSCRIPTION_ID__"
client_id = "__ARM_CLIENT_ID__"
client_secret = "__ARM_CLIENT_SECRET__"
tenant_id = "__ARM_TENANT_ID__"
env = "__ENV__"
{% endhighlight %}

This still doesn't contain the actual value for the subscriptionId and other values we need, but we'll define these as variables in our VSTS release and then do a find and replace with a 'Replace Tokens' task, which you can find in the VSTS marketplace. 

![resources]({{ "assets/2018-08/vars.PNG" | absolute_url }}){: .img-fluid}

![resources]({{ "assets/2018-08/replace.PNG" | absolute_url }}){: .img-fluid}

You will notice that I also defined a env and storageaccount variable. The env variable I will use to differentiate my different environments. I will also use this in the terraform script to get unique naming for all resources. For example:

{% highlight terraform %}
#create a resource group
resource "azurerm_resource_group" "rg" {
    name = "${var.env}-mcw-serverless-architecture"
    location = "West Europe"
}

resource "azurerm_storage_account" "sa" {
  name                     = "${var.env}mcwserverlessarchsa"
  resource_group_name      = "${azurerm_resource_group.rg.name}"
  location                 = "West Europe"
  account_tier             = "Standard"
  account_replication_type = "RAGRS"
  account_kind             = "BlobStorage"
  access_tier              = "Hot"
}
{% endhighlight %}

The storageaccount variable will be used by the Terraform state provider. When Terraform creates an environment, it needs to know what already got deployed and what the differences are with a next deployment. For this it keeps track of state. In the first post in this series we kept state in a local file called terraform.tfstate. On our build agent we can no longer use a local file since the build agent will be recreated with every run. Instead we will keep state in a shared storage account on Azure. For this I created a storage account, put the name of the storage account in the STORAGE_ACCOUNT variable in the VSTS release pipeline and the primary key of the storage account in the ARM_ACCESS_KEY variable. 

In the setup directory I added an additional file state.tf which tells terraform to keep its state in Azure. 

{% highlight terraform %}
terraform {
  backend "azurerm" {

  }
}
{% endhighlight %}

The actual values for the state are in the backend.tfvars file also in the setup directory.

{% highlight terraform %}
storage_account_name = "__STORAGE_ACCOUNT__"
container_name = "terraform-state"
key = "mcw-serverless-arch.terraform.tfstate"
access_key = "__ARM_ACCESS_KEY__"
{% endhighlight %}

The values in this file will also be picked up by the 'Replace Tokens' release task. 

To run our terraform deploy, the run.sh file needs to be altered just a bit, so it will take the .tfvars files as input. 

{% highlight terraform %}
#!/bin/bash -e

echo "************ Initialize provider"
terraform init -backend-config=./backend.tfvars

echo "************ Run apply"
terraform apply --var-file=./release.tfvars -auto-approve 
{% endhighlight %}

And that's it, with these steps in place, you can commit and ^push all files to the VSTS git repo. This will trigger a build and a release of the infrastructure. 

And before you think, well this was an easy process, this ilustrates how many builds and release I needed to get all the pieces of the puzzle together :D 

![resources]({{ "assets/2018-08/releases.PNG" | absolute_url }}){: .img-fluid}

[terraformstart]: https://www.terraform.io/
