# Managing login credentials using SecretsOps

--- 

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-050
```

---

Before we proceed to authenticate users, let's prepare a way to securely deploy sensitive information to our Kubernetes cluster. In our case, it currently involves login credentials for the database and a Personal Access Token for the repository. As sensitive information grows, managing them on the local disk without encryption becomes less efficient. The solution for us is to use the [_Secrets Ops_][sops] method. Its principle involves using asymmetric keys to encrypt sensitive information using a public key and the ability to decrypt them only using the private key. The private key is manually stored on the respective cluster. The public key and encrypted data can then be safely stored and archived in our repository. At the same time, we will leverage the built-in feature of [Flux CD](https://fluxcd.io/flux/guides/mozilla-sops/), allowing Flux to automatically decrypt this data when deploying it to the cluster.

1. To use this technique, you need to have the [sops] tool installed from [https://github.com/getsops/sops/releases](https://github.com/getsops/sops/releases). In this exercise, we will use [AGE] as the encryption tool, which you can install from [https://github.com/FiloSottile/age/releases/tag/v1.1.1](https://github.com/FiloSottile/age/releases). Both tools can also be installed using the [Chocolatey] package manager. The [AGE] tool can be installed with the `apt-get` command on Linux systems.

>info:> The [sops] tool also supports other encryption and key storage methods, such as [GPG](https://www.gnupg.org/) or [Azure KeyVault](https://learn.microsoft.com/en-us/azure/key-vault/general/), etc. Depending on the target requirements, you can use a different encryption tool; the process will be similar in all cases, except for configuring sops parameters.

2. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params/repository-pat.env` with content corresponding to your Personal Access Token for the repository. You should have this information in the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/repository-pat.yaml`. The content of the `repository-pat.env` file should look like this:


```env
username=<github-id>
password=<your-pat> @_important_@
```

>info:> Generate a _Personal Access Token_ following the [guide](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) or the procedure in the [Continuous Deployment using Flux](../01.Web-Components/081-flux.md) chapter.

Delete the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/repository-pat.yaml` - **YAML**, and modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/kustomization.yaml` to contain:

```yaml
...
resources: @_remove_@
  - repository-pat.yaml  @_remove_@

secretGenerator:        @_add_@
  - name: repository-pat        @_add_@
    type: Opaque        @_add_@
    envs:       @_add_@
      - params/repository-pat.env       @_add_@
    options:        @_add_@
        disableNameSuffixHash: true     @_add_@
```

3. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params/mongodb-auth.env` with the following content:

```env
username=admin
password=<silne-heslo> @_important_@
```

>warning:> When changing the password, it may be necessary to delete the original Persistent Volume Claim in the cluster - Mongo DB stores the initialization password on disk, and upon subsequent startup, it ignores the password set in the environment variable. For demonstration purposes, we recommend using the original password.

Modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/kustomization.yaml` and add:

```yaml
...
secretGenerator:
  - name: repository-pat
    ...
  - name: mongodb-auth    @_add_@
    type: Opaque    @_add_@
    envs:    @_add_@
      - params/mongodb-auth.env    @_add_@
    options:    @_add_@
    disableNameSuffixHash: true    @_add_@
```

Now we need to delete the original `mongodb-auth` object, which is added to the configuration along with mongodb. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/patches/mongodb-auth.secret.yaml` with the following content:

```yaml
$patch: delete  @_important_@
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-auth
```

and modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml`:

```yaml
...
patches:
  - path: patches/ambulance-webapi.service.yaml
  - path: patches/mongodb-auth.secret.yaml   @_add_@
```

4. Generate a new pair of encryption keys using the command:

```ps
age-keygen
```

The output of the command will look something like this:

```ps
# created: 2024-01-01T00:00:00Z
# public key: public-key
AGE-SECRET-KEY-private-key
```

Copy and securely store the private key - you will need it even when you want to redeploy your cluster. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params/.sops.yaml` with the following content:

```yaml
creation_rules:
- age: age_verejny_hexa_kluc 
```

This file is used to encrypt keys stored in this directory.

>warning:> Each cluster can have one or more associated encryption keys, but for security reasons, these keys should not be shared between different clusters. Perhaps the only exception is when you share the private key among product developers so that they can each deploy their own local cluster, assuming only test data is present.

For the same reason, we create all _Secret_ objects independently for each cluster, and their current values should differ - i.e., unique password, username, or client identifier.

