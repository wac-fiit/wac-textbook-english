# Testing in Go

---

>info:>
Template for the pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-api-050`

---

Testing is an integral part of programming with equal importance as the source code itself.

In the [Go](https://go.dev/doc/tutorial/add-a-test) language, there is an integrated simple testing framework consisting of the [go test](https://pkg.go.dev/cmd/go#hdr-Test_packages) command and the [testing](https://pkg.go.dev/testing) package.
The `go test` command runs all functions that meet the following conditions:

* The function has the signature `func (t *testing.T)`.
* The function name starts with `Test` and continues with any text starting with a capital letter, for example, `TestExample`.
* The file name ends with the `_test.go` extension, for example, `api-admins_test.go`.

>info:> In addition to functions starting with `Test`, the `go test` command also runs [other types of functions](https://pkg.go.dev/cmd/go#hdr-Testing_functions) such as `benchmark` and `example` functions.

Similar to the development of the web component, here we will show an example of creating a unit test. We will use the [testify](https://pkg.go.dev/github.com/stretchr/testify#section-directories) library, which provides extensions to the `testing` package, especially with regard to evaluating test results and creating mock instances. We will also try to demonstrate the recommended process for creating tests, suitable for the [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) method.

1. Create a file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_waiting_list_test.go` with the following content:

```go
package ambulance_wl

import (
    "testing"

    "github.com/stretchr/testify/suite"
)

type AmbulanceWlSuite struct {
    suite.Suite
}

func TestAmbulanceWlSuite(t *testing.T) {
    suite.Run(t, new(AmbulanceWlSuite))
}

func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
    // ARRANGE

    // ACT

    // ASSERT
}
```

We have created the basic structure of our test and testing setup. The file `impl_ambulance_waiting_list_test.go` has the `_test.go` suffix, which allows the GO language tools to recognize it as a file containing test functions. The function `TestAmbulanceWlSuite(t *testing.T)` adheres to the rules for test functions mentioned above. The function `Test_UpdateWl_DbServiceUpdateCalled()` is our test function, which again meets the internal requirements of the _testify_ library.

We have divided the test function into sections `// ARRANGE`, `// ACT`, and `// ASSERT`. This approach is recommended because it improves the readability of the test function and also allows for easier test creation. In the `// ARRANGE` section, we create all the necessary objects and set them to the desired state. In the `// ACT` section, we perform the _action_ or set of actions whose functionality we are trying to verify.

>info:> If you encounter errors in the editor indicating missing libraries in the following steps, you can add them to the dependencies and install them with the `go mod tidy` command. We will also execute this command in step 7.

2. We start in the `// ASSERT` section, where we try to determine how we will verify whether the result or side effects correspond to our expectations. In this case, we want to verify that after the request to update the waiting list, the `UpdateDocument` function of the `DbService` class is called (where instead of the real instance of `DbService`, we will use its `mock`). Add the following code to the `// ARRANGE` section:

```go
...
func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
    // ARRANGE

    // ACT

    // ASSERT
    suite.dbServiceMock.AssertCalled(suite.T(), "UpdateDocument", mock.Anything, "test-ambulance", mock.Anything)  @_add_@
}
```

>info:> We will clarify the meaning of the function call later. At the beginning of the test creation, feel free to improvise, which is even desirable in the [TDD] technique because questions like _How will I verify the functionality?_, _How will I invoke the required functionality?_, or _How will I prepare the environment for the execution of the required functionality?_ help create a understandable, meaningful, and, most importantly, testable structure of the resulting code.

3. Next, continue with the `// ACT` section, where we specify how the desired functionality should be invoked - in our case, it will be a call to the `UpdateWaitingListEntry` function of our `implAmbulanceWaitingListAPI` class. In this section, we always use functions whose invocation is expected from external entities - entities outside our tested unit. Typically, these are functions labeled as application interfaces, public methods, and the like. Create the following code in the `// ACT` section:

```go
func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
    // ARRANGE

    // ACT
    sut.UpdateWaitingListEntry(ctx) @_add_@

    // ASSERT
    suite.dbServiceMock.AssertCalled(suite.T(), "UpdateDocument", mock.Anything, "test-ambulance", mock.Anything)
}
```

