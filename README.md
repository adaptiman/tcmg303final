<a id="top"></a>
# TCMG303 Practicum Final: Gotta Run!

<img src="images/Gordon_Ramsay.jpg" alt="Gordon Ramsay is about to kick our @ss." />

Dear SysAdmin:

Gordon Ramsay met with out company owners yesterday and dropped a bomb (along with a few f-bombs). He wants us to create a new site for him THIS WEEKEND. To make matters worse, he heard about this neat thing called "open source software" and thinks it would be really chic to use it. You may be aware that Mr. Ramsay can be a bit tempermental and demanding, so we have one shot to get this right.

> Your task is to create and deploy a functional Web app in Azure Cloud Services using the [open source project OpenEats](https://github.com/open-eats/OpenEats/).

Summarizing Mr. Ramsay's requirements, the application must be:

* Based upon the Open Eats project located at https://github.com/open-eats/OpenEats
* Built using Docker and Docker Compose tools according to the OpenEats documentation
* Populated with example data
*  Served over port 80
* Connected to a persistent Azure for MySQL Database Server
* Deployed as an Azure Web App

I've looked over the OpenEats repo and have a few suggestions that may help you in the deployment:

1. Develop the solution in three stages - 1) Deploy the app "locally" on a fresh Azure VM named OpenEatsDocker with the necessary packages installed according to the documentation to prove the concept. 2) Create a persistent database service using Azure Database for MySQL Server, and then connect the local app to it. 3) Migrate the local app containers excluding the containerized database to Azure App Services.

Looking over the project, the app is deployed using four containers with the following functions:

