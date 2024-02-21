# Containerization of the Application and Continuous Integration

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-api-060`

---

We have successfully created a functional web API. To deploy it, we need to create a software container and ensure continuous integration of the code.

## Creating an image of the software container for the web API service

One of the main advantages of a containerized application is its easy and consistent deployment across different environments. Similar to the web component, we will use a multi-stage image creation approach.

1. Create a file `${WAC_ROOT}/ambulance-webapi/build/docker/Dockerfile` and insert the following content:

```dockerfile
# use specific versions of images
FROM openapitools/openapi-generator-cli:v7.0.1 as api

WORKDIR /local

COPY api api
COPY scripts scripts
COPY .openapi-generator-ignore .openapi-generator-ignore

RUN docker-entrypoint.sh generate -c /local/scripts/generator-cfg.yaml

# not used normally but redefine entrypoint for the case of checking this stage results
ENTRYPOINT ["bash"]

############################################
```

In the first phase, we will use the image [openapitools/openapi-generator-cli](https://hub.docker.com/r/openapitools/openapi-generator-cli) to generate source files for the web API using our configuration and templates created in the previous chapters. The result is the `/local/internal/ambulance_wl` directory containing the source files for the web API.

2. Add instructions for the second phase of image creation to the file `${WAC_ROOT}/ambulance-webapi/build/docker/Dockerfile`:

```dockerfile
# use specific versions of images if you want
FROM openapitools/openapi-generator-cli:v7.0.1 as api
...
############################################
FROM golang:latest AS build    @_add_@
   @_add_@
WORKDIR /app   @_add_@
   @_add_@
# download dependencies - low frequency of changes   @_add_@
COPY go.mod go.sum ./   @_add_@
RUN go mod download   @_add_@
   @_add_@
# copy sources - higher frequency of changes   @_add_@
COPY internal/ internal/   @_add_@
COPY cmd/ cmd/   @_add_@
COPY --from=api /local/ ./   @_add_@
   @_add_@
# ensure tests are passing   @_add_@
RUN go test ./...   @_add_@
   @_add_@
# create executable - ambulance-webapi-srv   @_add_@
# we want to use scratch image so setting    @_add_@
# the build options in the way that will link all dependencies statically   @_add_@
RUN CGO_ENABLED=0 GOOS=linux \   @_add_@
      go build \    @_add_@
      -ldflags="-w -s" \   @_add_@
      -installsuffix 'static' \   @_add_@
      -o ./ambulance-webapi-srv ./cmd/ambulance-api-service   @_add_@
      @_add_@
############################################   @_add_@
```

In this section, we use the [golang:latest](https://hub.docker.com/_/golang) image to compile the source files of the web API. Note that we first used only the `go.mod` and `go.sum` files to fetch the libraries on which our code depends. These files tend to change infrequently, so there's a relatively high chance that when generating locally, we can take advantage of Docker's caching system instead of having to regenerate this layer. Next, we copied the source files, including the files generated in the previous phase, and executed tests. Only then did we compile the source files into an executable file. During compilation, we used settings to create an executable file that contains no dependencies on dynamic libraries, which would need to be installed in the target container. The result is the `ambulance-webapi-srv` file, which we will use in the third phase.

3. Add instructions for the third phase to the file `${WAC_ROOT}/ambulance-webapi/build/docker/Dockerfile`:

```dockerfile
...
############################################
FROM golang:latest AS build
...
############################################
FROM scratch        @_add_@
         @_add_@
# see https://github.com/opencontainers/image-spec/blob/main/annotations.md for details     @_add_@
LABEL org.opencontainers.image.authors="Your Name"      @_add_@
LABEL org.opencontainers.image.title="Ambulance Waiting List WebAPI Service"        @_add_@
LABEL org.opencontainers.image.description="WEBAPI for managing entries in ambulances` waiting list"        @_add_@
         @_add_@