Apply the private key to the cluster using the command:


```ps
$agekey="AGE-SECRET-KEY-private-key" @_important_@
kubectl create secret generic sops-age --namespace wac-hospital --from-literal=age.agekey="$agekey"
```

>warning:> You will need to perform this step before the initial deployment of Flux CD from now on.

Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/secrets.kustomization.yaml` with the following content:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: secrets
  namespace: wac-hospital
spec:
  wait: true
  interval: 42s
  path: clusters/localhost/secrets
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
```

and modify the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/kustomization.yaml`:

```yaml
  ...
  resources:
  - prepare.kustomization.yaml
  - cd.kustomization.yaml
  - install.kustomization.yaml
  - secrets.kustomization.yaml   @_add_@
  ...

  patches:    @_add_@
  - target:    @_add_@
      group: kustomize.toolkit.fluxcd.io    @_add_@
      version: v1    @_add_@
      kind: Kustomization    @_add_@
    patch: |-    @_add_@
      - op: add @_add_@
        path: /spec/decryption @_add_@
        value: @_add_@
          provider: sops @_add_@
          secretRef: @_add_@
            name: sops-age @_add_@
```

This modification adds configuration for decrypting files using the [sops] tool with our created [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/) object `sops-age` to all objects of type [_Kustomization_](https://fluxcd.io/flux/components/kustomize/kustomizations/). Additionally, we have added automation for deploying sensitive data to the cluster, which we had to do manually before.

5. Encrypt files with sensitive data. Open a command prompt in the directory `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params` and execute the commands:


```ps
sops --encrypt --in-place repository-pat.env
sops --encrypt --in-place mongodb-auth.env
```

If you now open the files `repository-pat.env` and `mongodb-auth.env`, you will see that their contents are encrypted. The added variables allow identifying which key and version of the [sops] tool were used for encryption.

6. Delete the files `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/.gitignore`. Archive the changes and submit them to the remote repository. In the directory `${WAC_ROOT}/ambulance-gitops`, execute the commands:

```ps
git add .
git commit -m "SecretsOps"
git push
```

7. Verify if our objects of type [Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/) are correctly deployed:

```ps
kubectl get kustomization -n wac-hospital
```

From now on, we can make changes to files with sensitive data, and after encrypting and submitting them to the repository, they will be automatically deployed to the cluster. In the case of redeployment, we only need to deploy the private key, which we saved in a secure place. When deploying to an empty cluster for the first time, we need to deploy the unencrypted `repository-pat` object as well so that FluxCD can download the source code from the repository. However, we can already maintain all other sensitive data in encrypted form directly in the repository, including `repository-pat` and theoretically the _Secret `sops-age`, but due to its potential inclusion of various key categories, we do not recommend it.

>warning:> In team collaboration, it may happen that a team member unintentionally archives unencrypted passwords. In such a case, it is necessary to change the login credentials. We also recommend implementing a suitable [_pre-commit hook_](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) in the repository to prevent the submission of unencrypted files.

## Automation of cluster deployment for developers

For your information, here is a script that you can use to automate the deployment to an empty cluster for developers. This script assumes that team members share a private key for deploying to the local cluster. The script automates the deployment of `sops-age` and `repository-pat`, subsequent deployment of Flux CD, and deployment of _kustomization_ objects when creating or refreshing the local cluster. You can save the script to a file, for example, `${WAC_ROOT}/ambulance-gitops/scripts/developer-deploy.ps1`:

```ps
param (
    $cluster ,
    $namespace,
    $installFlux = $true
)

if ( -not $cluster ) {
    $cluster = "localhost"
}

if ( -not $namespace ) {
    $namespace = "wac-hospital"
}

$ProjectRoot = "${PSScriptRoot}/.."
echo "ScriptRoot is $PSScriptRoot"
echo "ProjectRoot is $ProjectRoot"

$clusterRoot = "$ProjectRoot/clusters/$cluster"


$ErrorActionPreference = "Stop"

$context = kubectl config current-context

if ((Get-Host).version.major -lt 7 ){
    Write-Host -Foreground red "PowerShell Version must be minimum of 7, please install latest version of PowerShell. Current Version is $((Get-Host).version)"
    exit -10
}
$pwsv=$((Get-Host).version)
try {if(Get-Command sops){$sopsVersion=$(sops -v)}} 
Catch {
    Write-Host -Foreground red "sops CLI must be installed, use 'choco install sops' to install it before continuing."
    exit -11
}

# check if $cluster folder exists
if (-not (Test-Path -Path "$clusterRoot" -PathType Container)) {
    Write-Host -Foreground red "Cluster folder $cluster does not exist"
    exit -12
}

$banner = @"
THIS IS A FAST DEPLOYMENT SCRIPT FOR DEVELOPERS!
---

The script shall be running **only on fresh local cluster** **!
After initialization, it **uses gitops** controlled by installed flux cd controller.
To do some local fine tuning get familiarized with flux, kustomize, and kubernetes

Verify that your context is coresponding to your local development cluster:

* Your kubectl *context* is **$context**.
* You are installing *cluster* **$cluster**.
* *PowerShell* version is **$pwsv**.
* *Mozilaa SOPS* version is **$sopsVersion**.
* You got *private SOPS key* for development setup.
"@
    
$banner = ($banner | ConvertFrom-MarkDown -AsVt100EncodedString) 
Show-Markdown -InputObject $banner
Write-Host "$banner"
$correct = Read-Host "Are you sure to continue? (y/n)"

if ($correct -ne 'y')
{
    Write-Host -Foreground red "Exiting script due to the user selection"
    exit -1
}

function read-password($prompt="Password", $defaultPassword="")
{
    $p = "${prompt} [${defaultPassword}]"
    $password = Read-Host -MaskInput -Prompt $p
    if (-not $password) { $password = $defaultPassword}
    return $password
}

$agekey = read-password "Enter master key of SOPS AGE (for developers)" 

# create a namespace
Write-Host -Foreground blue "Creating namespace $namespace"
kubectl create namespace $namespace
Write-Host -Foreground green "Created namespace $namespace"

# generate AGE key pair and create a secret for it
Write-Host -Foreground blue "Creating sops-age private secret in the namespace ${namespace}"

kubectl delete secret sops-age --namespace "${namespace}"
kubectl create secret generic sops-age --namespace "${namespace}" --from-literal=age.agekey="$agekey"

Write-Host -Foreground green "Created sops-age private secret in the namespace ${namespace}"

# unencrypt gitops-repo secrets to push it into cluster
Write-Host -Foreground blue "Creating gitops-repo secret in the namespace ${namespace}"

$patSecret = "$clusterRoot/secrets/params/repository-pat.env"
if (-not (Test-Path -Path $patSecret)) {
    $patSecret = "$clusterRoot/../localhost/secrets/params/gitops-repo.env"
    if (-not (Test-Path -Path $patSecret)) {
        Write-Host -Foreground red "gitops-repo secret not found in $clusterRoot/secrets/params/gitops-repo.env or $clusterRoot/../localhost/secrets/params/gitops-repo.env"
        exit -13
    }
}

$oldKey=$env:SOPS_AGE_KEY
$env:SOPS_AGE_KEY=$agekey
$envs=sops --decrypt $patSecret

# check for error exit code
if ($LASTEXITCODE -ne 0) {
    Write-Host -Foreground red "Failed to decrypt gitops-repo secret"
    exit -14
}

# read environments from env
$envs | Foreach-Object {
    $env = $_.split("=")
    $envName = $env[0]
    $envValue = $env[1]
    if ($envName -eq "username") {
        $username = $envValue
    }
    if ($envName -eq "password") {
        $password = $envValue
    }    
}
$env:SOPS_AGE_KEY="$oldKey"
$agekey=""
kubectl delete secret repository-pat --namespace $namespace
kubectl create secret generic  repository-pat `
  --namespace $namespace `
  --from-literal username=$username `
  --from-literal password=$password `

$username=""
$password=""
Write-Host -Foreground green "Created gitops-repo secret in the namespace ${namespace}"

if($installFlux)
{
    Write-Host -Foreground blue "Deploying the Flux CD controller"
    # first ensure crds exists when applying the repos
    kubectl apply -k $ProjectRoot/infrastructure/fluxcd --wait

    if ($LASTEXITCODE -ne 0) {
        Write-Host -Foreground red "Failed to deploy fluxcd"
        exit -15
    }

    Write-Host -Foreground blue "Flux CD controller deployed"
}

Write-Host -Foreground blue "Deploying the cluster manifests"
kubectl apply -k $clusterRoot --wait
Write-Host -Foreground green "Bootstrapping process is done, check the status of the GitRepository and Kustomization resource in namespace ${namespace} for reconcilation updates"

```
