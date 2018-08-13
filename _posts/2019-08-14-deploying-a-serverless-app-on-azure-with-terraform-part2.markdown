---
layout: post
title:  "Deploying a serverless app to azure with Terraform - Part 2"
date:   2018-08-14 14:15:05
categories: azure Terraform Serverless
background: "/assets/ip-restrictions.png"
draft: true
---



First thing you need is a VSTS project for the automation. Open up your vsts (someaccount.visualstudio.com) and click 'Create Project'. As you can see, I am already using the new VSTS layout, which I like a lot better compared to the old look and feel.

![resources]({{ "assets/2018-08/createproject.PNG" | absolute_url }}){: .img-fluid}

Fill out the name of your new VSTS project, choose whether you want your repo to be public or private, choose your source control system and the template you want to use. The default will often do. After filling this out click Create.

![resources]({{ "assets/2018-08/createproject2.PNG" | absolute_url }}){: .img-fluid}

Once your project got created, initialize your repo with a default readme file. I choose the one for VisualStudio, since I will be mainly doing .Net development for this demo. We will later add additional statements for specific Terraform files.

![resources]({{ "assets/2018-08/repoinit.PNG" | absolute_url }}){: .img-fluid}


[terraformstart]: https://www.terraform.io/
[vstsstart]: https://visualstudio.microsoft.com/team-services/
[armstart]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates
[IaC]: https://docs.microsoft.com/en-us/azure/devops/what-is-infrastructure-as-code
[mcwserverless]: https://github.com/Microsoft/MCW-Serverless-architecture 
[terraformdownload]: https://www.terraform.io/downloads.html
[terraforminstall]: https://www.terraform.io/intro/getting-started/install.html
[azurecli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest