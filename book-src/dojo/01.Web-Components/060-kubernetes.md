## Deployment to Kubernetes Cluster

---

>info:>
Template for a pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-060`

---

In the previous example, we introduced one of the ways to deploy our application, where we used [_Platform as a Service_][paas] services in the [Microsoft Azure][azure] data center. Although this method of deploying the application is functional, it is suitable only for very small projects. In projects where several teams collaborate, and the final work consists of various subsystems, this deployment method is insufficient.

In this exercise, we will show you how to deploy our application to a [Kubernetes] cluster manually. In the next part, we'll cover how to ensure continuous deployment of the application using the [GitOps] technique with the help of the [Flux] application.

Before starting, make sure you have Kubernetes support enabled in the `Docker Desktop` application, as shown in the image:

![Kubernetes in Docker for Desktop](./img/060-01-docker-kubernetes.png)

1. In the `${WAC_ROOT}` directory, create a new folder named `ambulance-gitops`. In VS Code, add this folder to the current workspace using the menu option _File->Add Folder to Workspace..._.

2. In the `${WAC_ROOT}/ambulance-gitops` folder, create the following directory structure:


```plain
ambulance-gitops
|
|- apps
|  L <pfx>-ambulance-ufe
|
|- infrastructure
|  L ufe-controller
|
|- components
|- clusters
    L localhost
        
```

We will deploy the application using [kustomize] for managing configurations in the Kubernetes system. [Kustomize] is a native part of the Kubernetes system and is also integrated into the `kubectl` application. An often-used alternative to this application is [Helm]. The purpose of use is the same for these applications; Helm focuses on template techniques that are parameterized with configuration for specific deployments. [Kustomize] rejects templates as they are not directly deployable or usable, and it works on the principle of customizing functional manifests using partial modifications - _patches_ - for specific deployments. In this exercise, we prefer [Kustomize], mainly because manifests, in their basic form, maintain syntactic and semantic accuracy defined by the [Kubernetes API][k8s-api].

The folder structure is hierarchical. The `clusters` folder will contain the system configuration for individual specific clusters. Most of the time during the exercise, we will work with the local cluster labeled as `localhost`.

The `infrastructure` folder will contain system components that are either installed typically only once during the cluster's lifecycle or shared among teams and are necessary for the operation of other subsystems. What exactly is part of the `infrastructure` folder may vary from case to case; in essence, it includes shared services with other applications that are already prepared on some clusters and cannot be "reinstalled" within a specific project/team. On other clusters, they need to be pre-installed to ensure the functionality of the delivered application.

The `apps` folder contains system components that the project team provides for all target environments - clusters - and has full control over them.

The `components` folder allows sharing configuration between different deployment variants. For example, if our system has the option to use different DBMS systems, we could place the relevant configurations and modifications of existing components in this folder. In our case, we will primarily use this folder to define different versions of our system.

>info:> In these exercises, we use the so-called [_MonoRepo_](https://fluxcd.io/flux/guides/repository-structure/#monorepo) repository structure for continuous deployment. However, it is not the only option, and more possibilities are described in the [flux documentation](https://fluxcd.io/flux/guides/repository-structure/).

3. In the `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe` folder, create a file named `deployment.yaml` with the following content (you can omit comments):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <pfx>-ambulance-ufe-deployment      # name of the deployment object, the name of the pod will be derived from this name
spec:
  replicas: 2 @_important_@
  selector:
    matchLabels:
        pod: <pfx>-ambulance-ufe-label    # this is a label key-value pair pod=ambulance ufe
                                          # used for selecting pods with the same label
  template:                    # template for creating pods
    metadata:
      labels: 
        pod: <pfx>-ambulance-ufe-label    # label for selecting pods
    spec:
      containers:
      - name: ambulance-ufe-container #  name of the container, pod can contain more containers
        image: <pfx-hub>/ambulance-ufe
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080       # port on which the container listens
        resources:                  # resource requests and limits
                                    # for the container
                                    # it is important to set these values so that the pod does not consume all resources
            requests:               
                memory: "32M"
                cpu: "0.1"
            limits:
                memory: "320M"
                cpu: "0.3"
```

