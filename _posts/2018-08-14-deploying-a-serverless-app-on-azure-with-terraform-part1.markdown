---
layout: post
title:  "Deploying a serverless app to azure with Terraform - Part 1"
date:   2018-08-14 14:15:05
categories: azure Terraform Serverless
background: "/assets/2018-08/serverlessapp.PNG"
---
Recently I was going through one of our cloud workshops and got the idea of redoing the infrastructure setup for this with [Terraform][terraformstart] and with a fully automated build and release pipeline on [VSTS][vstsstart]. Terraform will let you, just like [ARM templating][armstart] will, setup your infractructure in a fully automated fashion. It will give you the same ability to write [infrastructure as code][IaC]. With the added value that Terraform can be used to not only automate Azure environments, but also those for other cloud providers. Also, from this first experience, it feels less tedious than doing the infrastructure configuration in Json. 

In this post I will walk you through the steps needed for setting this up. The cloud workshop I will be using as a basis for this is the one on [serverless architecture][mcwserverless]. It combines a couple of Azure services like Azure Functions, CosmosDb, Azure Event Grid, Logic Apps, ... just to name a few. I will not perform the entire setup in this blog post, but it will be enough to get you started on Terraform, Azure and VSTS.

In part 1 we will start building our Terraform script. In part 2 we will automate this script in VSTS. 

Before we start writing our Terraform scripts, lets first download Terraform from [here][terraformdownload]. Install instructions can be found [here][terraforminstall]. Basically for windows, you unzip the download and alter your PATH variable so it can find the terraform.exe.

With this being done, let's write our first Terraform file. I created a /setup directory in the project directory where I want to work in. In the /setup directory I created a setup.tf file. First thing I want to do is create a Azure Resource Manager resource group. For this you need to indicate you want to use the azure provider and next indicate you want to create a resourcegroup: 

{% highlight Terraform %}
#configure the azure provider
provider "azurerm" {
  
}

#create a resource group
resource "azurerm_resource_group" "rg" {
    name = "mcw-serverless-architecture"
    location = "West Europe"
}
{% endhighlight %}

You will notice we don't give any credentials in the azurerm provider of the setup file. This is because when running locally, Terraform will use the credentials of my command line. We will login to Azure in one of the next steps. Also the "rg" name used in the script is the name by which you can later in the Terraform script refer to this resource group. "mcw-serverless-architecture" will be the resourcegroup name as you will see it in Azure. 

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

Make sure to check whether the correct subscription is indicated as being your default subscription, because in the next step we will actually start creating resources. If this is not the case, you can change this with:

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
      name:     "mcw-serverless-architecture"
      tags.%:   <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
  
azurerm_resource_group.mcw-serverless-architecture: Creating...
  location: "" => "westeurope"
  name:     "" => "mcw-serverless-architecture"
  tags.%:   "" => "<computed>"
azurerm_resource_group.mcw-serverless-architecture: Creation complete after 0s (ID: /subscriptions/xxxxxxxx-xxxx-xxxx-xx
xx-xxxxxxxxxxxx/resourceGroups/mcw-serverless-architecture)
{% endhighlight %}

Terraform will perform a check of which steps it needs to execute to create the resources of your .tf files. It will also ask you for confirmation whether it can go on in creating them.

With this being succesful, you can now check out the newly created resource group in the Azure portal. 

![resources]({{ "assets/2018-08/resourcegroup.PNG" | absolute_url }}){: .img-fluid}

You will also notice that in the folder where you created the setup.tf file a terraform.tfstate file got added. This file will be used by Terraform to keep track of the resources it created. This means that if you now reissue the terraform apply command, nothing will really happen, because all resources were already created. Also, this behaviour is different from ARM templating, where the actual Azure setup is used to make a comparison between what you already have and what you want to achieve. 

Next step will be to add extra resources to our resource group. Let's start by creating a storage account. Add the following to the bottom of the setup.tf file:

{% highlight Terraform %}
resource "azurerm_storage_account" "sa" {
  name                     = "mcwserverlessarchsa"
  resource_group_name      = "${azurerm_resource_group.rg.name}"
  location                 = "West Europe"
  account_tier             = "Standard"
  account_replication_type = "RAGRS"
  account_kind             = "BlobStorage"
  access_tier              = "Hot"
}
{% endhighlight %}

This let's us create a storage account in the resourcegroup we just created. We reference the resourcegroup with ${azurerm_resource_group.rg.name}. Using the [documentation on terraform Azure storage][storagedocs] it is quite easy to build up the configuration based on what you need. Issue another terraform apply to get this resource created. 

Next it's really easy to add the storage containers images and export to this storage account. 

{% highlight Terraform %}
resource "azurerm_storage_container" "imagescontainer" {
  name                  = "images"
  resource_group_name   = "${azurerm_resource_group.rg.name}"
  storage_account_name  = "${azurerm_storage_account.sa.name}"
  container_access_type = "private"
}

resource "azurerm_storage_container" "exportcontainer" {
  name                  = "export"
  resource_group_name   = "${azurerm_resource_group.rg.name}"
  storage_account_name  = "${azurerm_storage_account.sa.name}"
  container_access_type = "private"
}
{% endhighlight %}

A terraform apply again does the trick. And with this, I must say, the script/config format for Terraform is really concise and contains less clutter than ARM templates do. I am already really liking this format and the quickness of using it. 

For the next part, I did have to alter my config a bit. I filed a [github issue][githubissue] for the problem I encountered (seems to have something to do with the casing of the function app name).  

{% highlight Terraform %}
resource "azurerm_storage_account" "functionsa" {
  name                     = "mcwfunctionsa"
  resource_group_name      = "${azurerm_resource_group.rg.name}"
  location                 = "${azurerm_resource_group.rg.location}"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_app_service_plan" "asp" {
  name                = "azure-functions-mcw-service-plan"
  location            = "${azurerm_resource_group.rg.location}"
  resource_group_name = "${azurerm_resource_group.rg.name}"
  kind                = "FunctionApp"

  sku {
    tier = "Dynamic"
    size = "Y1"
  }
}

resource "azurerm_function_app" "function" {
  name                      = "mcw-tollboothevents-func"
  location                  = "${azurerm_resource_group.rg.location}"
  resource_group_name       = "${azurerm_resource_group.rg.name}"
  app_service_plan_id       = "${azurerm_app_service_plan.asp.id}"
  storage_connection_string = "${azurerm_storage_account.functionsa.primary_connection_string}"
}
{% endhighlight %}

I did reuse the same storage account and app service plan for the second function. If afterwards it turns out this is a bad choice, I can easily change that in the config. 

Next up: the Event Grid topic. 

{% highlight Terraform %}
resource "azurerm_eventgrid_topic" "egtopic" {
  name                = "mcwTollboothTopic"
  location            = "${azurerm_resource_group.rg.location}"
  resource_group_name = "${azurerm_resource_group.rg.name}"
}
{% endhighlight %}

And the Azure CosmosDb account:

{% highlight Terraform %}
resource "azurerm_cosmosdb_account" "cosmosdb" {
    name                = "mcwTollboothDb"
    location            = "${azurerm_resource_group.rg.location}"
    resource_group_name = "${azurerm_resource_group.rg.name}"
    offer_type          = "Standard"
    kind                = "GlobalDocumentDB"

    enable_automatic_failover = true

    consistency_policy {
        consistency_level       = "BoundedStaleness"
        max_interval_in_seconds = 10
        max_staleness_prefix    = 200
    }

    geo_location {
        location          = "North Europe"
        failover_priority = 1
    }

    geo_location {
        location          = "${azurerm_resource_group.rg.location}"
        failover_priority = 0
    }
}
{% endhighlight %}

As a next step, we should be able to add the necessary collections to our CosmosDb, but at the moment of this writing, creating CosmosDb collections is still an [open feature request][cosmosdbfeature] in the terraform github repo. Also, this is not possible through an ARM template either. Check out [this feature request][cosmosdfeature2] where the product team discontinues this request. So for now, I will create these through azure CLI commands.

{% highlight Console %}
az cosmosdb database create --db-name LicensePlates --name mcw-tollbooth-db --resource-group-name mcw-serverless-architecture
az cosmosdb collection create --collection-name Processed --db-name LicensePlates --resource-group-name "mcw-serverless-architecture" --throughput 5000 --name mcw-tollbooth-db
az cosmosdb collection create --collection-name NeedsManualReview --db-name LicensePlates --resource-group-name "mcw-serverless-architecture" --throughput 5000 --name mcw-tollbooth-db
{% endhighlight %}

