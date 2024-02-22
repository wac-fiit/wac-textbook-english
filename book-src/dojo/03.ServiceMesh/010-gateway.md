# Routing Requests Using Gateway API

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-010
```

---

Our services are served from different subdomains - different host addresses. The frontend is deployed at `http://localhost:30331`, and the WebAPI at `http://localhost:30081`. In larger systems with dozens of microservices, this approach would be unsustainable. Most networks filter the HTTP protocol on ports other than `80` or `443`, meaning each microservice must have its own subdomain. This would require managing dozens of domains, securing them, and reduce flexibility in system evolution. One way to address this problem is to integrate a "reverse proxy," sometimes referred to as a "Gateway" or "API Gateway." The terms differ, as does the functionality included in these services. For our purpose, it is essential for this service to be able to route requests to individual microservices based on specific rules, typically based on the URL path specified in the request.

Typical representatives of such reverse proxies are applications like [nginx](https://www.nginx.com) and [Envoy Proxy](https://www.envoyproxy.io/), but they are by no means the only ones. Since routing requests is one of the fundamental functionalities in orchestrating microservices, the Kubernetes team introduced the standardized API called [Ingress], but left its implementation to third parties. However, this API did not cover all real-world cases in its basic version, so individual implementations extend the API with proprietary annotations.

Based on experiences from implementing [Ingress], a new set of [Gateway API] has started to be prepared. Although [Ingress] is currently the most widely used way to control request routing in Kubernetes systems, we will demonstrate an approach based on [Gateway API].

[Gateway API] provides several objects for configuring systems. In production deployment, it is assumed that the `Gateway Class` type will be provided by the infrastructure provider, such as the operator of a public data center. The `Gateway` type will be provided by the cluster administrator, such as the customer's IT department. Application developers will typically provide objects of type `HTTPRoute`. However, for local development purposes, it is necessary to deploy all types of objects.

## Deploying Gateway API

1. In this step, we will create a configuration for deploying [Envoy Gateway], an implementation of [Gateway API]. Create a directory `${WAC_ROOT}/ambulance_gitops/infrastructure/envoy-gateway`, and within it, create a file `gateway-class.yaml` with the following content:


```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: wac-hospital-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

The [Envoy Gateway] controller allows the assignment of exactly one instance of [GatewayClass](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/). It is possible to deploy multiple [Envoy Gateway] controllers, but each must have a different `controllerName`. In our case, we will use the name `gateway.envoyproxy.io/gatewayclass-controller`, which is used in the standard configuration of [Envoy Gateway].

2. Next, create the file: `${WAC_ROOT}/ambulance_gitops/infrastructure/envoy-gateway/gateway.yaml` with the following content:


```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
name: wac-hospital-gateway
namespace: wac-hospital
spec:
gatewayClassName: wac-hospital-gateway-class
listeners:
  - name: http
    protocol: HTTP
    port: 80
```

This configuration will create, within the cluster, an endpoint where the [Gateway API] implementation will wait for incoming requests and process them according to the rules defined in objects of type [`HTTPRoute`](https://gateway-api.sigs.k8s.io/api-types/httproute/).

3. Create the file: `${WAC_ROOT}/ambulance_gitops/infrastructure/envoy-gateway/kustomization.yaml` with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/envoyproxy/gateway/releases/download/latest/install.yaml
- gateway-class.yaml
- gateway.yaml
```

This manifest contains the objects necessary for deploying [Envoy Gateway], implementing [Gateway API], and preparing the cluster for the deployment of individual microservices.

Modify the file: `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`

```yaml
...

resources:
...
- ../../../infrastructure/envoy-gateway @_add_@

patches: 
...
```

5. Modify the file: `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/prepare.kustomization.yaml`

```yaml
...
spec:
  wait: true @_remove_@
  # targeted healthcheck
  healthChecks:    @_add_@
    - apiVersion: apps/v1    @_add_@
      kind: Deployment    @_add_@
      name: envoy-gateway    @_add_@
      namespace: envoy-gateway-system    @_add_@
    - apiVersion: apps/v1    @_add_@
      kind: Deployment    @_add_@
      name: ufe-controller    @_add_@
      namespace: wac-hospital         @_add_@
  interval: 42s
  ...
```

The reason for this modification is that [Flux CD] cannot correctly evaluate the deployment status of objects of type [_Job_](https://kubernetes.io/docs/concepts/workloads/controllers/job/), which are deleted from the Kubernetes API after successful execution. The `wait: true` property was specifying that it should check if all objects that are part of the configuration are in the _Ready_ state. The [Envoy Gateway] implementation also includes objects of type _Job_ that are executed, and Flux CD cannot subsequently verify whether they are in the _Ready_ state. With the above modification, we turned off this property and instead added an explicit list of objects to be verified.

6. Save the changes to the remote repository:

```ps
git add .
git commit -m "Add Gateway API"
git push
```

After [Flux CD] applies the changes, you can verify whether the objects were successfully deployed:

```ps
kubectl get gatewayclass
kubectl get gateway
```

and optionally open the page `http://localhost` in the browser, which will currently only provide a `404 Not Found` error message.

## Routing Requests

In our cluster, we have the following services available that we could connect to from an external network:

- **ufe-controller** - provides our user interface
- **ambulance-webapi** - provides an interface for accessing data
- **swagger-ui** - provides a description of our web API

We will gradually create routes - `HTTPRoute` - for all services that will be accessible from the external network.

1. Create the file: `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller/http-route.yaml`

```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: ufe-controller
  spec:
    parentRefs:
      - name: wac-hospital-gateway
        namespace: wac-hospital
      - name: wac-hospital-gateway  # Hack to make it work for common cluster
        namespace: wac-hospital-system
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /ui
        backendRefs:
          - group: ""
            kind: Service
            name: ufe-controller
            port: 80
        
      - matches:
        - path:
            type: Exact
            value: /
        filters:
        - type: RequestRedirect
          requestRedirect:
            path:
              type: ReplaceFullPath
              replaceFullPath: /ui
            scheme: https
            port: 443
```

In general, each request must be processed by one or none of the rules specified in the `HTTPRoute` objects for a given `Gateway` object. This manifest specifies that all requests with a path starting with the `/ui` segment will be redirected to the `ufe-controller` service. Requests to the root document `/` will be returned to the client with a `303 - Redirect` status, redirecting to the path `/ui`.

There is a need to modify the [_base URL_] for the `ufe-controller` service, which will now be accessible from the root directory '/' of requests but from the subdirectory '/ui/'. Create the file: `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller/patches/ufe-controller.deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ufe-controller
spec:
  template: 
    spec:
      containers:
      - name: ufe-controller
        env:
          - name: BASE_URL
            value: /ui/
```

and finally, modify the file: `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller/kustomization.yaml`:

```yaml
...
resources:
- https://github.com/milung/ufe-controller//configs/k8s/kustomize
- http-route.yaml @_add_@
@_add_@
patches: @_add_@
- path: patches/ufe-controller.deployment.yaml @_add_@
```

2. Create the file: `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/http-route.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: <pfx>-ambulance-webapi
spec:
parentRefs:
  - name: wac-hospital-gateway
    namespace: wac-hospital-system
  - name: wac-hospital-gateway  # Hack to make it work for common cluster
    namespace: wac-hospital-system
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /<pfx>-api
    filters: 
      - type: URLRewrite    @_important_@
        urlRewrite:    @_important_@
          path:    @_important_@
            type: ReplacePrefixMatch    @_important_@
            replacePrefixMatch: /api    @_important_@
    backendRefs:
      - group: ""
        kind: Service
        name: <pfx>-ambulance-webapi
        port: 80
```

This manifest specifies that all requests with a path starting with the `/<pfx>-api` segment will be redirected to the `<pfx>-ambulance-webapi` service. Note the `filters` section - here, we specify that the path `/<pfx>-api` should be replaced with `/api` before the request is handed over to the `<pfx>-ambulance-webapi` service. This is necessary because the `<pfx>-ambulance-webapi` service expects API requests to be sent to the `/api` path.

Next, add the new rule to the same file:


```yaml
...
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /<pfx>-api
      ...
    - matches:    @_add_@
        - path:    @_add_@
            type: PathPrefix    @_add_@
            value: /<pfx>-openapi-ui    @_add_@
      backendRefs:    @_add_@
        - group: ""    @_add_@
          kind: Service    @_add_@
          name: <pfx>-openapi-ui    @_add_@
          port: 80    @_add_@
```

Requests to the path `/<pfx>-openapi-ui` will be redirected to the `<pfx>-openapi-ui` service, which we still need to configure. This is a service that will expose the [Swagger UI] container configured in our webapi service manifests. Create the file: `${WAC_ROOT}/ambulance-gitops/infrastructure/<pfx>-ambulance-webapi/openapi-ui.service.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: <pfx>-openapi-ui
spec:  
  selector:
    pod: <pfx>-ambulance-webapi-label
  ports:
  - name: http
    protocol: TCP
    port: 80  
    targetPort: 8081
```

Once again, modify the file: `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/http-route.yaml`, this time adding handling for the path `/<pfx>-openapi` from which Swagger UI will load the OpenAPI specification:

```yaml
...
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /<pfx>-api
      ...
    - matches:
        - path:
            type: PathPrefix
            value: /<pfx>-openapi-ui
      ...
    - matches:   @_add_@
        - path:   @_add_@
            type: Exact   @_add_@
            value: /<pfx>-openapi   @_add_@
      filters:    @_add_@
      - type: URLRewrite   @_add_@
        urlRewrite:   @_add_@
          path:   @_add_@
            type: ReplaceFullPath   @_add_@
            replaceFullPath: /openapi   @_add_@
      backendRefs:   @_add_@
        - group: ""   @_add_@
          kind: Service   @_add_@
          name: <pfx>-ambulance-webapi   @_add_@
          port: 80   @_add_@
```

Since we have modified the path from which [Swagger UI] will load the OpenAPI specification and where it will be handled, compared to the manifests in the `ambulance-webapi` repository, we need to adjust the configuration of the [Swagger UI] container. These modifications are necessary to deploy our services to a shared cluster without conflicts with the services of other students. Create the file: `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/patches/ambulance-webapi.deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <pfx>-ambulance-webapi 
spec:
  template:
    spec:
      containers:
        - name: openapi-ui
          env:
            - name: URL
              value: /<pfx>-openapi
            - name: BASE_URL
              value: /<pfx>-openapi-ui
```

Finally, modify the file: `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/kustomization.yaml`:

```yaml
...
resources:
- 'https://github.com/milung/ambulance-webapi//deployments/kustomize/install' # ?ref=v1.0.1
- openapi-ui.service.yaml @_add_@
- http-route.yaml @_add_@
@_add_@
patches: @_add_@
- path: patches/ambulance-webapi.deployment.yaml @_add_@
```

>homework:> Modify the manifests in the `ambulance-webapi` repository to include the added configurations for the _Service_ object `<pfx>-openapi` and the _HTTPRoute_ object `<pfx>-ambulance-webapi`, where the _HTTPRoute_ object will be an optional configuration component `with-gateway-api`. Subsequently, adjust the configuration in the `ambulance-gitops` repository to apply these configurations.

3. Finally, we will modify the declaration of our micro frontend application so that its `api-base` attribute points to the path of the webapi on the same host machine:

```yaml
...
  navigation:
    - element: <pfx>-ambulance-wl-app
      ...
      attributes:
        - name: api-base
          value: http://localhost:5000/api @_remove_@
          # use absolute path on the same host
          value: /<pfx>-api @_add_@
...
```

4. Verify the correctness of your configuration:

```ps
kubectl kustomize clusters/localhost/prepare
kubectl kustomize clusters/localhost/install
kubectl kustomize clusters/localhost
```

5. Archive the changes to the remote repository.

```ps
git add .
git commit -m "Add HTTPRoutes"
git push
```

After [flux] applies changes in your local cluster, open the page [http://localhost](http://localhost) in your browser. You should see our application, which is capable of communicating with WebAPI. If you navigate to the page [http://localhost/<pfx>-openapi](http://localhost/<pfx>-openapi), you should see the description of our WebAPI in the [Swagger UI](https://swagger.io/tools/swagger-ui/) interface.


