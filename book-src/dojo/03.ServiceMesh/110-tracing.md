# Distributed Tracing with Jaeger Tracing

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-110
```

---

Metrics and log analysis significantly facilitate monitoring the system's state and analyzing any deviations from the specified system behavior. However, outside the development environment, when the system is under load, we will encounter situations where it is necessary to analyze how individual requests are processed across multiple microservices, or we need to find where the cause of an error occurred, observed as a failure of a request in one of the services where it is no longer possible to perform a corrective operation. Essentially, we need to find a relationship between individual operational records - logs - in different parts of our system, while such a system can simultaneously process tens to thousands of requests. The method of such analysis is called distributed tracing - [_distributed tracing_](https://microservices.io/patterns/observability/distributed-tracing.html).

Simply put, in the context of distributed tracing, an incoming request into the system (or an individual request generated within the system itself) is assigned a so-called _trace-id_, which is propagated during calls to individual subsystems - microservices, libraries, or microservice components. Each subsystem then defines the scope of the computation - [_span_](https://opentelemetry.io/docs/concepts/signals/traces/#spans), to which it assigns the relevant [attributes](https://opentelemetry.io/docs/concepts/signals/traces/#attributes) needed to identify the service, type of computation, or computation parameters and events occurring during the computation. A _span_ has an associated [_span context_](https://opentelemetry.io/docs/concepts/signals/traces/#span-context), also called [_trace context_](https://opentelemetry.io/docs/concepts/signals/traces/#span-context), which contains the identifier of the original request - _trace_id_ - in whose context the computation is executed, as well as the scope of the computation - _span-id_, which includes the respective computation. These records are then sent to the service - _collector_ - which sends these records for permanent storage for later analysis.

Similar to metrics, the [OpenTelemetry] library is most commonly used for distributed tracing. In this part of the exercise, we will show how to deploy [Jaeger] services for collecting and analyzing distributed records in our system, and in the following part, we will modify the ambulance-webapi service to generate its own computation scope.

> Info: Similar to metrics, there are various alternatives for products and services that provide this functionality in the case of distributed tracing. A basic overview of available services can be found, for example, on the [CNCF Cloud Native Landscape](https://landscape.cncf.io/card-mode?category=tracing&grouping=category) pages. In any case, we recommend focusing primarily on services and products that natively support the [OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/).


1. We will install the [Jaeger] service using the [Jaeger Helm Chart](https://github.com/jaegertracing/helm-charts). Create the directory `${WAC_ROOT}/ambulance-gitops/infrastructure/jaegertracing` and in it, create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/jaegertracing/helm-repository.yaml` with the following content:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: jaegertracing
  namespace: wac-hospital
spec:
  interval: 1m
  url: https://jaegertracing.github.io/helm-charts
```

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/jaegertracing/helm-release.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: jaeger
  namespace: wac-hospital
spec:
  interval: 1m
  chart:
    spec:
      chart: jaeger

      sourceRef:
        kind: HelmRepository
        name: jaegertracing
        namespace: wac-hospital
      interval: 1m
      reconcileStrategy: Revision
  values:
    provisionDataStore:
      cassandra: false
    storage:
      type: opensearch
    collector:
      enabled: true
      extraEnv:
        # see https://opensearch.org/docs/latest/observing-your-data/trace/trace-analytics-jaeger/
        - name: SPAN_STORAGE_TYPE
          value: opensearch
        - name: ES_TAGS_AS_FIELDS_ALL
          value: "true"
        - name: ES_SERVER_URLS
          value: http://monitoring-opensearch.wac-hospital:9200 @_important_@
        - name: ES_TLS_ENABLED
          value: "false"
      service:
        otlp:    @_important_@
          grpc:    @_important_@
            name: otlp-grpc     @_important_@
            port: 4317    @_important_@
          http:
            name: otlp-http
            port: 4318
    query:
      enabled: true
      basePath: /jaeger   @_important_@
      extraEnv:
        - name: SPAN_STORAGE_TYPE
          value: opensearch
        - name: ES_SERVER_URLS
          value: http://monitoring-opensearch.wac-hospital:9200 @_important_@
        - name: ES_TLS_ENABLED
          value: "false"
```

