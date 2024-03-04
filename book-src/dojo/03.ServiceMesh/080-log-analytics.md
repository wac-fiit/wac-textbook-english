# Log Collection and Analysis with Fluentbit and OpenSearch

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-080
```

---

Our system is gradually expanding with new microservices that handle various aspects of our application. Simultaneously, we anticipate the expansion of the application's functionality itself, leading to the addition of additional microservices to the system. Despite all efforts to deliver the highest quality components, we must assume that during the operation of the system, situations will arise where the system's behavior deviates from the expected specified behavior. In such situations, it is necessary to have tools available to determine what is happening in the system and where the problem lies. Additionally, we need information on how the current system is being used and stressed so that we can proactively address potential issues. In the context of [DevOps](https://en.wikipedia.org/wiki/DevOps) development, these capabilities and activities are expected from the development team itself. In this and the following sections, we will demonstrate how to deploy such tools into the system and how to support system monitoring during the implementation of microservices.

Most software solutions generate records of activity - _logs_ - in some way, ideally writing them directly to standard output. In the first step, we will deploy tools for log collection and analysis into the cluster. In our case, these will be [FluentBit] and [Opensearch].

1. The [FluentBit] service is used for collecting and processing data streams, optimized for log processing, typically in Kubernetes and microservices environments. The processed records are then sent for further storage and analysis. For the specific destination where this data should be stored, we can use one of the existing plugins. [FluentBit] needs to be installed in the Kubernetes cluster as a [_DaemonSet_](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) object, ensuring that one replica of the [FluentBit] service is created on each [_Node_](https://kubernetes.io/docs/concepts/architecture/nodes/). This replica handles data collection and sending it for further processing.

When configuring [FluentBit], we will use an example from the [Kubernetes Logging with Fluent Bit](https://github.com/fluent/fluent-bit-kubernetes-logging) repository, which we will customize for our needs. Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/fluentbit/kustomization.yaml` with the following content:


```yaml
resources:
- namespace.yaml
- https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml
- https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-1.22.yaml
- https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding-1.22.yaml
- https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-configmap.yaml
- https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml
# for minikube use the following line  instead of the above one
# - https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/   fluent-bit-ds-minikube.yaml
  
images:
  - name: fluent/fluent-bit
    newName: fluent/fluent-bit
    newTag: latest
commonLabels:
    app.kubernetes.io/component: fluentbit
patches: 
- path: patches/fluent-bit-config.config-map.yaml
- path: patches/fluent-bit.daemon-set.yaml
```

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/fluentbit/namespace.yaml` with the following content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
```

We could install FluentBit into the `wac-hospital` _namespace_, but for better isolation and clarity, let's install FluentBit into a separate namespace called `logging`.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/fluentbit/patches/fluent-bit-config.config-map.yaml` with the following content:


```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: fluent-bit-config
    namespace: logging
    labels:
      k8s-app: fluent-bit
  data:
    # Configuration files: server, input, filters and output
    # ======================================================
    fluent-bit.conf: |
      [SERVICE]
          Flush         1
          Log_Level     info
          Daemon        off
          Parsers_File  parsers.conf
          HTTP_Server   On
          HTTP_Listen   0.0.0.0
          HTTP_Port     2020

      @INCLUDE input-kubernetes.conf
      @INCLUDE filter-kubernetes.conf
      @INCLUDE output-opensearch.conf

    output-opensearch.conf: |  
      [OUTPUT]
          Name                opensearch
          Match               *
          Host                ${FLUENT_OPENSEARCH_HOST}
          Port                ${FLUENT_OPENSEARCH_PORT}
          Suppress_Type_Name  On
          Replace_Dots        On
          Retry_Limit         2

    output-elasticsearch.conf: null
```

