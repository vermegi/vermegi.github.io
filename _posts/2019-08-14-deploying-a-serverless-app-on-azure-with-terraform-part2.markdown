---
layout: post
title:  "Deploying a serverless app to azure with Terraform - Part 2"
date:   2018-08-14 14:15:05
categories: azure Terraform Serverless
background: "/assets/ip-restrictions.png"
draft: true
---

In the previous post we build a [Terraform][terraformstart] template for deploying multiple resources to Azure. In this second part I will automate the deployment with a VSTS build and release pipeline. 

First thing you need is a VSTS project for the automation. Open up your vsts (someaccount.visualstudio.com) or create a new account and click 'Create Project'. As you can see, I am already using the new VSTS layout, which I like a lot better compared to the old look and feel.

![resources]({{ "assets/2018-08/createproject.PNG" | absolute_url }}){: .img-fluid}

Fill out the name of your new VSTS project, choose whether you want your repo to be public or private, choose your source control system and the template you want to use. The default will often do. After filling this out click Create.

![resources]({{ "assets/2018-08/createproject2.PNG" | absolute_url }}){: .img-fluid}

Once your project got created, initialize your repo with a default readme file. I choose the one for VisualStudio, since I will be mainly doing .Net development for this demo. We will later add additional statements for specific Terraform files, like the Terraform state and the Azure provider folder.

![resources]({{ "assets/2018-08/repoinit.PNG" | absolute_url }}){: .img-fluid}

There are a couple of ways of getting your /setup folder into this repo. Either you clone this repo locally, copy the /setup folder to it, alter the .gitignore file and do a commit and push. Or you perform a git init inside the folder where you created the /setup folder, you add the VSTS repo as a remote, perform a pull, alter the .gitignore file and commit and push. Whatever process you prefer, VSTS gives you a regular git repo, where you can add the /setup folder to. 

I added the following statements to the .gitignore file:

{% highlight shell_session %}
# Terraform
.terraform/
.terraform.tfstate.lock.info
*.log
*.tfstate*
{% endhighlight %}

So, we now will create a build and a release pipeline. The build pipeline for now will just fetch our sources, so they are available for our release pipeline. 

... todo ... build pipeline part

We can now create our release pipeline. Click on releases in the menu on the left and click the + sign and 'create a release pipeline'. 

![resources]({{ "assets/2018-08/release1.PNG" | absolute_url }}){: .img-fluid}

From the choices you now get, choose for 'empty process', since none of the predefined pipelines will actually be exactly what we are looking for.

![resources]({{ "assets/2018-08/release-emptyprocess.PNG" | absolute_url }}){: .img-fluid}

Give your pipeline a name, I called mine Dev, since more environments will follow.

![resources]({{ "assets/2018-08/release-dev.PNG" | absolute_url }}){: .img-fluid}

Once you renamed your pipeline, you can close this side-window and click the '1 phase, 0 tasks' link of the pipeline. 

![resources]({{ "assets/2018-08/release-dev2.PNG" | absolute_url }}){: .img-fluid}

Your release pipeline already has 1 phase. I renamed the default phase to 'deploy environment'. As for a build agent for this phase, we have a couple of options. The default agent type is VS2017, but there are also other options, like Ubuntu, windows container, ... If you click the 'Manage' link, this will bring you to the agent pools page, where you can manage your agent types. Here you can also see the capabilities of the already installed agents. 

![resources]({{ "assets/2018-08/agentpools.PNG" | absolute_url }}){: .img-fluid}

If you run through the capabilities of each agent, you will notice that none of the agents has Terraform installed. So either you create your own agent with Terraform on it, or you add an extra task to your pipeline for downloading terraform. Either process will work, each with their pro's and cons. I will opt for the first way of working, since it will allow me to reuse this agent in other projects and since I'll be running the agent on Azure Container Instance, it will be a cost effective way of running my agent. My custom build agent is based on [this one][dockeragent], but I altered the agent a bit. 

To create the custom build agent, make sure you have docker installed and it is running Linux containers. I started with a new folder with the following Dockerfile:

{% highlight Terraform %}
FROM microsoft/vsts-agent:latest

# Build-time metadata as defined at http://label-schema.org
LABEL org.label-schema.name="VSTS Agent with Terraform Tools" \
    org.label-schema.url="https://github.com/vermegi/" \
    org.label-schema.vcs-url="https://github.com/vermegi/vsts-agent-terraform" \
    org.label-schema.schema-version="1.0"
                
ENV TERRAFORM_VERSION 0.11.8

# Install Terraform
RUN echo "===> Installing Terraform ${TERRAFORM_VERSION}..." \
 && wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
 &&	unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
 && mv terraform /usr/local/bin/terraform \
 && rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip 
{% endhighlight %}

This starts from the microsoft/vsts-agent, which is an Ubuntu based build agent, which already has tools like Azure CLI, .Net Core SDK, ... installed. All it's capabilities can be found [here][vstsagent]. This base agent also takes in a couple of environment variables for your VSTS account, an authentication token, a VSTS agent pool, ... We will use this later to link this container to our release pipeline. 
Next in the Dockerfile, we indicate the Terraform version we want. At the time of this writing, the version just changed from 0.11.7 to 0.11.8. And next, download terraform, unzip it and make it available. 

Once you have the Dockerfile, you can build it, tag it and push it to your docker registry. In my case that's the gittetitter registry.

{% highlight Terraform %}
docker build -t vsts-agent-terraform .
docker login
docker tag vsts-agent-terraform gittetitter/vsts-agent-terraform
docker push gittetitter/vsts-agent-terraform
{% endhighlight %}

Next in VSTS we'll create an agent pool. This pool can be used to group all of your custom ACI agents together. To do this, on the main VSTS screen, go to 'Admin Settings' (bottom left of the screen) and then under 'Build and Release' you'll find 'Agent Pools'. Here you can click 'New pool'. Once we run the ACI VSTS agent on the fly, you will see it pop up in this agent pool. 

![resources]({{ "assets/2018-08/agentpools2.PNG" | absolute_url }}){: .img-fluid}

Give your pool a name and click OK.

![resources]({{ "assets/2018-08/ACIPool.PNG" | absolute_url }}){: .img-fluid}




[terraformstart]: https://www.terraform.io/
[vstsstart]: https://visualstudio.microsoft.com/team-services/
[armstart]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates
[IaC]: https://docs.microsoft.com/en-us/azure/devops/what-is-infrastructure-as-code
[mcwserverless]: https://github.com/Microsoft/MCW-Serverless-architecture 
[terraformdownload]: https://www.terraform.io/downloads.html
[terraforminstall]: https://www.terraform.io/intro/getting-started/install.html
[azurecli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
[dockeragent]: https://hub.docker.com/r/lenisha/vsts-agent-infrastructure/
[vstsagent]: https://hub.docker.com/r/microsoft/vsts-agent/