# Service Mesh Administrator - Linkerd

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-130
```

---

In the previous sections, we demonstrated how to secure our system composed of multiple microservices and ensure request routing and system activity monitoring. Each of these features adds its own set of microservices. Therefore, it is appropriate to understand our system as a structure of interconnected (micro-)services, where each of these services represents a building block of the system, independently versioned and deployable in various systems and environments. Such a system of interconnected services is also called a [_Service Mesh_](https://en.wikipedia.org/wiki/Service_mesh).

In this section, we will show how to deploy additional tools for managing the _Service Mesh_ into our cluster. These tools are referred to by the same name - _Service Mesh_. In practice, it is good to explicitly distinguish whether you mean the software architectural pattern by the term _Service Mesh_ or a specific tool for managing systems. In our case, we will use the tool [Linkerd], which is one of the most commonly used services for managing [_Service Mesh_](https://en.wikipedia.org/wiki/Service_mesh) in Kubernetes clusters. Other popular tools include [Istio](https://istio.io/latest/) or [Open Service Mesh](https://openservicemesh.io/), as well as others that you can find on the [CNCF Landscape](https://landscape.cncf.io/card-mode?category=service-mesh&grouping=category) page.

_Service Mesh_ tools provide various complementary services for microservices-based systems, and the specific list depends on the tool used. Common features include request flow control between services, fault tolerance, automatic communication security, access control management, and system monitoring. In this exercise, we will demonstrate how to deploy the [Linkerd] tool into our cluster and use it to enhance system reliability and secure communication between services. Linkerd will automatically provide us with communication monitoring at the data flow level between individual services, adding further information to our distributed tracing from previous sections.

When installing [Linkerd], we will follow the guide on the [_Installing Linkerd with Helm_](https://linkerd.io/2.14/tasks/install-helm/) page.

1. Create a folder `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd` and within it, create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/helm-repository.yaml` with the content

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: linkerd
  namespace: linkerd
spec:
  interval: 1m
  url: https://helm.linkerd.io/stable
```

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/crds.helm-release.yaml` with the content

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: linkerd-crds
  namespace: linkerd
spec:
  interval: 1m
  chart:
    spec:
      chart: linkerd-crds
      sourceRef:
        kind: HelmRepository
        name: linkerd
        namespace: linkerd
      interval: 1m
      reconcileStrategy: Revision
  values:
    # we already have Gateway API subsystem installed
    enableHttpRoutes: false
```

This file will provide the [_Custom Resource Definition_](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) for the [Linkerd] service. These definitions extend the basic functionality of Kubernetes with new objects, thus exposing the functionality of the [Linkerd] tool through declarative configuration.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/control-plane.helm-release.yaml` with the content

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: linkerd-control-plane
  namespace: linkerd
spec:
  interval: 1m
  dependsOn:  @_important_@
  - name: linkerd-crds @_important_@
  chart:
    spec:
      chart: linkerd-control-plane
      sourceRef:
        kind: HelmRepository
        name: linkerd
        namespace: linkerd
      interval: 1m
      reconcileStrategy: Revision
  values:
    prometheusUrl: http://prometheus-server.wac-hospital
  valuesFrom:
    valuesFrom:
    # identity trust anchor certificate (shared accross clusters)
    - kind: Secret
      name: linkerd-trust-anchor
      valuesKey: tls.crt
      targetPath: identityTrustAnchorsPEM
```

During the installation of the [Linkerd] service, it is necessary to specify certificates from [_certificate authorities_](https://en.wikipedia.org/wiki/Certificate_authority) that will be used to verify the authenticity of certificates issued by [Linkerd] services. These are specified here as installation parameters.

Add the installation of `jaeger-injector` to the file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/jaeger-injector.helm-release.yaml` with the content

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: linkerd-jaeger-injector
  namespace: linkerd
spec:
  interval: 1m
  dependsOn:  
  - name: linkerd-control-plane   @_important_@
  chart:
    spec:
      chart: linkerd-jaeger
      sourceRef:
        kind: HelmRepository
        name: linkerd
        namespace: linkerd
      interval: 1m
      reconcileStrategy: Revision
  values:
    # we already have Jaeger collector, installing only jaeger injector webhook
    # to ensure linkerd proxies are configured to send traces to existing collector
    collector:
      enabled: false   @_important_@
    jaeger:
      enabled: false   @_important_@
    webhook:
      collectorSvcAddr: "jaeger-collector.wac-hospital:14268"
      collectorSvcName: jaeger-collector.wac-hospital
```

With this configuration, we ensure that the [Linkerd] proxies are configured to send the computation scope - _span_ - to the [Jaeger] service we installed in the previous section.

>info:> In the exercise, we could have proceeded by adding the installation of [Linkerd] to the cluster and leveraging provided features like [Gateway API], metrics, and distributed tracing. However, for didactic reasons, we chose to incrementally add individual features, explaining their contributions to the overall system and understanding their functionality as separate entities. In practice, it is possible to proceed in the opposite way and deploy primarily one of the _Service Mesh_ tools, which provides all these features as part of the installation.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/namespace.yaml` with the content

```yaml
ApiVersion: v1
kind: Namespace
metadata:
name: linkerd
```

and integrate the manifests in the file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization    

namespace: linkerd    

resources:
- namespace.yaml
- helm-repository.yaml
- crds.helm-release.yaml
- control-plane.helm-release.yaml
- jaeger-injector.helm-release.yaml
```

2. Before deploying [Linkerd] into the cluster, we need to create the certificates for the `linkerd-trust-anchor` object. This certificate from the [_certificate authority_](https://en.wikipedia.org/wiki/Certificate_authority) will be used to verify the authenticity of identities in communication between different clusters. In a production environment, such an intermediary CA certificate would be obtained from your organization's cybersecurity department. Here, we will generate our own certificate pair for our development server. Install the _step CLI_ tool following the instructions on the [smallstep - Install step](https://smallstep.com/docs/step-cli/installation/#windows) page. For example, open a command prompt in a directory outside your repository and execute the commands

```ps
curl.exe -LO https://dl.smallstep.com/cli/docs-cli-install/latest/step_windows_amd64.zip
Expand-Archive -LiteralPath .\step_windows_amd64.zip -DestinationPath .
step_windows_amd64\bin\step.exe version
$env:PATH += ";$pwd\step_windows_amd64\bin"
```

>info:> Instructions for installing the _step CLI_ tool on other platforms can be found on the [smallstep - Install step](https://smallstep.com/docs/step-cli/installation/) page.

In the command prompt window, navigate to the `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params` directory and generate the certificate for the global [_certificate authority_](https://en.wikipedia.org/wiki/Certificate_authority) using the following command.

```ps
step certificate create root.linkerd.cluster.local linkerd-ca.crt linkerd-ca.key --profile root-ca --no-password --insecure
```

If you are using [_SecretsOps for Managing Credentials_](./050-secrets-ops.md), encrypt the created certificates with the following commands.

```ps
sops --encrypt --in-place ./linkerd-ca.crt
sops --encrypt --in-place ./linkerd-ca.key
```

If you are not using _SecretOps_, ensure that the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params/linkerd-issuer.key` is not archived into the git repository - modify the `.gitignore` file.

Edit the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/kustomization.yaml` and add the following

```yaml
...   
secretGenerator:
  ...    
  - name: linkerd-trust-anchor    @_add_@
    type: kubernetes.io/tls    @_add_@
    options:    @_add_@
        disableNameSuffixHash: true    @_add_@
    files:    @_add_@
    - tls.crt=params/linkerd-ca.crt    @_add_@
    - tls.key=params/linkerd-ca.key    @_add_@
```

In addition to the [_certificate authority_](https://en.wikipedia.org/wiki/Certificate_authority) certificate, it is necessary to create a certificate for the local [_certificate authority_](https://en.wikipedia.org/wiki/Certificate_authority) for the [Linkerd] service, which will issue certificates for individual services (_linkerd-proxy_) within our cluster. In this case, we will use the already installed [cert-manager] service, which will ensure the creation of a certificate for the intermediary [_certificate authority_](https://en.wikipedia.org/wiki/Certificate_authority) verified by the certificate stored in the `linkerd-trust-anchor` object. The advantage of using the [cert-manager] service is that it will automatically renew the certificate in case of expiration, and its validity can be short-lived, increasing the system's security.

>info:> It would be possible to create the `linkerd-trust-anchor` object using the [cert-manager] service, similar to what we did in the chapter [Secure Connection to the Application using HTTPS](./040-secure-connection.md). However, this would not reflect the fact that this certificate is intended for verifying identities between different clusters.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/control-plane.issuer.yaml` with the content

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  ca:
    secretName: linkerd-trust-anchor   @_important_@
```

and the file `${WAC_ROOT}/ambulance-gitops/infrastructure/linkerd/control-plane.certificate.yaml` with the content

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-identity-issuer 
spec:
  secretName: linkerd-identity-issuer @_important_@
  duration: 48h
  renewBefore: 25h
  issuerRef:
    name: linkerd-trust-anchor  @_important_@
    kind: Issuer
  commonName: identity.linkerd.cluster.local @_important_@
  dnsNames:
  - identity.linkerd.cluster.local
  isCA: true @_important_@
  privateKey:
    algorithm: ECDSA
  usages:
  - cert sign
  - crl sign
  - server auth
  - client auth
```

3. Edit the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml` and add the following

```yaml
...
resources:
...
- ../../../infrastructure/linkerd   @_add_@

patches: 
...
```

4. Save the modified files and commit the changes to the remote repository

```ps
git add .
git commit -m "Add linkerd"
git push
```

Wait for the changes to take effect in the cluster and then verify the installation correctness with the following commands

```ps
kubectl get helmreleases -n linkerd
kubectl get pods -n linkerd
```