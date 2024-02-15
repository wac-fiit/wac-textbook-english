# Semi-automatic Deployment (Automatic Update of Docker Image from Docker Hub)

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-042`

---

In the previous section, we manually deployed a containerized web application to
Azure cloud. Now, let's demonstrate a simple way to set up automatic
updates for the application when its Docker image changes on Docker Hub.

1. Go back to the [Azure portal][azure-portal], to your web application, and in the _Deployment Center_ tab, switch the _Continuous deployment_ option to `On`, and copy the value from the _Webhook URL_ field. Save the setting by pressing the _Save_ button:

![Deployment Center for Azure Web Application](./img/050-01-azurewebapp-cd.png)

Go to the [Docker Hub][docker-hub], open the details of your image
`ambulance-ufe`, and navigate to the _Webhooks_ tab. Create a new webhook, name it
_Azure WebApp_, and set the URL value to the one copied from the Azure portal.

![Creating a webhook entry on DockerHub](./img/050-02-dockerhub-webhook.png)

With these settings, whenever a new image with the tag `<your-dockerhub-account>/ambulance-ufe:latest` is pushed to the Docker Hub registry, the web service created on
the Microsoft Azure platform will automatically get the latest container image with that
tag.

2. We will add a step to the _CI pipeline_ to publish a new version of the Docker image after a successful
   build. This time, we will manually modify the continuous integration script.

In the [Containerization of the Application](./041-ufe-containerization.md) chapter, we created an image using the `docker build` command. This command created an image for the current platform of our environment - `linux/amd64` in a Windows or Linux environment on Intel/AMD processors, or `linux/arm64/v8` on newer Apple Mac models. To make our generated image usable on various platforms, we need to create a [multi-platform build](https://docs.docker.com/build/building/multi-platform/), which will create different images and register them in the registry under a common name (a manifest with multiple references). Locally, we would use the command `docker buildx build --platform linux/amd64,linux/arm64/v8 --push -t <your-dockerhub-account>/ambulance-ufe .`. Details about this process can be found [here](https://docs.docker.com/build/building/multi-platform/). In continuous integration, we will create such multi-platform images.

Open the file `${WAC_ROOT}/ambulance-ufe/.github/workflows/ci.yml`, and add new steps to the script at the end:


```yaml
...
    - run: npm test

    - name: Set up QEMU @_add_@
      uses: docker/setup-qemu-action@v1  @_add_@
      @_add_@
    - name: Set up Docker Buildx @_add_@
      id: buildx @_add_@
      uses: docker/setup-buildx-action@v1   @_add_@
```

These steps add support for multi-platform builds to the pipeline. In the next step, we will add a command to log in to the Docker Hub registry:

```yaml
...
- name: Login to DockerHub
  uses: docker/login-action@v1 
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

We will set the variables `secrets.DOCKERHUB_USERNAME` and `secrets.DOCKERHUB_TOKEN` later in this chapter. The next step will create [metadata](https://github.com/docker/metadata-action) for image creation, specifically the `tags` information which is important for us:


```yaml
...
- name: Docker meta
  id: meta
  uses: docker/metadata-action@v3
  with:
    images: |
      <your-dockerhub-account>/ambulance-ufe @_important_@
    tags: |
      type=schedule
      type=ref,event=branch
      type=ref,event=branch,suffix={{date '.YYYYMMDD.HHmm'}} # napr `main.20210930.1200` @_important_@
      type=ref,event=tag
      type=semver,pattern={{version}} # napr pri tagu  `v1.0.0`
      type=semver,pattern={{major}}.{{minor}} # napr `1.0`
      type=semver,pattern={{major}}
      type=raw,value=latest,enable={{is_default_branch}} # `latest` pre každý komit do main vetvy @_important_@
```

>info:> In this exercise, we always create an image with the tag `<your-dockerhub-account>/ambulance-ufe:latest` for the sake of simplifying the subsequent steps. In real-world projects, the `latest` tag is typically created only for official, tested releases of a new version - for example, by adding a tag in the format `v1.0.1`.

Next, we will add a step to create a multi-platform image:


```yaml
...

- uses: docker/build-push-action@v2
  with:
    context: .
    platforms: linux/amd64,linux/arm64/v8
    file: ./build/docker/Dockerfile
    push: true
    tags: ${{ steps.meta.outputs.tags }} @_important_@
    labels: ${{ steps.meta.outputs.labels }}
```

3. To ensure the successful execution of the continuous integration, you need to set the variables `secrets.DOCKERHUB_USERNAME` and `secrets.DOCKERHUB_TOKEN`. Go to the [Docker Hub] page, expand the menu labeled with your account name, and select _Account Settings_. In the _Security_ tab, you will find the _New Access Token_ button.

![Creating a new token for Docker Hub](./img/050-01-AccountSecurity.png)

Create a new token with the name `ambulance-ufe CI` and assign it the `Read, Write, Delete` permissions. Press the _Generate_ button and copy the generated token to your clipboard.

Now, go to your repository `<pfx>/ambulance-ufe` on the [GitHub] page. In the top menu, choose the _Settings_ tab and then on the side panel, select _Secrets and Variables_ -> _Actions_.
On this page, press the _New repository secret_ button and create a new variable named `DOCKERHUB_TOKEN`, pasting the copied token as its value. Again, press the _New repository secret_ button and create a variable named `DOCKERHUB_USERNAME`, with your Docker Hub username as its value.

![Variables and keys for continuous integration](./img/050-02-GithubSecrets.png)

The created variables are now available for the further execution of our continuous integration.

4. In the `${WAC_ROOT}/ambulance-ufe` directory, commit and push the source code changes using the following commands:

```ps
git add .
git commit -m "ci - publish docker image"
git push
```

5. On the [GitHub] page in your repository `<pfx>/ambulance-ufe`, navigate to the _Actions_ tab and check that the new run of continuous integration completes successfully. After its completion, you can also verify the status of the image on the Docker Hub page, where you can see new version tags and platforms for your image.

This way, we have ensured the continuous deployment of this web application to the [Azure][azure-portal] environment. Similarly, you could deploy other services or leverage existing services provided on the Azure platform. Detailed guides on how to proceed if you want to primarily create solutions on existing Azure services can be found, for example, [here](https://learn.microsoft.com/en-us/azure/architecture/).

Next, in this exercise, we will demonstrate how to create an application using the micro Front End technique and deploy it to the [Kubernetes] environment using the [GitOps](https://www.gitops.tech/) technique.