The `deployment.yaml` file is a declaration of deploying our service - the so-called [_workload_](https://kubernetes.io/docs/concepts/workloads/) - to the cluster. Note that it requests the deployment of two replicas - meaning that two _pods_ will be created in the cluster, and within each of them, a running container with our application will be instantiated. Also, notice that in the template for creating the pod, a label `pod: <pfx>-ambulance-ufe-label` is defined, which is used in the selector for selecting pods to be part of this deployment. This label is also used in other manifests that we will create. All objects (_resources_) in the Kubernetes system can be labeled, and they can be searched using selectors that use these labels. The suffixes `-label`, `-deployment`, and `-container` are not standard conventions; they are mainly included here for better orientation on how individual names in the manifest relate.

4. The manifest created in this way is functional and can be deployed to the cluster. We assume that you have the [Kubernetes system](https://kubernetes.io/) installed on your computer and have selected an [active context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#define-clusters-users-and-contexts) for this cluster. You can find out which context is active by executing the command:

```ps
kubectl config get-contexts
```

that will display the results in the form:

```plain
CURRENT   NAME           CLUSTER           AUTHINFO 
*         aks            aks-cluster       user-admin
          docker-desktop docker-desktop    docker-desktop
```

where the `*` character indicates the currently active context. If the desired context is not active, you can change it with the command:

```ps
kubectl config use-context docker-desktop
```

>info:> We recommend installing [kubectx], [kubens], [OpenLens][lens] on your computer and learning how to use them when working with Kubernetes clusters. The exercises provide commands based on the [kubectl] tool, which is part of the installed [Kubernetes] system.

Now, navigate to the `${WAC_ROOT}/ambulance-gitops` directory and run the command:

```ps
kubectl apply -f apps/<pfx>-ambulance-ufe/deployment.yaml
```

Subsequently, you can execute the command (repeatedly):

```ps
kubectl get pods
```

and repeat it until you see the output in the form

```plain
NAME                                        READY   STATUS    RESTARTS   AGE
ambulance-ufe-deployment-7f45494d6b-qj58z   1/1     Running   0          37s
ambulance-ufe-deployment-7f45494d6b-slppc   1/1     Running   0          61s
```

>build_circle:> The status may temporarily show `Container Creating`. If the column indicates any error, correct the `deployment.yaml` manifest and apply it again with the command `kubectl apply -f apps/<pfx>-ambulance-ufe/deployment.yaml`.

The output indicates that we now have two [pods](https://kubernetes.io/docs/concepts/workloads/pods) running in our cluster - essentially, two virtual machines connected to the virtual network of the Kubernetes cluster.

5. Currently, these pods are only accessible on the virtual network of the Kubernetes system. If we want to access them, we can use the [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) feature to forward ports from the local system to the target pod (or service) of the active cluster. Execute the command:


```ps
kubectl port-forward <pfx>-ambulance-ufe-deployment-<your-hash> 8081:8080
```

if we open the page [http://localhost:8081](http://localhost:8081) in the browser, we will see our list of waiting patients.

The `port-forward` function also works with remote clusters and is suitable for debugging supporting microservices that are not accessible from external networks outside the cluster. In our case, only the service integrating individual micro-front-end components will be exposed, as shown below.

>info:> The [kubectl] tool acts as a web client accessing the [REST API]/[Kubernetes API][k8s-api] of the Kubernetes system. It is possible to access this API directly, but we would have to ensure the correct authentication and authorization for our requests, which [kubectl] performs automatically based on the configuration typically stored in the `~/.kube` directory.

Finally, we will remove the objects we created using the command:

```ps
kubectl delete -f apps/<pfx>-ambulance-ufe/deployment.yaml
```

6. Now, create a file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/service.yaml` with the following content:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: <pfx>-ambulance-ufe   #  name of the service, it is also used as a DNS record
spec:  
  selector:
    pod: <pfx>-ambulance-ufe-label    # label of the pods that will be part of this service, matches the label in
                                      # deployment.yaml
  ports:
  - name: http
    protocol: TCP
    port: 80            # port on which the service listens, it can be different from the port on which the container listens
    targetPort: http    # port on which the service forwards requests to the pods, see the name in deployment.yaml
```

This object in the Kubernetes system declares a [network service](https://kubernetes.io/docs/concepts/services-networking/service/) that also implements a simple load balancer for distributing HTTP requests between two replicas of our web service. It also implements [_service discovery_](https://en.wikipedia.org/wiki/Service_discovery) based on the [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) system. This means that the service name `<pfx>-ambulance-ufe` simultaneously represents a DNS record within the virtual network. Therefore, our web service will be internally accessible within the Kubernetes system at the address `http://<pfx>-ambulance-ufe` within the same namespace, or at `http://<pfx>-ambulance-ufe.<namespace>` from any [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).


7. Next, create a file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml` with the following content:

```yaml
apiVersion: fe.milung.eu/v1
kind: WebComponent
metadata: 
  name: <pfx>-ambulance-ufe
spec:   
  module-uri: http://<pfx>-ambulance-ufe.wac-hospital/build/ambulance-ufe.esm.js  
                    # URL to the web component module
                    # this URL is used to load the module into the browser
  navigation:
    - element: <pfx>-ambulance-wl-list    # element name of the web component
      path: <pfx>-ambulance-wl            # path to the web component, used in the URL
      title: Waiting list <pfx>      # title of the web component, used in the browser
      details: List of waiting patients <pfx> # description of the web component, used in the browser
  preload: false                    # whether the module should be preloaded
  proxy: true                       # whether the module should be proxied, for components that are not exposed
                                    # to the public network
  hash-suffix: v1alpha1             # hash suffix for the module, used for versioning and for cache busting
```

> warning: The name of the element `<pfx>-ambulance-wl-list` must correspond to the component we created earlier. Refer to the file `${WAC_ROOT}\ambulance-ufe\src\components\<pfx>-ambulance-wl-list\<pfx>-ambulance-wl-list.tsx`.

This file is a custom object - _Custom Resource_ - in the Kubernetes system. In the next step, we will create manifests for the micro-front-end controller, which will manage these objects. Essentially, we are registering the micro-application - web component - into the main application wrapper.

8. Finally, create the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/kustomization.yaml`. Configuration management based on the [Kustomize] system uses `kustomization.yaml` files as the integration point for individual objects declared in other manifests. Other manifests remain usable, and they can be directly used as an argument for the `kubectl` application. The `kustomization.yaml` file is essentially a prescription for automating the _editing_ of the original files to deploy individual objects in different environments - meaning in different clusters - and these modifications can be hierarchically accumulated. In this case, it is simply a list of individual manifests and the assignment of common labels.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:                  # list of resources to be deployed
- deployment.yaml
- service.yaml
- webcomponent.yaml

commonLabels:               # common labels for all resources
  app.kubernetes.io/component: <pfx>-ambulance-ufe
```

At this moment, we will not deploy our application yet; we need to prepare manifests for the cluster infrastructure first. However, if we want to see the result of the "customization" of our application, we can obtain it by executing the command:

```yaml
kubectl kustomize ./apps/<pfx>-ambulance-ufe/
```

9. In this step, we will prepare manifests for objects in our _infrastructure_. In our case, this primarily involves the micro-application controller, which acts as an application wrapper for displaying individual web components. Upon examining the current state, we did not find an implementation of a [micro-front-end system][micro-fe] that seamlessly combines technologies such as [Web Components][webc], [Kubernetes], and [micro-Front-Ends][micro-fe]. This combination appears to be the most suitable given current development trends, as it provides space for independent development of individual web components based on widely accepted standards. These components can then be deployed declaratively in various situations, aligning with the philosophy of microservices. Existing implementations include [bit] or [single-spa], but their integration often requires tight coupling of micro-front-end services.

To leverage the declarative principles of the Kubernetes API and the independence of teams developing individual micro-applications, a simple [_Kubernetes controller_][k8s-controller] has been created for the purposes of this exercise. This controller handles custom objects in the Kubernetes system defined using [_Custom Resource Definition_][k8s-crd]. The controller continuously monitors changes in declared objects - _webcomponents_ - and provides them to its built-in web server, which implements the application wrapper. Individual web components are dynamically loaded as needed and displayed based on the specifications in these custom objects. The controller is implemented in Prolog (back-end) and TypeScript (front-end), and its source files are available at [https://github.com/milung/ufe-controller](https://github.com/milung/ufe-controller). It serves as both motivation and an example of the extensibility of the [Kubernetes API][k8s-api], which focuses on declarative descriptions of the desired state rather than procedural descriptions of how to achieve the desired state. In the exercises, we will use its containerized version `milung/ufe-controller:latest`.

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller/kustomization.yaml` with the following content:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/milung/ufe-controller//configs/k8s/kustomize

namespace: wac-hospital # namespace for the objects

commonLabels:
  app.kubernetes.io/component: ufe-controller
```

By following this approach, we have created a configuration based on a pre-prepared configuration for the _controller_. We have also specified that all relevant objects will be placed under the cluster subdomain `wac-hospital`.

You can view the contents of the files in the repository [here](https://github.com/milung/ufe-controller/tree/master/configs/k8s/kustomize). The files `deployment.yaml` and `service.yaml` are similar to those in our `ambulance-ufe` microservice. The `crd.yaml` file declares a custom object in the Kubernetes system - _Custom Resource Definition_ - for declaring web components. Among other things, it defines the schema based on which the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml` was created.

The controller needs access to objects of type `webcomponents`, which we defined in the previous file. Therefore, the configuration also includes definitions for [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), allowing our pod to access objects of `webcomponents.fe.milung.eu`. In the `deployment.yaml` file, we specified `serviceAccountName: ufe-controller-account` for our pod, indicating that it will use the `ufe-controller-account` account defined in the `serviceaccount.yaml` file. This account is assigned to the _ClusterRole_ `webcomponents-reader`, defined in the `clusterrole.yaml` file. This _ClusterRole_ is assigned to the _ClusterRoleBinding_ `ufe-controller`, defined in the `clusterrolebinding.yaml` file. This way, we have created access rights for our pod to objects of type `webcomponents.fe.milung.eu`.

>info:> Such direct dependency on external manifests without specifying the corresponding version is undesirable in practice, as it risks changes that may disrupt the functionality of our application. In practice, these manifests would either be copied to a local repository, or a specific version (branch/tag on GitHub) would be specified for use in the `kustomization.yaml` file.

10. In the previous steps, we created declarations for our `ambulance-ufe` application and for the infrastructure application `ufe-controller`. Now, let's move on to declaring configuration for specific environments - clusters. Since we will be deploying to our local Kubernetes cluster, we need to deploy both applications and create a cluster subdomain - [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) - to place them in. Deployment to the cluster will be divided into two steps:

Step `prepare` - creating a _namespace_ and deploying cluster infrastructure. Here, we will deploy services that are a prerequisite for preparing our cluster for deploying the application itself;

Step `install` - deploying services of the application(s) itself into the cluster. If we have the cluster completely prepared, it is sufficient to deploy this step.

Implicitly, we also assume that there are dependencies between these steps - for example, objects of type `webc` in the `install` step assume the existence of Kubernetes API extensions of type `webcomponents.fe.milung.eu` from the `prepare` step.

>info:> We try to limit the number of steps, but sometimes it may be necessary to divide deployment into multiple steps to meet certain prerequisites for deploying our application - typically the availability of services that define how to deploy other objects or objects to which we want to limit access for better coordination between teams.

Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/namespace.yaml` with the following content:


```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wac-hospital
```

Next, create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- namespace.yaml                            # namespace for the objects
- ../../../infrastructure/ufe-controller       # ufe-controller infrastructure

patches: 
- path: patches/ufe-controller.service.yaml    # patch for the ufe-controller service
```

This "kustomization" additionally specifies that we will use a modification from the file `patches/ufe-controller.service.yaml`.

Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/patches/ufe-controller.service.yaml` with the following content:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: ufe-controller
spec:  
  type: NodePort
  ports:
  - name: http
    protocol: TCP
    port: 80
    nodePort: 30331
```

This modification changes the service type of `ufe-controller`, which is originally specified in the file `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller/service.yaml`. The original specification implicitly uses the `ClusterIP` type, which makes the service accessible only on the internal network of the Kubernetes cluster. The modified version uses the `NodePort` type and sets the parameter `nodePort: 30331`. This means that the `ufe-controller` service can be accessed on port `30331` of the host computer of the cluster. This [strategic-merge patch](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#patch-using-path-strategic-merge) file is not a complete declaration of the _Service_; it only contains identifiers to locate the specific record (`kind`, `apiVersion`, `name`, `protocol`) and the items to be modified.

Another type of _service_ that we could use is the `LoadBalancer` type. The configuration of this type depends on the cluster provider; in the case of [Docker Desktop][docker-desktop], the service would be available on port 80 of our computer. However, in this case, only one `LoadBalancer` type service can be used across the entire cluster. (In the case of clusters in Azure or AWS, each `LoadBalancer` type service is assigned a separate _public_ IP address).

Now, we need to deploy the `ambulance-ufe` application into the cluster. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml` with the following content:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital  # namespace for the objects, it will override the namespace from the resources

commonLabels:
  app.kubernetes.io/part-of: wac-hospital

resources:
- ../../../apps/<pfx>-ambulance-ufe
```

11. Verify that your configuration is correct by executing the command in the `${WAC_ROOT}/ambulance-gitops` directory:

```ps
kubectl kustomize clusters/localhost/prepare
```

followed by the command:

```ps
kubectl kustomize clusters/localhost/install
```

If your configurations are correct, the applied YAML documents will be displayed; otherwise, you need to correct the error according to the displayed output.

Next, apply the preparatory step using the command in the `${WAC_ROOT}/ambulance-gitops` directory:

```ps
  kubectl apply -k clusters/localhost/prepare
```

It's possible that when executing this command, you may receive warnings like: `Warning: resource ... is missing the kubectl.kubernetes.io/last-applied-configuration annotation ...`. You can ignore these warnings; the `last-applied-configuration` will be implicitly created by the previous command.

Next, apply the command:

```ps
kubectl apply -k clusters/localhost/install
```

and verify that the newly created pods reach the `Running` state with the command:
  
```ps
kubectl get pods --namespace wac-hospital
```

In your browser, open the page [http://localhost:30331](http://localhost:30331), where you will see the application shell with an integrated micro-application. After clicking on the link "Waiting List", you should see the following output:

![Integrated Waiting List](./img/060-02-appshell-list.png)

>build_circle:> On some systems, `NodePort` may not be accessible at `localhost`. Check how they are accessible on your system. On [Docker Desktop][docker-desktop], you can use the address `host.docker.internal` or `localhost` with port `30331`.

The following image illustrates the deployment and communication scheme of the deployed application.

![Communication between micro-front-end controller and deployed WebComponent](./img/060-03-k8s-ufe-communication.png)

We have verified that the manual deployment of our application into the Kubernetes cluster works. In the next section, we will demonstrate how to achieve continuous deployment using the Flux application.

Before that, let's remove all manually deployed objects from the cluster:

```ps
kubectl delete -k ./clusters/localhost/install
kubectl delete -k ./clusters/localhost/prepare
```