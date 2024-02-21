# Deployment of the application on a production Kubernetes cluster

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-111`

---

For security reasons, the GitOps git repository for the production cluster is private, and only the gitops team can make changes to it. It has a similar structure to our `ambulance-gitops` repository. Unlike the local cluster, an external GitOps team ensures the presence of infrastructure in this cluster. Students have configuration available for the cluster, through which they can access the cluster and deploy their applications to the `wac-hospital` namespace. In this step, we will prepare the configuration for deployment to the cluster. We assume that your configuration repository and software container images are publicly accessible, and authentication is not required. If not, you need to add the relevant authentication credentials to the configuration in the form of Kubernetes objects, specifically a _Secret_.

1. In this configuration, we assume that you want to explicitly control which versions of software containers will be deployed to the shared cluster, assuming it is a _production_ cluster. Let's assume that our application is ready for production deployment. Therefore, as a first step, we will create a new release of our application. On the [GitHub] page, go to the `ambulance-ufe` repository and in the _Code_ section, click on the `0 tags` link, and then on the _Create a new release_ button. In the _Choose a tag_ dropdown, enter the text `v1.0.0` and click the _+ Create new tag: v1.0.0 on publish_ button. In the _Release title_ field, enter the text `v1.0.0`, and in the _Describe this release_ field, enter the text `Initial release of the waiting list application with CRUD operations`. Press the _Publish release_ button.

![Creating a new release](./img/111-01-CreateRelease.png)

In the previous sections, we set up continuous integration. In the file `${WAC_ROOT}/ambulance-ufe/.github/workflows/ci.yml`, we specified a trigger for creating a tag in the format `v1*`.


```yaml
name: Ambulance uFE CI
on:
push:
    branches: [ "main" ]
    tags: @_important_@
    - 'v1*'  @_important_@
pull_request:
    branches: [ "main" ]
```

to trigger the continuous integration process automatically after creating a new release. Go to the [GitHub] page for your `ambulance-ufe` repository, navigate to the _Actions_ section, and ensure that a new run of the continuous integration process completes successfully. After its completion, you can also verify the status of the image on the [Docker Hub] page, where you can see new version tags and platforms for your image.

2. Create a new file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/install/kustomization.yaml` with the following content:


```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

commonLabels:
app.kubernetes.io/part-of: wac-hospital
app.kubernetes.io/name: <pfx>-ambulance-wl

resources:
- ../../../apps/<pfx>-ambulance-ufe


components: 
- ../../../components/version-release @_important_@
```

