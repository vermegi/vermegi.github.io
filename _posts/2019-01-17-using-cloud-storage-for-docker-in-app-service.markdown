---
layout: post
title:  "Using Cloud Storage for Docker in Azure App Service for Linux"
date:   2019-01-16 14:15:05
categories: azure AppService docker storage
background: "/assets/2018-08/dev.PNG"
---

Recently I got the question from a customer to help him run [Apache superset][superset] on Azure. Apache superset is an open source projects which allows you to create and run PowerBI dashboards on top of different data sources. It is build in Python and according to the installation documentation you can run this in a docker container to get quickly started. So, my plan was to get this docker container up and running on Azure App Service, since it needs a web endpoint. Also, along the way, I needed to map storage for this container. This post will guide you through my efforts and setup.

First, to make a long story short on how much time I lost on getting the official docker container build based on the [superset documentation][supersetdocs] and [official superset git repo][supersetrepo]: After 2 days of sweating and swearing, I switched to [this prebuild docker image][supersetdocker] on dockerhub. Based on this image and accompanying [github repo with examples][supersetgit], I was able to at least build and run superset locally with docker. For this I used [docker for windows][dockerwindows], set it to use Linux containers and walked through the [superset postgres example][postgresexample]. 

So, once I had this example running locally, I wanted to get this up and running in Azure. First thing I did for this, was create an Azure Database for PostgreSQL:

{% highlight cli %}
#login to Azure
az login

#if needed select the correct subscription

#create a resource group for your setup
az group create --name superset_tryout --location "West Europe"

#create the PostgreSQL server
az postgres server create --resource-group superset_tryout --name supersettryout --location "West Europe" --admin-user admin --admin-password yadadadatralala --sku-name B_Gen4_1

#create a firewall rule for the database (a very relaxed rule for this demo, but feel free to make it much more restrictive)
az postgres server firewall-rule create --resource-group superset_tryout --server-name supersettryout --start-ip-address=0.0.0.0 --end-ip-address=0.0.0.0 --name AllowAllAzureIPs
az postgres server firewall-rule create --resource-group superset_tryout --server-name supersettryout --start-ip-address=<your_ip_address> --end-ip-address=<your_ip_address> --name AllowLocalClient
{% endhighlight %}

Once you created the posgreSQL server, you can create a database for superset:

{% highlight cli %}
psql -h supersettryout.postgres.database.azure.com -U admin@supersettryout postgres

CREATE DATABASE superset;
{% endhighlight %}

Once you created this database, you can remove the postgres service from the docker-compose file of the superset examples. The file now looks like this:

{% highlight cli %}
version: '3'
services:
  redis:
    image: redis
    restart: always
    volumes:
      - redis:/data
  superset:
    image: amancevice/superset
    restart: always
    depends_on:
      - redis
    environment:
      MAPBOX_API_KEY: ${MAPBOX_API_KEY}
    ports:
      - "8088:8088"
    volumes:
      - ./superset_config.py:/etc/superset/superset_config.py
volumes:
  postgres:
    external: false
  redis:
    external: false
{% endhighlight %}

[superset]: https://superset.incubator.apache.org/
[supersetdocs]: https://superset.incubator.apache.org/installation.html#start-with-docker
[supersetrepo]: https://github.com/apache/incubator-superset
[supersetdocker]: https://hub.docker.com/r/amancevice/superset/
[supersetgit]: https://github.com/amancevice/superset 
[dockerwindows]: https://docs.docker.com/docker-for-windows/ 
[postgresexample]: https://github.com/amancevice/superset/tree/master/examples 