# Creating custom operational metrics using OpenTelemetry SDK

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-100
```

---

In the previous section, we deployed [Prometheus] and [Grafana] services into our system, enabling us to collect and display various operational metrics of our system, mostly obtained by monitoring Kubernetes system parameters. However, these metrics are not always sufficient as they do not include metrics from our application, which are essential for us. In this section, we will demonstrate how to create custom operational metrics. To achieve this, we need to modify our source code. In our case, we will be using the [OpenTelemetry] library, allowing us to create custom metrics and export them in a format compatible with the [Prometheus] service.

The [OpenTelemetry] library, also known as SDK, is the result of integrating various projects focused on application monitoring. It provides a unified library for creating metrics, distributed tracing, and logging, collectively referred to as [_Signals_](https://opentelemetry.io/docs/concepts/signals/metrics/). These signals can then be exported in various formats. The format and [OpenTelemetry specification](https://opentelemetry.io/docs/specs/) are supported by most well-known libraries and can be integrated with services from different providers. The SDK is implemented in various programming languages, and in our case, we will be using the implementation for the [Go](https://opentelemetry.io/docs/instrumentation/go/) language. In this section, we will focus only on a few aspects.

1. In the first step, we need to prepare the instrumentation (collection of various measurements and signals) for the program. Open the file `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go` and modify it:

```go
import (
    "context"
    _ "embed"
    "log" @_add_@
    "net/http"
    "os"
    "strings"
    
    "github.com/gin-gonic/gin"
    "github.com/milung/ambulance-webapi/api"
    "github.com/milung/ambulance-webapi/internal/ambulance_wl"
    "github.com/milung/ambulance-webapi/internal/db_service"
    "github.com/prometheus/client_golang/prometheus/promhttp"  @_add_@
    "github.com/technologize/otel-go-contrib/otelginmetrics"  @_add_@
    "go.opentelemetry.io/otel"     @_add_@
    "go.opentelemetry.io/otel/attribute"     @_add_@
    "go.opentelemetry.io/otel/exporters/prometheus"     @_add_@
    "go.opentelemetry.io/otel/sdk/metric"     @_add_@
    "go.opentelemetry.io/otel/sdk/resource"     @_add_@
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"     @_add_@
)

// initialize OpenTelemetry instrumentations   @_add_@
func initTelemetry() error {   @_add_@
    ctx := context.Background()   @_add_@
    res, err := resource.New(ctx,   @_add_@
    resource.WithAttributes(semconv.ServiceNameKey.String("Ambulance WebAPI Service")),   @_add_@
    resource.WithAttributes(semconv.ServiceNamespaceKey.String("WAC Hospital")),   @_add_@
    resource.WithSchemaURL(semconv.SchemaURL),   @_add_@
    resource.WithContainer(),   @_add_@
    )   @_add_@
    @_add_@
    if err != nil {   @_add_@
    return err   @_add_@
    }   @_add_@
    @_add_@
    metricExporter, err := prometheus.New()   @_add_@
    if err != nil {   @_add_@
    return err   @_add_@
    }   @_add_@
    @_add_@
    metricProvider := metric.NewMeterProvider(metric.WithReader(metricExporter), metric.WithResource(res))   @_add_@
    otel.SetMeterProvider(metricProvider)   @_add_@
    return nil @_add_@
}   @_add_@
```

The `initTelemetry()` function prepares a global instance of the metric provider - [_metric provider_](https://opentelemetry.io/docs/concepts/signals/metrics/#meter-provider), which will be responsible for collecting metrics and exporting them. All our metrics (and later [_traces_](https://opentelemetry.io/docs/concepts/signals/traces/)) will be associated with an object described by an instance created by calling the `resource.New` function, to which we assign attributes that uniquely identify this object.

Simultaneously, we have created an instance of `metricExporter`, which is responsible for providing - [_export_](https://opentelemetry.io/docs/concepts/components/#exporters) of metrics in a format that the [Prometheus] service can process.

Next, modify the `main()` function in the same file `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go`:


```go
func main() {
    log.Printf("Server started")

    port := os.Getenv("AMBULANCE_API_PORT")
    if port == "" {
        port = "8080"
    }

    environment := os.Getenv("AMBULANCE_API_ENVIRONMENT")
    if !strings.EqualFold(environment, "production") { // case insensitive comparison
        gin.SetMode(gin.DebugMode)
    }
    engine := gin.New()
    engine.Use(gin.Recovery())

    // setup telemetry    @_add_@
    initTelemetry()    @_add_@
    @_add_@
    // instrument gin engine
    engine.Use(otelginmetrics.Middleware(    @_add_@
        "Ambulance WebAPI Service",    @_add_@
        // Custom attributes    @_add_@
        otelginmetrics.WithAttributes(func(serverName, route string, request *http.Request) []attribute.KeyValue {    @_add_@
            return append(otelginmetrics.DefaultAttributes(serverName, route, request))    @_add_@
        }),    @_add_@
    ))    @_add_@

    // setup context update  middleware
    dbService := db_service.NewMongoService[ambulance_wl.Ambulance](db_service.MongoServiceConfig{})
    defer dbService.Disconnect(context.Background())
    engine.Use(func(ctx *gin.Context) {
        ctx.Set("db_service", dbService)
        ctx.Next()
    })

    // request routings
    ambulance_wl.AddRoutes(engine)

    // openapi spec endpoint
    engine.GET("/openapi", api.HandleOpenApi)

    // metrics endpoint   @_add_@
    promhandler := promhttp.Handler()   @_add_@
    engine.Any("/metrics", func(ctx *gin.Context) {   @_add_@
        promhandler.ServeHTTP(ctx.Writer, ctx.Request)   @_add_@
    })   @_add_@

    engine.Run(":" + port)
}
```

In addition to calling the `initTelemetry()` function itself, we have ensured the instrumentation of the [gin] library using the `otelginmetrics.Middleware()` function. This function is responsible for collecting metrics for each request passing through our server, such as the volume of transferred data or the number of active requests at a given time. Finally, we added an endpoint `/metrics`, which will serve for reading metrics by the [Prometheus] service.

If we were to finish our modifications now, we would obtain a set of additional metrics that we could display in the [Grafana] service. Simultaneously, we could access the `/metrics` path, where we would see individual measurement records in the _Prometheus_ service format.

2. Our goal is to generate two additional metrics: the total time spent waiting for a database response and the current number of waiting patients in individual clinics. The latter of these metrics is not quite an _operational_ metric; we will mainly show it as an example of an asynchronous metric. [OpenTelemetry] distinguishes between synchronous and asynchronous metrics. Synchronous measurements are created and stored synchronously with the ongoing computation, while asynchronous measurements are obtained only if the client needs them - the measurement is triggered asynchronously using a _callback_ function.

Modify the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/utils_ambulance_updater.go`:


