---
layout: post
title:  "Running superset on Azure App Service for Linux"
date:   2019-01-16 14:15:05
categories: azure AppService docker storage
background: "/assets/2018-08/dev.PNG"
---

Recently I got the question from a customer to help him run [Apache superset][superset] on Azure. Apache superset is an open source projects which allows you to create and run PowerBI dashboards on top of different data sources. It is build in Python and according to the installation documentation you can run this in a docker container to get quickly started. So, my plan was to get this docker container up and running on Azure App Service, since it needs a web endpoint. 

In this post will guide you through my efforts and setup. I will explain to you:

* how to run superset locally
* how to connect superset to an Azure Database for PostgreSQL
* how to connect superset to an Azure Redis Cache

In a next post I will go through:

* how to deploy superset to Azure App Service for Linux
* how to attach cloud storage for the config file(s)

First, to make a long story short on how much time I lost on getting the official docker container build based on the [superset documentation][supersetdocs] and [official superset git repo][supersetrepo]: After 2 days of sweating and swearing, I switched to [this prebuild docker image][supersetdocker] on dockerhub. Based on this image and accompanying [github repo with examples][supersetgit], I was able to at least build and run superset locally with docker. For this I used [docker for windows][dockerwindows], set it to use Linux containers and walked through the [superset postgres example][postgresexample]. 

This example makes use of a docker-compose file to spin up 3 containers, one for superset, one for redis cache and one for a PostgreSQL database. The docker-compose file also maps the superset_config.py file to a local path in the superset container. This superset_config.py files contains the configuration for helping superset connect to the redis cache and PostgreSQL database. 

Next to that there are a superset-init and superset-demo file that are copied into the original docker container. These scripts initialize superset with a login user and load it with demo data. 

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

Once you have created this database, you can remove the postgres service from the docker-compose file of the superset examples. The file now looks like this:

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
  redis:
    external: false
{% endhighlight %}

But, what we also need, is give superset the connection info to this azure postgreSQL database. This is done in the superset_config.py file. You need to change the SQLALCHEMY_DATABASE_URI. The format for the URI is:

{% highlight cli %}
postgresql://database_user@servername:thesuperduperpassword@servername.postgres.database.azure.com:5432/databasename'
{% endhighlight %}

So for my example this will be:

{% highlight cli %}
SQLALCHEMY_DATABASE_URI = \
    'postgresql://superset@supersettryout:thesuperduperpassword@supersettryout.postgres.database.azure.com:5432/superset'
{% endhighlight %}

Figuring out the correct syntax and values for this connection string took some trial and error, but once you have this you can run everything with:

{% highlight bash %}
# Start Redis service (no need anymore to start postgress service)
docker-compose up -d redis
# Wait for services to come up fully...

# Start Superset
docker-compose up -d superset
# Wait for Superset to come up fully...

# Initialize demo
docker-compose exec superset superset-demo
# or `docker-compose exec superset superset-init` if no demo data needed

# Play around in demo...

# Bring everything down
docker-compose down -v
{% endhighlight %}

Next step will be to replace the redis container with an Azure Redis Cache. 

{% highlight cli %}
az redis create --name superset --resource-group superset_tryout --location "West Europe" --sku Basic --vm-size C0
{% endhighlight %}

Once you have your cache setup, you can also alter the docker-compose file and get rid of the redis service. 

{% highlight cli %}
version: '3'
services:
  superset:
    image: amancevice/superset
    restart: always
    environment:
      MAPBOX_API_KEY: ${MAPBOX_API_KEY}
    ports:
      - "8088:8088"
    volumes:
      - ./superset_config.py:/etc/superset/superset_config.py
{% endhighlight %}

And again, in the superset_config.py file alter the connection information. Now, this is quite tricky, since Azure Redis Cache only allows connections over SSL. For this you need special syntax. 

{% highlight python %}
import os
import redis

r = redis.StrictRedis(host='superset.redis.cache.windows.net', port=6380, db=0, password='yadayadadada', ssl=True)

MAPBOX_API_KEY = os.getenv('MAPBOX_API_KEY', '')
CACHE_CONFIG = {
    'CACHE_TYPE': 'redis',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_HOST': r}
SQLALCHEMY_DATABASE_URI = \
    'obfuscated'
SQLALCHEMY_TRACK_MODIFICATIONS = True
SECRET_KEY = 'thisISaSECRET_1234'
{% endhighlight %}

You need to use redis.StrictRedis to create a connection object and then pass this object to the HOST parameter. This does the trick. And now you can again spin everything up and try it out: 

{% highlight cli %}
# no need anymore to start redis or postgress

# Start Superset
docker-compose up -d superset
# Wait for Superset to come up fully...

# demo already initialized (since we now use posgreSQL database, our initialization and demo data is persisted across runs)

# Play around in demo...

# Bring everything down
docker-compose down -v
{% endhighlight %}

Ok, so now we already have our local superset container run with an Azure Redis Cache and an Azure Database for PostgreSQL. Next step will be to get the superset container running in Azure App Service. 

[superset]: https://superset.incubator.apache.org/
[supersetdocs]: https://superset.incubator.apache.org/installation.html#start-with-docker
[supersetrepo]: https://github.com/apache/incubator-superset
[supersetdocker]: https://hub.docker.com/r/amancevice/superset/
[supersetgit]: https://github.com/amancevice/superset 
[dockerwindows]: https://docs.docker.com/docker-for-windows/ 
[postgresexample]: https://github.com/amancevice/superset/tree/master/examples 