# list all variables and their default values for clarity       @_add_@
ENV AMBULANCE_API_ENVIRONMENT=production        @_add_@
ENV AMBULANCE_API_PORT=8080	        @_add_@
ENV AMBULANCE_API_MONGODB_HOST=mongo        @_add_@
ENV AMBULANCE_API_MONGODB_PORT=27017        @_add_@
ENV AMBULANCE_API_MONGODB_DATABASE=pfx-ambulance        @_add_@
ENV AMBULANCE_API_MONGODB_COLLECTION=ambulance      @_add_@
ENV AMBULANCE_API_MONGODB_USERNAME=root     @_add_@
ENV AMBULANCE_API_MONGODB_PASSWORD=     @_add_@
ENV AMBULANCE_API_MONGODB_TIMEOUT_SECONDS=5     @_add_@
         @_add_@
COPY --from=build /app/ambulance-webapi-srv ./      @_add_@
         @_add_@
# Actual port may be changed during runtime     @_add_@
# Default using for the simple case scenario        @_add_@
EXPOSE 8080     @_add_@
ENTRYPOINT ["./ambulance-webapi-srv"]       @_add_@
```

The last stage is based on the [scratch](https://hub.docker.com/_/scratch) image, which is an image that contains no added layers. Since our container will only contain one executable file, it is an ideal candidate. In addition to setting environment variables that will be used when running the container and adding labels describing the image content, this stage includes only one layer created by the command `COPY --from=build /app/ambulance-webapi-srv ./`.

4. Create a file `${WAC_ROOT}/ambulance-webapi/.dockerignore` and insert the following content:

```text
internal/ambulance_wl/api_*    @_important_@
internal/ambulance_wl/model_*     @_important_@
internal/ambulance_wl/routers.go    @_important_@
build/
deployments/
```

This file defines which files and folders should be ignored when creating the container image. In our case, it includes files generated in the first phase of image creation.

5. Open the file `${WAC_ROOT}/ambulance-webapi/scripts/run.ps1` and add the command to create the container image:

```powershell
...
switch ($command) {
   ...
   "mongo" {
      mongo up
   }
   "docker" {    @_add_@
         docker build -t <docker-id>/ambulance-wl-webapi:local-build -f ${ProjectRoot}/build/docker/Dockerfile .    @_add_@
   }    @_add_@
   default {
   ...
```

Modify the text `<docker-id>` to include your user id on the [Docker Hub] page and save the changes. Start the Docker subsystem and, on the command line in the directory `${WAC_ROOT}/ambulance-webapi`, execute the command

```powershell
.\scripts\run.ps1 docker
```

After successfully completing the command, you will have a new software container image named `<docker-id>/ambulance-wl-webapi:local-build`. Verify this using the command:

```powershell
docker inspect <docker-id>/ambulance-wl-webapi:local-build
```

You can publish the image on the [Docker Hub] page using the command:

```powershell
docker login
docker push <docker-id>/ambulance-wl-webapi:local-build
```

6. Archive the changes with commands in the directory `${WAC_ROOT}/ambulance-webapi`

```powershell
git add .
git commit -m "Added dockerization"
git push
```

## Continuous Integration

Continuous Integration is a process that ensures automatic execution of tests and generation of resulting artifacts (library packages, software container images, etc.), and in some cases, even the automatic deployment of the application to the target environment. Similar to the web component, here we will use the [GitHub Actions](https://github.com/features/actions) service. The sufficient step would be to ensure the creation of a software container that already includes test execution. For illustrative purposes, however, we will create a workflow that generates the web API scaffold, runs tests, and only then creates a software container. This time, we will show how to create a workflow using the graphical editor on the [GitHub] page.

1. Go to the [GitHub] page and open the `ambulance-webapi` repository. In the top bar, select the `Actions` tab. On the _Get started with GitHub Actions_ page, choose the _Configure_ button in the _GO_ entry in the list of recommended workflows.

2. On the next page, the workflow editor for continuous integration will be displayed. At the top, change the name from `go.yaml` to `ci.yaml`. Modify the name and trigger of the workflow as follows:

```yaml
...
name: Go @_remove_@
# WEBAPI docker 
name: Test and Publish WebAPi Container Image @_add_@

on:
   push:
      branches: [ "main" ]
      tags: [ "v1*" ] @_add_@
   pull_request:
      branches: [ "main" ]
   workflow_dispatch:  {} # allow manually trigger workflow @_add_@
   ...      
```

3. For the automatic retrieval of the `go` language version from the `go.mod` file, we will use the following 2 steps:

```yaml
...
build:
   runs-on: ubuntu-latest
   steps:
   - name: Checkout repository
      uses: actions/checkout@v3

   - name: Get Go version from go.mod    @_add_@
      id: get_go_version    @_add_@
      run: echo "go_version=$(grep -m1 'go ' go.mod | awk '{print $2}')" >> $GITHUB_OUTPUT    @_add_@
      
   - name: Set up Go    @_add_@
      uses: actions/setup-go@v4    @_add_@
      with:    @_add_@
      go-version: ${{ steps.get_go_version.outputs.go_version }}    @_add_@
...      
```

4. Now, modify the `Build` step:

```yaml
...
jobs:
   ...
   - name: Build
      run: go build -v ./... @_remove_@
      # build api service 
      run: go build -v ./cmd/ambulance-api-serviceň @_add_@
   ...      
```

5. On the right side in the _Marketplace_ tab, enter the text _openapi_ in the search bar, and select the item _openapi-generator-generate-action_ from the list. In the subsequent display, copy the code from the _Installation_ section and paste it into your code before the `- name: Build` block. Modify the inserted code into the following form:

```yaml
...
      - name: Set up Go
      uses: actions/setup-go@v4
      with:
         go-version: '1.21'

      - name: Generate api controllers interfaces    @_add_@
      uses: craicoverflow/openapi-generator-generate-action@v1.2.1    @_add_@
      with:    @_add_@
         # version: 7.0.0-beta - at time of writing this text only prerelease was available    @_add_@
         generator: go-gin-server            @_add_@
         input: api/ambulance-wl.openapi.yaml    @_add_@
         additional-properties: apiPath=internal/ambulance_wl,packageName=ambulance_wl    @_add_@
         template: scripts/templates    @_add_@

      - name: Build
...

```

![OpenApi Generator Action](./img/060-01-GeneratorAction.png)

6. Go back to the search in the _Marketplace_ (click on the _Marketplace_ link in the text _Marketplace/Search results_) and search for actions using the word _docker_. Select the item _Docker Setup QUEMU_ from the list and copy the code of this action to the end of your workflow. Modify it into the following form:

```yaml
...
      - name: Test
      run: go test -v ./...

      - name: Docker Setup QEMU  @_add_@
      uses: docker/setup-qemu-action@v2.2.0  @_add_@
```

Go back to the search results (click on the link _Search results_), and using a similar process, copy the code for the actions _Docker Setup Buildx_, _Docker metadata action_, _Docker Login_, and finally _Build and push Docker Images_. Modify the resulting code into the following form:

```yaml
...
      - name: Docker Setup QEMU
      uses: docker/setup-qemu-action@v2.2.0 

      - name: Docker Setup Buildx     @_add_@
      uses: docker/setup-buildx-action@v2.9.1      @_add_@
      @_add_@
      - name: Docker Metadata action     @_add_@
      id: meta    @_add_@
      uses: docker/metadata-action@v4.6.0    @_add_@
      with:    @_add_@
         images: <docker-id>/ambulance-wl-webapi     @_add_@
         tags: |      @_add_@
            type=schedule     @_add_@
            type=ref,event=branch      @_add_@
            type=ref,event=branch,suffix={{date '.YYYYMMDD.HHmm'}}      @_add_@
            type=ref,event=tag      @_add_@
            type=semver,pattern={{version}}     @_add_@
            type=semver,pattern={{major}}.{{minor}}      @_add_@
            type=semver,pattern={{major}}          @_add_@
            type=raw,value=latest,enable={{is_default_branch}}    @_add_@
      @_add_@
      - name: Docker Login   @_add_@
      uses: docker/login-action@v2.2.0   @_add_@
      with:   @_add_@
         username: ${{ secrets.DOCKERHUB_USERNAME }}   @_add_@
         password: ${{ secrets.DOCKERHUB_TOKEN }}   @_add_@
         @_add_@
      - name: Build and push Docker images     @_add_@
      uses: docker/build-push-action@v4.1.1     @_add_@
      with:    @_add_@
         context: .      @_add_@
         file: ./build/docker/Dockerfile      @_add_@
         labels: ${{ steps.meta.outputs.labels }}            @_add_@
         platforms: linux/amd64,linux/arm64/v8      @_add_@
         push: true      @_add_@
         tags: ${{ steps.meta.outputs.tags }}       @_add_@
```

>info:> Most of the steps involving copying code from the _Marketplace_ tab could be skipped, and you could work directly with the file `${WAC_ROOT}/ambulance-webapi/.github/workflows/ci.yml`. However, the intention was to demonstrate how to work with the graphical workflow editor on the [GitHub] page, where, in addition to obtaining current workflow templates, you can also get an overview of reusable actions created for automating various tasks in the software development process.

7. Press the _Commit changes ..._ button at the top of the page, choose _Commit directly to main_, and again press the _Commit changes_ button. In the repository, go to the _Actions_ tab and check that the _Test and Publish WebAPi Container Image_ workflow has been triggered. At this moment, the run will be unsuccessful, specifically the step of publishing the image to the Docker Hub registry due to the unavailability of authorization rights.

8. For a successful continuous integration run, it is necessary to set the variables `secrets.DOCKERHUB_USERNAME` and `secrets.DOCKERHUB_TOKEN`. Go to the [Docker Hub] page, expand the menu labeled with your account name, and select _Account Settings_. In the _Security_ tab, you will find the _New Access Token_ button.

![Creating a new token for Docker Hub](./img/060-03-AccountSecurity.png)

Create a new token named `ambulance-webapi CI` and assign it the permissions `Read, Write, Delete`. Press the _Generate_ button. Copy the generated token to the clipboard.

Now, go to your repository `<github-id>/ambulance-webapi` on the [GitHub] page. In the top bar, select the _Settings_ tab, and then on the side panel, choose _Secrets and Variables_ -> _Actions_. On this page, press the _New repository secret_ button and create a new variable named `DOCKERHUB_TOKEN`, using the copied token from the clipboard as the value. Again, press the _New repository secret_ button and create a variable named `DOCKERHUB_USERNAME`, and as the value, enter your Docker Hub username.

![Variables and keys for continuous integration](./img/060-04-GithubSecrets.png)

The created variables are now available for the next run of our continuous integration.

9. In the repository, go to the _Code_ tab, press the link _0 tags_, and then click on the _Create new release_ button. Enter `v1.0.0` into the _Choose tag_ field, enter `v1.0.0` into the _Release Title_ field, and enter `Initial version of Ambulance Waiting List API` into the _Describe this release_ field. Press the _Publish release_ button.

![GitHub Actions](./img/060-02-NewRelease.png)

In the repository, go to the _Actions_ tab and check that the _Test and Publish WebAPi Container Image_ workflow has been triggered for the newly created tag. After its completion, you can verify on the [Docker Hub] page that a new software container image with the name `<docker-id>/ambulance-wl-webapi:v1.0.0` has been created in the repository.

10. Fetch the changes from your [GitHub] repository. In the `${WAC_ROOT}/ambulance-webapi` directory, execute the command:

```powershell
git pull
```
