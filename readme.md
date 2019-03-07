# Hands-on Containers op Azure


## Challenge:

Run 2 containers, a frontend which is calling a backend, on:

1. Azure Container Instances (ACI)
2. Webapp for Container (multicontainer)
   1. Using Docker Compose
   2. Using Kubernetes pod file
3. Extra: storage

## Common information:

The containers:

**Backend:**
pascalnaber/dutchazuremeetupwebapi:4

The database already exists, you just need the connection string.

Environment variables:

```
ConnectionStrings__DutchAzureMeetupContext= "Server=tcp:dutchazuremeetup.database.windows.net,1433;Initial Catalog=dutchazuremeetup;Persist Security Info=False;User ID=dutchazuremeetup;Password=We<3@zure;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
```

**Frontend:**
pascalnaber/dutchazuremeetupwebapp:4

Environment variables:
```
BackendUri=<URI of the webapi>
```

### 1. Azure Container Instances (ACI)

You are free to use the Portal, CLI, Powershell or an ARM Template to deploy the 2 containers in a container group.

More information: https://docs.microsoft.com/en-us/azure/container-instances/#5-minute-quickstarts

To run an .NET Core application on port 5000 for example:
```
ASPNETCORE_URLS=http://*:5000
```
The BackendUri environment variable should be like this:

```
BackendUri="http://localhost:5000"
```

Navigate to the URI of the ACI instance and you should see a working webapplication.

You can use the folling ARM Template for inspiration (**there's still something to change in it**):

```
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [
        {
            "name": "DAM",
            "type": "Microsoft.ContainerInstance/containerGroups",
            "apiVersion": "2018-10-01",
            "location": "westeurope",
            "identity": {
                "type": "SystemAssigned"
            },
            "properties": {
                "containers": [
                    {
                        "name": "webapi",
                        "properties": {
                            "command": [],
                            "environmentVariables": [
                                {
                                  "name": "ConnectionStrings__DutchAzureMeetupContext",
                                  "value": "Server=tcp:dutchazuremeetup.database.windows.net,1433;Initial Catalog=dutchazuremeetup;Persist Security Info=False;User ID=dutchazuremeetup;Password=We<3@zure;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
                                },
                                {
                                    "name": "ASPNETCORE_URLS",
                                    "value": "http://*:5000"
                                }
                            ],
                            "image": "<WEBAPI_IMAGE>",
                            "ports": [
                                {
                                    "protocol": "TCP",
                                    "port": 80
                                }
                            ],
                            "resources": {
                                "requests": {
                                    "cpu": 1,
                                    "memoryInGB": 2
                                }
                            },
                            "volumeMounts": []
                        }
                    },
                    {
                        "name": "webapp",
                        "properties": {
                            "command": [],
                            "environmentVariables": [
                                {
                                  "name": "BackendUri",
                                  "value": "http://localhost:5000"
                                }
                            ],
                            "image": "<WEBAPP_IMAGE>",
                            "resources": {
                                "requests": {
                                    "cpu": 1,
                                    "memoryInGB": 2
                                }
                            },
                            "volumeMounts": []
                        }
                    }
                ],
                "osType": "Linux",
                "ipAddress": {
                    "type": "Public",
                    "ports": [
                        {
                            "protocol": "TCP",
                            "port": 80
                        }
                    ]
                },
                "volumes": []
            }
        }
    ],
    "outputs": {
        "containerIPv4Address": {
            "type": "string",
            "value": "[reference(resourceId('Microsoft.ContainerInstance/containerGroups/','DAM')).ipAddress.ip]"
        }
    }
}
```

### 2. Webapp for Container (multicontainer)

### Docker Compose

Below you can find the sample Docker Compose from Microsoft. Change it to run both the meetup containers.

Note: Only the container that needs to be public should contain a ports configuration.

Docker compose on Web app for container uses a bridge network. This means that each container gets it's own IP address.


```
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

### Kubernetes

Below you can find the sample Kubernetes configuration from Microsoft. Change it to run both the meetup containers.

Note: Both containers are running on port 80 by default. That's not possible within a Pod. A Pod shares the IP address and the port makes them unique. Containers in the Pod can call each other using localhost.


```
apiVersion: v1
kind: Pod
metadata:
  name: wordpress
  labels:
    name: wordpress
spec:
  containers:

   - image: redis:3-alpine
     name: redis

   - image: microsoft/multicontainerwordpress
     name: wordpress
     ports:
     - containerPort: 80
       name: wordpress
     volumeMounts:
      - name: appservice-storage
        mountPath: /var/www/html
        subPath: /site/wwwroot
  volumes:
    - name: appservice-storage
      hostConfig:
        path: ${WEBAPP_STORAGE_HOME}
```

### 3. Extra

Mount a volume for both ACI and Webapp