Our shared cluster is named `wac-aks`, indicating deployment to the [Azure Kubernetes Services](https://azure.microsoft.com/en-us/products/kubernetes-service). The content is similar to the content from the `localhost` cluster, but we have changed the component in the `components` section.

3. Create a file `${WAC_ROOT}\ambulance-gitops/components/version-release/kustomization.yaml` with the following content:


```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
images:
- name: <docker-id>/ambulance-ufe
    newName: <docker-id>/ambulance-ufe 
    newTag: 1.0.0  #current version that you want to deploy

replacements: 
- targets:
    - select: 
        group: fe.milung.eu
        version: v1 
        kind: WebComponent
        name: <pfx>-ambulance-ufe 
    fieldPaths:
        - spec.hash-suffix @_important_@
    source: 
    version: v1
    kind: Deployment
    name:  <pfx>-ambulance-ufe-deployment  
    fieldPath: spec.template.spec.containers.0.image @_important_@
    options: 
        delimiter: ':'
        index: 1
```

In this cluster, we will not use [_Image Update Automation_](https://fluxcd.io/flux/guides/image-update/) [Flux CD] operator, but we want explicit control over the version of Docker images being deployed. Therefore, we created a new component `version-release` that contains configuration specific to the official release of our software.

Additionally, we used the [_Replacement Transformer_](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/) to copy the Docker image version from our deployment to the definition of the web component for the purpose of cache busting. Whenever we change the Docker image version tag, the hash in the web component definition and the hash in the resulting JavaScript file offered by the `ufe-controller` service will also change. As a result, when the user reloads the page, the version of our component is refreshed in the browser.

>info:> We could also use semantic versioning and define a component for each version, for example, `version-1.0.3`, and in the `version-release` component, create a reference to the current release's component. For the sake of simplicity, we will skip this approach in the exercise.

4. Create the directory `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/gitops` and within it, create the file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/gitops/git-repository.yaml` with the following content:


```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
name: <pfx>-gitops-repo # must be unique as there are multiple instances of this object in common cluster @_important_@
namespace: wac-hospital
spec:
interval: 1m0s
ref:
    branch: main
timeout: 1m0s
url: https://github.com/<your-account>/ambulance-gitops @_important_@

# don't forget to create access secrets for your repository if it is private 
# secretRef:
#    name: <pfx>-repository-pat
```

This manifest defines access to your repository for the [Flux CD] operator. Pay attention to the use of unique names for objects within the cluster. Next, create the file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/gitops/install.kustomization.yaml` defining the configuration of your application:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
name: <pfx>-install  # must be unique as there are multiple instances of this object in common cluster @_important_@
namespace: wac-hospital
spec:
wait: true
interval: 42s
path: clusters/wac-aks/install @_important_@
prune: true
sourceRef:
    kind: GitRepository
    name: <pfx>-gitops-repo @_important_@
```

Add the file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/gitops/cd.kustomization.yaml` with the content for the configuration of the continuous deployment of your application:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
    name: <pfx>-cd # v spoločnom klastri je nasadených viacero takýchto objektov @_important_@
    namespace: wac-hospital
spec:
    wait: true
    interval: 42s
    path: clusters/wac-aks
    prune: true
    sourceRef:
        kind: GitRepository
        name: <pfx>-gitops-repo @_important_@
```

Add the file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/gitops/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
    app.kubernetes.io/part-of: wac-hospital
    app.kubernetes.io/name: <pfx>-ambulance-wl

namespace: wac-hospital

resources:
- cd.kustomization.yaml
- install.kustomization.yaml
- git-repository.yaml
```

Finally, add the file `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization


commonLabels:
app.kubernetes.io/name: <pfx>-apps
app.kubernetes.io/instance: release
app.kubernetes.io/version: "1.0.0"

resources:
    - gitops  
```

>info:> If you are using a private repository, don't forget to create objects for authentication details. Also, refer to point 8 in the chapter [Continuous Deployment using Flux - Configuration Preparation](./081-flux)

5. Save the changes and verify the correctness of the configuration - in the directory `${WAC_ROOT}/ambulance-gitops`, run the command:


```ps
kubectl kustomize clusters/wac-aks/install
kubectl kustomize clusters/wac-aks
```

If both commands were able to generate the configuration without errors, archive the changes in the `${WAC_ROOT}/ambulance-gitops` folder to the repository. In the `${WAC_ROOT}/ambulance-gitops` directory, run the command:


```ps
git add .
git commit -m "Configuration for deployment to the production cluster"
git push
```

6. To activate continuous deployment for our configuration, we need to deploy it manually for the first time. For this, you will need access to the shared cluster. Based on the _Course Instructions_, download the configuration for the shared cluster - `wac-202?-student.kubeconfig.yaml` and [merge it with the existing configuration](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/). In the long term, it is probably most practical to maintain configurations for individual clusters in separate files and set the `KUBECONFIG` variable in your environment to point to the list of individual clusters. Copy the configuration for the shared cluster to the file `$HOME/.kube/wac-student.kubeconfig.yaml` and set the `$env:KUBECONFIG` variable in your environment to `$HOME\.kube\config;$HOME\.kube\wac-student.kubeconfig.yaml`. In a Linux environment, use the `:` character as the path separator instead of `;`:


```ps
cp wac-202?-student.kubeconfig.yaml $HOME/.kube/wac-student.kubeconfig.yaml
$env:KUBECONFIG="$HOME/.kube/config;$HOME/.kube/wac-student.kubeconfig.yaml"
```

>info:> For Linux and Mac systems, use ':' as the path separator instead of ';'

The provided configuration adds a new `wac-student` context to the kubectl environment, allowing you to connect to the shared cluster.

>info:> We recommend having the `$env:KUBECONFIG` variable set persistently, not just for the current terminal instance. You can use the `echo $profile` command to get the path to your PowerShell profile and add the command `$env:KUBECONFIG="$HOME/.kube/config:$HOME/.kube/wac-student.kubeconfig.yaml"` to it, so it runs every time you start the PowerShell terminal. Alternatively, similar initialization files exist for other command line interpreters.


7. In the `${WAC_ROOT}/ambulance-gitops` directory, run the commands:

```ps
kubectl config use-kontext wac
kubectl apply -k clusters/wac-aks
```

Verify the deployment status of the application with the commands:

```ps
kubectl get gitrepository <pfx>-git-repo -n wac-hospital
kubectl get kustomization <pfx>-cd -n wac-hospital
kubectl get kustomization <pfx>-install -n wac-hospital
```

If the `READY` status is `True`, it means that Flux CD has successfully deployed the configuration to the cluster.

>build_circle:> If the `READY` status is `False`, check the `Status` item in the output of the command `kubectl describe kustomization <name> -n wac-hospital` and correct any errors. Apply the fix with a commit to your repository. If the error concerns the `gitops-repo` object, also execute the command `kubectl apply -k clusters/localhost`; otherwise, a commit to the repository is sufficient.

Finally, verify if all deployed objects are ready and if all pods are in the `Running` state:


```ps
kubectl get all -n wac-hospital
```

The output should include, among other things, your deployment `<pfx>-ambulance-ufe-deployment` and the corresponding pod. If both are deployed, you can proceed to the shared cluster page, where you will see your application in the list of applications.

Change the kubectl context back to the local cluster:

```ps
kubectl config use-kontext docker-desktop
```

Thanks to the continuous deployment configuration, making changes in your `ambulance-gitops` repository will be sufficient. The [Flux CD] operator will ensure the application of changes directly to the cluster. However, remember that if you want to change the version of the Docker image, you need to create a new release in the `ambulance-ufe` repository and update the version in the file `${WAC_ROOT}/ambulance-gitops/components/version-release/kustomization.yaml`.
