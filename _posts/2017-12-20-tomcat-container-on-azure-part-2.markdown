---
layout: post
title:  "Tomcat Container on Azure Part 2"
date:   2017-12-20 14:15:05
categories: azure docker ACI WebApps
background: "/assets/logo docker.png"
---
In a [previous post][previous] I ran a docker container on [Azure Container Instance][aci]. In this post, I will reuse this container and run it on [Web Apps on Containers][webapps].

For this we will create a new Web app on containers in the Azure portal. 

![new]({{ "assets/03.JPG" | absolute_url }}){: .img-fluid}

Give your webapp a unique name, select a resource group or create a new one, create a new app service plan and link your container. 

![new]({{ "assets/04.JPG" | absolute_url }}){: .img-fluid}

Once the app has been created, open this resource in the Azure portal. Last step you need is add a app setting for the 8080 port. 

![new]({{ "assets/04.JPG" | absolute_url }}){: .img-fluid}

That's it. Once this is done, you can click the URL of your app in the overview tab and this will show you the same Tomcat admin page. 


[previous]: http://vermegi.github.io/azure/docker/aci/webapps/2017/12/20/tomcat-container-on-azure/
[aci]:        https://azure.microsoft.com/en-us/services/container-instances/ 
[webapps]:        https://docs.microsoft.com/en-us/azure/app-service/containers/