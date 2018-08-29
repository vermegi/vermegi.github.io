---
layout: post
title:  "Deploying a serverless app to azure with Terraform - addendum"
date:   2018-08-29 14:15:05
categories: azure Terraform Serverless
background: "/assets/2018-08/release4.PNG"
---

In the previous 3 posts we build a [Terraform][terraformstart] template for deploying multiple resources to Azure and we created a build agent in a container which we can run on the fly and we fully automated the deployment through VSTS. In this small addendum, I just want to add a little side note of an alteration I made to the process of the docker build agent. 

Originally I started of with the debian:stretch base image, but because I needed some additional tools when trying out service principals for the third part of the posts, I switched the base image to microsoft/vsts-agent:ubuntu-14.04. This is the base image which is also used by the standard vsts-agent which has all the tools like Java, azure cli, cmake, ... The standard vsts-agent however was way too big and bloated for the job I wanted to get done (more than 8Gb). However, if you start from the same docker file and strip the things you don't need, you still end up with a lean docker image, which will start up quite quickly. 

So at the moment, this is my docker file:

{% highlight docker %}
FROM microsoft/vsts-agent:ubuntu-14.04

# To make it easier for build and release pipelines to run apt-get,
# configure apt to not require confirmation (assume the -y argument by default)
ENV DEBIAN_FRONTEND=noninteractive
RUN echo "APT::Get::Assume-Yes \"true\";" > /etc/apt/apt.conf.d/90assumeyes

# Install basic command-line utilities
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    curl \
    dnsutils \
    file \
    ftp \
    iproute2 \
    iputils-ping \
    locales \
    openssh-client \
    rsync\
    shellcheck \
    sudo \
    telnet \
    time \
    unzip \
    wget \
    zip \
    tzdata \
 && rm -rf /var/lib/apt/lists/*

 # Setup the locale
ENV LANG en_US.UTF-8
ENV LC_ALL $LANG
RUN locale-gen $LANG \
 && update-locale

# Accept EULA - needed for certain Microsoft packages like SQL Server Client Tools
ENV ACCEPT_EULA=Y

# Install essential build tools
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    build-essential \
 && rm -rf /var/lib/apt/lists/*
 
 # Install Terraform
RUN TERRAFORM_VERSION=$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | jq -r .current_version) \
 && curl -LO https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
 && unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d /usr/local/bin \
 && rm -f terraform_${TERRAFORM_VERSION}_linux_amd64.zip
{% endhighlight %}

It is copy pasted from [this one][docker] and I kept only the bits and pieces I needed (still a bit too much btw at the moment, but you get the point). 

What I also needed to do was rename some of the environment variables in my Azure function, so VSTS_TOKEN instead of VSTS_AGENT_INPUT_TOKEN. But apart from that, this agent works smoothly. 

That's it, small addendum. 

[terraformstart]: https://www.terraform.io/
[docker]: https://github.com/microsoft/vsts-agent-docker/blob/e1d3d411ddd55ac4f0385ad3291439d8a3949614/ubuntu/14.04/standard/Dockerfile
