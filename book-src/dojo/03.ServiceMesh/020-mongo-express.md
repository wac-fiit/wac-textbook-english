# (Optional - Independent Work) Deployment of Mongo Express

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-030
```

---

When developing an application, it's convenient to have the ability to monitor the database's status. For this purpose, you can use the tool [Mongo Express][mongoexpress], which can be deployed into a Kubernetes cluster. At this point, you already have all the necessary knowledge to accomplish this independently. Try deploying Mongo Express into the Kubernetes cluster based on the information you have, so that this application is served at the path `/mongo-express`. Afterwards, compare the result with the procedure provided here, which also includes deploying access to the application through a micro frontend application.

1. In the directory `${WAC_ROOT}/ambulance-gitops/apps/mongo-express`, create a file `deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:  
  name: &PODNAME mongo-express
  annotations: 
spec:
  replicas: 1  
  selector:
    matchLabels:
      pod: *PODNAME
  template:
    metadata:
      labels: 
        pod: *PODNAME
    spec:
      containers:
      - image: mongo-express
        name: mongo-express
        env:
        - name: ME_CONFIG_SITE_BASEURL @_important_@
          value: /mongo-express/ @_important_@
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:  
            secretKeyRef: 
              name: mongodb-auth @_important_@
              key: username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:  
            secretKeyRef: 
              name: mongodb-auth @_important_@
              key: password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-connection @_important_@
              key: host
        # authentication and authorization at cluster level
        - name: ME_CONFIG_BASICAUTH_USERNAME
          value: "" @_important_@
        - name: ME_CONFIG_BASICAUTH_PASSWORD
          value: "" @_important_@
        ports:
        - name: http
          containerPort: 8081
        resources:
          limits:
            cpu: '1'
            memory: '512M'
          requests:
            cpu: '0.01'
            memory: '128M'
```

Notice that we reference the `mongodb-auth` secret and `mongodb-connection` ConfigMap. We have already taken these objects from the configuration of the webapi service. Here, we leverage the fact that we suppressed the generation of a hashed name for these objects. By setting the environment variables `ME_CONFIG_BASICAUTH_...` to an empty string, we simultaneously suppress authentication when accessing this service. We assume that authentication and authorization will be secured at the Kubernetes cluster level - see the following chapters. By setting the environment variable `ME_CONFIG_SITE_BASEURL`, we configure the path for the [&lt;base&gt; element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base), ensuring correct loading of resources for this application.

2. Create the file `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/service.yaml` with the following content:


```yaml
apiVersion: v1
kind: Service
metadata:
  name: &SVCNAME mongo-express
spec:
  ports:
  - name: http
    protocol: TCP
    port: 8081
    targetPort: 8081
  selector:
    pod: *SVCNAME
```

3. Create the file `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/http-route.yaml`, which will route requests to the path `/mongo-express` to the `mongo-express` service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mongo-express
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mongo-express
      backendRefs:
        - group: ""
          kind: Service
          name: mongo-express
          port: 8081
```

4. Create the file `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/webcomponent.yaml` with the following content:

```yaml
apiVersion: fe.milung.eu/v1
kind: WebComponent
metadata: 
  name: mongo-express
spec:
  module-uri: built-in  @_important_@
  navigation:
    - element: ufe-frame @_important_@
      path: mongo-express
      title: Mongo Express
      details: UI Access to local Mongo DB
      attributes:
        - name: src
          value: /mongo-express @_important_@
```

This configuration will expose the [MongoExpress] application as part of our micro frontend interface. The application itself will be accessible through an `iframe` element with the `src="/mongo-express"` attribute set.

5. Create the file `${WAC_ROOT}/ambulance-gitops/apps/mongo-express/kustomization.yaml` with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- http-route.yaml
- webcomponent.yaml
```

6. Modify the file: `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml`

```yaml
...
resources:
- ../../../apps/pfx-ambulance-ufe
- ../../../apps/pfx-ambulance-webapi
- ../../../apps/mongo-express @_add_@
...
```

7. Open a command prompt in the directory `${WAC_ROOT}/ambulance-gitops` and verify the correctness of your configuration.

```ps
kubectl kustomize clusters/localhost/install
```

8. Archive the changes to the git repository and submit them to the remote repository.

```ps
git add .
git commit -m "Mongo Express deployment"
git push
```

After applying the changes with [FluxCD][flux], go to the page [http://localhost/ui](http://localhost/ui) and verify that the [Mongo Express][mongoexpress] application is also displayed.
