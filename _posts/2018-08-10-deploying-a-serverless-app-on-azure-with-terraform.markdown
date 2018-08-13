---
layout: post
title:  "Deploying a serverless app to azure with Terraform"
date:   2018-08-10 14:15:05
categories: azure Terraform Serverless
background: "/assets/ip-restrictions.png"
draft: true
---
Recently I was going through our serverless cloud workshop and got the idea of redoing the infrastructure setup for this with Terraform and with a fully automated build and release pipeline on VSTS. In this post I will walk you through the steps needed for setting this up. 

First thing you need is a VSTS project for the automation. Open up your vsts (someaccount.visualstudio.com) and click 'Create Project'. As you can see, I am already using the new VSTS layout, which I like a lot better compared to the old look and feel. 

![resources]({{ "assets/2018-08/createproject.PNG" | absolute_url }}){: .img-fluid}

Fill out the name of your new VSTS project, choose whether you want your repo to be public or private, choose your source control system and the template you want to use. The default will often do. After filling this out click Create.

![resources]({{ "assets/2018-08/createproject2.PNG" | absolute_url }}){: .img-fluid}

Once your project got created, initialize your repo with a default readme file for VisualStudio. 

![resources]({{ "assets/2018-08/repoinit.PNG" | absolute_url }}){: .img-fluid}