And finally the Computer Vision API, this one as well is not supported through terraform yet, although this can be created through an ARM template, which we can embed in a terraform template deployment resource. This does complicate the setup and I would only suggest to use this for resources which are indeed unsupported by Terraform at the time. Also, bare in mind that resources created this way cannot be tracked by Terraform. 

{% highlight Terraform %}
resource "azurerm_template_deployment" "customvision" {
  name                = "mcwtollboothvision"
  resource_group_name = "${azurerm_resource_group.rg.name}"

  template_body = <<DEPLOY
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "accountName": {
      "defaultValue": "mcwtollboothvision",
      "metadata": {
        "description": "Display name of Computer Vision API account"
      },
      "type": "string"
    },
    "SKU": {
      "type": "string",
      "metadata": {
        "description": "SKU for Computer Vision API"
      },
      "defaultValue": "F0",
      "allowedValues": [
        "F0",
        "S1"
      ]
    },
    "cognitiveServicesLocation": {
      "metadata": {
        "description": "The location for the Computer Vision API"
      },
      "type": "string",
      "minLength": 1,
      "allowedValues": [
        "westeurope",
        "eastus2",
        "southeastasia",
        "centralus",
        "westus"
      ],
      "defaultValue": "westeurope"
    }
  },
  "variables": {
    "cognitiveservicesid": "[concat(resourceGroup().id,'/providers/','Microsoft.CognitiveServices/accounts/', parameters('accountName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.CognitiveServices/accounts",
      "sku": {
        "name": "[parameters('SKU')]"
      },
      "kind": "ComputerVision",
      "name": "[parameters('accountName')]",
      "apiVersion": "2016-02-01-preview",
      "location": "[parameters('cognitiveServicesLocation')]",
      "properties": {},
      "dependsOn": []
    }
  ],
  "outputs": {
    "cognitivekeys": {
      "type": "object",
      "value": "[listKeys(variables('cognitiveservicesid'),'2016-02-01-preview')]"
    },
    "cognitivekey1": {
      "type": "string",
      "value": "[listKeys(variables('cognitiveservicesid'),'2016-02-01-preview').key1]"
    },
    "cognitivekey2": {
      "type": "string",
      "value": "[listKeys(variables('cognitiveservicesid'),'2016-02-01-preview').key2]"
    },
    "endpoint": {
      "type": "string",
      "value": "[reference(variables('cognitiveservicesid'),'2016-02-01-preview').endpoint]"
    }
  }
}
DEPLOY

  # these key-value pairs are passed into the ARM Template's `parameters` block
  parameters {
    "accountName" = "mcwtollboothvision"
    "SKU" = "S1"
  }

  deployment_mode = "Incremental"
}

output "cognitivekey1" {
  value = "${azurerm_template_deployment.customvision.outputs["cognitivekey1"]}"
}
{% endhighlight %}

Only thing I needed to do here is drop the [quickstart template for Computer Vision][template] in the Terraform resource deployment and alter some default settings. You can also send values to the different parameters if needed, or do something with the output of the ARM template. So although we bumped into an issue, this is still quite flexible. 

That concludes our basic Terraform template fo the infrastructure setup. In a next part we will use VSTS to deploy our infractructure resources instead of doing it from my dev machine. And except for a couple of not yet supported features it was a very nice experience using Terraform for this process. 

[terraformstart]: https://www.terraform.io/
[vstsstart]: https://visualstudio.microsoft.com/team-services/
[armstart]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates
[IaC]: https://docs.microsoft.com/en-us/azure/devops/what-is-infrastructure-as-code
[mcwserverless]: https://github.com/Microsoft/MCW-Serverless-architecture 
[terraformdownload]: https://www.terraform.io/downloads.html
[terraforminstall]: https://www.terraform.io/intro/getting-started/install.html
[azurecli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
[storagedocs]: https://www.terraform.io/docs/providers/azurerm/r/storage_account.html
[githubissue]: https://github.com/hashicorp/terraform/issues/18669 
[cosmosdbfeature]: https://github.com/terraform-providers/terraform-provider-azurerm/issues/548
[cosmosdbfeature2]: https://feedback.azure.com/forums/263030-azure-cosmos-db/suggestions/13427997-create-db-collections-using-arm-template
[template]: https://github.com/Azure/azure-quickstart-templates/tree/master/101-cognitive-services-Computer-vision-API