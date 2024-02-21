# Web API Service Skeleton

---

>info:>
Template for a pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-api-010`

---

1. Go to the [GitHub] page and under your account, create a new repository named "ambulance-webapi." Leave the settings empty.

![Create Repository](img/001-01-CreateRepository.png)

After pressing the _Create repository_ button, a page with instructions on how to clone your created repository to your computer will appear. However, we are currently interested in the path to your repository, which will be used as the identifier for the [Golang module](https://go.dev/doc/modules/developing) of our project. In your browser, copy the address to your repository without the leading scheme `https://`, meaning the string should be in the form `github.com/<github-id>/ambulance-webapi`.

2. Create a directory `${WAC_ROOT}/ambulance-webapi` and execute the following command in this directory on the command line:

```ps
go mod init github.com/<pfx>/ambulance-webapi
```

The command will create a file `${WAC_ROOT}/ambulance-webapi/go.mod` with the content in the following format:

```go
module github.com/<pfx>/ambulance-webapi

go 1.21
```

3. In our project, we will follow the file organization structure described in the document [Standard Go Project Layout](https://github.com/golang-standards/project-layout/tree/master#readme). Our first step will be to provide users of our service access to the API specification. At the same time, this will create a basic functional skeleton for our service.

Create a directory `${WAC_ROOT}/ambulance-webapi/api` and copy the file `${WAC_ROOT}/ambulance-ufe/api/ambulance-wl.openapi.yaml` into it.

>info:> Later, we will remove the specification from the `ambulance-ufe` project, but for now, keep it there.

4. Create a file `${WAC_ROOT}/ambulance-webapi/cmd/ambulance-api-service/main.go` and insert the following content into it:


```go
package main

import (
    "log"
    "os"
    "strings"
    "github.com/gin-gonic/gin"
    "github.com/<github-id>/ambulance-webapi/api" @_important_@
)

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
    // request routings
    engine.GET("/openapi", api.HandleOpenApi) @_important_@
    engine.Run(":" + port)
}
```

The `main` function in the `main` package serves as the entry point for the program in the [Go] language. In our case, it creates an instance of an HTTP server using the [gin-go][gin] library, which will listen on the port defined in the environment variable `AMBULANCE_API_PORT`. Routing of requests to individual functions is handled through the `engine.GET` function. In our case, we registered the `api.HandleOpenApi` function, which will process requests to retrieve the API specification. Also, note that we reference the package `github.com/<github-id>/ambulance-webapi/api`, which is essentially a reference to the directory we created at `${WAC_ROOT}/ambulance-webapi/api`.

>info:> We recommend having the [golang.go](https://marketplace.visualstudio.com/items?itemName=golang.Go) extension installed in Visual Studio Code. We will address errors in the list of imported packages in the next step.

5. Create a file `${WAC_ROOT}/ambulance-webapi/api/openapi.go` with the following content:


```go
package api

import (
    _ "embed"
    "net/http"

    "github.com/gin-gonic/gin"
)

//go:embed ambulance-wl.openapi.yaml @_important_@
var openapiSpec []byte

func HandleOpenApi(ctx *gin.Context) {
    ctx.Data(http.StatusOK, "application/yaml", openapiSpec)
}
```

In this file, we added functionality for our package - `github.com/<github-id>/ambulance-webapi/api`. We utilized the functionality of the [embed](https://pkg.go.dev/embed) library, which ensures that the `ambulance-wl.openapi.yaml` file will be embedded with the binary file of our program.

6. In the previous files, we left error messages indicating that the specified package is not available. All dependencies of our module must be listed in the file `${WAC_ROOT}/ambulance-webapi/go.mod` and loaded into the local cache. To avoid manually adding individual dependencies, we will use the `go mod tidy` command, which will add them automatically. Save the changes and execute the following command in the command line in the `${WAC_ROOT}/ambulance-webapi` directory:


```ps
go mod tidy
```

Subsequently, execute the command that will compile our program and hand over control to its entry point - the `main` function in the `main` package:

```ps
go run ./cmd/ambulance-api-service
```

Open a new terminal and enter the command in the command line:

```ps
curl http://localhost:8080/openapi
```

The response will be the output of our [OpenAPI] specification.

7. Create a file `${WAC_ROOT}/ambulance-webapi/.gitignore` with the following content:

```text
*.exe
*.exe~
./ambulance-api-service
```

Initialize and archive the Git repository:

```sh
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<github-id>/ambulance-webapi.git
git push -u origin main
```
