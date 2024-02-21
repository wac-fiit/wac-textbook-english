# Implementation of WEB API and Data Persistence

---

>info:>
Template for a pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-api-040`

---

Document databases essentially store documents, which are organized into collections, and these collections are then stored in databases. Documents are essentially JSON objects that can contain any structure. In our case, we will be storing objects of type `Ambulance` in the `ambulances` collection. These objects will contain both information about the respective ambulance for the needs of our application, as well as information about the list of waiting patients and the list of symptoms from which patients choose an item when registering in the waiting list. All requests to our WEB API will then work with the corresponding document that corresponds to the selected ambulance.

1. In the first step, we will modify our API to include the definition of the corresponding object and allow creating or removing a record for the respective ambulance. Open the file `${WAC_ROOT}/ambulance-webapi/api/ambulance-wl.openapi.yaml` and add a new type `Ambulance` to the section `components/schemas`:


```yaml
...
components:
    schemas:
    WaitingListEntry: 
        ...
    Condition:
        ...
    Ambulance:   @_add_@
        type: object   @_add_@
        required: [ "id", "name", "roomNumber"]   @_add_@
        properties:   @_add_@
        id:   @_add_@
            type: string   @_add_@
            example: dentist-warenova   @_add_@
            description: Unique identifier of the ambulance   @_add_@
        name:   @_add_@
            type: string   @_add_@
            example: Zubná ambulancia Dr. Warenová   @_add_@
            description: Human readable display name of the ambulance   @_add_@
        roomNumber:   @_add_@
            type: string   @_add_@
            example: 356 - 3.posch   @_add_@
        waitingList:   @_add_@
            type: array   @_add_@
            items:   @_add_@
            $ref: '#/components/schemas/WaitingListEntry'   @_add_@
        predefinedConditions:   @_add_@
            type: array   @_add_@
            items:   @_add_@
            $ref: '#/components/schemas/Condition'   @_add_@
        example:   @_add_@
        $ref: "#/components/examples/AmbulanceExample"   @_add_@
examples: 
    ...
```

Next, add an example for the `Ambulance` type in the `examples` section.

```yaml
...
components: 
....
examples: 
    ...
    WaitingListEntriesExample:
    ...
    AmbulanceExample:           @_add_@
        summary: Sample GP ambulance          @_add_@
        description: |            @_add_@
        Example of GP ambulance with waiting list and predefined conditions         @_add_@
        value:            @_add_@
        id: gp-warenova         @_add_@
        name: Ambulancia všeobecného lekárstva Dr. Warenová         @_add_@
        roomNumber: 356 - 3.posch           @_add_@
        waitingList:            @_add_@
            - id: x321ab3         @_add_@
            name: Jožko Púčik           @_add_@
            patientId: 460527-jozef-pucik           @_add_@
            waitingSince: "2038-12-24T10:05:00.000Z"            @_add_@
            estimatedStart: "2038-12-24T10:35:00.000Z"          @_add_@
            estimatedDurationMinutes: 15            @_add_@
            condition:          @_add_@
            value: Teploty            @_add_@
            code: subfebrilia         @_add_@
            reference: "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/"          @_add_@
            - id: x321ab4         @_add_@
            name: Ferdinand Trety           @_add_@
            patientId: 780907-ferdinand-tre         @_add_@
            waitingSince: "2038-12-24T10:25:00.000Z"            @_add_@
            estimatedStart: "2038-12-24T10:50:00.000Z"          @_add_@
            estimatedDurationMinutes: 25            @_add_@
            condition:          @_add_@
            value: Nevoľnosť          @_add_@
            code: nausea          @_add_@
            reference: "https://zdravoteka.sk/priznaky/nevolnost/"            @_add_@
        predefinedConditions:           @_add_@
            - value: Teploty          @_add_@
            code: subfebrilia           @_add_@
            reference: "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/"            @_add_@
            typicalDurationMinutes: 20          @_add_@
            - value: Nevoľnosť            @_add_@
            code: nausea            @_add_@
            reference: "https://zdravoteka.sk/priznaky/nevolnost/"          @_add_@
            typicalDurationMinutes: 45          @_add_@
            - value: Kontrola         @_add_@
            code: followup          @_add_@
            typicalDurationMinutes: 15          @_add_@
            - value: Administratívny úkon         @_add_@
            code: administration            @_add_@
            typicalDurationMinutes: 10          @_add_@
            - value: Odber krvi           @_add_@
            code: blood-test            @_add_@
            typicalDurationMinutes: 10          @_add_@
```

Add a new tag `ambulances` to the `tags` section.

```yaml
...
tags:
- name: ambulanceWaitingList
    description: Ambulance Waiting List API
- name: ambulanceConditions
    description: Patient conditions and synptoms handled in the ambulance
- name: ambulances @_add_@
    description: Ambulance details   @_add_@
