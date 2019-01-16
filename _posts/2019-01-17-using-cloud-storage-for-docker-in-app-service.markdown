---
layout: post
title:  "Using Cloud Storage for Docker in Azure App Service for Linux"
date:   2019-01-16 14:15:05
categories: azure AppService docker storage
background: "/assets/2018-08/superset.PNG"
---

In a previous post I walked you through how to get [apache superset][superset] up and running together with Azure Database for PostgreSQL and Azure Redis Cache. 
In this post I will walk you through getting the superset container up in Azure App Service for linux. I will explain to you the following steps:

* create an Azure App Service for Linux based on a docker-compose file
* map storage to you app service and use it in your container. This is a new [feature][feature] for app service
 
First, we will need to create an app service plan, and a webapp. 

{% highlight cli %}
#create appservice plan
az appservice plan create --name supersetplan --resource-group superset_tryout --sku S1 --is-linux

#create webapp
az webapp create --resource-group superset_tryout --plan supersetplan --name supersettryout --multicontainer-config-type compose --multicontainer-config-file docker-compose.yml
{% endhighlight %}

This second line will create a web app in Azure based on your docker-compose file. The docker-compose file looks like this:

{% highlight docker %}
version: '3'
services:
  superset:
    image: gittetitter/superset:1
    restart: always
    environment:
      MAPBOX_API_KEY: ${MAPBOX_API_KEY}
    ports:
      - "8088:8088"
    volumes:
      - ./superset_config.py:/etc/superset/superset_config.py
{% endhighlight %}

(As you can notice in the docker-compose file I switched to my own docker container, that I published to docker hub. The only change I made here was an updated version of the werkzeug library. I switched this to version 0.14. I needed this update somewhere along the way, but I can't remember what the error was I was trying to fix.)

This setup works, more or less, however, notice the volume mapping which is in there. This will not work in app service, since the superset_config.py file is not there. So the superset service will have no config to start from. To fix this, we can map a storage container to our app service. 

{% highlight cli %}
#create storage account
az storage account create --name supersettryout --resource-group superset_tryout

#create storage container
az storage container create --name superset --account-name supersettryout

#upload superset_config.py
az storage blob upload-batch -d superset --account-name supersettryout --account-key "yadayadathekey" -s F:\dev\github\superset\examples\gitte_postgres --pattern *.py
{% endhighlight %}

Now that we have a storage container with our superset_config.py file, we can associate this container to our app service. 

{% highlight cli %}
#attach storage account to webapp
az webapp config storage-account add --resource-group superset_tryout --name supersettryout --custom-id CustomId --storage-type AzureBlob --share-name superset --account-name supersettryout --access-key "yadadadathekey" --mount-path /etc/superset
{% endhighlight %}

The tricky parts of this command are:

* --share-name: which can also be a storage container name, so here use the name of your storage container you just create
* --mount-path: use the same as the path where you want it to end up in your container
* --custom-id: you will later use this id in your docker-compose file

Now you can alter your docker-compose file to use this storage:

{% highlight docker %}
version: '3'
services:
  superset:
    image: gittetitter/superset:1
    restart: always
    environment:
      MAPBOX_API_KEY: ${MAPBOX_API_KEY}
    ports:
      - "80:8088"
    volumes:
      - CustomId:/etc/superset
{% endhighlight %}

So in the volumes parameter, you use the --custom-id and the --mount-path again. 

Last step is update your web app:

{% highlight cli %}
az webapp config container set --resource-group superset_tryout --name supersettryout --multicontainer-config-type compose --multicontainer-config-file docker-compose.yml
{% endhighlight %}

And that should get it all up and running. 

[superset]: https://superset.incubator.apache.org/
[supersetdocs]: https://superset.incubator.apache.org/installation.html#start-with-docker
[supersetrepo]: https://github.com/apache/incubator-superset
[supersetdocker]: https://hub.docker.com/r/amancevice/superset/
[supersetgit]: https://github.com/amancevice/superset 
[dockerwindows]: https://docs.docker.com/docker-for-windows/ 
[postgresexample]: https://github.com/amancevice/superset/tree/master/examples 
[feature]: https://blogs.msdn.microsoft.com/appserviceteam/2018/09/24/announcing-bring-your-own-storage-to-app-service/