Notice the configuration of the services `jaeger-collector` and `jaeger-query`. This configuration allows us to use the `monitoring-opensearch` service as the storage for distributed tracing records and, at the same time, makes the trace analysis functionality available in the `monitoring-opensearch-dashboards` service. Additionally, in the `collector` service, we have enabled support for the [OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/), which simplifies the connection to external services.

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/jaegertracing/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

resources:
- helm-repository.yaml
- helm-release.yaml
```

2. Create the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/jaeger-query.http-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: jaeger-query
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /jaeger
      backendRefs:
        - group: ""
          kind: Service
          name: jaeger-query
          port: 80
```

and the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/jaeger-query.webcomponent.yaml`:

```yaml
apiVersion: fe.milung.eu/v1
kind: WebComponent
metadata:
  name: jaeger
spec:
  module-uri: built-in
  navigation:
  - attributes:
    - name: src
      value: /jaeger
    details: Analýza spracovania požiadaviek v distribuovanom systéme
    element: ufe-frame
    path: jaeger
    priority: 0
    title: Distribuované trasovanie
  preload: false
  proxy: true
```

Modify the file `${WAC_ROOT}/ambulance-gitops/apps/observability-webc/kustomization.yaml`:

```yaml
...
resources:
..,
- jaeger-query.webcomponent.yaml   @_add_@
- jaeger-query.http-route.yaml   @_add_@
```

3. Distributed tracing records must be actively generated in the service we want to monitor and sent in batches to a service designated for collecting these records, in our case, to the `jaeger-collector` service. This requires us to modify the relevant settings of the services that support distributed tracing.

[Envoy Gateway] allows configuring instances of dynamically created [envoy proxy] pods using objects of type [EnvoyProxy](https://gateway.envoyproxy.io/v0.6.0/api/extension_types/#envoyproxy). In this object, it is possible to set entry points fo

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: wac-hospital-proxy
  namespace: envoy-gateway-system
spec:
  telemetry:
    tracing:
      # sample 100% of requests - downrate in production, see https://opentelemetry.io/docs/concepts/sampling/
      samplingRate: 100 @_important_@
      provider:
        host: jaeger-collector.wac-hospital @_important_@
        port: 4317
        type: OpenTelemetry @_important_@
```