```go
package ambulance_wl

import (
    "context"
    "fmt" @_add_@
    "log" @_add_@
    "net/http"
    "time"  @_add_@

    "github.com/gin-gonic/gin"
    "github.com/milung/ambulance-webapi/internal/db_service"
    "go.opentelemetry.io/otel"   @_add_@
    "go.opentelemetry.io/otel/attribute"   @_add_@
    "go.opentelemetry.io/otel/metric"   @_add_@
)

var (    @_add_@
    dbMeter           = otel.Meter("waiting_list_access")    @_add_@
    dbTimeSpent       metric.Float64Counter    @_add_@
    waitingListLength = map[string]int64{}    @_add_@
)    @_add_@
    @_add_@
// package initialization - called automaticaly by go runtime when package is used    @_add_@
func init() {    @_add_@
    // initialize OpenTelemetry instrumentations    @_add_@
    var err error    @_add_@
    dbTimeSpent, err = dbMeter.Float64Counter(    @_add_@
        "ambulance_wl_time_spent_in_db",    @_add_@
        metric.WithDescription("The time spent in the database requests"),    @_add_@
        metric.WithUnit("ms"),    @_add_@
    )    @_add_@
    @_add_@
    if err != nil {    @_add_@
        panic(err)    @_add_@
    }    @_add_@
}    @_add_@
```

