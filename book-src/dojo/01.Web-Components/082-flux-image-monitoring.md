# Flux - monitoring and applying changes to the version of the Docker image

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-081`

---

In the previous chapter, we implemented continuous deployment based on the configuration in our repository. This configuration always deploys the latest version of our container image (_latest_) and dependent containers, such as `ufe-controller`. In the event of a change in the API during the release of any of the containers, it can affect the functionality of our solution. Therefore, in production deployment, specific releases of software containers or even specific variants of containers are always specified using an SHA signature. This helps maintain stability and cybersecurity in the production deployment. The choice of specific versions for a deployment can be realized, for example, using the [ImageTransformer](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_imagetagtransformer_) in the Kustomize configuration of our cluster.

Unlike the production system, when deploying for the development team, we want to deploy the latest versions of software containers that have successfully passed continuous integration. In this case, using the label `latest` (or similar) is not sufficient because the Kubernetes orchestrator does not know when there is a change in the image with this label in the container registry. Specifying a specific image version is possible, but if we want to deploy a newer version, we would have to manually change the configuration in the `ambulance-gitops` repository. Especially in larger teams, this step might be forgotten. Ideally, the change in the image version in the repository should automatically reflect in the cluster. In this chapter, we will show how to achieve this using Flux.

Currently, we have deployed the `latest` version of the container (see the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/deployment.yaml`). When we create a new version of the container, we have two options to automatically deploy it to the cluster.

1. We tag the container with a unique tag (e.g., buildversion), and as part of continuous integration, we add a step that modifies the mentioned `deployment.yaml` file and sets the container version in it. The integration step commits the modified file to the repository, and [Flux] registers the change and makes updates in the Kubernetes cluster.
2. The second option is to configure [Flux] to monitor container changes on [Docker Hub] and to take a new version if available. In this case, [Flux] itself will modify the relevant manifest, set the latest suitable version of the container in it, and save the change to the repository. This way, the GitOps principle is still preserved, stating that the _only configuration object is a git repository_. Only after changing the version of the Docker image in the corresponding branch of the git repository will an update occur in the cluster.