1. [openeats/openeats-nginx](https://github.com/open-eats/openeats-nginx) - This is a customized nginx web server that acts as a [reverse proxy](https://www.nginx.com/resources/glossary/reverse-proxy-server/). This container serves as the web front-end for the app but doesn't contain any data or website templates.
2. [openeats/openeats-api](https://github.com/open-eats/openeats-api) - This is the core app that contains all of the website logic and templates. It's built on the [Django](https://www.djangoproject.com/) web framework.
3. [openeats/openeats-web](https://github.com/open-eats/openeats-web) - This is a customized state container for the app based upon [React and Redux](https://react-redux.js.org/). A state container is responsible for "bundling" the data and serving it as javascript to the front-end. 
4. [mariadb](https://hub.docker.com/_/mariadb) - This is an unmodified containerized database common in many microservice apps. Mariadb is a lightweight version of MySQL and acts almost identically.

<img src="images/Phase 1.png" alt="Phase 1 Deployment Model" />
<img src="images/Phase 2.png" alt="Phase 2 Deployment Model" />
<img src="images/Phase 3.png" alt="Phase 3 Deployment Model" />

I've written a little program for you contained in this repo that will check your deployment as you go. Run it occasionally and check your work.



Unfortunately, my vacation starts today, so I won't be here this weekend to help you. Since I'm hiking in the Dolomites, my mobile reception will probably be spotty. I might be able to respond on Slack, but you never know. Well, I'm off. In case I don't see you for a while, have fun and GOOD LUCK!

<img src="images/boots-mountain.jpg" alt="I'm outta here!" />
This lab was originally written by [Mangesh Sangapu](https://github.com/msangapu) at Microsoft and adapted for TCMG-303 by [David Sweeney](https://adaptiman.com).


[Web App for Containers](app-service-linux-intro.md) provides a flexible way to use Docker images. In this tutorial, you'll learn how to create a multi-container app using WordPress and MySQL. You should complete this lab from one of your running Ubuntu VM's. While you can complete the tutorial with any [Azure CLI](/cli/azure/install-azure-cli) command-line tool (2.0.32 or later), the grading program included in this repo *only* works on unix-based systems.

In this lab, you will learn how to:
> * Convert a Docker Compose configuration to work with Web App for Containers
> * Deploy a multi-container app to Azure
> * Add application settings
> * Use persistent storage for your containers
> * Connect to Azure Database for MySQL
> * Troubleshoot errors

<a href="#top" class="top" id="table-of-contents">Top</a>
## Table of Contents
-  [Prerequisites](#pre)
-  [Clone the repo](#clone)
-  [Create a resource group](#rg)
-  [Create an Azure App Service Plan](#serviceplan)
-  [Create a Docker Compose app](#dockerapp)
-  [Connect to production database](#proddb)
-  [Add persistent storage](#persist)
-  [Add Redis container](#redis)
-  [Find Docker container logs](#logs)
-  [Submitting the Lab](#submitlab)
-  [Clean up the deployment](#cleanup)
-  [Summary](#summary)

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="pre"></a>
## Prerequisites

To complete this lab, you need experience with [Docker Compose](https://docs.docker.com/compose/).

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="clone"></a>
## Clone the repo

Run the following commands to clone the lab repository then change to the `lab11` directory.

```bash
git clone https://github.com/adaptiman/lab11
cd lab11
```
For this lab, you start with the compose file from [Docker](https://docs.docker.com/compose/wordpress/#define-the-project), but you'll modify it to include Azure Database for MySQL, persistent storage, and Redis. The configuration file can be found at [Azure Samples](https://github.com/Azure-Samples/multicontainerwordpress). For supported configuration options, see [Docker Compose options](https://docs.microsoft.com/en-us/azure/app-service/containers/configure-custom-container#docker-compose-options).

```yaml
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
    db_data:
```

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="rg"></a>
## Create a resource group

Create a resource group with the [`az group create`](/cli/azure/group?view=azure-cli-latest#az-group-create) command. The following example creates a resource group named *myResourceGroup* in the *South Central US* location. To see all supported locations for App Service on Linux in **Standard** tier, run the [`az appservice list-locations --sku S1 --linux-workers-enabled`](/cli/azure/appservice?view=azure-cli-latest#az-appservice-list-locations) command.

```
az group create --name myResourceGroup --location "South Central US"
```

For the purposes of this lab, create all of your resources in the "South Central US" region.

When the command finishes, a JSON output shows you the resource group properties.
```
{
  "adminSiteName": null,
  "appServicePlanName": "myAppServicePlan",
  "geoRegion": "South Central US",
  "hostingEnvironmentProfile": null,
  "id": "/subscriptions/0000-0000/resourceGroups/myResourceGroup/providers/Microsoft.Web/serverfarms/myAppServicePlan",
  "kind": "linux",
  "location": "South Central US",
  "maximumNumberOfWorkers": 1,
  "name": "myAppServicePlan",
  < JSON data removed for brevity. >
  "targetWorkerSizeId": 0,
  "type": "Microsoft.Web/serverfarms",
  "workerTierName": null
}
```

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="serviceplan"></a>
## Create an Azure App Service plan

Create an App Service plan in the resource group with the `az appservice plan create` command.



The following example creates an App Service plan named `myAppServicePlan` in the **Standard** pricing tier (`--sku B1`) and in a Linux container (`--is-linux`).

```
az appservice plan create --name myAppServicePlan --resource-group myResourceGroup --sku B1 --is-linux
```

When the App Service plan has been created, your shell shows information similar to the following example:

```
{
  "adminSiteName": null,
  "appServicePlanName": "myAppServicePlan",
  "geoRegion": "South Central US",
  "hostingEnvironmentProfile": null,
  "id": "/subscriptions/0000-0000/resourceGroups/myResourceGroup/providers/Microsoft.Web/serverfarms/myAppServicePlan",
  "kind": "linux",
  "location": "South Central US",
  "maximumNumberOfWorkers": 1,
  "name": "myAppServicePlan",
  ...
  "targetWorkerSizeId": 0,
  "type": "Microsoft.Web/serverfarms",
  "workerTierName": null
}
```

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="dockerapp"></a>
## Create a Docker Compose app

Create a multi-container [web app](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-intro) in the `myAppServicePlan` App Service plan with the [az webapp create](https://docs.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-create) command. Don't forget to replace _\<app-name>_ with a unique app name.

```
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name <app-name> --multicontainer-config-type compose --multicontainer-config-file docker-compose-wordpress.yml
```

When the web app has been created, your shell shows output similar to the following example:
```
{
  "additionalProperties": {},
  "availabilityState": "Normal",
  "clientAffinityEnabled": true,
  "clientCertEnabled": false,
  "cloningInfo": null,
  "containerSize": 0,
  "dailyMemoryTimeQuota": 0,
  "defaultHostName": "<app-name>.azurewebsites.net",
  "enabled": true,
...
}
```

### Browse to the app

Browse to the deployed app at (`http://<app-name>.azurewebsites.net`). The app may take a few minutes to load. If you receive an error, allow a few more minutes then refresh the browser. If you're having trouble and would like to troubleshoot, review [container logs](#logs).

<img src="images/azure-multi-container-wordpress-install.png" alt="Wordpress setup screen." />

**Congratulations**, you've created a multi-container app in Web App for Containers. Next you'll configure your app to use Azure Database for MySQL. Don't install WordPress at this time.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="proddb"></a>
## Connect to production database

It's not recommended to use database containers in a production environment. The local containers aren't scalable. Instead, you'll use Azure Database for MySQL which can be scaled.

### Create an Azure Database for MySQL server

Create an Azure Database for MySQL server with the [`az mysql server create`](https://docs.microsoft.com/en-us/cli/azure/mysql/server?view=azure-cli-latest#az-mysql-server-create) command.

In the following command, substitute your MySQL server name where you see the _<mysql-server-name>_ placeholder (valid characters are `a-z`, `0-9`, and `-`). This name is part of the MySQL server's hostname  (`<mysql-server-name>.database.windows.net`), it needs to be globally unique.

```
az mysql server create --resource-group myResourceGroup --name <mysql-server-name>  --location "South Central US" --admin-user adminuser --admin-password My5up3rStr0ngPaSw0rd! --sku-name B_Gen5_2 --version 5.7
```

Creating the server may take a few minutes to complete. When the MySQL server is created, your shell shows information similar to the following example:

```
{
  "administratorLogin": "adminuser",
  "administratorLoginPassword": null,
  "fullyQualifiedDomainName": "<mysql-server-name>.database.windows.net",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myResourceGroup/providers/Microsoft.DBforMySQL/servers/<mysql-server-name>",
  "location": "southcentralus",
  "name": "<mysql-server-name>",
  "resourceGroup": "myResourceGroup",
  ...
}
```

### Configure server firewall

Create a firewall rule for your MySQL server to allow client connections by using the [`az mysql server firewall-rule create`](https://docs.microsoft.com/en-us/cli/azure/mysql/server/firewall-rule?view=azure-cli-latest#az-mysql-server-firewall-rule-create) command. When both starting IP and end IP are set to 0.0.0.0, the firewall is only opened for other Azure resources.

```
az mysql server firewall-rule create --name allAzureIPs --server <mysql-server-name> --resource-group myResourceGroup --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

> You can be even more restrictive in your firewall rule by [using only the outbound IP addresses your app uses](https://docs.microsoft.com/en-us/azure/app-service/overview-inbound-outbound-ips?toc=/azure/app-service/containers/toc.json#find-outbound-ips).
>

### Create the WordPress database

```
az mysql db create --resource-group myResourceGroup --server-name <mysql-server-name> --name wordpress
```

When the database has been created, your shell shows information similar to the following example:

```
{
  "additionalProperties": {},
  "charset": "latin1",
  "collation": "latin1_swedish_ci",
  "id": "/subscriptions/12db1644-4b12-4cab-ba54-8ba2f2822c1f/resourceGroups/myResourceGroup/providers/Microsoft.DBforMySQL/servers/<mysql-server-name>/databases/wordpress",
  "name": "wordpress",
  "resourceGroup": "myResourceGroup",
  "type": "Microsoft.DBforMySQL/servers/databases"
}
```

### Configure database variables in WordPress

To connect the WordPress app to this new MySQL server, you'll configure a few WordPress-specific environment variables, including the SSL CA path defined by `MYSQL_SSL_CA`. The [Baltimore CyberTrust Root](https://www.digicert.com/digicert-root-certificates.htm) from [DigiCert](https://www.digicert.com/) is provided in the [custom image](https://docs.microsoft.com/azure/app-service/containers/tutorial-multi-container-app#use-a-custom-image-for-mysql-ssl-and-other-configurations) below.

To make these changes, use the [az webapp config appsettings set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/appsettings?view=azure-cli-latest#az-webapp-config-appsettings-set) command in your shell. App settings are case-sensitive and space-separated.

```
az webapp config appsettings set --resource-group myResourceGroup --name <app-name> --settings WORDPRESS_DB_HOST="<mysql-server-name>.mysql.database.azure.com" WORDPRESS_DB_USER="adminuser@<mysql-server-name>" WORDPRESS_DB_PASSWORD="My5up3rStr0ngPaSw0rd!" WORDPRESS_DB_NAME="wordpress" MYSQL_SSL_CA="BaltimoreCyberTrustroot.crt.pem"
```

When the app setting has been created, your shell shows information similar to the following example:

```
[
  {
    "name": "WORDPRESS_DB_HOST",
    "slotSetting": false,
    "value": "<mysql-server-name>.mysql.database.azure.com"
  },
  {
    "name": "WORDPRESS_DB_USER",
    "slotSetting": false,
    "value": "adminuser@<mysql-server-name>"
  },
  {
    "name": "WORDPRESS_DB_NAME",
    "slotSetting": false,
    "value": "wordpress"
  },
  {
    "name": "WORDPRESS_DB_PASSWORD",
    "slotSetting": false,
    "value": "My5up3rStr0ngPaSw0rd!"
  },
  {
    "name": "MYSQL_SSL_CA",
    "slotSetting": false,
    "value": "BaltimoreCyberTrustroot.crt.pem"
  }
]
```

For more information on environment variables, see [Configure environment variables](https://docs.microsoft.com/en-us/azure/app-service/containers/configure-custom-container#configure-environment-variables).

### Use a custom image for MySQL SSL and other configurations

By default, SSL is used by Azure Database for MySQL. WordPress requires additional configuration to use SSL with MySQL. The WordPress 'official image' doesn't provide the additional configuration, but a [custom image](https://github.com/Azure-Samples/multicontainerwordpress) has been prepared for your convenience. In practice, you would add desired changes to your own image.

The custom image is based on the 'official image' of [WordPress from Docker Hub](https://hub.docker.com/_/wordpress/). The following changes have been made in this custom image for Azure Database for MySQL:

* [Adds Baltimore Cyber Trust Root Certificate file for SSL to MySQL.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L61)
* [Uses App Setting for MySQL SSL Certificate Authority certificate in WordPress wp-config.php.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L163)
* [Adds WordPress define for MYSQL_CLIENT_FLAGS needed for MySQL SSL.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L164)

The following changes have been made for Redis (to be used in a later section):

* [Adds PHP extension for Redis v4.0.2.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/Dockerfile#L35)
* [Adds unzip needed for file extraction.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L71)
* [Adds Redis Object Cache 1.3.8 WordPress plugin.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L74)
* [Uses App Setting for Redis host name in WordPress wp-config.php.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L162)

To use the custom image, you'll update your docker-compose-wordpress.yml file. In your shell, type `nano docker-compose-wordpress.yml` to open the nano text editor. Change the `image: wordpress` to use `image: microsoft/multicontainerwordpress`. You no longer need the database container. Remove the  `db`, `environment`, `depends_on`, and `volumes` section from the configuration file. Your file should look like the following code:

```yaml
version: '3.3'

services:
   wordpress:
     image: microsoft/multicontainerwordpress
     ports:
       - "8000:80"
     restart: always
```

Save your changes and exit nano. Use the command `^O` to save and `^X` to exit.

### Update app with new configuration

In your shell, reconfigure your multi-container [web app](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-intro) with the [az webapp config container set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/container?view=azure-cli-latest#az-webapp-config-container-set) command. Don't forget to replace _\<app-name>_ with the name of the web app you created earlier.

```
az webapp config container set --resource-group myResourceGroup --name <app-name> --multicontainer-config-type compose --multicontainer-config-file docker-compose-wordpress.yml
```

When the app has been reconfigured, your shell shows information similar to the following example:

```
[
  {
    "name": "DOCKER_CUSTOM_IMAGE_NAME",
    "value": "COMPOSE|dmVyc2lvbjogJzMuMycKCnNlcnZpY2VzOgogICB3b3JkcHJlc3M6CiAgICAgaW1hZ2U6IG1zYW5nYXB1L3dvcmRwcmVzcwogICAgIHBvcnRzOgogICAgICAgLSAiODAwMDo4MCIKICAgICByZXN0YXJ0OiBhbHdheXM="
  }
]
```

### Browse to the app

Browse to the deployed app at (`http://<app-name>.azurewebsites.net`). The app is now using Azure Database for MySQL.

<img src="images/azure-multi-container-wordpress-install.png" alt="Wordpress setup screen." />


<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="persist"></a>
## Add persistent storage

Your multi-container is now running in Web App for Containers. However, if you install WordPress now and restart your app later, you'll find that your WordPress installation is gone. This happens because your Docker Compose configuration currently points to a storage location inside your container. The files installed into your container don't persist beyond app restart. In this section, you'll [add persistent storage](https://docs.microsoft.com/en-us/azure/app-service/containers/configure-custom-container#use-persistent-shared-storage) to your WordPress container.

### Configure environment variables

To use persistent storage, you'll enable this setting within App Service. To make this change, use the [az webapp config appsettings set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/appsettings?view=azure-cli-latest#az-webapp-config-appsettings-set) command in your shell. App settings are case-sensitive and space-separated.

```
az webapp config appsettings set --resource-group myResourceGroup --name <app-name> --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=TRUE
```

When the app setting has been created, your shell shows information similar to the following example:

```
[
...
  {
    "name": "WORDPRESS_DB_NAME",
    "slotSetting": false,
    "value": "wordpress"
  },
  {
    "name": "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
    "slotSetting": false,
    "value": "TRUE"
  }
]
```

### Modify configuration file

In the your shell, type `nano docker-compose-wordpress.yml` to open the nano text editor.

The `volumes` option maps the file system to a directory within the container. `${WEBAPP_STORAGE_HOME}` is an environment variable in App Service that is mapped to persistent storage for your app. You'll use this environment variable in the volumes option so that the WordPress files are installed into persistent storage instead of the container. Make the following modifications to the file:

In the `wordpress` section, add a `volumes` option so it looks like the following code:

```yaml
version: '3.3'

services:
   wordpress:
     image: microsoft/multicontainerwordpress
     volumes:
      - ${WEBAPP_STORAGE_HOME}/site/wwwroot:/var/www/html
     ports:
       - "8000:80"
     restart: always
```

### Update app with new configuration

In your shell, reconfigure your multi-container [web app](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-intro) with the [az webapp config container set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/container?view=azure-cli-latest#az-webapp-config-container-set) command. Don't forget to replace _\<app-name>_ with a unique app name.

```
az webapp config container set --resource-group myResourceGroup --name <app-name> --multicontainer-config-type compose --multicontainer-config-file docker-compose-wordpress.yml
```

After your command runs, it shows output similar to the following example:

```
[
  {
    "name": "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
    "slotSetting": false,
    "value": "TRUE"
  },
  {
    "name": "DOCKER_CUSTOM_IMAGE_NAME",
    "value": "COMPOSE|dmVyc2lvbjogJzMuMycKCnNlcnZpY2VzOgogICBteXNxbDoKICAgICBpbWFnZTogbXlzcWw6NS43CiAgICAgdm9sdW1lczoKICAgICAgIC0gZGJfZGF0YTovdmFyL2xpYi9teXNxbAogICAgIHJlc3RhcnQ6IGFsd2F5cwogICAgIGVudmlyb25tZW50OgogICAgICAgTVlTUUxfUk9PVF9QQVNTV09SRDogZXhhbXBsZXBhc3MKCiAgIHdvcmRwcmVzczoKICAgICBkZXBlbmRzX29uOgogICAgICAgLSBteXNxbAogICAgIGltYWdlOiB3b3JkcHJlc3M6bGF0ZXN0CiAgICAgcG9ydHM6CiAgICAgICAtICI4MDAwOjgwIgogICAgIHJlc3RhcnQ6IGFsd2F5cwogICAgIGVudmlyb25tZW50OgogICAgICAgV09SRFBSRVNTX0RCX1BBU1NXT1JEOiBleGFtcGxlcGFzcwp2b2x1bWVzOgogICAgZGJfZGF0YTo="
  }
]
```

### Browse to the app

Browse to the deployed app at (`http://<app-name>.azurewebsites.net`).

The WordPress container is now using Azure Database for MySQL and persistent storage.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="redis"></a>
## Add Redis container

 The WordPress 'official image' does not include the dependencies for Redis. These dependencies and additional configuration needed to use Redis with WordPress have been prepared for you in this [custom image](https://github.com/Azure-Samples/multicontainerwordpress). In practice, you would add desired changes to your own image.

The custom image is based on the 'official image' of [WordPress from Docker Hub](https://hub.docker.com/_/wordpress/). The following changes have been made in this custom image for Redis:

* [Adds PHP extension for Redis v4.0.2.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/Dockerfile#L35)
* [Adds unzip needed for file extraction.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L71)
* [Adds Redis Object Cache 1.3.8 WordPress plugin.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L74)
* [Uses App Setting for Redis host name in WordPress wp-config.php.](https://github.com/Azure-Samples/multicontainerwordpress/blob/5669a89e0ee8599285f0e2e6f7e935c16e539b92/docker-entrypoint.sh#L162)

Add the redis container to the bottom of the configuration file so it looks like the following example:

```yaml
version: '3.3'

services:
   wordpress:
     image: microsoft/multicontainerwordpress
     ports:
       - "8000:80"
     restart: always

   redis:
     image: redis:3-alpine
     restart: always
```

### Configure environment variables

To use Redis, you'll enable this setting, `WP_REDIS_HOST`, within App Service. This is a *required setting* for WordPress to communicate with the Redis host. To make this change, use the [az webapp config appsettings set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/appsettings?view=azure-cli-latest#az-webapp-config-appsettings-set) command in your shell. App settings are case-sensitive and space-separated.

```
az webapp config appsettings set --resource-group myResourceGroup --name <app-name> --settings WP_REDIS_HOST="redis"
```

When the app setting has been created, your shell shows information similar to the following example:

```
[
...
  {
    "name": "WORDPRESS_DB_USER",
    "slotSetting": false,
    "value": "adminuser@<mysql-server-name>"
  },
  {
    "name": "WP_REDIS_HOST",
    "slotSetting": false,
    "value": "redis"
  }
]
```

### Update app with new configuration

In your shell, reconfigure your multi-container [web app](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-intro) with the [az webapp config container set](https://docs.microsoft.com/en-us/cli/azure/webapp/config/container?view=azure-cli-latest#az-webapp-config-container-set) command. Don't forget to replace _\<app-name>_ with a unique app name.

```
az webapp config container set --resource-group myResourceGroup --name <app-name> --multicontainer-config-type compose --multicontainer-config-file compose-wordpress.yml
```

After your command runs, it shows output similar to the following example:

```
[
  {
    "name": "DOCKER_CUSTOM_IMAGE_NAME",
    "value": "COMPOSE|dmVyc2lvbjogJzMuMycKCnNlcnZpY2VzOgogICBteXNxbDoKICAgICBpbWFnZTogbXlzcWw6NS43CiAgICAgdm9sdW1lczoKICAgICAgIC0gZGJfZGF0YTovdmFyL2xpYi9teXNxbAogICAgIHJlc3RhcnQ6IGFsd2F5cwogICAgIGVudmlyb25tZW50OgogICAgICAgTVlTUUxfUk9PVF9QQVNTV09SRDogZXhhbXBsZXBhc3MKCiAgIHdvcmRwcmVzczoKICAgICBkZXBlbmRzX29uOgogICAgICAgLSBteXNxbAogICAgIGltYWdlOiB3b3JkcHJlc3M6bGF0ZXN0CiAgICAgcG9ydHM6CiAgICAgICAtICI4MDAwOjgwIgogICAgIHJlc3RhcnQ6IGFsd2F5cwogICAgIGVudmlyb25tZW50OgogICAgICAgV09SRFBSRVNTX0RCX1BBU1NXT1JEOiBleGFtcGxlcGFzcwp2b2x1bWVzOgogICAgZGJfZGF0YTo="
  }
]
```

### Browse to the app

Browse to the deployed app at (`http://<app-name>.azurewebsites.net`).

Complete the steps and install WordPress.

### Connect WordPress to Redis

Sign in to WordPress admin. In the left navigation, select **Plugins**, and then select **Installed Plugins**.

<img src="images/wordpress-plugins.png" alt="Select Wordpress plugins." />

In the plugins page, find **Redis Object Cache** and click **Activate**.

<img src="images/activate-redis.png" alt="Activate Redis." />

Click on **Settings**.

<img src="images/redis-settings.png" alt="Click on Redis settings." />

Click the **Enable Object Cache** button.

<img src="images/enable-object-cache.png" alt="Click Enable Object Cache Button." />

WordPress connects to the Redis server. The connection **status** appears on the same page.

<img src="images/redis-connected.png" alt="WordPress connects to the Redis server. The connection **status** appears on the same page." />

**Congratulations!**, you've connected WordPress to Redis. The production-ready app is now using **Azure Database for MySQL, persistent storage, and Redis**. You can now scale out your App Service Plan to multiple instances.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="logs"></a>
## Find Docker Container logs

If you run into issues using multiple containers, you can access the container logs by browsing to: `https://<app-name>.scm.azurewebsites.net/api/logs/docker`.

You'll see output similar to the following example:

```
[
   {
      "machineName":"RD00XYZYZE567A",
      "lastUpdated":"2018-05-10T04:11:45Z",
      "size":25125,
      "href":"https://<app-name>.scm.azurewebsites.net/api/vfs/LogFiles/2018_05_10_RD00XYZYZE567A_docker.log",
      "path":"/home/LogFiles/2018_05_10_RD00XYZYZE567A_docker.log"
   }
]
```

You see a log for each container and an additional log for the parent process. Copy the respective `href` value into the browser to view the log.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="submitlab"></a>
## Submitting the lab

To submit the lab, run:
```
$ history -a
$ ~/lab11/grader.sh
```
The program will look for evidence that you completed this lab. Additionally, it will check for a running Wordpress installation on a container. The program will display your grade. You can run the grader as many times as you like until you're happy with your score. The grader automatically reports your grade to the instructor.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="cleanup"></a>
## Clean up the deployment
After you have submitted the lab, the following command can be used to remove the resource group and all resources associated with it:
```
az group delete --name myResourceGroup
```
The lab grader only runs on unix. It will not work on Windows or MacOS.

<a href="#table-of-contents" class="top" id="preface">Top</a>
<a id="summary"></a>
## Summary

In this lab, you learned how to:
> * Convert a Docker Compose configuration to work with Web App for Containers
> * Deploy a multi-container app to Azure
> * Add application settings
> * Use persistent storage for your containers
> * Connect to Azure Database for MySQL
> * Troubleshoot errors
