# Adding Distributed Tracing to Web API Service

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-120
```

---

In the previous section, we saw how to analyze individual requests using distributed tracing. However, the recorded traces were quite coarse, and we still lack additional details regarding the ongoing computation. In this section, we will demonstrate how to add custom computation scopes - _spans_ - to the application.

1. Open the file `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go` and make the following modifications:

```go
package main

import (
    ...
    "time"   @_add_@
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"   @_add_@
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"   @_add_@
    "go.opentelemetry.io/otel/sdk/trace"   @_add_@
    "go.opentelemetry.io/otel/propagation"   @_add_@
)

// initialize OpenTelemetry instrumentations
func initTelemetry()  error { @_remove_@
func initTelemetry() (func(context.Context) error, error) { @_add_@
    ...
    metricProvider := metric.NewMeterProvider(metric.WithReader(metricExporter), metric.WithResource(res))
    otel.SetMeterProvider(metricProvider)

    // setup trace exporter, only otlp supported      @_add_@
    // see also https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/exporters/autoexport      @_add_@
    traceExportType := os.Getenv("OTEL_TRACES_EXPORTER")      @_add_@
    if traceExportType == "otlp" {      @_add_@
        ctx, cancel := context.WithTimeout(ctx, time.Second)      @_add_@
        defer cancel()      @_add_@
        // we will configure exporter by using env variables defined      @_add_@
        // at https://opentelemetry.io/docs/concepts/sdk-configuration/otlp-exporter-configuration/      @_add_@
        traceExporter, err := otlptracegrpc.New(ctx)      @_add_@
        if err != nil {      @_add_@
            return nil, err      @_add_@
        }      @_add_@
        @_add_@
        traceProvider := trace.NewTracerProvider(      @_add_@
            trace.WithResource(res),      @_add_@
            trace.WithSyncer(traceExporter))      @_add_@
        @_add_@
        otel.SetTracerProvider(traceProvider)      @_add_@
        otel.SetTextMapPropagator(propagation.TraceContext{})      @_add_@
        // Shutdown function will flush any remaining spans      @_add_@
        return traceProvider.Shutdown, nil      @_add_@
    } else {      @_add_@
        // no otlp trace exporter configured      @_add_@
        noopShutdown := func(context.Context) error { return nil }      @_add_@
        return noopShutdown, nil      @_add_@
    }      @_add_@

}
```

Also, simultaneously modify the return values in the omitted lines.

We have added initialization of the [_TraceProvider_](https://opentelemetry.io/docs/concepts/signals/traces/#tracer-provider) in the `initTelemetry()` function. The actual configuration will be done later using environment variables. It is crucial to note that we create this section only if the environment variable `OTEL_TRACES_EXPORTER` is explicitly set to the value `otlp` - we do not support any other way of storing records. Since the [_Trace Exporter_](https://opentelemetry.io/docs/concepts/signals/traces/#trace-exporters) stores records asynchronously and in batches, it is necessary to ensure that all records are stored before the application exits. This is achieved by the `Shutdown()` function, the instance of which is the return value of the `initTelemetry()` function. If the environment variable `OTEL_TRACES_EXPORTER` is not set to the value `otlp`, we return a function that does nothing.

In the same file `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go`, modify the `main()` function:

```go
func main() {
    
    // setup telemetry
    err := initTelemetry() @_remove_@
    shutdown, err := initTelemetry() @_add_@
    if err != nil {
        log.Fatalf("Failed to initialize telemetry: %v", err)
    }
    defer func() { _ = shutdown(context.Background()) }() @_add_@

    // instrument gin engine
    engine.Use(
        otelginmetrics.Middleware(
            ...
        ),
        otelgin.Middleware(serverName), @_add_@
    )

    ...
}
```

The `main()` function has not changed much, except that we ensured the call to the `Shutdown()` function upon program termination and added instrumentation for the [Gin] library.

These modifications should ensure that all requests handled by our WebAPI will now generate traces and send them to a eventually configured server supporting the [OTLP](https://opentelemetry.io/docs/specs/otlp/) protocol.

2. Open the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/utils_ambulance_updater.go` and make the following modifications:

```go
...
var (
    dbMeter           = otel.Meter("waiting_list_access")
    dbTimeSpent       metric.Float64Counter
    waitingListLength = map[string]int64{}
    tracer            = otel.Tracer("ambulance-wl-api") @_add_@
)
...
func updateAmbulanceFunc(ctx *gin.Context, updater ambulanceUpdater) {
    // special handling for gin context    @_add_@
    // we need to extract the span context and create a new context to ensure span context propagation    @_add_@
    // to the updater function    @_add_@
    spanctx, span := tracer.Start(ctx.Request.Context(), "updateAmbulanceFunc")    @_add_@
    ctx.Request = ctx.Request.WithContext(spanctx)    @_add_@
    ..

    start := time.Now()
    ambulance, err := db.FindDocument(ctx, ambulanceId) @_remove_@
    ambulance, err := db.FindDocument(spanctx, ambulanceId) @_add_@
    dbTimeSpent.Add(ctx, float64(float64(time.Since(start)))/float64(time.Millisecond), metric.WithAttributes(
        attribute.String("operation", "find"),
        attribute.String("ambulance_id", ambulanceId),
        attribute.String("ambulance_name", ambulance.Name),
    ))

    if err != nil {  @_add_@
        span.SetStatus(codes.Error, err.Error())  @_add_@
    }  @_add_@

    switch err {
    ...

    if updatedAmbulance != nil {
        span.AddEvent("updateAmbulanceFunc: updating ambulance in database")
        start := time.Now()
        err = db.UpdateDocument(ctx, ambulanceId, updatedAmbulance) @_remove_@
        err = db.UpdateDocument(spanctx, ambulanceId, updatedAmbulance) @_add_@
        ...
    } else {
        err = nil // redundant but for clarity
    }

    if err != nil {   @_add_@
        span.SetStatus(codes.Error, err.Error())   @_add_@
    }   @_add_@
    
    switch err {
    ...
}
```

At the beginning of the `updateAmbulanceFunc()` function, we created a new computation scope. [_Span Context_](https://opentelemetry.io/docs/concepts/signals/traces/#span-context), meaning the _trace id_ and the parent _span id_, is passed to the `tracer.Start()` function as an attribute of the variable `ctx`. The computation scope is concluded by calling the `span.End()` function. Notice how we created a new computation scope for the calls to the `db.FindDocument()` and `db.UpdateDocument()` functions as well. In case of an error, we set the computation scope status to `Error` and add the error message to the `status` attribute. These attributes will later be visible in the [Jaeger](https://www.jaegertracing.io/) tool.

Modify the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/`. We will demonstrate how to add records for computation scopes to the `UpdateWaitingListEntry()` function, and adjust the other functions similarly:

```go
package ambulance_wl

import (
    ...
    "go.opentelemetry.io/otel/attribute"   @_add_@
    "go.opentelemetry.io/otel/trace"   @_add_@
)

// UpdateWaitingListEntry - Updates specific entry
func (this *implAmbulanceWaitingListAPI) UpdateWaitingListEntry(ctx *gin.Context) {
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance, interface{}, int) {
        // special handling for gin context  @_add_@
        // we need to extract the span context and create a new context to ensure span context propagation  @_add_@
        // to the updater function  @_add_@
        spanctx, span := tracer.Start(  @_add_@
            c.Request.Context(),  @_add_@
            "UpdateWaitingListEntry",  @_add_@
            trace.WithAttributes(  @_add_@
                attribute.String("ambulance_id", ambulance.Id),  @_add_@
                attribute.String("ambulance_name", ambulance.Name),  @_add_@
            ),  @_add_@
        )  @_add_@
        c.Request = c.Request.WithContext(spanctx)  @_add_@
        defer span.End()   @_add_@
        ...
        ambulance.reconcileWaitingList()  @_remove_@
        ambulance.reconcileWaitingList(spanctx) @_add_@
        return ambulance, ambulance.WaitingList[entryIndx], http.StatusOK
    })
}
```

In most cases, we will create a new computation scope whenever we enter a new function, and we will use the `defer` statement to ensure the completion of the computation scope. The [_Span Context_](https://opentelemetry.io/docs/concepts/signals/traces/#span-context) is propagated between functions using the variable `ctx`.

> Homework: Independently add records for computation scopes to the remaining functions in the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/`. If relevant, set the error status of the scope using the `span.SetStatus()` function, and add important computation events using the `span.AddEvent()` function.

