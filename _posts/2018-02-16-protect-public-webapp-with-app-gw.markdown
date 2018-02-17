---
layout: post
title:  "Protect multi-tenant web app with application gateway and WAF"
date:   2018-02-16 14:15:05
categories: azure WebApps WAF
background: "/assets/ip-restrictions.png"
---
The Azure docs contain a walkthrough of [configuring an Application Gateway in front of a multi-tenant web app][walkthrough]. This is all good and well, but after going through these powershell scripts, a user is still able to directly hit your public website. Meaning, users are not necessarily forced to pass through the application gateway. 

To enable this behaviour, you need to combine the above walkthrough with the [IP Restrictions][iprestrictions] capability of web apps. 

If you follow the second [script][script] on the application gateway configuration documentation item, you end up with a resource group which has the following resources:

![resources]({{ "assets/resources.JPG" | absolute_url }}){: .img-fluid}

I altered this script a bit to take a different region closer to me. Also, I altered the script to use the Standard app service plan SKU, since IP restrictions are not possible in the free or shared SKU's. 

Next, you need to get the public IP of your application gateway. You can find this either in the application gateway resource, or in the public IP resource. 

Once you have this, in your web app, you can go to networking:

![networking]({{ "assets/networking.JPG" | absolute_url }}){: .img-fluid}

And there open the IP restrictions blade. Here you can now add a IP restriction rule for the IP of the application gateway. 

Once you have this, directly going to the URL of the web app will not be possible anymore.

![noaccess]({{ "assets/noaccess.JPG" | absolute_url }}){: .img-fluid}

While going through the application gateway is no problem.

![access]({{ "assets/access.JPG" | absolute_url }}){: .img-fluid}

And that's all you need to protect a public web app with a application gateway and/or a WAF. (for the latter, just enable the feature on the app gateway). 


[walkthrough]:      https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-web-app-powershell
[iprestrictions]:   https://docs.microsoft.com/en-us/azure/app-service/app-service-ip-restrictions