In the first step, we determine which [repository](https://fluxcd.io/flux/components/image/imagerepositories/) of the software container Flux should monitor. Our _ambulance-ufe_ Docker image is publicly accessible, so there is no need for authentication. Create a file `${WAC_ROOT}/ambulance-gitops/cluster/localhost/gitops/ambulance-ufe.image-repository.yaml` with the following content:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: ambulance-ufe
  namespace: wac-hospital
spec:
  image: <docker-id>/ambulance-ufe @_important_@
  interval: 1m0s
```

#warning: Replace <docker-id>!

Create a file `${WAC_ROOT}/ambulance-gitops/cluster/localhost/gitops/ufe-controller.image-repository.yaml` with the following content:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: ufe-controller
  namespace: wac-hospital
spec:
  image: milung/ufe-controller
  interval: 15m0s
```

2. Another Flux component [_ImagePolicy_](https://fluxcd.io/flux/components/image/imagepolicies/) sets the criterion for selecting the appropriate version of the Docker image. Create a file `${WAC_ROOT}/ambulance-gitops/cluster/localhost/gitops/ambulance-ufe.image-policy.yaml` with the following content:


```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: ambulance-ufe
  namespace: wac-hospital
spec:
  imageRepositoryRef:
    name: ambulance-ufe # refers to ImageRepository from the previous step @_important_@
  filterTags:
      pattern: "main.*" # selects all versions that start with main- (e.g., main-20240315.1200) @_important_@
  policy:
  alphabetical:
    order: asc
    
```

and file `${WAC_ROOT}/ambulance-gitops/cluster/localhost/gitops/ufe-controller.image-policy.yaml`


```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: ufe-controller
  namespace: wac-hospital
spec:
  imageRepositoryRef:
    name: ufe-controller # referuje ImageRepository z predchádzajúceho kroku @_important_@
  policy:
    semver:
      range: "^1.*.*" @_important_@
```

3. Modify all files where we want Flux to update the version of the Docker image. This is done by adding a special marker `# {"$imagepolicy": "POLICY_NAMESPACE:POLICY_NAME"}` to the line that needs to be modified. In our case, we could modify the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/deployment.yaml` and adjust the configuration in the directory `${WAC_ROOT}/ambulance-gitops/infrastructure/ufe-controller`. However, it is more advantageous in this case to have all container versions in one place and also have the ability to control container versions for individual releases of our system. To achieve this, we will use so-called [_Kustomize components_](https://kubectl.docs.kubernetes.io/guides/config_management/components/), which allow combining different configuration variants.

Create a file `${WAC_ROOT}/ambulance-gitops/components/version-developers/kustomization.yaml` with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1  
kind: Component 

images:
- name: <docker-id>/ambulance-ufe  @_important_@
  newName: <docker-id>/ambulance-ufe # {"$imagepolicy":  "wac-hospital:ambulance-ufe:name"} @_important_@
  newTag: main # {"$imagepolicy": "wac-hospital:ambulance-ufe:tag"} @_important_@

- name: milung/ufe-controller
  newName: milung/ufe-controller # {"$imagepolicy":  "wac-hospital:ufe-controller:name"} @_important_@
  newTag: latest # {"$imagepolicy": "wac-hospital:ufe-controller:tag"} @_important_@
```

Notice the markers in the comments - it is important to refer to the correct names of the _image policies_ created in the previous step.

Now, modify the files `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml` and `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`. Add the following section to both:

```yaml
...
components: 
- ../../../components/version-developers
```

This way, through customization, we will modify all occurrences in referenced YAML files where the image `<docker_id>/ambulance-ufe` or `milung/ufe-controller` is found.

4. Create a new component `ImageUpdateAutomation`, where we define the locations of files to be modified. Create a file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/image-update-automation.yaml` with the following content:


```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: image-updater
  namespace: wac-hospital
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: gitops-repo # repository that we configured in previous chapter @_important_@
  git:
    checkout:
      ref:
        branch: main @_important_@
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcd_bot
      messageTemplate: |
        Automated image update
        
        Automation name: {{ .AutomationObject }}
        
        Files:
        {{ range $filename, $_ := .Updated.Files -}}
        - {{ $filename }}
        {{ end -}}
        
        Images:
        {{ range .Updated.Images -}}
        - {{.}}
        {{ end -}}
    push:
      branch: main  @_important_@
  update:
    path: ./components/version-developers  # path where we want the update to happen @_important_@
    strategy: Setters
```

> warning:> If you are using the `master` branch or another, update the name in the `main` entries.

This object, within the `gitops-repo` Git repository in the `main` branch and the `/components/version-developers` directory, will update all files containing the mentioned marker. It then archives the files. The new version of the repository will include the comment specified in the `commit` section.

> info:> In theory, the `ImageUpdateAutomation` object could be deployed in multiple clusters, but it may cause conflicts. It is better to have only one of the deployed clusters modify the relevant configuration - in our case, `components/version-developers`. In practice, this could be a development cluster used by developers and testers for quick feedback on changes made.

5. Finally, modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/kustomization.yaml` and in the `resources` section, specify the newly created files:

  
```yaml
...
resources: 
...
- ambulance-ufe.image-repository.yaml @_add_@
- ambulance-ufe.image-policy.yaml @_add_@
- ufe-controller.image-repository.yaml @_add_@
- ufe-controller.image-policy.yaml @_add_@
- image-update-automation.yaml @_add_@
```
  
6. Save all files and verify the correctness of your configurations. In the directory `${WAC_ROOT}/ambulance-gitops`, execute the following commands:
  
```ps
kubectl kustomize clusters/localhost/install
kubectl kustomize clusters/localhost/prepare
kubectl kustomize clusters/localhost
```

Archive the changes and synchronize with the remote repository.

```ps
git add .
git commit -m "fluxcd - image update automation"
git push
```

This way, we have simultaneously applied our changes to the cluster. After a while, the container version in the cluster will automatically change to `main-YYYYMMDD.HHMM`, and in the repository, you can see a new commit authored by `fluxcd_bot`.

![Container version change](./img/082-01-FluxBotCommit.png)

Explicitly verify the entire CI/CD cycle. In the `${WAC_ROOT}/ambulance-ufe` directory, change the code of the `ambulance-ufe` component, for example, change the patient's name in the list, commit, and synchronize the changes. After a while, when CI runs and a new image is created on DockerHub, check the history on the [GitHub] page in the _ambulance-gitops_ repository (click on the _N commits_ label at the top of the file list). Then, enter the address [http://localhost:30331/](http://localhost:30331/) in your browser and check the list of patients.

After verification, refresh the repository state - flux cd has recorded the new changes into it:

```ps
git fetch
git pull
```

---

>build_circle:> In case of a cluster failure, for example, during the reinstallation of a computer, you can restore the deployment using the configuration in the repository. Create a local copy of the repository. In the `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets` directory, create a file `repository-pat.yaml` as described in the previous chapter. Then, follow the instructions given in the previous chapter in the _Bootstrapping Flux_ section. It is sufficient to make other changes to our configuration exclusively in the Git repository. However, continue to monitor the state of individual objects continuously, which is a standard practice in software development using DevOps. You can also use objects from the [_Flux CD Notification Controller_](https://fluxcd.io/flux/components/notification/), which allow monitoring the state of individual objects and sending notifications to the relevant team communication channel in case of changes.

---

For better understanding, the following image illustrates the components mentioned in this chapter and their interconnection.

![Flux CD Components and Collaboration](./img/082-02-FluxCD.svg)