In this configuration, we specified that all requests should be sent to the `jaeger-collector` service on port `4317`. The variable `samplingRate` determines the proportional amount of requests to be traced and sent to the `jaeger-collector` service. In this case, for demonstration purposes, we set the value to 100%, but in a production system, this may unnecessarily load the system and can be cumbersome for analyzing any deviations since the majority of requests may not be of interest. If we wanted to trace only every tenth request, we would set the `samplingRate` value to `10`. More information about the options for setting the sampling rate can be found in the [documentation](https://opentelemetry.io/docs/concepts/sampling/).

Add a reference to the created [EnvoyProxy](https://gateway.envoyproxy.io/v0.6.0/api/extension_types/#envoyproxy) object to the file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/gateway-class.yaml`:

```yaml
...
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:   @add_@
    group: gateway.envoyproxy.io   @_add_@
    kind: EnvoyProxy   @_add_@
    name: wac-hospital-proxy   @_add_@
    namespace: envoy-gateway-system   @_add_@
```

Modify the file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/kustomization.yaml`:

```yaml
...
resources:
...
- envoy-proxy.yaml @_add_@

configMapGenerator:
...
```

Similarly, we will configure [Open Policy Agent] to send all requests to the `jaeger-collector` service. Modify the file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/params/opa-config.yaml`:

```yaml
plugins:
...
decision_logs:
  console: true
distributed_tracing:    @_add_@
  type: grpc    @_add_@
  address: jaeger-collector.wac-hospital:4317    @_add_@
  service_name: open-policy-agent    @_add_@
  # downrate in production, see https://opentelemetry.io/docs/concepts/sampling/    @_add_@
  sample_percentage: 100    @_add_@
  encryption: "off"    @_add_@
```

Further modifications are required in the [Grafana](https://grafana.com/docs/grafana/v9.3/setup-grafana/configure-grafana/#tracingopentelemetry) service. Open the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/params/grafana.ini` and add configuration for distributed tracing:

```ini
...
[tracing.opentelemetry]
sampler_type = rateLimiting
sampler_param=10
sampling_server_url = 

[tracing.opentelemetry.otlp]
address= jaeger-collector.wac-hospital:4317
propagation= w3c
```

Modify the file `${WAC_ROOT}/ambulance-gitops/infrastructure/grafana/deployment.yaml`:

   ```yaml
   ...
   spec:
     template:
       spec:
         ...
         containers:
           - name: grafana
             image: grafana/grafana:latest
             imagePullPolicy: IfNotPresent
             env:    @_add_@
               # workaround for issue https://github.com/grafana/grafana/issues/58608   @_add_@
               - name: JAEGER_AGENT_HOST   @_add_@
                 value: ""   @_add_@
               - name: JAEGER_AGENT_PORT   @_add_@
                 value: ""   @_add_@
              ...
   ```

4. Verify the correctness of the configuration commands in the directory `${WAC_ROOT}/ambulance-gitops`.

```ps
kubectl kustomize clusters/localhost/prepare
kubeclt kustomize clusters/localhost/install
```

and archive the changes in the Git repository:

```ps
git add .
git commit -m "Added jaeger tracing"
git push
```

Verify that the new configurations have been applied in the cluster:

```ps
kubectl get kustomization -n wac-hospital
```

5. Go to the page [https://wac-hospital.loc/ui](https://wac-hospital.loc/ui), into your _Waiting List_ application, and perform several operations on the list. Then, navigate to the _Distributed Tracing_ application at [https://wac-hospital.loc/ui/jaeger](https://wac-hospital.loc/ui/jaeger). The application [Jaeger Query][jaeger] search window will appear. In the drop-down field _Service_, select the service `wac-hospital-gateway.wac-hospital` and press the _Find Traces_ button. The results of tracing corresponding to your previous activity will be displayed on the right side of the page.

![Search Traces in Distributed Tracing](./img/110-01-SearchTraces.png)

Click on one of the records—preferably one that includes the scope `open-policy-agent`. In a new window, you can analyze how individual computation scopes overlap, what attributes are assigned to individual scopes, and what is the time contribution of individual operations to the overall computation scope. Also, try different ways of displaying records using the drop-down field in the upper right corner.

![Analysis of Tracing Records](./img/110-02-TraceDetails.png)

Now, go to the page [https://wac-hospital.loc/http-echo?am-i-admin=yes](https://wac-hospital.loc/http-echo?am-i-admin=yes). In the resulting JSON file, pay attention to the headers `traceparent` and `tracestate`. These are generated by each service participating in fulfilling requests and represent [span context](https://opentelemetry.io/docs/concepts/signals/traces/#span-context). The initial trace and span are created by the [Envoy Gateway] service in our case, and parent spans are gradually passed to other services within the system. The format of these headers is standardized; you can find more about this format in the [W3C Trace Context](https://www.w3.org/TR/trace-context/#traceparent-header) specification.

```json
{
"path": "/http-echo",
"headers": {
    "host": "wac-hospital.loc",
    ...
    "traceparent": "00-132e2f75d9acb963a7b615142af630bc-ed213bfdbcdae197-01", @_important_@
    "tracestate": "" @_important_@
},
...
}
```
