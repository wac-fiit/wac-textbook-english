# Installation and Setup of MongoDB Database

---

>info:>
Template for the pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-api-030`

---

The data that our WEB API will manage needs to be permanently stored somewhere. In this exercise, we will show you how to store them in a [document-oriented database](https://en.wikipedia.org/wiki/Document-oriented_database), specifically in [MongoDB]. At the same time, we will use the [MongoExpress] application, which allows us to manage the connected [MongoDB] from a user interface. We will secure the database with a password, reflecting its real-world use.

Besides a document-oriented database, we could have chosen a [relational database](https://en.wikipedia.org/wiki/Relational_database), a [graph-oriented database](https://en.wikipedia.org/wiki/Graph_database), or a [database geared towards working with time-series data](https://en.wikipedia.org/wiki/Time_series_database). The choice of a database depends on the specific use case and application requirements. In our case, we chose a document-oriented database because of its simplicity and flexibility.

## Local Usage of MongoDB

When using MongoDB, we have several options. For example, we can install the application directly on the local machine or use container technology. Alternatively, we can deploy MongoDB to our local cluster and access the database through the `kubectl proxy-forward` command. Before that, however, we will show you how to start several containerized applications for local development using the [Docker Compose] tool. Our goal is to prepare a configuration that can be easily started in a local environment where the Docker subsystem is available.

1. Create a file `${WAC_ROOT}/ambulance-webapi/deployments/docker-compose/compose.yaml` with the following content:

```yaml
services: 
    mongo_db:
        image: mongo:7.0-rc
        container_name: mongo_db @_important_@
        restart: always
        ports:
        - 27017:27017
        volumes:
        - db_data:/data/db @_important_@
        environment:
            MONGO_INITDB_ROOT_USERNAME: ${AMBULANCE_API_MONGODB_USERNAME}
            MONGO_INITDB_ROOT_PASSWORD: ${AMBULANCE_API_MONGODB_PASSWORD} 
    mongo_express:
        image: mongo-express
        container_name: mongo_express
        restart: always
        ports:
        - 8081:8081
        environment:
            ME_CONFIG_MONGODB_ADMINUSERNAME: ${AMBULANCE_API_MONGODB_USERNAME}
            ME_CONFIG_MONGODB_ADMINPASSWORD: ${AMBULANCE_API_MONGODB_PASSWORD}
            ME_CONFIG_MONGODB_SERVER: mongo_db
            ME_CONFIG_BASICAUTH_USERNAME: mexpress
            ME_CONFIG_BASICAUTH_PASSWORD: mexpress
        links:
        - mongo_db
volumes:
    db_data: {} @_important_@
```

In this specification, we specify that we want to start two services - `mongo-db` and `mongo-express`. The [MongoExpress] service will provide us with a user interface through which we can verify the functionality of our application. In the `volumes` section, we specified the name of a persistent storage, a so-called [_persistent docker volume_](https://docs.docker.com/storage/volumes/), where the data of our database will be stored. The database access credentials will be stored in environment variables.

Create a file `${WAC_ROOT}/ambulance-webapi/deployments/docker-compose/.env` with the following content:

```env
AMBULANCE_API_MONGODB_USERNAME=root
AMBULANCE_API_MONGODB_PASSWORD=neUhaDnes
```

In the directory `${WAC_ROOT}/ambulance-webapi`, execute the following command:

```ps
docker compose --file ./deployments/docker-compose/compose.yaml up
```

Next, go to the page [http://localhost:8081](http://localhost:8081) in your browser. Log in using the credentials specified in the environment variables of Mongo Express in the `compose.yaml` file, `ME_CONFIG_BASICAUTH_USERNAME` and `ME_CONFIG_BASICAUTH_USERNAME`. You should see the [MongoExpress] user interface.

>info:> In case you are unable to connect to the MongoExpress user interface and receive an `Unauthorized` error in the browser, try clearing the `Basic Authentication Details` in the browser.

![Mongo Express User Interface](./img/003-01.MongoExpress.png)

>info:> [Docker Compose] allows you to create more complex configurations providing various additional environment parameters. In our case, we decided to use a simple configuration that is sufficient for local development and captures the main idea of using docker compose.

2. In the MongoExpress user interface, create a new database named `<pfx>-ambulance-wl`. Enter the text `<pfx>-ambulance-wl` in the _Database Name_ field and press the _+ Create Database_ button. Then press the _View_ button next to the name `<pfx>-ambulance-wl`. Enter `ambulances` in the _Collection name_ field and press the `Create collection` button. This sets up our database for further development.

3. We will modify the way our application is started. Open the file `${WAC_ROOT}\ambulance-webapi\scripts\run.ps1` and adjust it:


```ps
...
$env:AMBULANCE_API_PORT="8080"
$env:AMBULANCE_API_MONGODB_USERNAME="root"    @_add_@
$env:AMBULANCE_API_MONGODB_PASSWORD="neUhaDnes"    @_add_@
    @_add_@
function mongo {    @_add_@
    docker compose --file ${ProjectRoot}/deployments/docker-compose/compose.yaml $args    @_add_@
}    @_add_@

switch ($command) {
    "openapi" {
        docker run --rm -ti  -v ${ProjectRoot}:/local openapitools/openapi-generator-cli generate -c /local/scripts/generator-cfg.yaml 
    }
    "start" {
        try {    @_add_@
            mongo up --detach    @_add_@
            go run ${ProjectRoot}/cmd/ambulance-api-service
        } finally {    @_add_@
            mongo down    @_add_@
        }    @_add_@
    }
    "mongo" {    @_add_@
    mongo up    @_add_@
    }    @_add_@
    default {
        throw "Unknown command: $command"
    }
}
```

The command `scripts/run.ps1 start` now starts our application and the database. The command `scripts/run.ps1 mongo` starts only the database.

4. Stop the running process in which Mongo is running and execute the command:

```ps
docker compose --file ./deployments/docker-compose/compose.yaml down
```

This command does the exact opposite of the up variant. All containers created by the `up` command will be stopped and removed. All network interfaces created by the `up` command will be removed.

5. Save the changes and commit them to the git repository. In the directory `${WAC_ROOT}/ambulance-webapi`, execute the commands:

```ps
git add .
git commit -m "Add mongodb compose file"
git push
```
