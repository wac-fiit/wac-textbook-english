# Deployment to the Production Kubernetes Cluster

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-030
```

---

Similarly to the web client, the infrastructure for the web API was deployed centrally. This means that services like _MongoDb_, _ufe-container_, and _Envoy Gateway_ are running in the cluster.

A route rule was created for Mongo Express, and the application is accessible at [https://wac-24.westeurope.cloudapp.azure.com/ui/mongo-express/](https://wac-24.westeurope.cloudapp.azure.com/ui/mongo-express/).
Since this is a publicly accessible resource, access is protected by a username and password, which you can obtain from your tutor.

1. Verify if your client application (web component) is still working at [https://wac-24.westeurope.cloudapp.azure.com/ui](https://wac-24.westeurope.cloudapp.azure.com/ui/mongo-express/).

2. If you have made any changes to your front-end application - `ambulance-ufe` in the meantime, you need to create a new release for it. Similar to the [initial deployment of our Web component](../01.Web-Components/111-production-deployment.md), go to the GitHub repository for `ambulance-ufe`, navigate to the _Code_ section, click on the `1 tag` link, and then click on the _Create a new release_ button. In the dropdown list _Choose a tag_, enter the text `v1.1.0` - or any higher value following [semantic versioning][semver]. In the _Release title_ field, enter the text `v1.1.0`, and in the _Describe this release_ field, enter text describing the changes you made. Press the _Publish release_ button.

This will trigger the continuous integration process to start automatically after creating a new release. On the GitHub page for your `ambulance-ufe` repository, go to the _Actions_ section and check that the new continuous integration run completes successfully. After its completion, you can also verify the status of the image on the [Docker Hub] page, where you can see new version labels and platforms for your image.

3. Similarly, create a new release for the web API. Go to the GitHub repository for `ambulance-webapi`, navigate to the _Code_ section, click on the `1 tag` link, and then click on the _Create a new release_ button. In the dropdown list _Choose a tag_, enter the text `v1.1.0` - or any higher value following [semantic versioning][semver]. In the _Release title_ field, enter the text `v1.1.0`, and in the _Describe this release_ field, enter _Kustomize manifests_. Press the _Publish release_ button.

![Creating a new release](./img/020-01-Create-WebApi-Release.png)

On the GitHub page for your `ambulance-webapi` repository, go to the _Actions_ section and check that the new continuous integration run completes successfully. After its completion, you can also verify the status of the image on the [Docker Hub] page, where you can see new version labels and platforms for your image.

4. Open the file `${WAC_ROOT}/ambulance-gitops/components/version-release/kustomization.yaml` and modify it. Replace the used tags with the tags created in the previous two steps, and replace `<docker-id>` with your Docker Hub account name.

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
images:
- name: <docker-id>/ambulance-ufe
   newName: <docker-id>/ambulance-ufe 
   newTag: 1.1.0 @_important_@
- name: <docker-id>/ambulance-wl-webapi   @_add_@
   newName: <docker-id>/ambulance-wl-webapi   @_add_@
   newTag: 1.1.0   @_add_@
replacements: 
...
```

5. Modify the file: `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/install/kustomization.yaml`

```yaml
...
resources:
- ../../../apps/<pfx>-ambulance-ufe
- ../../../apps/<pfx>-ambulance-webapi @_add_@
...
```

6. Archive your changes to the remote repository:

```ps
git add .
git commit -m "Release of webapi to production cloud"
git push
```

Once the changes reach the `main` branch and Flux deploys the new components, verify that your pods are running - either using the _Lens_ tool or the `kubectl` command. (Don't forget to change the context!)

```ps
kubectl config get-contexts
kubectl config use-context <meno-kontextu-na-produkcny-k8s>
kubectl get pods -n wac-hospital
```

7. You can try accessing your web component through the page [https://wac-24.westeurope.cloudapp.azure.com/ui/](https://wac-24.westeurope.cloudapp.azure.com/ui/).

---

>info:> Similarly, follow these steps when deploying your semester project. Create the necessary manifests, update the release version based on its stability, and deploy it to the production cluster.