The original configuration used an output plugin for [Elasticsearch](https://www.elastic.co/), which we replaced with a plugin for [Opensearch](https://opensearch.org/). In the configuration, it is necessary to set the address and port where the [Opensearch] API will be accessible.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/fluentbit/patches/fluent-bit.daemon-set.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  template:
    spec:
      containers:
      - name: fluent-bit
        env:
        - name: FLUENT_OPENSEARCH_HOST
          value: monitoring-opensearch.wac-hospital @_important_@
        - name: FLUENT_OPENSEARCH_PORT
          value: "9200"
```

When setting up `FLUENT_OPENSEARCH_HOST`, it is necessary to provide the server address along with the `namespace` resolution, as we will be deploying this service in the `wac-hospital` namespace. In case FluentBit is deployed in the `logging` namespace, we could specify only `monitoring-opensearch`.

2. Prepare the configuration for the [OpenSearch] service. This consists of two services - the backend service providing storage, indexing, and data retrieval services through the REST API, and the [OpenSearch Dashboards](https://opensearch.org/docs/latest/dashboards/index/) service that provides a user interface for searching and visualizing data. The content and purpose of each file should now be clear to you.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/server.deployment.yaml`

```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: &PODNAME monitoring-opensearch
  spec:
    replicas: 1
    strategy:
      # recreate to avoid simultanopus locking of the data volume during updates
      type: Recreate
    selector:
      matchLabels:
        app.kubernetes.io/component: *PODNAME
    template:
      metadata:
        labels:
          app.kubernetes.io/component: *PODNAME
      spec:
        volumes:
        - name: *PODNAME
          persistentVolumeClaim:
            claimName: *PODNAME
        initContainers:
        - name: fsgroup-volume
          image: busybox:latest
          imagePullPolicy: IfNotPresent
          command: ['sh', '-c']
          args:
          # change data ownership to avoid "permision denied" errors
          - 'chown -R 1000:1000 /usr/share/opensearch/data'
          securityContext:
          runAsUser: 0
          resources:
          requests:
            cpu: 1m
            memory: 32Mi
          limits:
            cpu: 10m
            memory: 128Mi
          volumeMounts:
          - mountPath: /usr/share/opensearch/data
            name: *PODNAME
        containers:
        - name: *PODNAME
          image: opensearchproject/opensearch:latest
          volumeMounts:
            - mountPath: /usr/share/opensearch/data @_important_@
              name: *PODNAME
          env:
            - name: discovery.type
              value: single-node  @_important_@
            - name: OPENSEARCH_JAVA_OPTS
              value: -Xms512m -Xmx512m
            - name: DISABLE_INSTALL_DEMO_CONFIG @_important_@
              value: "true"
            - name: DISABLE_SECURITY_PLUGIN @_important_@
              value: "true"
          ports: 
            - name: api
              containerPort: 9200 
            - name: performance
              containerPort: 9600
```

The configuration is intentionally simplified, assuming the use of cluster-level security mechanisms, and is deployed in single-node mode. If needed, this configuration can be extended with additional parameters available in the [documentation](https://opensearch.org/docs/latest/install/index/). In production, you might, for instance, utilize [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) to ensure high availability.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/server.service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: &PODNAME monitoring-opensearch
spec: 
  selector:
    app.kubernetes.io/component: *PODNAME
  ports:
  - name: api
    port: 9200
    targetPort: 9200
  - name: performance
    port: 9600
    targetPort: 9600
```

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: monitoring-opensearch
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/dashboard.deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: &PODNAME monitoring-dashboards
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: *PODNAME
  template:
    metadata:
      labels:
        app.kubernetes.io/component: *PODNAME
    spec:
      containers:
      - name: *PODNAME
        image: opensearchproject/opensearch-dashboards:latest
        env:
          - name: OPENSEARCH_HOSTS
            value:  '["http://monitoring-opensearch:9200"]' @_important_@
          - name: DISABLE_SECURITY_DASHBOARDS_PLUGIN
            value: "true"
          - name: SERVER_BASEPATH
            value: /monitoring @_important_@
          - name: SERVER_REWRITEBASEPATH
            value: "true"
        ports: 
          - name: web
            containerPort: 5601
        resources:
        limits:
            cpu: '0.5'
            memory: '1Gi'
        requests:
            cpu: '0.1'
            memory: '512M'
```

The [OpenSearch Dashboards](https://opensearch.org/docs/latest/dashboards/index/) service will be accessible through our [Gateway API][gatewayapi], so we set `SERVER_BASEPATH` to `/monitoring`.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/dashboard.service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: &PODNAME monitoring-dashboards
spec: 
  selector:
    app.kubernetes.io/component: *PODNAME
  ports:
  - name: web
    port: 80
    targetPort: 5601
```

Finally, create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/monitoring-opensearch/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

commonLabels:
  app.kubernetes.io/part-of: wac-hospital

resources:
- server.deployment.yaml
- server.service.yaml
- pvc.yaml
- dashboard.deployment.yaml
- dashboard.service.yaml
```

3. Create a directory `${WAC_ROOT}/ambulance-gitops/apps/observability-webc` and within it, create the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/monitoring-opensearch.webcomponent.yaml`:

```yaml
apiVersion: fe.milung.eu/v1
kind: WebComponent
metadata: 
  name: monitoring-dashboards
spec:   
  module-uri: built-in
  navigation:
    - element: ufe-frame 
      path: monitoring
      title: Analýza logov
      details: Analytické nástroje pre prácu so záznamami systému
      attributes:  
        - name: src
          value: /monitoring
```

and create the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/monitoring-opensearch.http-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: monitoring-dashboards
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /monitoring
      backendRefs:
        - group: ""
          kind: Service
          name: monitoring-dashboards
          port: 80
```

Create a file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

resources:
- monitoring-opensearch.webcomponent.yaml
- monitoring-opensearch.http-route.yaml
```

We create _WebComponent_ and _HTTPRoute_ objects as part of the application to avoid cross-dependencies between these objects and the installation of user-defined object types - [_Custom Resource Definition_](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). An alternative would be to split the deployment of infrastructure into additional interdependent steps, such as `prepare-crd` and `prepare-instances`. However, for simplification, we chose to place the integration with the microfrontend in the _apps_ section, which more or less corresponds to the semantics of registering these microfrontend components into our application.

4. Modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`

```yaml
...
resources:
...
- ../../../infrastructure/fluentbit   @_add_@
- ../../../infrastructure/monitoring-opensearch   @_add_@
```

and the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml`:

```yaml
...
resources:
...
- ../../../apps/http-echo
- ../../../apps/observability-webc  @_add_@
```

Verify the correctness of the configuration using the command in the `${WAC_ROOT}/ambulance-gitops/` directory:

```bash
kubectl kustomize ./clusters/localhost/prepare
kubectl kustomize ./clusters/localhost/install
```

5. Archive the change and save it to the remote repository:

```bash
git add .
git commit -m "Add fluentbit and opensearch"
git push
```

Wait until the changes are applied to the cluster and verify them using the command:

```bash
kubectl get kustomization -n wac-hospital
kubectl get pods -n logging
kubectl get pods -n wac-hospital
```
>info:> If pods fail to start, it's possible that the cluster doesn't have enough disk space. For example, in the case of docker-desktop, it may be necessary to increase disk limits in the application settings.

6. Go to the page [http://localhost/monitoring](http://localhost/monitoring) and verify its availability. When initially accessing, choose the option _Explore on my own_ and dismiss any pop-up window. Open the side menu, select _Index Management_, and then choose _Indices_. In the list of indices, you should see the _fluent-bit_ index.

![Index Management](./img/080-01-IndexManagement.png)

Open the side panel and select _Discover_. You will see a window with instructions to create an _Index Pattern_.

![Discover Initial Window](./img/080-02-DiscoverFirstTime.png)

Press the _Create Index Pattern_ button, and in the _Index pattern name_ field, enter the text `fluent-bit`.

![Create Index Pattern](./img/080-03-CreateIndexPattern.png)

>info:> This action could also be triggered by selecting _Dashboards Management_ in the side menu and then selecting _Index patterns_.

Press the _Next step_ button, and in the dropdown field _Time field_, choose `@timestamp`. Press the _Create index pattern_ button.

![Create Index Pattern](./img/080-04-CreateIndexPattern-Timefield.png)

7. In the side menu of the _OpenSearch Dashboards_ application, select _Discover_. Now, you will see a window with a log output in your cluster for the last 15 minutes. In the left side panel, you can modify which log data will be displayed. In the top panel, you can specify a search phrase for system log entries, or filter the output based on different criteria, or set a different time range for system log entries. In case of unexpected system behavior, you can use this functionality to trace a possible cause of such behavior. Additional log analysis options can be found in the side menu under "Logs". For more information on using OpenSearch Dashboard for system analysis and monitoring, refer to the [documentation](https://opensearch.org/docs/latest/dashboards/index/).

![Log Output and Analysis for the `wac-hospital` namespace](./img/080-05-LogAnalysis.png)

To make log analysis truly effective, pay attention to the logs you generate and how useful the information in them is. Overloading logs with unnecessary information can make it challenging to analyze them when needed. Always label entries appropriately to easily filter them, and include information that allows you to monitor data processing in different parts of the system - having the information that your API was called 500 times isn't as useful unless you can identify which of these calls are related to suspicious system behavior.

The [OpenSearch] application also allows adding more advanced visualizations. For example, our logs might contain some important status indicators of individual services. We won't delve into these options here; in the next chapter, we will show you how to provide various metrics and activity indicators for the system.