4. Finally, prepare the conditions for testing. In the `// ARRANGE` section, create an instance of the `implAmbulanceWaitingListAPI` class. Create an instance of the `gin.Context` class and modify its properties to match a real call to the `UpdateWaitingListEntry` method. In the code, we will use placeholder objects provided by the [gin] library, as well as placeholder objects from the [httptest](https://pkg.go.dev/net/http/httptest) library. In the `// ARRANGE` section, create the following code:

```go
func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
    // ARRANGE
    json := `{ @_add_@
        "id": "test-entry", @_add_@
        "patientId": "test-patient", @_add_@
        "estimatedDurationMinutes": 42 @_add_@
    }`    @_add_@
    @_add_@
    gin.SetMode(gin.TestMode) @_add_@
    recorder := httptest.NewRecorder() @_add_@
    ctx, _ := gin.CreateTestContext(recorder) @_add_@
    ctx.Set("db_service", suite.dbServiceMock) @_add_@
    ctx.Params = []gin.Param{ @_add_@
        {Key: "ambulanceId", Value: "test-ambulance"}, @_add_@
        {Key: "entryId", Value: "test-entry"}, @_add_@
    } @_add_@
    ctx.Request = httptest.NewRequest("POST", "/ambulance/test-ambulance/waitinglist/test-entry", strings.NewReader(json)) @_add_@

    sut := implAmbulanceWaitingListAPI{} @_add_@

    // ACT
    sut.UpdateWaitingListEntry(ctx)

    // ASSERT
    suite.dbServiceMock.AssertCalled(suite.T(), "UpdateDocument", mock.Anything, "test-ambulance", mock.Anything)
}
```

5. In the `\\ ASSERT` section, we used the `suite.dbServiceMock` object. Here, we will demonstrate how to create a substitute object - a _mock_ - and simplify test writing by placing common initialization in the `SetupTest` method. In the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_waiting_list_test.go`, declare a new structure `DbServiceMock` and implement the methods of the `db_service.DbService` interface on it. The `DbServiceMock` structure will be derived from the type - _contains_ - [mock.Mock](https://pkg.go.dev/github.com/stretchr/testify@v1.8.4/mock#Mock).


```go
...
type AmbulanceWlSuite struct {
    suite.Suite
    dbServiceMock *DbServiceMock[Ambulance] @_add_@
}

func TestAmbulanceWlSuite(t *testing.T) {
    suite.Run(t, new(AmbulanceWlSuite))
}

type DbServiceMock[DocType interface{}] struct {    @_add_@
    mock.Mock              @_add_@
}              @_add_@
            @_add_@
func (this *DbServiceMock[DocType]) CreateDocument(ctx context.Context, id string, document *DocType) error {              @_add_@
    args := this.Called(ctx, id, document)             @_add_@
    return args.Error(0)               @_add_@
}              @_add_@
            @_add_@
func (this *DbServiceMock[DocType]) FindDocument(ctx context.Context, id string) (*DocType, error) {               @_add_@
    args := this.Called(ctx, id)               @_add_@
    return args.Get(0).(*DocType), args.Error(1)               @_add_@
}              @_add_@
            @_add_@
func (this *DbServiceMock[DocType]) UpdateDocument(ctx context.Context, id string, document *DocType) error {              @_add_@
    args := this.Called(ctx, id, document)             @_add_@
    return args.Error(0)               @_add_@
}              @_add_@
            @_add_@
func (this *DbServiceMock[DocType]) DeleteDocument(ctx context.Context, id string) error {             @_add_@
    args := this.Called(ctx, id)               @_add_@
    return args.Error(0)               @_add_@
}              @_add_@
            @_add_@
func (this *DbServiceMock[DocType]) Disconnect(ctx context.Context) error {            @_add_@
    args := this.Called(ctx)               @_add_@
    return args.Error(0)               @_add_@
}              @_add_@

func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
...
```

Implemented methods simply invoke the `Called` function, whose implementation is provided by the embedded object `mock.Mock`. The actual result is of type `mock.Arguments` and is determined using the `mock.Mock.On` method, which we will use in the next step.

6. Most of our tests will use the same object `dbServiceMock`, so we will create it in the `SetupTest` method. Additionally, we will prepare a response for the `FindDocument` method, where we will provide a simple instance of type `Ambulance`. In the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_waiting_list_test.go`, add the following code:

```go
...
func (this *DbServiceMock[DocType]) Disconnect(ctx context.Context) error {
    ...
}

func (suite *AmbulanceWlSuite) SetupTest() {    @_add_@
    suite.dbServiceMock = &DbServiceMock[Ambulance]{}                @_add_@
                @_add_@
    // Compile time Assert that the mock is of type db_service.DbService[Ambulance]              @_add_@
    var _ db_service.DbService[Ambulance] = suite.dbServiceMock              @_add_@
                @_add_@
    suite.dbServiceMock.                 @_add_@
        On("FindDocument", mock.Anything, mock.Anything).                @_add_@
        Return(              @_add_@
            &Ambulance{              @_add_@
                Id: "test-ambulance",                @_add_@
                WaitingList: []WaitingListEntry{                 @_add_@
                    {                @_add_@
                        Id:                       "test-entry",              @_add_@
                        PatientId:                "test-patient",                @_add_@
                        WaitingSince:             time.Now(),                @_add_@
                        EstimatedDurationMinutes: 101,               @_add_@
                    },               @_add_@
                },               @_add_@
            },               @_add_@
            nil,                 @_add_@
        )                @_add_@
}                @_add_@

func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
...
```

Similarly to how we prepared the invocation of the `FindDocument` method, we will also prepare the invocation of the `UpdateDocument` method. This time, we will perform this preparation in the `// ARRANGE` section of our test:

```go
func (suite *AmbulanceWlSuite) Test_UpdateWl_DbServiceUpdateCalled() {
    // ARRANGE
    suite.dbServiceMock.  @_add_@
        On("UpdateDocument", mock.Anything, mock.Anything, mock.Anything).  @_add_@
        Return(nil)  @_add_@

    json := `{
    ...
```

It is up to us to decide which methods to prepare in the `SetupTest` method and which ones in the `// ARRANGE` section. In general, it is advisable to prepare methods that will be used in most tests in the `SetupTest` method, while other methods can be prepared in the `// ARRANGE` section. If a particular test requires different behavior for a specific method, we can invoke the `Unset` function on the object returned when calling the `On` method.
  
7. In the file `${WAC_ROOT}/ambulance-webapi/internal/ambulance_wl/impl_ambulance_waiting_list_test.go`, modify references to external libraries:

```go
package ambulance_wl

import (
    "context"   @_add_@
    "net/http/httptest"   @_add_@
    "strings"   @_add_@
    "testing"
    "time"    @_add_@
    
    "github.com/gin-gonic/gin"    @_add_@
    "github.com/stretchr/testify/mock"    @_add_@
    "github.com/<github-id>/ambulance-webapi/internal/db_service"    @_add_@
    "github.com/stretchr/testify/suite"
)
```

In the directory `${WAC_ROOT}/ambulance-webapi`, execute the command:

```ps
go mod tidy
```

8. Verify if the test execution completes successfully: In the directory `${WAC_ROOT}/ambulance-webapi`, execute the command:

```ps
go test ./...
```

You should see results similar to this:

```text
?       github.com/milung/ambulance-webapi/api  [no test files]
?       github.com/milung/ambulance-webapi/cmd/ambulance-api-service    [no test files]
?       github.com/milung/ambulance-webapi/internal/db_service  [no test files]
ok      github.com/milung/ambulance-webapi/internal/ambulance_wl        0.031s @_important_@
```

>info:> If you have the [golang.go](https://marketplace.visualstudio.com/items?itemName=golang.Go) extension installed in Visual Studio Code, you can execute or debug tests directly from the VS Code environment.

If we strictly follow the [TDD] method, at this step, our `UpdateWaitingListEntry` method would not be implemented yet - it would return the error code `501 - Not Implemented`. We would gradually add functionality only based on new requirements, creating necessary tests for it. Additionally, we might refactor the source code while maintaining the original required functionality - already verified by existing tests. The tests would reflect both the required and implemented functionality. A common shortcoming of poor test suites is that they try to describe and verify how the code is implemented but do not consider whether the implemented code is actually what is required and what functionality it is based on.

9. Modify the file `${WAC_ROOT}/ambulance-webapi/scripts/run.ps1` - add the command for running tests:

```ps
...
switch ($command) {
    ...
    "start" {
        ...
    }
    "test" {     @_add_@
        go test -v ./...     @_add_@
    }     @_add_@
    "mongo" {
    ...
```

10. Save the changes and commit them to the Git repository. In the directory `${WAC_ROOT}/ambulance-webapi`, execute the following commands:

```ps
git add .
git commit -m "ambulance waiting list first test"
git push
```

>homework:> Independently, add at least one test for the `ambulance-webapi` service.
