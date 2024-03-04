# Monitoring System Status Using Metrics with Prometheus and Grafana Services

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-090
```

---

If our logs are well-structured, we can extract enough information about the current system state from them, besides just troubleshooting analysis. However, logs primarily do not address this task, and in most cases, they are not the most suitable way to monitor various performance parameters of the system and other system metrics in general. In this section, we will show how to enrich our system with metric collection in the form of time series of the current system state. First, we'll demonstrate how to collect these metrics for the cluster itself, and in the next section, we'll show the process of generating specific metrics for our application. For metric collection, we will use the [Prometheus] tool, and for visualization, the [Grafana](https://grafana.com/) tool.

The [Prometheus] tool is among the most commonly used tools for monitoring the state of complex systems. It records measurements obtained from various sources, assigning a timestamp to each measurement, creating a time series of such measurements. Measurements can include the volume of allocated memory, response time delay for an HTTP request, and many other indicators. Individual measurements can then be retrieved, combined, or aggregated using the [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) query language. [Prometheus] also includes a simple interface for obtaining and visualizing data, but specialized tools are generally used for system status visualization, in our case, it will be the [Grafana] tool.

>info:> Metrics from the [Prometheus] service can also be accessed in the [OpenSearch] service, but as of the writing of this text - November 2023 - this functionality was still limited and not quite stable. Check the [documentation](https://opensearch.org/docs/latest/observing-your-data/prometheusmetrics/) for the current status and usage instructions.

1. When deploying [Prometheus] on a Kubernetes cluster, it is necessary to choose a suitable deployment method. In our case, we will use the [Helm] chart [prometheus-community/prometheus](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus), which, in addition to configuring the _Prometheus_ server, also ensures the deployment of other components. In our case, the most important service is [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics#readme), which ensures the collection of metrics from the Kubernetes cluster. The deployment will be performed using the objects of the [FluxCD] operator.

Create the directory `${WAC_ROOT}/ambulance-gitops/infrastructure/prometheus` and within it, create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/prometheus/helm-repository.yaml` with the following content:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: wac-hospital
spec:
  interval: 1m
  url: https://prometheus-community.github.io/helm-charts
```

The [_HelmRepository_](https://fluxcd.io/flux/components/source/helmrepositories/) object serves as a reference source - repository - for [FluxCD] to Helm packages.

Next, create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/prometheus/helm-release.yaml` with the following content:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
name: prometheus
namespace: wac-hospital
spec:
interval: 1m
chart:
  spec:
    chart: prometheus   

    sourceRef:
      kind: HelmRepository
      name: prometheus-community @_important_@
      namespace: wac-hospital
    interval: 1m
    reconcileStrategy: Revision
values:
  prometheus-node-exporter:
    hostRootFsMount:
      mountPropagation: false # required on docker-desktop kluster 
```

Using the [_HelmRelease_](https://fluxcd.io/flux/components/source/helmcharts/) object of [FluxCD], we specify that the `prometheus` package should be installed on the target cluster from the repository defined by the `prometheus-community` object of type [_HelmRepository_](https://fluxcd.io/flux/components/source/helmrepositories/).

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/prometheus/kustomization.yaml` and integrate the previous manifests.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

resources:
- helm-repository.yaml
- helm-release.yaml
```

2. Prepare the configuration for the [Grafana] service. Create the directory `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana` and within it, create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  template:
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
        - name: grafana
          image: grafana/grafana:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http-grafana
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /robots.txt
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 2
          livenessProbe:
            failureThreshold: 3
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 3000
            timeoutSeconds: 1
          resources:
            requests:
              cpu: 250m
              memory: 750Mi
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-pv
            - mountPath: /etc/grafana
              name: config
            - mountPath: /etc/grafana/provisioning/datasources
              name: datasources
      volumes:
        - name: grafana-pv
          persistentVolumeClaim:
            claimName: grafana  @_important_@
        - name: config
          configMap:
            name: grafana-config  @_important_@
        - name: datasources
          configMap:
            name: grafana-datasources @_important_@
```