```

Add new paths and operations to the `paths` section.

```yaml
...
paths:
...
    "/waiting-list/{ambulanceId}/condition":
    ...
    "/ambulance":     @_add_@
    post:     @_add_@
        tags:     @_add_@
        - ambulances     @_add_@
        summary: Saves new ambulance definition     @_add_@
        operationId: createAmbulance     @_add_@
        description: Use this method to initialize new ambulance in the system     @_add_@
        requestBody:     @_add_@
        content:     @_add_@
            application/json:     @_add_@
            schema:     @_add_@
                $ref: "#/components/schemas/Ambulance"     @_add_@
            examples:     @_add_@
                request-sample:      @_add_@
                $ref: "#/components/examples/AmbulanceExample"     @_add_@
        description: Ambulance details to store     @_add_@
        required: true     @_add_@
        responses:     @_add_@
        "200":     @_add_@
            description: >-     @_add_@
            Value of stored ambulance     @_add_@
            content:     @_add_@
            application/json:     @_add_@
                schema:     @_add_@
                $ref: "#/components/schemas/Ambulance"     @_add_@
                examples:     @_add_@
                updated-response:      @_add_@
                    $ref: "#/components/examples/AmbulanceExample"     @_add_@
        "400":     @_add_@
            description: Missing mandatory properties of input object.     @_add_@
        "409":     @_add_@
            description: Entry with the specified id already exists     @_add_@
    "/ambulance/{ambulanceId}":     @_add_@
    delete:     @_add_@
        tags:     @_add_@
        - ambulances     @_add_@
        summary: Deletes specific ambulance     @_add_@
        operationId: deleteAmbulance     @_add_@
        description: Use this method to delete the specific ambulance from the system.     @_add_@
        parameters:     @_add_@
        - in: path     @_add_@
            name: ambulanceId     @_add_@
            description: pass the id of the particular ambulance     @_add_@
            required: true     @_add_@
            schema:     @_add_@
            type: string     @_add_@
        responses:     @_add_@
        "204":     @_add_@
            description: Item deleted     @_add_@
        "404":     @_add_@
            description: Ambulance with such ID does not exist     @_add_@
```

2. Generate a new version of the skeleton for the server-side of our WEB API. In the directory `${WAC_ROOT}/ambulance-webapi`, execute the following command:

```ps
./scripts/run.ps1 openapi
```

In the project, you will find a new file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/api_ambulances.go`. Create a file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulances.go` with the following content:

```go
package ambulance_wl

import (
"net/http"

"github.com/gin-gonic/gin"
)

// CreateAmbulance - Saves new ambulance definition
func (this *implAmbulancesAPI) CreateAmbulance(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented)
}

// DeleteAmbulance - Deletes specific ambulance
func (this *implAmbulancesAPI) DeleteAmbulance(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented)
}
```

This implementation is temporary. Before its completion, we will prepare a helper class responsible for database access.

3. To access and work with documents, we will create new classes in a separate `db_service` package. Serialization of objects will use annotations generated by the openapi code generator. For example, the type `Condition` in the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/model_condition.go` is defined as follows:

```go
type Condition struct {

    Value string `json:"value"`

    Code string `json:"code,omitempty"`

    // Link to encyclopedical explanation of the patient's condition
    string `json:"reference,omitempty"`

    TypicalDurationMinutes int32 `json:"typicalDurationMinutes,omitempty"`
}
```

The `json:"value"` annotation specifies that the `Value` property will be serialized into the JSON object under the key `value`. The mongo library supports both `json` and `bson` (binary JSON) annotations. This will allow using this class directly for serialization and deserialization of objects from the database.

> Because the [Go] programming language does not allow creating cyclic dependencies between individual packages, the implementation of `db_service` will use type templates - [generics]. This means it will not use any types from the `ambulance_wl` package.

Create a file `${WAC_ROOT}/ambulance-webapi/internal/db_service/mongo_svc.go` with the following content:


```go
package db_service

import (
    "context"
    "fmt"
    "log"
    "os"
    "strconv"
    "sync"
    "sync/atomic"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type DbService[DocType interface{}] interface {
    CreateDocument(ctx context.Context, id string, document *DocType) error
    FindDocument(ctx context.Context, id string) (*DocType, error)
    UpdateDocument(ctx context.Context, id string, document *DocType) error
    DeleteDocument(ctx context.Context, id string) error
    Disconnect(ctx context.Context) error
}

var ErrNotFound = fmt.Errorf("document not found")
var ErrConflict = fmt.Errorf("conflict: document already exists")

type MongoServiceConfig struct {
    ServerHost string
    ServerPort int
    UserName   string
    Password   string
    DbName     string
    Collection string
    Timeout    time.Duration
}

type mongoSvc[DocType interface{}] struct {
    MongoServiceConfig
    client     atomic.Pointer[mongo.Client]
    clientLock sync.Mutex
}
```