The `dbMeter` instance represents the measurement range - [instrumentation scope](https://opentelemetry.io/docs/concepts/instrumentation-scope/) labeled as `waiting_list_access`, in which we subsequently create the `dbTimeSpent` metric.

In the same file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/utils_ambulance_updater.go`, modify the `updateAmbulanceFunc` function:


```go
func updateAmbulanceFunc(ctx *gin.Context, updater ambulanceUpdater) {
    ...
    db, ok := value.(db_service.DbService[Ambulance])
    if !ok {
        ...
    }

    ambulanceId := ctx.Param("ambulanceId")

    start := time.Now() @_add_@
    ambulance, err := db.FindDocument(ctx, ambulanceId)
    dbTimeSpent.Add(ctx, float64(float64(time.Since(start)))/float64(time.Millisecond), metric.WithAttributes(   @_add_@
        attribute.String("operation", "find"),   @_add_@
        attribute.String("ambulance_id", ambulanceId),   @_add_@
        attribute.String("ambulance_name", ambulance.Name),   @_add_@
    ))   @_add_@   
    ...
    updatedAmbulance, responseObject, status := updater(ctx, ambulance)

    if updatedAmbulance != nil {
        start := time.Now() @_add_@
        err = db.UpdateDocument(ctx, ambulanceId, updatedAmbulance)

        // update metrics    @_add_@
        dbTimeSpent.Add(ctx, float64(float64(time.Since(start)))/float64(time.Millisecond), metric.WithAttributes(    @_add_@
            attribute.String("operation", "update"),    @_add_@
            attribute.String("ambulance_id", ambulanceId),    @_add_@
            attribute.String("ambulance_name", ambulance.Name),    @_add_@
        ))@_add_@
        @_add_@
        // demonstration of possible handling of async instruments:    @_add_@
        // not really an operational metric, it would be more of a business metric/KPI.    @_add_@
        // also UpDownCounter may be of better use in practical cases.    @_add_@
        if _, ok := waitingListLength[ambulanceId]; !ok {    @_add_@
            newGauge, err := dbMeter.Int64ObservableGauge(    @_add_@
                fmt.Sprintf("%v_waiting_patients", ambulanceId),    @_add_@
                metric.WithDescription(fmt.Sprintf("The length of the waiting list for the ambulance %v", ambulance.Name)),    @_add_@
                metric.WithUnit("{patient}"),    @_add_@
            )    @_add_@
            if err != nil {    @_add_@
                log.Printf("Failed to create waiting list length gauge for ambulance %v: %v", ambulanceId, err)    @_add_@
            }    @_add_@
            waitingListLength[ambulanceId] = 0    @_add_@
            @_add_@
            _, err = dbMeter.RegisterCallback(func(_ context.Context, o metric.Observer) error {    @_add_@
                // we could have looked up the ambulance in the database here, but we already have it in memory    @_add_@
                // so use the latest snapshots to update the gauge    @_add_@
                o.ObserveInt64(newGauge, waitingListLength[ambulanceId])    @_add_@
                return nil    @_add_@
            }, newGauge)    @_add_@
            @_add_@
            if err != nil {    @_add_@
                log.Printf("Failed to register callback for waiting list length gauge for ambulance %v: %v", ambulanceId, err)    @_add_@
            }    @_add_@
        }@_add_@
        @_add_@
        // set the gauge snapshot    @_add_@
        waitingListLength[ambulanceId] = int64(len(updatedAmbulance.WaitingList))    @_add_@

    } else {
        err = nil // redundant but for clarity
    }
    ...
}
```

The changes in this function now measure the time spent calling the database when reading and modifying data in the waiting list, synchronously recording these changes in the `dbTimeSpent` metric. Simultaneously, an asynchronous metric `newGauge` is dynamically created for each clinic, and a _callback_ function is registered to provide the current number of waiting patients in that clinic based on the last known value. This metric is asynchronous because it is provided only when the client requests its value.

3. Save the changes and in the `${WAC_ROOT}/ambulance-webapi` directory, execute the following commands:


```ps
go get github.com/technologize/otel-go-contrib
go mod tidy
./scripts/run.ps1 start
```

In the browser, open the page [http://localhost:8080/metrics](http://localhost:8080/metrics), which will display existing metrics in [Prometheus] format. Currently, the metrics created for the `waiting_list_access` scope are not visible. Stop the process.

4. For the [Prometheus] system to be aware of the new metrics, we need to modify the deployment configuration of our application. Specifically, we need to add annotations under our pod specifications. The _Prometheus_ service is configured to search for objects of type [_Service_](https://kubernetes.io/docs/concepts/services-networking/service/) and type [_Pod_](https://kubernetes.io/docs/concepts/workloads/pods/) with annotations specified in the `prometheus-server` service configuration. Modify the file `${WAC_ROOT}/deployments/kustomize/install/deployment.yaml`:

```yaml
...
spec:
    replicas: 1
    selector:
    matchLabels:
        pod: milung-ambulance-webapi-label 
    template:
    metadata:
        labels:
        pod: milung-ambulance-webapi-label 
        annotations:   @_add_@
        prometheus.io/scrape: 'true'   @_add_@
        prometheus.io/path: '/metrics'   @_add_@
        prometheus.io/port: '8080'   @_add_@

    spec:
    ...
```

5. Save and archive the changes in the `${WAC_ROOT}/ambulance-webapi` directory:

```ps
git add .
git commit -m "Added custom metrics"
git push
```

Verify that our changes were successfully deployed with the command:

```ps
kubectl get kustomization -n wac-hospital
```

Go to the page [https://wac-hospital.loc/ui/](https://wac-hospital.loc/ui/) and in your _Waiting List <pfx>_ application, create several entries. Wait for approximately 2 minutes until the [Prometheus] service activates the metric loading process. Go to the _System Dashboards_ application, and in the left navigation panel, select _Explore_. In the _Metric_ field, search for measurements `bobulova_waiting_patients` and press the _Run query_ button. In the graph, you should see values corresponding to the current number of waiting patients. If you continue to create or delete entries in the waiting list, the graph values should respond accordingly, with the appropriate delay. Similarly, you can monitor the metric `ambulance_wl_time_spent_in_db_milliseconds_total`, which shows the total time spent reading and writing to the database.

![Graph of the metric bobulova_waiting_patients](./img/100-01-AmbulanceWaitingListMetric.png)

> Homework: Create dashboards for these metrics in the _System Dashboards_ application - [Grafana]. Also, try metrics starting with the prefix `http_server_` and `promhttp_`.