Finally, modify the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/ext_model_ambulance.go`:

```go
import (
    ...
    "context" @_add_@
    "go.opentelemetry.io/otel/attribute"   @_add_@
    "go.opentelemetry.io/otel/trace"   @_add_@
)

func (this *Ambulance) reconcileWaitingList() { @_remove_@
func (this *Ambulance) reconcileWaitingList(ctx context.Context) { @_add_@
    _, span := tracer.Start(ctx, "reconcileWaitingList",   @_add_@
        trace.WithAttributes(attribute.String("ambulanceId", this.Id)),   @_add_@
        trace.WithAttributes(attribute.String("ambulanceName", this.Name)),   @_add_@
    )   @_add_@
    defer span.End()   @_add_@

    slices.SortFunc(this.WaitingList, func(left, right WaitingListEntry) int {
    ...
}
```

To create a new computation scope, we need to have access to the [_Span Context_](https://opentelemetry.io/docs/concepts/signals/traces/#span-context). We obtain this from the added function argument.

3. Save the files and archive the changes in the directory `${WAC_ROOT}/ambulance-webapi`:

```ps
git add .
git commit -m "added tracing"
git push
```

4. Open the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-webapi/patches/ambulance-webapi.deployment.yaml` and add the environment variables needed for connecting the _Open Telemetry SDK_ to the `jaeger-collector` service:

```yaml
...
spec:
    template:
    spec:
        containers:
        - name: milung-ambulance-wl-webapi-container    @_add_@
            env:    @_add_@
            - name: OTEL_TRACES_EXPORTER    @_add_@
                value: otlp    @_add_@
            - name: OTEL_EXPORTER_OTLP_ENDPOINT    @_add_@
                value: http://jaeger-collector.wac-hospital:4317    @_add_@
            - name: OTEL_EXPORTER_OTLP_TRACES_INSECURE    @_add_@
                value: "true"    @_add_@
            - name: OTEL_EXPORTER_OTLP_PROTOCOL    @_add_@
                value: grpc    @_add_@
            - name: OTEL_TRACES_SAMPLER    @_add_@
                # see https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/#otel_traces_sampler    @_add_@
                value: "parentbased_always_on"    @_add_@
    - name: openapi-ui
```

Save the changes and archive them with commands in the directory `${WAC_ROOT}/ambulance-gitops`:

```ps
git add .
git commit -m "added webapi tracing"
git push
```

Wait for all changes to apply in the cluster.

5. Go to the page [https://wac-hospital.loc/ui](https://wac-hospital.loc/ui), open your _Waiting List for Patients_ application, and create, update, or change the list of waiting patients. Then, go to the _Distributed Tracing_ application - [https://wac-hospital.loc/ui/jaeger](https://wac-hospital.loc/ui/jaeger). In the drop-down field, select the service _Ambulance WEBAPI Service_ and search for records.

![Viewing records for the Ambulance WEBAPI Service](./img/120-01-Tracing-records.png)

Select one of the records, preferably one that shows the most spans, and view the details.

![Request computation scope details](./img/120-02-SpanDetails.png)

In the computation scope details for a specific request, you can see all the scopes and attributes assigned to them that we created in our application. You can see the contribution of each service to the overall computation time. Some service details are still missing - for example, for MongoDB. We will address this in the upcoming sections after deploying the [Linkerd] system.

> Homework: Analyze the requests to the _Ambulance WEBAPI Service_. Try to identify which parts of the code consume the most time and how these parts could be optimized. Are the informational records detailed enough? If not, try to add more information and scopes to your application. _Never include data in records that could compromise user privacy!_

&nbsp;

> Info: For [MongoDB], there is an option for instrumentation for distributed tracing, but we won't be using it here. More information can be found on the [Tracing & Logging](https://www.mongodb.com/docs/drivers/rust/current/fundamentals/tracing-logging/) page.
