# Creating reusable manifests for deploying the WEB API microservice into a Kubernetes cluster

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-api-060
```

---

The next logical step after creating the continuous integration pipeline is to ensure the continuous deployment of our WEB API service and its integration with the front-end application. We will handle continuous deployment using gitops, similar to what was done for the front-end application. During deployment, we will also demonstrate some techniques to extend our Kubernetes deployment/pod with additional services.

In this exercise, we will only partially address the integration itself. The complete integration into the target cluster will be shown in the next exercise, where we will create a [Service Mesh](https://buoyant.io/service-mesh-manifesto) solution.

Unlike the first exercise, we won't start creating our manifests directly in the `${WAC_ROOT}/ambulance-gitops` repository. Instead, we will create them as part of our project in the `${WAC_ROOT}/ambulance-webapi` directory. However, here we won't create them as system configuration but rather as a library for creating configuration in target environments.

1. Create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <pfx>-ambulance-webapi 
spec:
  replicas: 1
  selector:
      matchLabels:
        pod: <pfx>-ambulance-webapi-label 
  template:
      metadata:
        labels:
          pod: <pfx>-ambulance-webapi-label 
      spec:
        containers:
        - name: <pfx>-ambulance-wl-webapi-container 
          image: <pfx>/ambulance-wl-webapi:latest 
          imagePullPolicy: Always
          ports:
          - name: webapi-port
            containerPort: 8080
          env:
            - name: AMBULANCE_API_ENVIRONMENT
              value: production
            - name: AMBULANCE_API_PORT
              value: "8080"
            - name: AMBULANCE_API_MONGODB_HOST
              value: mongodb
            - name: AMBULANCE_API_MONGODB_PORT
              value: "27017"
              # change to actual value
            - name: AMBULANCE_API_MONGODB_USERNAME
              value: ""
              #change to actual value
            - name: AMBULANCE_API_MONGODB_PASSWORD
              value: ""
            - name: AMBULANCE_API_MONGODB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: <pfx>-ambulance-webapi-config
                  key: database
            - name: AMBULANCE_API_MONGODB_COLLECTION
              valueFrom:
                configMapKeyRef:
                  name: <pfx>-ambulance-webapi-config 
                  key: collection
            - name: AMBULANCE_API_MONGODB_TIMEOUT_SECONDS
              value: "5"
          resources:
            requests:
              memory: "64Mi"
              cpu: "0.01"
            limits:
              memory: "512Mi"
              cpu: "0.3"

```

We are already familiar with the structure of the manifest from the previous manifest for the front-end application. In this case, however, we have added the definition of environment variables that will be used when starting the container. Listing all variables will facilitate the work for users who are not familiar with the implementation of our service. All values in the `env` section must be of type string, so it is necessary to include numeric values in quotes as well. In the definition of environment variables, we also used references to configuration maps - [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/), which allows us to share settings among different containers.