The `DbService` interface is a generic interface, and the specific type of the instance will be determined when it is created by binding to the `DocType` type. In the rest of our implementation, we will assume that we are working with a [document-oriented database](https://en.wikipedia.org/wiki/Document-oriented_database), but we will not introduce explicit dependencies on a specific implementation. In our case, the concrete implementation will be the `mongoSvc` class, which will use the [mongo-go-driver](https://github.com/mongodb/mongo-go-driver) library. Note that this type is not visible outside the `db_service` package (it starts with a lowercase letter), which means we will only be able to use it within this package.

4. The `mongoSvc` type will be shared among individual requests coming to our server. Therefore, it will also include a synchronization mechanism that ensures only one instance of the `mongo.Client` class exists at a time. The code of the `mongo.Client` class is reentrant, so it is not necessary to synchronize access to its methods.

When creating a new instance, we will use either explicitly provided parameters in the `MongoServiceConfig` type or, if not provided, attempt to read them from environment variables or use default values. Add the following code to the end of the file `${WAC_ROOT}/ambulance-webapi/internal/db_service/mongo_svc.go`:


```go
func NewMongoService[DocType interface{}](config MongoServiceConfig) DbService[DocType] {
    enviro := func(name string, defaultValue string) string {
        if value, ok := os.LookupEnv(name); ok {
            return value
        }
        return defaultValue
    }

    svc := &mongoSvc[DocType]{}
    svc.MongoServiceConfig = config

    if svc.ServerHost == "" {
        svc.ServerHost = enviro("AMBULANCE_API_MONGODB_HOST", "localhost")
    }

    if svc.ServerPort == 0 {
        port := enviro("AMBULANCE_API_MONGODB_PORT", "27017")
        if port, err := strconv.Atoi(port); err == nil {
            svc.ServerPort = port
        } else {
            log.Printf("Invalid port value: %v", port)
            svc.ServerPort = 27017
        }
    }

    if svc.UserName == "" {
        svc.UserName = enviro("AMBULANCE_API_MONGODB_USERNAME", "")
    }

    if svc.Password == "" {
        svc.Password = enviro("AMBULANCE_API_MONGODB_PASSWORD", "")
    }

    if svc.DbName == "" {
        svc.DbName = enviro("AMBULANCE_API_MONGODB_DATABASE", "<pfx>-ambulance-wl")
    }

    if svc.Collection == "" {
        svc.Collection = enviro("AMBULANCE_API_MONGODB_COLLECTION", "ambulance")
    }

    if svc.Timeout == 0 {
        seconds := enviro("AMBULANCE_API_MONGODB_TIMEOUT_SECONDS", "10")
        if seconds, err := strconv.Atoi(seconds); err == nil {
            svc.Timeout = time.Duration(seconds) * time.Second
        } else {
            log.Printf("Invalid timeout value: %v", seconds)
            svc.Timeout = 10 * time.Second
        }
    }

    log.Printf(
        "MongoDB config: //%v@%v:%v/%v/%v",
        svc.UserName,
        svc.ServerHost,
        svc.ServerPort,
        svc.DbName,
        svc.Collection,
    )
    return svc
}
```

>info:> We recommend familiarizing yourself with the [viper] library, which provides more flexible ways of configuring the application in the target environment.

5. Further, add auxiliary functions for connecting to and disconnecting from the database in the file `${WAC_ROOT}/ambulance-webapi/internal/db_service/mongo_svc.go`. Notice how synchronization of access to the `mongo.Client` class instance is ensured. Although we could universally use the `clientLock` mutex, accessing it could trigger interruption of the computation thread, which would be inefficient in terms of overall performance. Therefore, we use [atomic pointer](https://pkg.go.dev/sync/atomic#Pointer), which allows atomic reading and writing of values, and only when we need to create or remove a client instance, we enter a critical section controlled by a mutex. Since the computation process may be terminated by an abrupt exception, we use the `defer` construct, ensuring that the specified expression is executed whenever we exit the given function.

```go
func (this *mongoSvc[DocType]) connect(ctx context.Context) (*mongo.Client, error) {
    // optimistic check
    client := this.client.Load()
    if client != nil {
        return client, nil
    }

    this.clientLock.Lock()
    defer this.clientLock.Unlock()
    // pesimistic check
    client = this.client.Load()
    if client != nil {
        return client, nil
    }

    ctx, contextCancel := context.WithTimeout(ctx, this.Timeout)
    defer contextCancel()

    var uri = fmt.Sprintf("mongodb://%v:%v", this.ServerHost, this.ServerPort)
    log.Printf("Using URI: " + uri)

    if len(this.UserName) != 0 {
        uri = fmt.Sprintf("mongodb://%v:%v@%v:%v", this.UserName, this.Password, this.ServerHost, this.ServerPort)
    }

    if client, err := mongo.Connect(ctx, options.Client().ApplyURI(uri).SetConnectTimeout(10*time.Second)); err != nil {
        return nil, err
    } else {
        this.client.Store(client)
        return client, nil
    }
}

func (this *mongoSvc[DocType]) Disconnect(ctx context.Context) error {
    client := this.client.Load()

    if client != nil {
        this.clientLock.Lock()
        defer this.clientLock.Unlock()

        client = this.client.Load()
        defer this.client.Store(nil)
        if client != nil {
            if err := client.Disconnect(ctx); err != nil {
                return err
            }
        }
    }
    return nil
}
```

6. Subsequently, add the implementation of the `DbService` interface for the `mongoSvc` type in the file `${WAC_ROOT}/ambulance-webapi/internal/db_service/mongo_svc.go`. In all methods, attempt to load a document (ambulance) with the corresponding id, and if it is not found in the database, return a predefined instance of the `error` type `ErrNotFoundE`. In the case of creating a document, return an error `ErrConflict` if such a document already exists.

Note the use of the [`context.Context`](https://pkg.go.dev/context) type. This type is commonly used as the first parameter of functions that utilize asynchronous data processing or implement long-running computations. `Context` allows propagating a request for premature termination of the computation to its subordinate contexts created by its `With...` methods. It also allows passing data shared across threads that are common to the entire computation. In our case, we use `Context` to time-limit the duration during which the client should attempt to retrieve a response from the connected database.

The `context.Cancel()` call sets the context to the `Done` state, which we could use if we were waiting for the asynchronous completion of the computation or if we wanted to prematurely terminate the context-controlled computation from another thread. However, we do not utilize this option here. Remember that newly created instances of the `context.Context` type are always in the `Done` state, so it is necessary to create them using the `context.With...` methods, and at the end of their lifecycle, the `context.Cancel()` function must be called. In our case, we achieve this using the `defer` construct and calling the provided `contextCancel()` function, which internally invokes the mentioned `context.Cancel()` method.

```go
func (this *mongoSvc[DocType]) CreateDocument(ctx context.Context, id string, document *DocType) error {
    ctx, contextCancel := context.WithTimeout(ctx, this.Timeout)
    defer contextCancel()
    client, err := this.connect(ctx)
    if err != nil {
        return err
    }
    db := client.Database(this.DbName)
    collection := db.Collection(this.Collection)
    result := collection.FindOne(ctx, bson.D{{Key: "id", Value: id}})
    switch result.Err() {
    case nil: // no error means there is conflicting document
        return ErrConflict
    case mongo.ErrNoDocuments:
        // do nothing, this is expected
    default: // other errors - return them
        return result.Err()
    }

    _, err = collection.InsertOne(ctx, document)
    return err
}

func (this *mongoSvc[DocType]) FindDocument(ctx context.Context, id string) (*DocType, error) {
    ctx, contextCancel := context.WithTimeout(ctx, this.Timeout)
    defer contextCancel()
    client, err := this.connect(ctx)
    if err != nil {
        return nil, err
    }
    db := client.Database(this.DbName)
    collection := db.Collection(this.Collection)
    result := collection.FindOne(ctx, bson.D{{Key: "id", Value: id}})
    switch result.Err() {
    case nil:
    case mongo.ErrNoDocuments:
        return nil, ErrNotFound
    default: // other errors - return them
        return nil, result.Err()
    }
    var document *DocType
    if err := result.Decode(&document); err != nil {
        return nil, err
    }
    return document, nil
}

func (this *mongoSvc[DocType]) UpdateDocument(ctx context.Context, id string, document *DocType) error {
    ctx, contextCancel := context.WithTimeout(ctx, this.Timeout)
    defer contextCancel()
    client, err := this.connect(ctx)
    if err != nil {
        return err
    }
    db := client.Database(this.DbName)
    collection := db.Collection(this.Collection)
    result := collection.FindOne(ctx, bson.D{{Key: "id", Value: id}})
    switch result.Err() {
    case nil:
    case mongo.ErrNoDocuments:
        return ErrNotFound
    default: // other errors - return them
        return result.Err()
    }
    _, err = collection.ReplaceOne(ctx, bson.D{{Key: "id", Value: id}}, document)
    return err
}

func (this *mongoSvc[DocType]) DeleteDocument(ctx context.Context, id string) error {
    ctx, contextCancel := context.WithTimeout(ctx, this.Timeout)
    defer contextCancel()
    client, err := this.connect(ctx)
    if err != nil {
        return err
    }
    db := client.Database(this.DbName)
    collection := db.Collection(this.Collection)
    result := collection.FindOne(ctx, bson.D{{Key: "id", Value: id}})
    switch result.Err() {
    case nil:
    case mongo.ErrNoDocuments:
        return ErrNotFound
    default: // other errors - return them
        return result.Err()
    }
    _, err = collection.DeleteOne(ctx, bson.D{{Key: "id", Value: id}})
    return err
}
```

With this, our database access approach is implemented.

7. In order to utilize our `DbService` interface in the request handling code, which we generated in the `ambulance_wl` package, we will add its instance to the context passed as an argument to the generated functions. To achieve this, we will use [_middleware_](https://gin-gonic.com/docs/examples/custom-middleware/) functions, which we will register in the `gin` library's _router_. Additionally, we will add CORS configuration.

Open the file `${WAC_ROOT}/workspaces/wac-test/ambulance-webapi/cmd/ambulance-api-service/main.go` and add the following code to the `main` function:

```go
package main

import (
    ...
    "github.com/<github_id>/ambulance-webapi/internal/db_service" @_add_@
    "context" @_add_@
    "time" @_add_@
    "github.com/gin-contrib/cors" @_add_@
)

func main() {
    ...
    engine := gin.New()
    engine.Use(gin.Recovery())

    corsMiddleware := cors.New(cors.Config{     @_add_@
        AllowOrigins:     []string{"*"},     @_add_@
        AllowMethods:     []string{"GET", "PUT", "POST", "DELETE", "PATCH"},     @_add_@
        AllowHeaders:     []string{"Origin", "Authorization", "Content-Type"},     @_add_@
        ExposeHeaders:    []string{""},     @_add_@
        AllowCredentials: false,     @_add_@
        MaxAge: 12 * time.Hour,     @_add_@
    })     @_add_@
    engine.Use(corsMiddleware)     @_add_@

    // setup context update  middleware     @_add_@
    dbService := db_service.NewMongoService[ambulance_wl.Ambulance](db_service.MongoServiceConfig{})     @_add_@
    defer dbService.Disconnect(context.Background())     @_add_@
    engine.Use(func(ctx *gin.Context) {     @_add_@
        ctx.Set("db_service", dbService)     @_add_@
        ctx.Next()     @_add_@
    })     @_add_@
    // request routings
    ambulance_wl.AddRoutes(engine)
    ...
}
```

8. Now, let's proceed with the implementation of the request handling. We'll start with the `implAmbulancesAPI` class. Open the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulances.go` and modify the `CreateAmbulance` method:

```go
package ambulance_wl

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid" @_add_@
    "github.com/<github_id>/ambulance-webapi/internal/db_service" @_add_@
)

// CreateAmbulance - Saves new ambulance definition
func (this *implAmbulancesAPI) CreateAmbulance(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented)  @_remove_@
    // get db service from context
    value, exists := ctx.Get("db_service")    @_add_@
    if !exists {   @_add_@
        ctx.JSON(   @_add_@
            http.StatusInternalServerError,   @_add_@
            gin.H{   @_add_@
                "status":  "Internal Server Error",   @_add_@
                "message": "db not found",   @_add_@
                "error":   "db not found",   @_add_@
            })   @_add_@
        return   @_add_@
    }   @_add_@
@_add_@
    db, ok := value.(db_service.DbService[Ambulance])   @_add_@
    if !ok {   @_add_@
        ctx.JSON(   @_add_@
            http.StatusInternalServerError,   @_add_@
            gin.H{   @_add_@
                "status":  "Internal Server Error",   @_add_@
                "message": "db context is not of required type",   @_add_@
                "error":   "cannot cast db context to db_service.DbService",   @_add_@
            })   @_add_@
        return   @_add_@
    }   @_add_@
@_add_@
    ambulance := Ambulance{}   @_add_@
    err := ctx.BindJSON(&ambulance)   @_add_@
    if err != nil {   @_add_@
        ctx.JSON(   @_add_@
            http.StatusBadRequest,   @_add_@
            gin.H{   @_add_@
                "status":  "Bad Request",   @_add_@
                "message": "Invalid request body",   @_add_@
                "error":   err.Error(),   @_add_@
            })   @_add_@
        return   @_add_@
    }   @_add_@
@_add_@
    if ambulance.Id == "" {   @_add_@
        ambulance.Id = uuid.New().String()   @_add_@
    }   @_add_@
@_add_@
    err = db.CreateDocument(ctx, ambulance.Id, &ambulance)   @_add_@
@_add_@
    switch err {   @_add_@
    case nil:   @_add_@
        ctx.JSON(   @_add_@
            http.StatusCreated,   @_add_@
            ambulance,   @_add_@
        )   @_add_@
    case db_service.ErrConflict:   @_add_@
        ctx.JSON(   @_add_@
            http.StatusConflict,   @_add_@
            gin.H{   @_add_@
                "status":  "Conflict",   @_add_@
                "message": "Ambulance already exists",   @_add_@
                "error":   err.Error(),   @_add_@
            },   @_add_@
        )   @_add_@
    default:   @_add_@
        ctx.JSON(   @_add_@
            http.StatusBadGateway,   @_add_@
            gin.H{   @_add_@
                "status":  "Bad Gateway",   @_add_@
                "message": "Failed to create ambulance in database",   @_add_@
                "error":   err.Error(),   @_add_@
            },   @_add_@
        )   @_add_@
    }   @_add_@
}
```

In this method, we first obtain an instance of `DbService` from the context, which we previously stored there. Subsequently, we retrieve the `Ambulance` object from the request body and attempt to save it to the database. Notice that a significant portion of the code is dedicated to handling possible errors and signaling the specific error in the response to the request.

>info:> Our openapi specification does not include all values for status errors and the return object used here. Independently update the specification to include all possible errors and return objects.

Furthermore, in the same file, modify the `DeleteAmbulance` method:

```go
...
func (this *implAmbulancesAPI) DeleteAmbulance(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // get db service from context
    value, exists := ctx.Get("db_service")    @_add_@
    if !exists {    @_add_@
        ctx.JSON(    @_add_@
            http.StatusInternalServerError,    @_add_@
            gin.H{    @_add_@
                "status":  "Internal Server Error",    @_add_@
                "message": "db_service not found",    @_add_@
                "error":   "db_service not found",    @_add_@
            })    @_add_@
        return    @_add_@
    }    @_add_@
    @_add_@
    db, ok := value.(db_service.DbService[Ambulance])    @_add_@
    if !ok {    @_add_@
        ctx.JSON(    @_add_@
            http.StatusInternalServerError,    @_add_@
            gin.H{    @_add_@
                "status":  "Internal Server Error",    @_add_@
                "message": "db_service context is not of type db_service.DbService",    @_add_@
                "error":   "cannot cast db_service context to db_service.DbService",    @_add_@
            })    @_add_@
        return    @_add_@
    }    @_add_@
    @_add_@
    ambulanceId := ctx.Param("ambulanceId")    @_add_@
    err := db.DeleteDocument(ctx, ambulanceId)    @_add_@
    @_add_@
    switch err {    @_add_@
    case nil:    @_add_@
        ctx.AbortWithStatus(http.StatusNoContent)    @_add_@
    case db_service.ErrNotFound:    @_add_@
        ctx.JSON(    @_add_@
            http.StatusNotFound,    @_add_@
            gin.H{    @_add_@
                "status":  "Not Found",    @_add_@
                "message": "Ambulance not found",    @_add_@
                "error":   err.Error(),    @_add_@
            },    @_add_@
        )    @_add_@
    default:    @_add_@
        ctx.JSON(    @_add_@
            http.StatusBadGateway,    @_add_@
            gin.H{    @_add_@
                "status":  "Bad Gateway",    @_add_@
                "message": "Failed to delete ambulance from database",    @_add_@
                "error":   err.Error(),    @_add_@
            })    @_add_@
    }    @_add_@
```

9. Save the changes. In the directory `${WAC_ROOT}/ambulance-webapi`, execute the following commands:

```ps
go mod tidy
./scripts/run.ps1 start
```

Open a new PowerShell command prompt and execute the following commands:

```ps
$Body = @{
    id = "bobulova"
    name = "Dr.Bobulová"
    roomNumber = "123"
    predefinedConditions = @(
        @{ value = "Nádcha"; code = "rhinitis" },
        @{ value = "Kontrola"; code = "checkup" }
    )
}

Invoke-RestMethod -Method Post -Uri http://localhost:8080/api/ambulance -Body ($Body | ConvertTo-Json) -ContentType "application/json"
```

The result should be displayed in this format:

```text
id       name         roomNumber predefinedConditions
--       ----         ---------- --------------------
bobulova Dr.Bobulov?? 123        {@{value=N??dcha; code=rhinitis}, @{value=Kontrola; code=checkup}}
```

>$linux:> V prípade linuxu/mac použite príkaz `curl`:
>
> ```sh
> curl -X POST -H "Content-Type: application/json" -d '{"id":"bobulova","name":"Dr.Bobulová","roomNumber":"123","predefinedConditions":>    [{"value":"Nádcha","code":"rhinitis"},{"value":"Kontrola","code":"checkup"}]}' http://localhost:8080/ambulance
> ```

By doing this, we have verified the functionality of the code implemented so far and also created a new ambulance. In the browser, open the URL `http://localhost:8081/db/<pfx>-ambulance-wl/ambulance` and click on the first entry in the document list.

10. For the remaining service methods, their functionality will be similar in many aspects. First, we need to retrieve the document for the given ambulance, modify it, and then save the modified document to the database. Meanwhile, we must handle possible error states. To reduce code duplication, we will use a helper function. Create a file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/utils_ambulance_updater.go` with the following content:

```go
package ambulance_wl

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/<github-id>/ambulance-webapi/internal/db_service"
)

type ambulanceUpdater = func( @_important_@
    ctx *gin.Context,  @_important_@
    ambulance *Ambulance,  @_important_@
) (updatedAmbulance *Ambulance, responseContent interface{}, status int)  @_important_@

func updateAmbulanceFunc(ctx *gin.Context, updater ambulanceUpdater) {
    value, exists := ctx.Get("db_service")
    if !exists {
        ctx.JSON(
            http.StatusInternalServerError,
            gin.H{
                "status":  "Internal Server Error",
                "message": "db_service not found",
                "error":   "db_service not found",
            })
        return
    }

    db, ok := value.(db_service.DbService[Ambulance])
    if !ok {
        ctx.JSON(
            http.StatusInternalServerError,
            gin.H{
                "status":  "Internal Server Error",
                "message": "db_service context is not of type db_service.DbService",
                "error":   "cannot cast db_service context to db_service.DbService",
            })
        return
    }

    ambulanceId := ctx.Param("ambulanceId")

    ambulance, err := db.FindDocument(ctx, ambulanceId)

    switch err {
    case nil:
        // continue
    case db_service.ErrNotFound:
        ctx.JSON(
            http.StatusNotFound,
            gin.H{
                "status":  "Not Found",
                "message": "Ambulance not found",
                "error":   err.Error(),
            },
        )
        return
    default:
        ctx.JSON(
            http.StatusBadGateway,
            gin.H{
                "status":  "Bad Gateway",
                "message": "Failed to load ambulance from database",
                "error":   err.Error(),
            })
        return
    }

    if !ok {
        ctx.JSON(
            http.StatusInternalServerError,
            gin.H{
                "status":  "Internal Server Error",
                "message": "Failed to cast ambulance from database",
                "error":   "Failed to cast ambulance from database",
            })
        return
    }

    updatedAmbulance, responseObject, status := updater(ctx, ambulance)

    if updatedAmbulance != nil {
        err = db.UpdateDocument(ctx, ambulanceId, updatedAmbulance)
    } else {
        err = nil // redundant but for clarity
    }

    switch err {
    case nil:
        if responseObject != nil {
            ctx.JSON(status, responseObject)
        } else {
            ctx.AbortWithStatus(status)
        }
    case db_service.ErrNotFound:
        ctx.JSON(
            http.StatusNotFound,
            gin.H{
                "status":  "Not Found",
                "message": "Ambulance was deleted while processing the request",
                "error":   err.Error(),
            },
        )
    default:
        ctx.JSON(
            http.StatusBadGateway,
            gin.H{
                "status":  "Bad Gateway",
                "message": "Failed to update ambulance in database",
                "error":   err.Error(),
            })
    }

}
```

Notice that the `updateAmbulanceFunc` function accepts another function as an input argument, declared as `ambulanceUpdater`. Functions in the [Go] language are full-fledged types, and thus, they can be used as types.

11. Now, open the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_conditions.go` and make the necessary modifications:

```go
...

// GetConditions - Provides the list of conditions associated with ambulance
func (this *implAmbulanceConditionsAPI) GetConditions(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(    @_add_@
        ctx *gin.Context,    @_add_@
        ambulance *Ambulance,    @_add_@
    ) (updatedAmbulance *Ambulance, responseContent interface{}, status int) {    @_add_@
        result := ambulance.PredefinedConditions   @_add_@
        if result == nil {   @_add_@
            result = []Condition{}   @_add_@
        }   @_add_@
        return nil, result, http.StatusOK   @_add_@
    })    @_add_@
}
```

In this method, we used the `updateAmbulanceFunc` function, providing an anonymous function as its argument that returns a list of conditions assigned to the ambulance. Notice that in the case where the ambulance has no assigned conditions, we return an empty list.

12. The `implAmbulanceWaitingListAPI` class modifies the waiting list in the selected ambulance. Part of the application logic is to ensure a consistent waiting list, meaning that after each modification, we need to adjust the expected entry time into the ambulance, which should not be earlier than the given moment and should not overlap between two patients. We achieve this using the `reconcileWaitingList` method, which we implement in the `Ambulance` class. We place the method in a new file to avoid manually rewriting the code when using the [openapi-generator] tool in the future. Create a file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/ext_model_ambulance.go` and insert the following code:

```go
package ambulance_wl

import (
    "time"

    "slices"
)

func (this *Ambulance) reconcileWaitingList() {
    slices.SortFunc(this.WaitingList, func(left, right WaitingListEntry) int {
        if left.WaitingSince.Before(right.WaitingSince) {
            return -1
        } else if left.WaitingSince.After(right.WaitingSince) {
            return 1
        } else {
            return 0
        }
    })

    // we assume the first entry EstimatedStart is the correct one (computed before previous entry was deleted)
    // but cannot be before current time
    // for sake of simplicity we ignore concepts of opening hours here

    if this.WaitingList[0].EstimatedStart.Before(this.WaitingList[0].WaitingSince) {
        this.WaitingList[0].EstimatedStart = this.WaitingList[0].WaitingSince
    }

    if this.WaitingList[0].EstimatedStart.Before(time.Now()) {
        this.WaitingList[0].EstimatedStart = time.Now()
    }

    nextEntryStart :=
        this.WaitingList[0].EstimatedStart.
            Add(time.Duration(this.WaitingList[0].EstimatedDurationMinutes) * time.Minute)
    for _, entry := range this.WaitingList[1:] {
        if entry.EstimatedStart.Before(nextEntryStart) {
            entry.EstimatedStart = nextEntryStart
        }
        if entry.EstimatedStart.Before(entry.WaitingSince) {
            entry.EstimatedStart = entry.WaitingSince
        }

        nextEntryStart =
            entry.EstimatedStart.
                Add(time.Duration(entry.EstimatedDurationMinutes) * time.Minute)
    }
}

```

>info:> In newer releases of the Go language, the `slices` package is part of the standard distribution. If the import of the `slices` package is not recognized, you may be using an older version of the Go language. In that case, either upgrade to a newer version or replace the import of the `slices` package with `golang.org/x/exp/slices`. To obtain the `golang.org/x/exp/slices` library, execute the command `go mod tidy` on the command line in the `${WAC_ROOT}/ambulance-webapi` directory.

13. Now, open the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_waiting_list.go` and modify the `CreateWaitingList` method:

```go
package ambulance_wl

import (
    "net/http" 

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"  @_add_@
    "slices"  @_add_@
)

// CreateWaitingListEntry - Saves new entry into waiting list
func (this *implAmbulanceWaitingListAPI) CreateWaitingListEntry(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance,  interface{},  int){    @_add_@
        var entry WaitingListEntry    @_add_@
    @_add_@
        if err := c.ShouldBindJSON(&entry); err != nil {    @_add_@
            return nil, gin.H{    @_add_@
                "status": http.StatusBadRequest,    @_add_@
                "message": "Invalid request body",    @_add_@
                "error": err.Error(),    @_add_@
            }, http.StatusBadRequest    @_add_@
        }    @_add_@
    @_add_@
        if entry.PatientId == "" {    @_add_@
            return nil, gin.H{    @_add_@
                "status": http.StatusBadRequest,    @_add_@
                "message": "Patient ID is required",    @_add_@
            }, http.StatusBadRequest    @_add_@
        }    @_add_@
    @_add_@
        if entry.Id == "" || entry.Id == "@new" {    @_add_@
            entry.Id = uuid.NewString()    @_add_@
        }    @_add_@
    @_add_@
        conflictIndx := slices.IndexFunc( ambulance.WaitingList, func(waiting WaitingListEntry) bool {    @_add_@
            return entry.Id == waiting.Id || entry.PatientId == waiting.PatientId     @_add_@
        })    @_add_@
    @_add_@
        if conflictIndx >= 0 {    @_add_@
            return nil, gin.H{    @_add_@
                "status": http.StatusConflict,    @_add_@
                "message": "Entry already exists",    @_add_@
            }, http.StatusConflict    @_add_@
        }    @_add_@
    @_add_@
        ambulance.WaitingList = append(ambulance.WaitingList, entry)    @_add_@
        ambulance.reconcileWaitingList()    @_add_@
        // entry was copied by value return reconciled value from the list    @_add_@
        entryIndx := slices.IndexFunc( ambulance.WaitingList, func(waiting WaitingListEntry) bool {    @_add_@
            return entry.Id == waiting.Id    @_add_@
        })    @_add_@
        if entryIndx < 0 {    @_add_@
            return nil, gin.H{    @_add_@
                "status": http.StatusInternalServerError,    @_add_@
                "message": "Failed to save entry",    @_add_@
            }, http.StatusInternalServerError    @_add_@
        }    @_add_@
        return ambulance, ambulance.WaitingList[entryIndx], http.StatusOK    @_add_@
    })    @_add_@
}
```

In this function, we proceed similarly to the `GetConditions` function. However, we need to handle situations where the request is not permissible, for example, rejecting a request to register an already waiting patient or a request with a duplicate identifier. In the case of successfully creating a record in the waiting list, we call the `reconcileWaitingList` method on the `Ambulance` object, which ensures the consistency of the timestamp of the waiting list.

Furthermore, in the same file, modify the `DeleteWaitingListEntry` method:

```go
...
// DeleteWaitingListEntry - Deletes specific entry
func (this *implAmbulanceWaitingListAPI) DeleteWaitingListEntry(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance, interface{}, int) {    @_add_@
        entryId := ctx.Param("entryId")    @_add_@
    @_add_@
        if entryId == "" {    @_add_@
            return nil, gin.H{    @_add_@
                "status":  http.StatusBadRequest,    @_add_@
                "message": "Entry ID is required",    @_add_@
            }, http.StatusBadRequest    @_add_@
        }    @_add_@
    @_add_@
        entryIndx := slices.IndexFunc(ambulance.WaitingList, func(waiting WaitingListEntry) bool {    @_add_@
            return entryId == waiting.Id    @_add_@
        })    @_add_@
    @_add_@
        if entryIndx < 0 {    @_add_@
            return nil, gin.H{    @_add_@
                "status":  http.StatusNotFound,    @_add_@
                "message": "Entry not found",    @_add_@
            }, http.StatusNotFound    @_add_@
        }    @_add_@
    @_add_@
        ambulance.WaitingList = append(ambulance.WaitingList[:entryIndx], ambulance.WaitingList[entryIndx+1:]...)    @_add_@
        ambulance.reconcileWaitingList()    @_add_@
        return ambulance, nil, http.StatusNoContent    @_add_@
    })    @_add_@
}
```

In this case, we are primarily concerned with whether the given record exists. If it does, we remove it from the list and call the `reconcileWaitingList` method on the `Ambulance` object.

The methods `GetWaitingList` and `GetWaitingListEntry` are relatively straightforward - they return the current waiting list or the requested entry if it exists:

```go
// GetWaitingListEntries - Provides the ambulance waiting list
func (this *implAmbulanceWaitingListAPI) GetWaitingListEntries(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance, interface{}, int) {  @_add_@
        result := ambulance.WaitingList     @_add_@
        if result == nil {     @_add_@
            result = []WaitingListEntry{}     @_add_@
        }     @_add_@
        // return nil ambulance - no need to update it in db   @_add_@
        return nil, result, http.StatusOK     @_add_@
    })   @_add_@
}

// GetWaitingListEntry - Provides details about waiting list entry
func (this *implAmbulanceWaitingListAPI) GetWaitingListEntry(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance, interface{}, int) {   @_add_@
        entryId := ctx.Param("entryId")   @_add_@
        @_add_@
        if entryId == "" {   @_add_@
            return nil, gin.H{   @_add_@
                "status":  http.StatusBadRequest,   @_add_@
                "message": "Entry ID is required",   @_add_@
            }, http.StatusBadRequest   @_add_@
        }   @_add_@
            @_add_@
        entryIndx := slices.IndexFunc(ambulance.WaitingList, func(waiting WaitingListEntry) bool {   @_add_@
            return entryId == waiting.Id   @_add_@
        })   @_add_@
        @_add_@
        if entryIndx < 0 {   @_add_@
            return nil, gin.H{   @_add_@
                "status":  http.StatusNotFound,   @_add_@
                "message": "Entry not found",   @_add_@
            }, http.StatusNotFound   @_add_@
        }   @_add_@
        @_add_@
        // return nil ambulance - no need to update it in db   @_add_@
        return nil, ambulance.WaitingList[entryIndx], http.StatusOK   @_add_@
    })   @_add_@
}
```

Finally, modify the implementation of the `UpdateWaitingListEntry` method:

```go
// UpdateWaitingListEntry - Updates specific entry
func (this *implAmbulanceWaitingListAPI) UpdateWaitingListEntry(ctx *gin.Context) {
    ctx.AbortWithStatus(http.StatusNotImplemented) @_remove_@
    // update ambulance document
    updateAmbulanceFunc(ctx, func(c *gin.Context, ambulance *Ambulance) (*Ambulance, interface{}, int) {        @_add_@
        var entry WaitingListEntry      @_add_@
        @_add_@
        if err := c.ShouldBindJSON(&entry); err != nil {        @_add_@
            return nil, gin.H{      @_add_@
                "status":  http.StatusBadRequest,       @_add_@
                "message": "Invalid request body",      @_add_@
                "error":   err.Error(),     @_add_@
            }, http.StatusBadRequest        @_add_@
        }       @_add_@
        @_add_@
        entryId := ctx.Param("entryId")     @_add_@
        @_add_@
        if entryId == "" {      @_add_@
            return nil, gin.H{      @_add_@
                "status":  http.StatusBadRequest,       @_add_@
                "message": "Entry ID is required",      @_add_@
            }, http.StatusBadRequest        @_add_@
        }       @_add_@
        @_add_@
        entryIndx := slices.IndexFunc(ambulance.WaitingList, func(waiting WaitingListEntry) bool {      @_add_@
            return entryId == waiting.Id        @_add_@
        })      @_add_@
        @_add_@
        if entryIndx < 0 {      @_add_@
            return nil, gin.H{      @_add_@
                "status":  http.StatusNotFound,     @_add_@
                "message": "Entry not found",       @_add_@
            }, http.StatusNotFound      @_add_@
        }       @_add_@
        @_add_@
        if entry.PatientId != "" {      @_add_@
            ambulance.WaitingList[entryIndx].PatientId = entry.PatientId        @_add_@
        }       @_add_@
        @_add_@
        if entry.Id != "" {     @_add_@
            ambulance.WaitingList[entryIndx].Id = entry.Id      @_add_@
        }       @_add_@
                @_add_@
        if entry.WaitingSince.After(time.Time{}) {      @_add_@
            ambulance.WaitingList[entryIndx].WaitingSince = entry.WaitingSince      @_add_@
        }       @_add_@
        @_add_@
        if entry.EstimatedDurationMinutes > 0 {     @_add_@
            ambulance.WaitingList[entryIndx].EstimatedDurationMinutes = entry.EstimatedDurationMinutes      @_add_@
        }       @_add_@
        @_add_@
        ambulance.reconcileWaitingList()        @_add_@
        return ambulance, ambulance.WaitingList[entryIndx], http.StatusOK       @_add_@
    })      @_add_@
}
```

Notice that, similar to the other methods, we aim to ensure the validity of the `PatientId` and `Id` properties of the `WaitingListEntry` object. However, from our schema, it is evident that these properties are mandatory, making this step seem redundant. An alternative solution could be to use a library for validating objects according to the [JSONSchema] specifications used in the OpenAPI specification, ideally already in the code generated by the [openapi-generator] tool. An example of such a library is [github.com/santhosh-tekuri/jsonschema/v5](https://github.com/santhosh-tekuri/jsonschema). Regardless, pay sufficient attention to data consistency in the database.

14. Save the changes and commit them to the Git repository. In the `${WAC_ROOT}/ambulance-webapi` directory, execute the following commands:


```ps
git add .
git commit -m "ambulance waiting list"
git push
```

Now, we have our WEB API implemented, and we can use it. In the next step, we will prepare a specification for its continuous integration, and then we will deploy it to a Kubernetes cluster.

>home_work:> In this exercise, we implemented the methods necessary for the functionality of our web component. Independently, complete the specification and implementation for editing and retrieving an ambulance and for working with the `predefinedConditions` list.
