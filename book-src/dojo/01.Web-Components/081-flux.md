# Continuous Deployment using Flux (locally in Kubernetes within Docker Desktop)

---

>info:>
Template for a pre-existing container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-070`

---

[Flux] is currently one of the most widely used tools for continuous deployment. It operates on the principle of _Pull Deployment_ and fully supports [GitOps] methodologies. In short, GitOps consists of three main components:

- IaC (Infrastructure as Code) - the description of infrastructure (_deployment_) is stored in files in a Git repository and serves as the "single source of truth."
- MRs (Merge Requests) - changes to infrastructure occur in code through merge requests. Typically, mandatory code reviews are set up here.
- CI/CD (Continuous Integration and Continuous Deployment) - automatic infrastructure updates are facilitated through CI/CD.

[Flux] is based on a set of extensions to the [Kubernetes API][k8s-api], known as [_Custom resources_][k8s-crd], which control how changes in Git repositories are applied to the Kubernetes cluster. Two basic objects we will work with are:

- [`GitRepository`](https://fluxcd.io/flux/components/source/gitrepositories/) object - mirrors configurations from a given Git repository.
- [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomization/) object - precisely specifies the directory within the repository and the synchronization frequency.

>warning:> `Flux Kustomization` object and `Kustomize Kustomization` object
> are two different objects. They will be mixed in the following text, so we will refer to them accurately each time.

The image illustrates Flux components.

![Flux GitOps](./img/081-01-flux-gitops-toolkit.png)

We will demonstrate the installation and configuration of Flux on a local Kubernetes cluster provided by Docker Desktop.

On a Kubernetes cluster in a data center (Azure, AWS, Google, ...), an administrator installs Flux.

>info:> At the end of the next chapter, there is an image depicting all mentioned components and their interconnection.

## Configuration Preparation

In this section, we will first configure all necessary files and
save them to a Git repository to prepare for continuous deployment to the Kubernetes cluster.

1. Create a new directory `${WAC_ROOT}/ambulance-gitops/infrastructure/fluxcd` and create a file `kustomization.yaml` in it with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/fluxcd/flux2//manifests/install?ref=v2.0.1
```

By doing this, we have created a dependency on a specific release of the [Flux CD][flux] system. Add a reference to the Flux CD configuration in the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`:

```yaml
...
resources:
- namespace.yaml
- ../../../infrastructure/ufe-controller
- ../../../infrastructure/fluxcd @_add_@
```

2. Create a directory `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops` and in it, create a file `git-repository.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: wac-hospital
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: repository-pat
  timeout: 1m0s
  url: https://github.com/<your-account>/ambulance-gitops @_important_@
```

This file defines the Git repository and branch that Flux CD will track and manage continuous deployment based on the configuration in this repository. It will use a personal access token (PAT) for access, which you will generate later.

3. In the same directory, create a file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/prepare.kustomization.yaml`:


```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prepare
  namespace: wac-hospital
spec:
  wait: true
  interval: 42s
  path: clusters/localhost/prepare
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
```

This file specifies that Flux CD will track changes in the `clusters/localhost/prepare` directory and, if any occur, apply them to the Kubernetes cluster.

4. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/install.kustomization.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: install
  namespace: wac-hospital
spec:
  wait: true
  force: true 
  dependsOn:  @_important_@
  - name: prepare @_important_@
  interval: 42s
  path: clusters/localhost/install
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
```

This file specifies that Flux CD will track changes in the `clusters/localhost/install` directory and, if any occur, apply them to the Kubernetes cluster. Notice that we have also specified that this configuration depends on the deployment and readiness of the `prepare` configuration.

5. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/cd.kustomization.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cd
  namespace: wac-hospital
spec:
  wait: true
  interval: 42s
  path: clusters/localhost
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
```

This file specifies that Flux CD will track the configuration in the `clusters/localhost/` directory, and if it changes, it will apply the changes to the Kubernetes cluster. This means that if we want to modify the details of continuous integration, it will be sufficient to write them to our repository.

6. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/kustomization.yaml`, which will integrate the above-mentioned files:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
app.kubernetes.io/part-of: wac-hospital

namespace: wac-hospital

resources:
- prepare.kustomization.yaml
- cd.kustomization.yaml
- install.kustomization.yaml
- git-repository.yaml
```

7. In the directory `${WAC_ROOT}/ambulance-gitops/clusters/localhost`, create a file `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization


commonLabels:
  app.kubernetes.io/name: wac-hospital
  app.kubernetes.io/instance: development
  app.kubernetes.io/version: "0.1"

resources:
  - gitops
```

This configuration refers to the `gitops` folder that we created in the previous step. This means that the cluster configuration is managed by Flux CD objects, which ensure continuous deployment based on the configuration in the Git repository at the respective paths.

8. If our repository is public, FluxCD can retrieve information without configuring access permissions. However, in practice, configurations are often specific and private. Therefore, we assume that this repository is private and we need to configure access credentials for FluxCD.

