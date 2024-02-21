# Deployment of Web API on Local Cluster

--- 

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-api-080
```

---

The next step is to deploy the prepared manifests into the local cluster. This time, we will use the manifests prepared in the previous exercise.

>info:> Remember that the [application configuration should be separated from the application source code](https://12factor.net/build-release-run). The manifests in the `ambulance-webapi` repository are therefore only recommended guidelines and not part of your system's configuration. In typical cases, this guideline may be suitable, but in other cases, it may need modification. This example serves only as a demonstration of configuration possibilities for individual components using distributed repositories.

1. Open the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/kustomization.yaml` and insert the following content into it - modify the repository name according to how you named it:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- 'https://github.com/<github-id>/ambulance-webapi//deployments/kustomize/install' # ?ref=v1.0.1
```

>info:> Two consecutive characters `//` separate the repository URL from the repository path. If you want to obtain a specific version - git tag or commit - add `?ref=<tag>` at the end of the URL.

2. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/patches/ambulance-webapi.service.yaml` with the following content:


```yaml
kind: Service
apiVersion: v1
metadata:
name: <pfx>-ambulance-webapi
spec:  
type: NodePort
ports:
- name: http
  protocol: TCP
  port: 80
  nodePort: 30081
```

This file will modify the service definition to make it accessible from the local network on port `30081`.

3. Open the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml` and insert the following content into it:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
...

resources:
- ../../../apps/<pfx>-ambulance-ufe
- ../../../apps/<pfx>-ambulance-webapi @_add_@

components: 
- ../../../components/version-developers
- https://github.com/<github-id>/ambulance-webapi//deployments/kustomize/components/mongodb @_add_@

patches: @_add_@
- path: patches/ambulance-webapi.service.yaml @_add_@
```

Because in our local cluster, we have only one service using [MongoDB], we directly apply the manifests from the `ambulance-gitops` repository. In a shared cluster or if there are multiple services using [MongoDB], we will proceed differently, and the manifests in the `ambulance-webapi` repository will serve as an example configuration.

4. Open the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml` and modify the `api-base` attribute:

```yaml
...
spec:   
  ...
  navigation:
    - element: pfx-ambulance-wl-app    
    ...
      attributes:
        - name: api-base
          value: http://localhost:30081/api @_add_@
        ...
```

By this, we have informed our micro frontend to communicate with the web API on port `30081`.

5. In this step, we will prepare manifests for container registry monitoring. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/ambulance-webapi.image-repository.yaml`:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: ambulance-webapi
  namespace: wac-hospital
spec:
  image: <docker-id>/ambulance-wl-webapi
  interval: 1m0s
```

Next, create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/ambulance-webapi.image-policy.yaml`:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: ambulance-webapi
  namespace: wac-hospital
spec:
  imageRepositoryRef:
    name: ambulance-webapi # referuje ImageRepository z predchádzajúceho kroku 
  filterTags:
    pattern: "main.*" # vyberie všetky verzie, ktoré začínajú na main- (napr. main-20240315.1200)
  policy:
    alphabetical:
      order: asc
```

Modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/kustomization.yaml`:

```yaml
...
resources:
..
- ambulance-ufe.image-repository.yaml
- ambulance-ufe.image-policy.yaml
- ambulance-webapi.image-repository.yaml @_add_@
- ambulance-webapi.image-policy.yaml @_add_@
...
```

And finally modify the file `${WAC_ROOT}/ambulance-gitops/components/version-developers/kustomization.yaml`:

```yaml
images:
- name: <docker-id>/ambulance-wl-webapi @_add_@
  newName: <docker-id>/ambulance-wl-webapi # {"$imagepolicy":  "wac-hospital:ambulance-webapi:name"} @_add_@
  newTag: main # {"$imagepolicy": "wac-hospital:ambulance-webapi:tag"} @_add_@

...
```

6. Open command line in `${WAC_ROOT}/ambulance-gitops` and validate the configuration using:

```ps
kubectl kustomize clusters/localhost/install
```

The output should be the manifests without any error messages.

7. Archive your changes.

```ps
git add .
git commit -m 'added webapi to localhost cluster'
git push
```

Make sure your changes were applied by flux on the cluster:

```ps
kubectl get pods  -n wac
```

8. Currently, our frontend is secured so that it allows requests to be loaded only from the same host. In order to be able to access the API on a different port, we have to modify the server's CSP header. Add patch for CSP header configuration to our local cluster. In the `clusters/localhost/prepare/kustomization.yaml` file, add the following lines:

```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  ...
  patches: 
  ...
  - patch: |- @_add_@
      - op: add @_add_@
        path: "/spec/template/spec/containers/0/env/-" @_add_@
        value: @_add_@
          name: "HTTP_CSP_HEADER" @_add_@
          value: "default-src 'self' 'unsafe-inline' https://fonts.googleapis.com/ https://fonts.gstatic.com/; font-src 'self' https://fonts.googleapis.com/ https://fonts.gstatic.com/; script-src 'nonce-{NONCE_VALUE}'; connect-src 'self' localhost:30331 localhost:30081" @_add_@
    target: @_add_@
      group: apps @_add_@
      version: v1 @_add_@
      kind: Deployment @_add_@
      name: ufe-controller @_add_@
  components:
  ...
```

This file modifies the deployment definition so that it is created with a CSP header that allows access to the local API on port `30081`.

9. In the browser, open the page [http://localhost:30331](http://localhost:30331), where you will see an application envelope with an integrated micro application. The micro app will try to load the data from the webapi, but it doesn't exist yet. Create them using the displayed interface. Try restarting your cluster and verify that the data is still available.

<hr/>

With this step, we have both front-end and webapi deployed in the local cluster. The disadvantage of this deployment is that these services are available on different ports, which from the browser's point of view are considered different instances of the server. Moreover, this approach would be problematic when deployed on a public URL, because most networks implicitly block HTTP access to ports other than ports 80 and 443. In the next part, we will explain how to solve this problem and build a so-called [Service Mesh] from individual microservices, which as a result it will form one consistent whole even from the user's point of view.