In this file, we defined the deployment of the container with the [Grafana] tool and assigned two configuration sources to it. The first is the `grafana-config` object of type [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/), which will contain the configuration of the [Grafana] tool itself. The second is the `grafana-datasources` object of type [_ConfigMap_](https://kubernetes.io/docs/concepts/configuration/configmap/), which will contain the configuration of data sources for the [Grafana] tool. We will create these objects in the following steps.

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/params/grafana.ini`:

```ini
[server]
domain = wac-hospital.loc @_important_@
root_url = https://wac-hospital.loc/grafana/ @_important_@
serve_from_sub_path = true @_important_@

[security]
# enable iframe embedding in ufe-controller
# or to embed graphs in external pages
allow_embedding = true

[users]
auto_assign_org=true
auto_assign_org_role=Admin

[auth]
disable_login_form = true

[auth.proxy]
enabled = true
header_name = x-auth-request-email @_important_@
header_property = email
auto_sign_up = true
sync_ttl = 60
whitelist =
headers = Name:x-auth-request-preferred-username Groups:x-auth-request-groups Role:x-auth-request-groups
enable_login_token = false
```

In the initialization file, we allowed anonymous access to the [Grafana] tool in the basic configuration. We also set basic parameters for authentication using the HTTP header `x-forwarded-email`. This header will contain the email address of the user who logs into the [Grafana] tool. This header will be created by the [oauth2-proxy] service, which we deployed in previous sections. We also specified that the [Grafana] tool will be available at the address `https://wac-hospital.loc/grafana/`.

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/params/prometheus-datasource.yaml`:


```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-server.wac-hospital @_important_@
    jsonData:
      httpMethod: POST
      manageAlerts: true
      prometheusType: Prometheus
      prometheusVersion: 2.47.0
      cacheLevel: 'High'
      disableRecordingRules: false
      incrementalQueryOverlapWindow: 10m
```

[Grafana] allows configuring data sources through the user interface or via automatic setup - [_provisioning_](https://grafana.com/docs/grafana/latest/administration/provisioning/), which allows preconfiguring known data source settings. This configuration is applied when the _Grafana_ service starts. In our case, we configured the data source for the [Prometheus] tool, which we deployed in the previous step.

>info:> [Grafana] also supports automatic configuration of data panels - _dashboards_. In the exercise, we will proceed with manual creation of data panels. You can then view, save, and use their configurations in JSON format for automatic provisioning. These configurations must be stored in the directory `/etc/grafana/provisioning/dashboards` - the _mountPath_ parameter of the `grafana` container.

Prepare the configuration for [_Persistent Volume Claim_](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/pvc.yaml`:


```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Next, configure the [_Service_](https://kubernetes.io/docs/concepts/services-networking/service/) object in the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: http-grafana
```

Finally, integrate all configuration files using the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

commonLabels:
  app.kubernetes.io/component: grafana
  
resources:
- deployment.yaml
- pvc.yaml
- service.yaml


configMapGenerator:
  - name: grafana-config
    files: 
    - params/grafana.ini
  - name: grafana-datasources
    files:
    - params/prometheus-datasource.yaml
```

3. Create the [_HTTPRoute_](https://gateway-api.sigs.k8s.io/docs/concepts/httproute/) object in the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/grafana.http-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /grafana
      backendRefs:
        - group: ""
          kind: Service
          name: grafana
          port: 80
```

Create the file `${WAC_ROOT}/apps/observability-webc/grafana.webcomponent.yaml`:

```yaml
apiVersion: fe.milung.eu/v1
kind: WebComponent
metadata:
  name: grafana
spec:
  module-uri: built-in
  navigation:
  - attributes:
    - name: src
      value: /grafana
    details: Aktuálny operačný stav systému
    element: ufe-frame
    path: grafana
    priority: 0
    title: System Dashboards
  preload: false
  proxy: true
```

and modify the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/kustomization.yaml`:

```yaml
...
resources:
- monitoring-opensearch.webcomponent.yaml
- monitoring-opensearch.http-route.yaml
- grafana.webcomponent.yaml     @_add_@
- grafana.http-route.yaml     @_add_@
```

4. Modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`:

```yaml
...
resources:
...
- ../../../infrastructure/fluentbit
- ../../../infrastructure/prometheus @_add_@
- ../../../infrastructure/grafana @_add_@
...
```

5. Verify the configuration correctness. Open a command prompt in the directory `${WAC_ROOT}/ambulance-gitops` and execute the command:

```ps
kubectl kustomize clusters/localhost/prepare
kubectl kustomize clusters/localhost/install
```

Archive your changes using the command:

```ps
git add .
git commit -m "Added prometheus and grafana"
git push
```

Wait for the changes to take effect in the cluster:

```ps
flux get kustomizations --namespace wac-hospital
```

6. Open your browser and visit the [Grafana] tool at [https://wac-hospital.loc/grafana](https://localhost/grafana). Open the side navigation panel, select _Connection_, and then select _Data Sources_. In the list of data sources, you should see the data source for [Prometheus], configured with the address `http://prometheus-server.wac-hospital`.

![Grafana - Data Sources](img/090-01-GrafanaDataSources.png)

Press the _Explore_ button in the _Prometheus_ section. A window will appear where you can search for individual available metrics and create aggregations and graphs. Switch the display mode from _Builder_ to _Code_ and enter the following query into the input field labeled _Enter PromQL query ..._:


```plain
irate(container_cpu_usage_seconds_total{namespace="wac-hospital"}[10m])
```

Switch the display mode back to _Builder_, and at the top of the panel, switch to _Explain_. Open the _Options_ panel, and in the _Legend_ field, choose _Custom_ and then enter {{pod}}. Finally, press the _Run Query_ button in the top right corner of the page. The resulting panel with the graph of current CPU usage by individual pods in the [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) `wac-hospital` should look like this:

![CPU Usage Graph for Pods in the wac-hospital namespace](img/090-02-CpuGraph.png)

Press the _Add to dashboard_ button and create a new data panel by pressing the _Open Dashboard_ button. A page with a data panel for current CPU usage will appear. Hover over the panel, and in the top right corner, display the _Menu_ and choose _Edit_. The panel edit mode will appear. Press the display options button - currently _Time series_ - and then the _Suggestion_ button. Choose your preferred display method, such as _Gauge_. Then press the _Apply_ button.

![Editing the Data Panel](img/090-03-CpuPanel.png)

Press the _Save dashboard_ icon and save the data panel with the name `WAC Hospital CPU`. Then press the _Save_ button. If you now go to the home page [https://wac-hospital.loc/grafana](https://wac-hospital.loc/grafana), you will see your saved panel `WAC Hospital CPU` in the list of dashboards.

>homework:> Expand the data panel with additional metrics, such as current memory usage or current CPU usage of individual pods in the [_namespace_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) `wac-hospital`. (Look for the metric `container_memory_usage_bytes`). Try various ways of visualizing metrics. More details about visualization options can be found in the [Grafana documentation](https://grafana.com/docs/grafana/latest/panels/visualizations/).

Metrics, in combination with log analysis, provide DevOps teams with an immediate insight into the operational state of an application deployed in a data center. They can monitor its load progression, system responses to changes in current load, and optimize the system implementation. The metrics currently available are mainly generated by the [_node-exporter_](https://github.com/prometheus/node_exporter) and [_kube-state-metrics_](https://github.com/kubernetes/kube-state-metrics) services. In the next section, we'll show you how to generate metrics specific to our application.

>info:> The tools mentioned here are examples of popular monitoring tools; they are not the only ones. If you are interested in other tools, you can search for them on the [CNCF Landscape](https://landscape.cncf.io/card-mode?category=observability-and-analysis&grouping=category) page.