Go to the [GitHub] page, log in, and then navigate to the [Developer Settings](https://github.com/settings/apps) page. _You can get there from the main page by clicking on your avatar icon, selecting "Settings" from the menu, and then "Developer settings"._ On the left side, choose _Personal access tokens->Tokens (classic)_ and click the _Generate new token_ button. Log in, enter a token name, e.g., _WAC-FluxCD token_, and set _Expiration_ to _Custom defined_ (date at least until the end of the semester). In the _Scopes_ section, check the _repo_ item. Press the _Generate token_ button, and __DO NOT FORGET TO SAVE__ the generated PAT to your clipboard.

![Creating Personal Access Token](./img/081-01-GitHubPAT.png)

Subsequently, we need to store this token in the Kubernetes cluster. In the context of GitOps, we now have several options on how to proceed. Here, we'll show a simpler approach for now, where passwords and other confidential information will not be part of our repository, but we will have access to them locally only. However, this approach becomes more complicated when we have to deal with more than one password or cluster. In such a case, it is much more appropriate to use [_SOPS - Service Operations_](https://fluxcd.io/flux/guides/mozilla-sops/), where passwords and public keys are encrypted directly in the repository, and the private key needed to decrypt confidential information is only accessible in the cluster itself, where it is manually deployed by its administrator. We will show the second method in another chapter.

Create a folder `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets` and in it, create a file `repository-pat.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repository-pat
  namespace: wac-hospital

type: Opaque
stringData:
  username: <your-acount> # zvyčajne môže byť akékoľvek meno @_important_@
  password: <your-personal-access-token>  # vygenerovaný PAT @_important_@
```

In the same directory, create a file `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  app.kubernetes.io/part-of: wac-hospital

namespace: wac-hospital

resources:
  - repository-pat.yaml
```

Finally, create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/.gitignore` with the content:

```plain
*
!.gitignore
!kustomization.yaml
```

By this last step, we have ensured that the `kustomization.yaml` file is present in the Git repository, but the `repository-pat.yaml` file is not. This way, we ensure that the password is not part of our Git repository. The downside is that in the event of a loss of local data, we would need to regenerate the password.

>warning:> Storing sensitive information in a file on disk is also exposed to the risk of unauthorized access. Make sure that only authorized individuals have access to your computer and the specified directory, or delete sensitive files after deployment and recreate them only when necessary.

9. Test the correctness of your configuration with commands executed from the `${WAC_ROOT}/ambulance-gitops/` directory:

```ps
kubectl kustomize clusters/localhost
kubectl kustomize clusters/localhost/prepare
kubectl kustomize clusters/localhost/install
kubectl kustomize clusters/localhost/secrets
```

If all commands ran without error messages, archive your repository:

```ps
git add .
git commit -m 'fluxcd configuration'
git push
```

## Bootstrapping Flux

To start using [Flux] services, we need to initially deploy them into the cluster manually. After the initial deployment, the state will be synchronized between the cluster and our repository. Subsequently, it will be sufficient to save configuration changes to the repository. One of the advantages of this approach is that we can control who from the development team needs access to individual deployments/clusters. In the case of DevOps-style development, we assume that this is the majority of developers, and at the same time, we can control the permissions granted to each member.

>info:> In this exercise, we deploy [Flux] by referencing published manifests in the [fluxcd/flux2](https://github.com/fluxcd/flux2) repository. Alternative installation methods are described in the [Flux documentation](https://fluxcd.io/flux/installation/).

1. Deploy the Flux operator into our cluster. Ensure you have selected the correct context - `kubectl config get-contexts`, navigate to the `${WAC_ROOT}/ambulance-gitops/` directory, and execute the command:

```ps
kubectl apply -k infrastructure/fluxcd
```

By this command, we have deployed Flux into the cluster. Check whether Flux has been deployed and whether everything is okay, or wait until all pods are in the `Running` state:
  
```ps
kubectl get all -n flux-system
```

2. We need to place the access credentials for our repository into the cluster. Still in the `${WAC_ROOT}/ambulance-gitops/` directory, execute the commands:

```ps
kubectl create namespace wac-hospital
kubectl apply -k clusters/localhost/secrets
```

3. Now we will deploy our configuration for the `localhost` cluster. In the `${WAC_ROOT}/ambulance-gitops/` directory, execute the command:

```ps
kubectl apply -k clusters/localhost
```

By this command, we directly deployed objects into the cluster from the `${WAC_ROOT}/ambulance-gitops/clusters/localhost\gitops` directory. Using the `gitops-repo` object of type `GitRepository`, Flux creates a local copy of the specified branch of our repository. Subsequently, using objects of type `cd` and `prepare`, both of type `Kustomization.kustomize.toolkit.fluxcd.io`, Flux applies the configuration in the `clusters/localhost` directory to the cluster. This ensures the restoration of the configuration of the continuous deployment specification itself. Simultaneously, using the `prepare` object, it also installs services into the cluster that our application implicitly assumes. In this case, it is the `ufe-controller` service and the [Flux CD][flux] operator, which can be updated to a newer version in this way.

After applying and preparing the configuration with the `prepare` object, the configuration specified in the `install` object of type `Kustomization.kustomize.toolkit.fluxcd.io` starts to apply, deploying our project's custom services and objects.

[Flux CD][flux] regularly checks for changes in the repository or whether the cluster state is different from the configuration specified by any of the `Kustomization` objects. Upon any detected change, it attempts to achieve a state identical to the one prescribed in the configuration. If the configuration in the repository changes, Flux CD automatically updates the configuration in the cluster.

>build_circle:> Sometimes we need to temporarily change the state of objects in the cluster - for example, when analyzing a reported issue, we may want to temporarily change the logging level generated by our microservice. If you add the annotation `kustomize.toolkit.fluxcd.io/reconcile: disabled` to an object in the cluster, the object's state will not change until you remove this annotation. You can apply the annotation, for example, with the command:
>
> ```ps
> kubectl annotate deployment <name> kustomize.toolkit.fluxcd.io/reconcile=disabled
> ```
>
> After you have finished your analysis, you can remove the annotation.

Check the status of the objects in the cluster:

```ps
kubectl get gitrepository -n wac-hospital
kubectl get kustomization -n wac-hospital
```
  
The output should look like this:
   
```plain
kubectl get gitrepository -n wac-hospital
NAME          URL                                          AGE    READY   STATUS
gitops-repo   https://github.com/milung/ambulance-gitops   119s   True    stored artifact for revision 'main@sha1:...'

kubectl get kustomization -n wac-hospital         
NAME      AGE   READY   STATUS
cd        16m   True    Applied revision: main@sha1...
install   16m   True    Applied revision: main@sha1:...
prepare   11m   True    Applied revision: main@sha1:...
```

If the `READY` status is set to `True`, it means that Flux CD has successfully deployed the configuration to the cluster.

>build_circle:> If the `READY` status is set to `False`, check the `Status` item in the output of the `kubectl describe kustomization <name> -n wac-hospital` command, and correct any errors. Apply the fix by committing your changes to the repository. If the error concerns the `gitops-repo` object, also execute the command `kubectl apply -k clusters/localhost`; otherwise, a commit to the repository is sufficient.

Finally, verify whether all deployed objects are ready and whether all pods are in the `Running` state:


```ps
kubectl get all -n wac-hospital
```
  
The output should look like this:

```plain
  NAME                                            READY   STATUS    RESTARTS   AGE
  pod/ambulance-ufe-deployment-64cfc4c9db-d46vq   1/1     Running   0          78m
  pod/ambulance-ufe-deployment-64cfc4c9db-f2cm7   1/1     Running   0          78m
  pod/ufe-controller-594bc6f989-45fjn             1/1     Running   0          78m
  pod/ufe-controller-594bc6f989-5b9jd             1/1     Running   0          78m

  NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT (S)        AGE
  service/ambulance-ufe    ClusterIP   10.105.190.116   <none>        80/ TCP         78m
  service/ufe-controller   NodePort    10.96.155.129    <none>        80:30331/ TCP   78m

  NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/ambulance-ufe-deployment   2/2     2            2           78m
  deployment.apps/ufe-controller             2/2     2            2           78m

  NAME                                                  DESIRED   CURRENT    READY   AGE
  replicaset.apps/ambulance-ufe-deployment-64cfc4c9db   2         2          2       78m
  replicaset.apps/ufe-controller-594bc6f989             2         2          2       78m
```

    For each of our two components (ambulance-ufe and ufe-controller), the following objects were created:
    - 1x service
    - 1x deployment
    - 1x replicaset (created automatically for each deployment)
    - 2x pods (specified 2 replicas in the deployment)

Enter the address [http://localhost:30331/](http://localhost:30331/) into the browser.
You should see a page with the application shell containing the integrated micro application. After clicking on the link _Zoznam čakajúcich_ (List of Patients), you should see the following output:

![Integrated list of patients](./img/060-02-appshell-list.png)

## Verification of Continuous Deployment Functionality

In principle, we have the first version of continuous deployment ready. If any of the YAML manifests in the `ambulance-gitops` repository is now changed (but must be referenced from the `${WAC_ROOT}/ambulance-gitops/clusters/localhost` directory), [Flux] will ensure that the changes are automatically reflected in the cluster. Let's try it.

In the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/deployment.yaml`, change the number of replicas to 1, save the changes, and archive them (commit and push) to the remote repository.

After a while, check that the changes have been reflected in the cluster. The output of the following command should show 1 pod with the name _ambulance-ufe-deployment_.

```ps
kubectl get pods -n wac-hospital
```