2. One of the requirements for our service is the ability to get an overview of the functionality of the WEB API service and the option to try it out. For this, we will use the containerized application image [swaggerapi/swagger-ui](https://hub.docker.com/r/swaggerapi/swagger-ui). We can deploy it independently as an optional component, or we can deploy it as a [sidecar](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar) to our service. We choose the second option and add another container to the list of containers in our pod.

>info:> All containers within the same pod behave as if they were on the same device within the virtual network created for that cluster

Add the following code to the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/deployment.yaml`:

```yaml
...
    spec:
        containers:
        - name: <pfx>-ambulance-wl-webapi-container
        ...
        - name: openapi-ui   @_add_@
          image: swaggerapi/swagger-ui   @_add_@
          imagePullPolicy: Always   @_add_@
          ports:   @_add_@
          - name: api-ui   @_add_@
            containerPort: 8081   @_add_@
          env:           @_add_@
            - name: PORT   @_add_@
              value: "8081"   @_add_@
            - name:   URL   @_add_@
              value: /openapi   @_add_@
            - name: BASE_URL   @_add_@
              value: /openapi-ui   @_add_@
            - name: FILTER   @_add_@
              value: 'true'   @_add_@
            - name: DISPLAY_OPERATION_ID   @_add_@
              value: 'true'   @_add_@
          resources:   @_add_@
              requests:   @_add_@
                  memory: "16M"   @_add_@
                  cpu: "0.01"   @_add_@
              limits:   @_add_@
                  memory: "64M"   @_add_@
                  cpu: "0.1"   @_add_@
```

The address `http://localhost:8080/.openapi/`, from the pod's perspective, points to the same device, meaning the same pod, but using a different socket port.

3. Our application assumes the existence of a database specified by the environment variable `AMBULANCE_API_MONGODB_DATABASE` and the existence of a document collection named by the environment variable `AMBULANCE_API_MONGODB_COLLECTION`. To avoid the need for manual creation of this database/collection, we will use an [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/). Init container processes are executed before the start of the processes of the actual containers in the pod, which are created only if the init container processes have completed with an exit code indicating successful execution. In our case, we want to execute a process that verifies whether the corresponding database and data collection exist, and if not, create them and insert initial data into the database. For this, we will use the [MongoDb-Shell](https://www.mongodb.com/docs/mongodb-shell/), which is part of the [mongo](https://hub.docker.com/_/mongo) container. Add the following code to the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/deployment.yaml`:

>info:> An alternative approach would be to create a manifest for a [Kubernetes job](https://kubernetes.io/docs/concepts/workloads/controllers/job/). In this case, however, we would have to address synchronization between the job and the container with our service and the container containing the MongoDB database, to ensure that the job starts whenever a new instance of the database is created and before our container is started. In some cases, these assumptions can be considered fulfilled.

```yaml
...
spec:
  ...
  template:
      ...
      spec:
        volumes:    @_add_@
        - name: init-scripts     @_add_@
          configMap:    @_add_@
            name: <pfx>-ambulance-webapi-mongodb-init @_add_@
        initContainers:        @_add_@
        - name: init-mongodb        @_add_@
          image: mongo:latest        @_add_@
          imagePullPolicy: Always        @_add_@
          command: ['mongosh', "--nodb", '-f', '/scripts/init-db.js']        @_add_@
          volumeMounts:        @_add_@
          - name: init-scripts        @_add_@
            mountPath: /scripts        @_add_@
          env:        @_add_@
            - name: AMBULANCE_API_PORT    @_add_@
              value: "8080"   @_add_@
            - name: AMBULANCE_API_MONGODB_HOST    @_add_@
              value: mongodb    @_add_@
            - name: AMBULANCE_API_MONGODB_PORT    @_add_@
              value: "27017"    @_add_@
            - name: AMBULANCE_API_MONGODB_USERNAME    @_add_@
              value: ""   @_add_@
            - name: AMBULANCE_API_MONGODB_PASSWORD    @_add_@
              value: ""   @_add_@
            - name: AMBULANCE_API_MONGODB_DATABASE     @_add_@
              valueFrom:     @_add_@
                configMapKeyRef:     @_add_@
                  name: <pfx>-ambulance-webapi-config     @_add_@
                  key: database     @_add_@
            - name: AMBULANCE_API_MONGODB_COLLECTION     @_add_@
              valueFrom:     @_add_@
                configMapKeyRef:     @_add_@
                  name: <pfx>-ambulance-webapi-config      @_add_@
                  key: collection     @_add_@
            - name: RETRY_CONNECTION_SECONDS    @_add_@
              value: "5"    @_add_@
          resources:        @_add_@
            requests:        @_add_@
              memory: "16Mi"        @_add_@
              cpu: "0.01"        @_add_@
            limits:        @_add_@
              memory: "64Mi"        @_add_@
              cpu: "0.1"        @_add_@
        containers:
        ...
```

The initialization process is defined by the command `command: ['mongosh', "--nodb", '-f', '/scripts/init-db.js']`, referring to the initialization script in the file `/scripts/init-db.js`. This file must be available within the container, so it needs to be added as a shared object using `volumeMounts`, mapping to the directory `/scripts` content of the object `<pfx>-ambulance-webapi-mongodb-init`.

For now, let's save the initialization script itself to the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/params/init-db.js` and it contains the following code:

```js
const mongoHost = process.env.AMBULANCE_API_MONGODB_HOST
const mongoPort = process.env.AMBULANCE_API_MONGODB_PORT

const mongoUser = process.env.AMBULANCE_API_MONGODB_USERNAME
const mongoPassword = process.env.AMBULANCE_API_MONGODB_PASSWORD

const database = process.env.AMBULANCE_API_MONGODB_DATABASE
const collection = process.env.AMBULANCE_API_MONGODB_COLLECTION

const retrySeconds = parseInt(process.env.RETRY_CONNECTION_SECONDS || "5") || 5;

// try to connect to mongoDB until it is not available
let connection;
while(true) {
    try {
        connection = Mongo(`mongodb://${mongoUser}:${mongoPassword}@${mongoHost}:${mongoPort}`);
        break;
    } catch (exception) {
        print(`Cannot connect to mongoDB: ${exception}`);
        print(`Will retry after ${retrySeconds} seconds`)
        sleep(retrySeconds * 1000);
    }
}

// if database and collection exists, exit with success - already initialized
const databases = connection.getDBNames()
if (databases.includes(database)) {    
    const dbInstance = connection.getDB(database)
    collections = dbInstance.getCollectionNames()
    if (collections.includes(collection)) {
      print(`Collection '${collection}' already exists in database '${database}'`)
        process.exit(0);
    }
}

// initialize
// create database and collection
const db = connection.getDB(database)
db.createCollection(collection)

// create indexes
db[collection].createIndex({ "id": 1 })

//insert sample data
let result = db[collection].insertMany([
    {
        "id": "bobulova",
        "name": "Dr.Bobulová",
        "roomNumber": "123",
        "predefinedConditions": [
            { "value": "Nádcha", "code": "rhinitis" },
            { "value": "Kontrola", "code": "checkup" }
        ]
    }
]);

if (result.writeError) {
    console.error(result)
    print(`Error when writing the data: ${result.errmsg}`)
}

// exit with success
process.exit(0);
```

This code reads the environment configuration and attempts to connect to the database. Upon successful connection, it verifies whether the prescribed database and collection exist, creating them and populating them with initial data if needed.

4. Next, create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/service.yaml` with the following content, specifying the definition of the [_Service_](https://kubernetes.io/docs/concepts/services-networking/service/) object. This will be used to provide access to our service from other objects within the Kubernetes cluster:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: <pfx>-ambulance-webapi   
spec:  
  selector:
    pod: <pfx>-ambulance-webapi-label
  ports:
  - name: http
    protocol: TCP
    port: 80  
    targetPort: webapi-port
```

5. Now, create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/kustomization.yaml`, which includes the previously mentioned configurations. Additionally, in this file, we will define a [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/) object for the database initialization script and a [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/) object for the configuration of our service:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

configMapGenerator:
  - name: <pfx>-ambulance-webapi-mongodb-init 
    files:
      - params/init-db.js
  - name: <pfx>-ambulance-webapi-config
    literals:
      - database=<pfx>-ambulance
      - collection=ambulance
patches:
- path: patches/webapi.deployment.yaml
  target:
    group: apps
    version: v1
    kind: Deployment
    name: <pfx>-ambulance-webapi
```

6. As we are creating reusable configuration, let's also create configuration for deploying MongoDB into the cluster. In this case, however, we don't know if the [MongoDB] database is already available on the preinstalled system or if it will be accessible in another form. Therefore, we define the configuration as a [Kustomize system component](https://kubectl.docs.kubernetes.io/guides/config_management/components/). Create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/components/mongodb/deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: &PODNAME mongodb   @_important_@
spec:
  replicas: 1
  selector:
      matchLabels:
        pod: *PODNAME  @_important_@
  template:
      metadata:
        labels:
          pod: *PODNAME  @_important_@
      spec:
        volumes: 
        - name: db-data
          persistentVolumeClaim: @_important_@
            claimName: mongo-pvc @_important_@
        containers:
        - name: *PODNAME
          image: mongo:latest
          imagePullPolicy: Always
          ports:
          - name: mongodb-port
            containerPort: 27017
          volumeMounts: @_important_@
          - name: db-data
            mountPath: /data/db
          env:
          - name: MONGO_INITDB_ROOT_USERNAME
            valueFrom:
                secretKeyRef: 
                name: mongodb-auth
                key: username
          - name: MONGO_INITDB_ROOT_PASSWORD
            valueFrom:
                secretKeyRef: 
                name: mongodb-auth
                key: password
          resources:
            requests:
              memory: "1Gi"
              cpu: "0.1"
            limits:
              memory: "4Gi"
              cpu: "0.5"
```

Notice how we declare the pod name, which is used in the configuration as a YAML variable. Also, note that we used the [_PersistentVolumeClaim_](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) object to provide persistent storage for the database. Additionally, to obtain authorization values for database access, we used values loaded from objects of type [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/), and some environment variables are obtained from various objects of type [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/), allowing for centralized configuration and usage in various places within our configuration.

Create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/components/mongodb/pvc.yaml` with the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  # storageClassName: default 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

The [_PersistentVolumeClaim_](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) object is used to define requirements for persistent storage. The storage object itself must be provided either by manual allocation of space using the [_PersistentVolume_](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes) object or by using the [_StorageClass_](https://kubernetes.io/docs/concepts/storage/storage-classes/) object. The storage object is specified by the `storageClassName` property. In our case, we used the predefined `default` object, which is available on most Kubernetes installations. If we wanted to use a different object, we would need to create it first.

Now, create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/components/mongodb/service.yaml` with the following content:


```yaml
kind: Service
apiVersion: v1
metadata:
    name: &PODNAME mongodb
spec:  
    selector:
      pod: *PODNAME
    ports:
    - name: mongo
      protocol: TCP
      port: 27017
    targetPort: mongodb-port
```

And finally, create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/components/mongodb/kustomization.yaml` with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- deployment.yaml
- service.yaml
- pvc.yaml

configMapGenerator:
- name: mongodb-connection
  options:   @_important_@
    disableNameSuffixHash: true   @_important_@
  literals:
    - host=mongodb
    - port=27017

secretGenerator:
- name: mongodb-auth
  options:   @_important_@
    disableNameSuffixHash: true   @_important_@
  literals:
  - username=admin
  - password=admin
```

Apart from the reference to the source configuration manifests, this file also contains the declaration of a [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/) and [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/). Note that these declarations have the default set to `disableNameSuffixHash: true`. Under normal circumstances, a generated configuration map includes a hash of its content, and the modified name is replaced everywhere it is used. This usually allows for an automatic pod restart when the manifest is changed—because the content change also changes the map name, and subsequently, the Deployment manifest where this map is used. In some cases, if the map is used in different manifest structures - which will be our case later when deploying to a shared cluster, it needs an abstract reference to the map or Secret with unknown content in advance. So, this option allows us to generate a map with a fixed name, which will be used in various manifest structures.

Furthermore, our file includes a reference to the modification manifest for deploying our webapi - `patches/webapi.deployment.patch.yaml`. This modification is necessary so that we can use values from the configuration map and [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/) object in configuring our service. In this case, we implement the modification using a [JSONPatch] manifest. The reason is mainly that we need to delete the original `value` properties in the environment variable definitions.

>info:> An alternative would be to create two [_strategic merge_](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patchesstrategicmerge/) patches, where the first would delete the original entries in the `env` section using the `$patch: delete` directive, and the second patch would add them using entries with the `valueFrom` property. For the sake of demonstrating the use of [JSONPatch], we did not choose this method, although it might be more appropriate for long-term development.

Create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/patches/webapi.deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <pfx>-ambulance-webapi 
spec:
  template:
  spec:
      initContainers:
        - name: init-mongodb
          env:
            - name: AMBULANCE_API_MONGODB_HOST
              value: null     @_important_@
              valueFrom:
                configMapKeyRef:
                  name: mongodb-connection
                  key: host
            - name: AMBULANCE_API_MONGODB_PORT
              value: null    @_important_@
              valueFrom:
                configMapKeyRef:
                  name: mongodb-connection
                  key: port
            - name: AMBULANCE_API_MONGODB_USERNAME
              value: null    @_important_@
              valueFrom:
                secretKeyRef: 
                  name: mongodb-auth
                  key: username
            - name: AMBULANCE_API_MONGODB_PASSWORD
              value: null    @_important_@
              valueFrom:
                secretKeyRef: 
                  name: mongodb-auth
                  key: password
      containers:
        - name: <pfx>-ambulance-wl-webapi-container 
          env:
            - name: AMBULANCE_API_MONGODB_HOST
              value: null    @_important_@
              valueFrom:
                configMapKeyRef:
                  name: mongodb-connection
                  key: host
            - name: AMBULANCE_API_MONGODB_PORT
              value: null    @_important_@
              valueFrom:
                configMapKeyRef:
                  name: mongodb-connection
                  key: port
            - name: AMBULANCE_API_MONGODB_USERNAME
              value: null    @_important_@
              valueFrom:
                secretKeyRef:
                  name: mongodb-auth
                  key: username
            - name: AMBULANCE_API_MONGODB_PASSWORD
              value: null    @_important_@
              valueFrom:
                secretKeyRef:
                  name: mongodb-auth
                  key: password
```

Essentially, it involves changing the values of `env` in the init container and in the webapi container in our original object `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install/deployment.yaml` with modified values of the respective environment variables, which are loaded from the configuration map and [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/). Note that we set the original `value:` fields to `null` to remove them from the resulting manifest.

Since this is a [_Component_](https://kubectl.docs.kubernetes.io/guides/config_management/components/) type configuration, the original object doesn't have to be part of this component's configuration, i.e., it doesn't have to be declared in any of the original objects loaded under the `resources` section, but it can be defined as part of another configuration that uses our component.

Finally, let's create a new configuration that uses this component. Create the file `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/with-mongo/kustomization.yaml` with the following content:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../install

components:
- ../components/mongodb
```

This configuration is derived from the one in the directory `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install`, to which we apply the component `${WAC_ROOT}/ambulance-webapi/deployments/kustomize/components/mongodb`.

7. Verify the correctness of the configuration by executing the following commands:

```ps
kubectl kustomize ${WAC_ROOT}/ambulance-webapi/deployments/kustomize/install
kubectl kustomize ${WAC_ROOT}/ambulance-webapi/deployments/kustomize/with-mongo
```

And archive your code.

```ps
git add .
git commit -m "Added kustomize configuration for webapi"
git push
```

In your browser, go to your repository on the [GitHub] page. In the repository, navigate to the _Code_ tab, click on the _1 tag_ link, and then click on the _Create new release_ button. Enter `v1.0.1` in the _Choose tag_ field, enter `v1.0.1` in the _Release Title_ field, and enter `Manifests for webapi deployment` in the _Describe this release_ field. Click on the _Publish release_ button.

>info:> If you have already created a tag with a higher semantic version, use a tag that continues the numbering.

>warning:> In a real environment, the correctness of the manifests must be verified by deploying them to a test cluster as part of automated integration tests.
