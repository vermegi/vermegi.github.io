---
layout: post
title:  "Deploying a serverless app to azure with Terraform"
date:   2018-08-10 14:15:05
categories: azure Terraform Serverless
background: "/assets/ip-restrictions.png"
draft: true
---
Recently I was going through our serverless cloud workshop and got the idea of redoing the infrastructure setup for this with Terraform and with a fully automated build and release pipeline on VSTS. In this post I will walk you through the steps needed for setting this up.

Before we write our Terraform script, lets first download Terraform from [here][terraformdownload]. Install instructions can be found [here][terraforminstall]. Basically for windows, you unzip the download and alter your PATH variable to find the terraform.exe.

With this being done, let's write our first Terraform file. I created a /setup directory in the project directory where I want to work in. In the /setup directory I created a setup.tf file. First thing I want to do is create a Azure Resource Manager resource group: 

{% highlight Terraform %}
#configure the azure provider
provider "azurerm" {
  
}

#create a resource group
resource "azurerm_resource_group" "mcw-serverless-architecture" {
    name = "dev"
    location = "West Europe"
}
{% endhighlight %}

Once you have this file, from the command line move to your /setup folder and issue the following command:

{% highlight Console %}
> terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "azurerm" (1.12.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.azurerm: version = "~> 1.12"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
{% endhighlight %}

This will initialize your /setup directory with the chosen Azure provider. 

Next step will be to execute our Terraform script against our Azure subscription. For this, you will first need to login. I will do this using the [Azure CLI commands][azurecli].

{% highlight Console %}
> az login
Note, we have launched a browser for you to login. For old experience with device code, use "az login --use-device-code"
You have logged in. Now let us find all subscriptions you have access to...
{% endhighlight %}

Make sure to check whether the correct subscription is inidcated as being your default subscription. If this is not the case, you can change this with:

{% highlight Console %}
> az account set 'subscription-id or name'
{% endhighlight %}

Once you have the right subscription set as the default subscription, you can perform the terraform apply command.

{% highlight Console %}
> terraform apply

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + azurerm_resource_group.mcw-serverless-architecture
      id:       <computed>
      location: "westeurope"
      name:     "dev"
      tags.%:   <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
  
azurerm_resource_group.mcw-serverless-architecture: Creating...
  location: "" => "westeurope"
  name:     "" => "dev"
  tags.%:   "" => "<computed>"
azurerm_resource_group.mcw-serverless-architecture: Creation complete after 0s (ID: /subscriptions/xxxxxxxx-xxxx-xxxx-xx
xx-xxxxxxxxxxxx/resourceGroups/dev)
{% endhighlight %}

With this being succesful, you can now check out the newly created resource group in the Azure portal. 

![resources]({{ "assets/2018-08/resourcegroup.PNG" | absolute_url }}){: .img-fluid}

First thing you need is a VSTS project for the automation. Open up your vsts (someaccount.visualstudio.com) and click 'Create Project'. As you can see, I am already using the new VSTS layout, which I like a lot better compared to the old look and feel.

![resources]({{ "assets/2018-08/createproject.PNG" | absolute_url }}){: .img-fluid}

Fill out the name of your new VSTS project, choose whether you want your repo to be public or private, choose your source control system and the template you want to use. The default will often do. After filling this out click Create.

![resources]({{ "assets/2018-08/createproject2.PNG" | absolute_url }}){: .img-fluid}

Once your project got created, initialize your repo with a default readme file. I choose the one for VisualStudio, since I will be mainly doing .Net development for this demo. We will later add additional statements for specific Terraform files.

![resources]({{ "assets/2018-08/repoinit.PNG" | absolute_url }}){: .img-fluid}



[terraformdownload]: https://www.terraform.io/downloads.html
[terraforminstall]: https://www.terraform.io/intro/getting-started/install.html
[azurecli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest