## User Authentication using OpenID Connect

--- 

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-060
```

---

In this chapter, we will demonstrate how to secure user identification using the [OpenID Connect](https://openid.net/connect/) protocol. We will not perform user authorization here, meaning we will not yet control user access to individual resources. We will only ensure that all users accessing the cluster must identify themselves so that we can uniquely determine their identity. We will use the [GitHub](https://github.com/) platform as the identity provider, but in a similar way, we could use other identity providers such as Google, Microsoft, Facebook, etc. If we wanted to set up our own identity provider, we could integrate one of the [Identity Provider](https://en.wikipedia.org/wiki/Identity_provider) service implementations into our system. In the realm of smaller projects, for example, the implementation [dex](https://dexidp.io/) is popular, but there are [many other implementations and libraries](https://openid.net/developers/certified/) available.

For authentication purposes, we will use the [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) service, and we will gradually configure the [Envoy Gateway] to use this service to verify the identity associated with the incoming request.

1. An important aspect of the OIDC protocol is the assumption of using a standard browser resistant to various security attacks. The browser plays a crucial role in the [Open ID Connect](https://openid.net/developers/how-connect-works/) protocol and guides the user between various providers - web application, identity provider, protected resource provider. The protocol assumes the creation of a multi-party contract between individual entities. Therefore, it is necessary to use unique identifiers for entities in this environment, which does not apply to the `localhost` domain, which refers to any computing resource.

Therefore, we need to assign different identifiers to our local computer, known as a [_Fully Qualified Domain Name (FQDN)_](https://en.wikipedia.org/wiki/Fully_qualified_domain_name). In the chapter [Securing Application Connection with HTTPS](./040-secure-connection.md), we assigned the name `wac-hospital.loc` to the `wac-hospital-gateway` object. We will use this name to refer to our local cluster; however, we still need to create a domain name for our computer.

Find out the IP address assigned to your computer, for example, using the `ipconfig` or `ifconfig` command for Linux OS. Open the file `C:\Windows\System32\drivers\etc\hosts` (`/etc/hosts` on Linux systems) and create a new entry in it:


```plain
<vaša IP adresa>  wac-hospital.loc
```

Save the file - you will be prompted to switch to privileged mode, or you must open and edit this file with administrator privileges.

>warning:> The IP address assigned to your computer may change, so each time you need to verify which IP address is assigned to your computer and update the entry in this file. In some networks, individual devices, including workstations, have permanently assigned FQDNs. In these cases, you can use this designation and do not need to edit the `etc/hosts` file. However, using the `localhost` designation will not work in the next exercise.

2. To gain access to the user identity of the GitHub platform, we need to register our application on this platform. Users will later be prompted to grant permission to share their identity with our application. Sign in with your account to the GitHub platform and go to [https://github.com/settings/developers](https://github.com/settings/developers). Choose the option _Register a new application_. Fill out the form on the displayed page:

* _Application name_: `WAC hospital`
* _Homepage URL_: `https://wac-hospital.loc`
* _Application description_: `An application created for WAC exercises - <your name>`
* _Authorization callback URL_: `https://wac-hospital.loc/authn/callback`

The first three items will be presented to users when granting permission to share information. The last item is important in the OIDC protocol itself - users will be redirected to this URL after authenticating on the GitHub page, and the identity provider accepts only authentication requests that redirect the user to one of the registered _authorization callback_ URLs. This prevents a malicious page from pretending to be your application and gaining access to user data without their prior consent.

![GitHub Application Registration](./img/github-oauth-app.png)

After filling out the form, press the _Register Application_ button, and on the next page, press the _Generate a new client secret_ button. Note the client identifier - _Client ID_ and the displayed password - _Client Secret_. Finally, press the _Update application_ button.

>info:> Guides and links to configure the used service with other identity providers can be found [here](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider).

3. On the page [https://www.random.org/strings/](https://www.random.org/strings/), create a random string exactly 32 characters long. We will use this string as the _cookie secret_ for the [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) service. Create the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params/oidc-client.env` with the following content (you must use values specific to your individual configuration):

```env
client-id=<client id z kroku 2>
client-secret=<client secret z kroku 2>
cookie-secret=<náhodný reťazec>
```

Open a command prompt window and navigate to the directory `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/params`. Create a _Secret_ using the command:

```powershell
sops --encrypt --in-place oidc-client.env
```

Edit the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/kustomization.yaml` and add the creation of a new object of type [_Secret_](https://kubernetes.io/docs/concepts/configuration/secret/):

```yaml
...
secretGenerator:
  - name: repository-pat
    ...
  - name: mongodb-auth
    ...
  - name: oidc-client     @_add_@
    type: Opaque     @_add_@
    envs:     @_add_@
      - params/oidc-client.env     @_add_@
    options:     @_add_@
        disableNameSuffixHash: true     @_add_@
```

4. Create the configuration for the microservice [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/). Create the directory `${WAC_ROOT}/ambulance-gitops/infrastructure/oauth2-proxy` and in it the file `${WAC_ROOT}/ambulance-gitops/infrastructure/oauth2-proxy/deployment.yaml` with the content:

```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: &PODNAME oauth2-proxy
  spec:
    replicas: 1
    selector:
      matchLabels:
        pod: *PODNAME
    template:
      metadata: 
        labels:
          pod: *PODNAME
      spec:
        containers:
        - name: oauth2-proxy  
          image: bitnami/oauth2-proxy
          args: 
          - --upstream="static://200" @_important_@
          - --set-xauthrequest @_important_@
          - --set-authorization-header
          - --silence-ping-logging
          env:
          - name: OAUTH2_PROXY_HTTP_ADDRESS
            # listen on standard interface - localhost will not work
            value: ":4180"
  
          - name: OAUTH2_PROXY_PROXY_PREFIX
            # oauth2-proxy route listens on path /authn
            value: /authn @_important_@

          - name: OAUTH2_PROXY_PROVIDER
            value: github @_important_@

          - name: OAUTH2_PROXY_CLIENT_ID
            valueFrom:
              secretKeyRef:
                name: oidc-client  @_important_@
                key: client-id
  
          - name: OAUTH2_PROXY_CLIENT_SECRET
            valueFrom:
              secretKeyRef:
                name: oidc-client @_important_@
                key: client-secret
  
          - name: OAUTH2_PROXY_REDIRECT_URL
            # must match redirection registered at GitHub Oauth2 Application form
            value: https://wac-hospital.loc/authn/callback     @_important_@
  
          - name: OAUTH2_PROXY_COOKIE_SECRET
            valueFrom:
              secretKeyRef:
                name: oidc-client
                key: cookie-secret
  
          - name: OAUTH2_PROXY_SESSION_STORE_TYPE
            # alternatively use redis and configure it
            value: cookie

          - name: OAUTH2_PROXY_COOKIE_PREFIX
            value: __Secure-
          - name: OAUTH2_PROXY_COOKIE_SAMESITE
            value: lax
            
          - name: OAUTH2_PROXY_EMAIL_DOMAINS
            # only authenticate - we will authorize users later
            value: "*"

          - name: OAUTH2_PROXY_SKIP_PROVIDER_BUTTON
            # change to true to skip provider selection page. Here false for    demonstration only
            value: "false"

          - name: OAUTH2_PROXY_SKIP_AUTH_ROUTES
            # regex of routes where anonymous users are allowed
            # either here or create separate gateway/listener for anonymous users
            value: (\/.well-known\/|\/favicon.ico)
  
          resources:
            limits:
              cpu: '0.2'
              memory: '320M'
            requests:
              cpu: '0.01'
              memory: '128M'
          livenessProbe:
            httpGet:
              path: /ready
              scheme: HTTP
              port: 4180
            initialDelaySeconds: 5
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              scheme: HTTP
              port: 4180
            initialDelaySeconds: 5
            periodSeconds: 5
```

Note that we reference values from the _Secret_ `oidc-client` created in the previous step. In this configuration, we use the so-called [_Secure Cookie_](https://en.wikipedia.org/wiki/Secure_cookie) for session management, where session data and the token identifying the user are encrypted. An alternative would be to store this information in a [redis] store, which we omitted for simplification.

Notice the option `OAUTH2_PROXY_EMAIL_DOMAINS=*`. This setting allows any authenticated users to enter the system. If we were to change it, for example, to `OAUTH2_PROXY_EMAIL_DOMAINS=stuba.sk`, we would restrict access only to users who own email addresses managed in the `stuba.sk` domain, i.e., students and employees of STU Bratislava. The variable `OAUTH2_PROXY_AUTHENTICATED_EMAILS_FILE` would allow us to further limit access only to specific users. However, we will not use this option in the exercise, and in the next step, we will show how to authorize users based on user information provided by the [oauth2-proxy] service.

In the container arguments, we used several [important configuration settings](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview). The option `--upstream="static://200"` specifies that, in the case of successful authentication, this service will return a response with a status code of 200. The option `--set-xauthrequest` specifies that the service will use headers of type `X-Auth-Request` to transfer information about the authenticated user. We will use these headers later for user authorization.

5. Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/oauth2-proxy/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: oauth2-proxy
spec: 
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 4180
```

Further, create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/oauth2-proxy/http-route.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: oauth2-proxy
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /authn
      backendRefs:
        - group: ""
          kind: Service
          name: oauth2-proxy
          port: 80
```

The `HTTPRoute` object will be referenced in the next step when modifying the _Gateway_ configuration.

Create the file `${WAC_ROOT}/ambulance-gitops/infrastructure/oauth2-proxy/kustomization.yaml` to integrate these configuration files.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

commonLabels:
  app.kubernetes.io/part-of: wac-hospital
  app.kubernetes.io/component: oauth2-proxy

resources:
- deployment.yaml
- service.yaml
- http-route.yaml
```

and include `oauth-proxy` in the cluster configuration by modifying the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`

```yaml
...
resources:
...
- ../../../infrastructure/cert-manager
- ../../../infrastructure/oauth2-proxy  @_add_@
```

6. In order to utilize the above-configured service, we need to modify the configuration of [Envoy Gateway]. From a technical perspective, [Envoy Gateway] implements the [Kubernetes Controller](https://kubernetes.io/docs/concepts/architecture/controller/) design pattern - it watches for changes in registered objects of the [Gateway API](https://gateway.envoyproxy.io/latest/docs/reference/api/), and subsequently creates a new instance of the [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/) service, whose configuration is determined based on the registered resources. [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/) is a highly efficient implementation of a reverse proxy that allows configuring the details of request processing and routing to the cluster (here in a general sense, not limited to just a Kubernetes cluster). One of the concepts of [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/) is the so-called [HTTP filter](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters), which allows adjusting the parameters of HTTP request processing. In our case, we will be using the [_External Authorization_](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) filter, which hands over control to our `oauth2-proxy` service, and only if this service returns a positive status code (less than 400), it allows further processing of the request, otherwise, it returns an error status `403 Forbidden`.

[Envoy Gateway](https://gateway.envoyproxy.io/latest/docs/reference/api/) allows extending the configuration of dynamically created instances of [envoy proxy](https://www.envoyproxy.io/docs/envoy/latest/) using the [_Envoy Patch Policy_](https://gateway.envoyproxy.io/latest/docs/reference/api/envoy.solo.io.v1.EnvoyPatchPolicy/) object. There is also a tool called [_egctl_](https://gateway.envoyproxy.io/latest/docs/reference/api/envoy.solo.io.v1.EnvoyPatchPolicy/), which allows obtaining the current configuration created based on the registered resources. We will use this type of object to add the [_External Authorization_](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter) configuration to the [Envoy Gateway](https://gateway.envoyproxy.io/latest/docs/reference/api/) configuration.

>info:> This is an advanced technique that assumes knowledge of [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/) configuration. Since it is likely that you will encounter [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/) when working with more complex systems in practice, we recommend familiarizing yourself with its details. More detailed information can be found in the [documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/intro).

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/envoy-patch-policy.yaml` with the following content

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata: 
    name: oauth2-ext-authz
    namespace: wac-hospital
spec:
    targetRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: wac-hospital-gateway
      namespace: wac-hospital
    type: JSONPatch
    jsonPatches:
      - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
        # The listener name is of the form <GatewayNamespace>/<GatewayName>/<GatewayListenerName>
        name:  wac-hospital/wac-hospital-gateway/fqdn  @_important_@
        operation:
          op: add
          # if there is only single listener per tls endpoint then replace "/filter_chains/0" 
          # with "/default_filter_chain" 
          # use config `egctl config envoy-proxy listener -A` to find out actual xDS configuration
          path: "/filter_chains/0/filters/0/typed_config/http_filters/0"  @_important_@
          value:
            name: authentication.ext_authz
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
              http_service:
                server_uri:
                  uri: http://oauth2-proxy.wac-hospital @_important_@
                  timeout: 30s
                  # The cluster name is of the form <RouteType>/<RouteNamespace>/<RouteName>/rule/   <RuleIndex>
                  # use  `egctl config envoy-proxy cluster -A` to find out actual xDS configuration
                  cluster: httproute/wac-hospital/oauth2-proxy/rule/0  @_important_@
                authorizationRequest:
                  allowedHeaders:
                    patterns:
                    - exact: authorization
                    - exact: cookie
                authorizationResponse:
                  allowedUpstreamHeaders:
                    patterns:
                    - exact: authorization
                    - prefix: x-auth
```

This patch is applied to the _listener_ `wac-hospital/wac-hospital-gateway/fqdn`, which means that the OpenID authorization is triggered only when accessing pages at the address `https://wac-hospital.loc`. In the file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/gateway.yaml`, you can see how the _listener_ with the name `fqdn` is configured.

>info:> In addition to the [External Authorization](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter.html) filter, we could use the [OAuth2](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/oauth2_filter.html) filter. However, for the purposes of this exercise, the first one listed is more suitable.

&nbsp;

>warning:> [Envoy Gateway](https://gateway.envoyproxy.io/latest/docs/reference/api/) is still actively developed, and it is possible that in the future, the OpenID protocol configuration will be added as part of the basic configuration.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/params/envoy-gateway.yaml` with the following content

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyGateway
gateway:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
logging:
  level:
    default: info
provider:
  type: Kubernetes
extensionApis:
  enableEnvoyPatchPolicy: true @_important_@
```

This file modifies the configuration file [_EnvoyGateway_](https://gateway.envoyproxy.io/latest/api/extension_types/#envoygateway), specifically allowing the use of the [_Envoy Patch Policy_](https://gateway.envoyproxy.io/latest/user/envoy-patch-policy/) object.

Modify the file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/kustomization.yaml` and add a reference to the newly created configuration


```yaml
resources:
...
- gateway.yaml
- envoy-patch-policy.yaml @_add_@
  @_add_@
configMapGenerator:    @_add_@
  - name: envoy-gateway-config    @_add_@
    namespace: envoy-gateway-system    @_add_@
    behavior: replace    @_add_@
    files:    @_add_@
      - params/envoy-gateway.yaml    @_add_@
```

7. Save the changes and archive them in a remote repository:

```ps
git add .
git commit -m "Add oauth2-proxy"
git push
```

Verify that the latest changes are applied in your cluster

```ps
kubectl -n wac-hospital get kustomization -w
```

Verify that the status of the _Envoy Patch Policy_ object is `Programmed`
>info:> If the status does not change to Programmed, it is necessary to restart the Envoy Gateway pods.

```ps
kubectl -n wac-hospital get epp
```

8. Open a new tab in your browser and go to _Developer Tools -> Network_. In this tab, navigate to the page [https://wac-hospital.loc](https://wac-hospital.loc) and choose the option _Persist Logs_ (or _Keep Log_). Remember that in the `etc/hosts` file, you must have the correct IP address assigned to the `wac-hospital.loc` record. The browser will alert you to a security risk due to the use of an untrusted TLS certificate. Choose _Continue_ and _Understand the security risk_.

>build_circle:> In some cases, the _Continue_ option may be unavailable. In such a case, leave the browser window as the active application and type `THISISUNSAFE` on the keyboard. This option (back-doors) is left in browsers like Google for well-informed professionals, such as software engineers.

On the screen, you will see the _OAuth2 Proxy_ login page (our configured service) with the option _Sign in with GitHub_. Press this option.

![OAuth2 Proxy Login Page](./img/oauth2-sign-in.png)

>info:> By configuring the OAuth2 Proxy service, you can modify the login page or skip this step and be redirected directly to GitHub pages.

Subsequently, you will be redirected to the GitHub page, where you will be prompted to grant permission to share your identity information with the _WAC Hospital_ application. Grant consent, after which you will be redirected to the application in your cluster.

Review the network communication log in the _Developer Tools_. You can see how the browser is redirected multiple times between different entities of the OIDC protocol. Part of the protocol takes place in the background between _OAuth2 Proxy_ and the GitHub identity provider.

_OAuth2 Proxy_ will now remember your login for the next 168 hours (cookie validity), and GitHub platform remembers the permission granted to your application. Therefore, upon reloading, you will be automatically redirected to the application pages, and only after a longer period of inactivity, you will be prompted to log in again. Alternatively, you can try logging in from a new private browser window that does not share your identity (cookies, etc.) with other browser windows.

9. The microservice _oauth2-proxy_ provides the identity of the logged-in user in the headers of forwarded requests. To verify this, we will add a simple service [http-echo](https://github.com/mendhak/docker-http-https-echo) to our cluster. Create a file `${WAC_ROOT}/ambulance-gitops/apps/http-echo/deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:  
  name: http-echo
spec:
  replicas: 1  
  selector:
    matchLabels:
      pod: http-echo
  template:
    metadata:
      labels: 
        pod: http-echo 
    spec:
      containers:
      - image: mendhak/http-https-echo
        name: http-echo        
        ports:
        - name: http
          containerPort: 8080
        resources:
          limits:
            cpu: '0.1'
            memory: '128M'
          requests:
            cpu: '0.01'
            memory: '16M'
```

Create file `${WAC_ROOT}/ambulance-gitops/apps/http-echo/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: http-echo
spec: 
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
```

Create file `${WAC_ROOT}/ambulance-gitops/apps/http-echo/http-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-echo
spec:
  parentRefs:
    - name: wac-hospital-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /http-echo
      backendRefs:
        - group: ""
          kind: Service
          name: http-echo
          port: 80
```

and file `${WAC_ROOT}/ambulance-gitops/apps/http-echo/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: 
- deployment.yaml
- service.yaml
- http-route.yaml

namespace: wac-hospital

commonLabels: 
  app.kubernetes.io/component: http-echo
```

Finally, in the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml`, add a reference to this service:

```yaml
...
resources: 
...
- ../../../apps/http-echo @_add_@
...
```

Save the changes and archive them in a remote repository:

```ps
git add .
git commit -m "Add http-echo"
git push
```

Verify that the latest changes are applied in your cluster

```ps
kubectl -n wac-hospital get kustomization -w
```

Go to the page [https://wac-hospital.loc/http-echo](https://wac-hospital.loc/http-echo) and inspect the generated JSON file. In the `headers` section, pay attention to the headers `x-auth-request-email` and `x-auth-request-user`, which were added to the request by the `oauth2-proxy` service.

> We recommend installing a browser extension for displaying JSON files, which is a useful tool in web application development. An example of such an extension for the Chrome browser is [JSONFormatter](https://github.com/callumlocke/json-formatter).

Our application is now capable of identifying users and, to some extent, controlling who can access our pages.
