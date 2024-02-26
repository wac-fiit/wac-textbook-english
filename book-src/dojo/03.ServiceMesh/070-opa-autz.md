# User Authorization with Open Policy Agent

---

```ps
devcontainer templates apply -t registry-1.docker.io/milung/wac-mesh-070
```

---

User Authentication alone does not address access control to individual resources in our application. In practice, we may, for example, require that only users with the role `hospital-supervisor` or `general-practitioner` have access to individual clinics - a concept known as [Role-Based Access Control](https://en.wikipedia.org/wiki/Role-based_access_control), or that only patients with the symptom `pregnant-women` can sign in to the waiting room on a certain day - known as [Attribute-Based Access Control](https://en.wikipedia.org/wiki/Attribute-based_access_control).

To manage access policies, not only within user authorization but also to define general policy rules, we will use [Open Policy Agent](https://www.openpolicyagent.org/). In our case, we will attempt to set a policy that denies access to the `http-echo` microservice for all users except those who have the "admin" role assigned by our access policy.

>info:> The [OPA Envoy Plugin](https://www.openpolicyagent.org/docs/latest/envoy-introduction/) can be used as a side-car for a specific service, meaning control of policy compliance just before accessing the specific microservice. This enhances security by preventing malicious actors from bypassing our authorization. Later, we will show how to prevent unwanted communication within our service mesh. The specific solution and system configuration will vary in practice depending on the specific requirements of the solution; often, it will be a combination of various policies and access control mechanisms.

1. Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/deployment.yaml` with the following content:

```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: &PODNAME opa-plugin
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
        volumes:   
        - name: opa-policy 
          configMap:
            name: opa-policy
        - name: opa-config
          configMap:
            name: opa-config
        containers:
        - name: *PODNAME
          image: openpolicyagent/opa:latest-envoy  @_important_@
          securityContext:
            runAsUser: 1111
          volumeMounts:
            - readOnly: true
              mountPath: /policy  @_important_@
              name: opa-policy
            - readOnly: true
              mountPath: /config
              name: opa-config @_important_@
          args:
            - "run"
            - "--server"
            - "--config-file=/config/config.yaml"  @_important_@
            - "--addr=localhost:8181"
            - "--diagnostic-addr=0.0.0.0:8282"
            - "--ignore=.*"
            - "/policy/policy.rego"  @_important_@
          ports: 
            - containerPort: 8181
              name: opa-rest
            - containerPort: 8282
              name: opa-diag
            - containerPort: 9191
              name: envoy-plugin   
          resources:
            limits:
              cpu: '0.5'
              memory: '320M'
            requests:
              cpu: '0.01'
              memory: '128M'
          livenessProbe:
            httpGet:
              path: /health?plugins
              scheme: HTTP
              port: 8282
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health?plugins
              scheme: HTTP
              port: 8282
            initialDelaySeconds: 5
            periodSeconds: 5  
```

Notice the `volumes` and `volumeMounts` entries, which will be assigning configuration maps to the file system.

Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/service.yaml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: opa-plugin
spec: 
  ports:
  - name: http
    port: 9191
    targetPort: 9191
```

2. Now, let's proceed with the configuration of the `opa-envoy-plugin` service. Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/params/opa-config.yaml`. Note that we assigned port `9191` for the `envoy_ext_authz_grpc` plugin, and we set the path to the authorization policy evaluation rule.

```yaml
plugins:
  envoy_ext_authz_grpc:
    addr: :9191
    path: wac/authz/result @_important_@
decision_logs:
  console: true  # v produkčnom prostredí nastavte na false
```

Notice that we assigned port `9191` for the `envoy_ext_authz_grpc` plugin - the same port as specified in the [_Service_](https://kubernetes.io/docs/concepts/services-networking/service/) object. Additionally, we set the path to the authorization policy evaluation rule - `wac/authz/result`.

3. The next step is to define the access policy for our system. Since we are working on a local cluster with a relatively simple application, our policy will serve as a demonstration for testing purposes. For this demonstration, we'll use the `http-echo` application and restrict access to only some of our colleagues or requests with a specific flag. Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/params/policy.rego`, and using the [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) language, we'll define our test policy. In the first step, we will create rules to retrieve information about the user who logged into the system.

>info:> For advanced Rego language work, you can use the [Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=tsandall.opa).


```rego
package wac.authz
import input.attributes.request.http as http_request   

default allow = false   

# define authenticated user
is_valid_user = true { http_request.headers["x-auth-request-email"] }   

user = { "valid": valid, "email": email, "name": name} {
    valid := is_valid_user
    email := http_request.headers["x-auth-request-email"]
    name := http_request.headers["x-auth-request-user"] 
}
```

The above code creates a rule `is_valid_user`, which is satisfied under the condition that the incoming request contains a header named `x-auth-request-email`. The `user` rule creates an object containing information about the user who logged into the system. Next, at the end of the same file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/params/policy.rego`, add the definition of rules for required permissions (roles) for specific requests:

```rego
# define required roles for paths
# admin role is allowed for any path
request_allowed_role["admin"] := true 

# /monitoring path requires monitoring role
request_allowed_role["monitoring"] := true {
    glob.match("/monitoring*", [], http_request.path)
}

# user may access anything except /monitoring and /http-echo
# !!! DEMONSTRATION ONLY: this is not a good idea, because user 
# !!! may access any path  that is not explicitely defined in request_allowed_role
# !!! in production use oposite logic: define white-listed paths
request_allowed_role["user"] := true {
  not glob.match("/monitoring*", [], http_request.path) 
  not glob.match("/http-echo*", [], http_request.path)
}
```

The first rule `request_allowed_role["admin"]` specifies that any request is allowed with the `admin` role. The next rule `request_allowed_role["monitoring"]` indicates that for access to paths starting with `/monitoring`, the `monitoring` role is required. The last rule `request_allowed_role["user"]` determines that for access to any path that is neither `/monitoring` nor `/http-echo`, the `user` role is needed.

>info:> In practice, we should define rules for explicitly allowed paths rather than for all others. Also, the rule file is highly simplified, it does not adhere to the [_Principle of Least Privilege_](https://en.wikipedia.org/wiki/Principle_of_least_privilege) and the granularity of access rights.

At the end of the same file, add rules for assigning roles

```rego
# define roles for user

# any user with valid email is user
user_role["user"] { 
    user.valid
}

# !!! DEMONSTRATION ONLY: backdoor for admin role @_important_@
user_role[ "admin" ] { 
    [_, query] := split(http_request.path, "?")
    glob.match("am-i-admin=yes", [], query)
}

# this are admin users
user_role[ "admin" ] { 
    user.email == "\<kolegov@email\>"
}

# this are users with access to monitoring actions
user_role[ "monitoring" ] { 
    user.email == "\<your github account\>" @_important_@
}
```

The first rule specifies that every logged-in user has the `user` role. The next two rules assign the `admin` role for requests with an explicitly defined parameter `am-i-admin=yes` in the [_Query_ parameters](https://en.wikipedia.org/wiki/Query_string) or for a user with the email of your colleague - this is for demonstration purposes in the following sections.

Further, at the end of the file, add rules for evaluating access rights:

```rego
# action is allowed if there is some role that is in user roles
# and path roles simultanously
action_allowed {
    some role 
    request_allowed_role[role]
    user_role[role]
}

# allow access if user is authenticated and action is allowed
allow {
    user.valid
    action_allowed
}
```

The rule `action_allowed` specifies that a given action is allowed if there exists at least one (`some`) role (`role`) that is in the access rights for the given path (`request_allowed_role[role]`), and at the same time, this role is assigned to the user (`user_role[role]`). The rule `allow` then determines that if the user is authenticated and the action is allowed, access to our system is granted.

Finally, add rules for creating a response for the [_External Authorization Filter_](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext_authz/v3/ext_authz.proto) in [Envoy Proxy]:

```rego
# set header to indicate that this policy was used to validate the request
headers["x-validated-by"] := "opa-checkpoint"

headers["x-auth-request-roles"] := concat(", ", [ role | 
    some r
    user_role[r] 
    role := r
])

# provide result to caller
result["allowed"] := allow
result["headers"] := headers
```

As you may have noticed, in full-stack development, a software engineer needs to be proficient in various programming languages. [Rego] is a logical and rule-based programming language based on [datalog](https://datalog.sourceforge.net/). In the above program, there are sets of rules used to reach the result of the `allow` rule - either `true` or `false`. The resulting value determines whether to allow further processing of the request. Additionally, we aim to create response headers containing information about the user and their permissions. In practice, rules like `request_allowed_role` and `user_role` are defined in external repositories (databases) and dynamically loaded into the service - for details, refer to the [guide](https://www.openpolicyagent.org/docs/latest/external-data/).

4. Create a file `${WAC_ROOT}/ambulance-gitops/infrastructure/opa-plugin/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital

commonLabels:
  app.kubernetes.io/part-of: wac-hospital
  app.kubernetes.io/component: opa-plugin

resources:
- deployment.yaml
- service.yaml

configMapGenerator:
- name: opa-config
  files:
    - config.yaml=params/opa-config.yaml
- name: opa-policy
  files: 
  - params/policy.rego
```

and add a reference to the file `${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml`:

```yaml
...
resources: 
...
- ../../../infrastructure/opa-plugin @_add_@

...
```

5. Similarly to the authentication with `oauth2-proxy`, we will add authorization among the list of filters in [Envoy Proxy]. Modify the file `${WAC_ROOT}/ambulance-gitops/infrastructure/envoy-gateway/envoy-patch-policy.yaml`:

```yaml
...
spec:
  ...
  jsonPatches:
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
    ...
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"             @_add_@
      name:  wac-hospital/wac-hospital-gateway/fqdn             @_add_@
      operation:             @_add_@
        op: add             @_add_@
        path: "/filter_chains/0/filters/0/typed_config/http_filters/1"             @_add_@
        value:             @_add_@
          name: authorization.ext_authz             @_add_@
          typed_config:             @_add_@
            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz             @_add_@
            transport_api_version: V3             @_add_@
            grpc_service:             @_add_@
              google_grpc:             @_add_@
                stat_prefix: opa             @_add_@
                target_uri: opa-plugin.wac-hospital:9191             @_add_@
              timeout: 3s             @_add_@
```

6. Save the changes and archive them in the remote repository:

```ps
git add .
git commit -m "Add open policy agent"
git push
```

Verify that the latest changes are applied in your cluster.

```ps
kubectl -n wac-hospital get kustomization -w
```

  Verify that the status of the _Envoy Patch Policy_ object is `Programmed`.

>info:> In case the status does not change to `Programmed`, it is necessary to restart the Envoy Gateway pods.

```ps
kubectl -n wac-hospital get epp
```

7. In the browser, go to the page [https://wac-hospital.loc](https://wac-hospital.loc), which should be displayed without any changes. Now go to the page [https://wac-hospital.loc/http-echo](https://wac-hospital.loc/http-echo) - the page is empty, and in the _Developer Tools -> Network_, you can see the response `403 Unauthorized`. Go to the page [https://wac-hospital.loc/http-echo?am-i-admin=yes](https://wac-hospital.loc/http-echo?am-i-admin=yes). In this case, you should see the response from the `http-echo` microservice. Similarly, you can open a private browser window and ask a colleague whose email you specified in the access policy to log in to the page [https://wac-hospital.loc/http-echo](https://wac-hospital.loc/http-echo). In this case, they should also see the response from the `http-echo` service.

Examine the response from the `http-echo` service. Among the headers of the request incoming to the `http-echo` service, you should find the following:

```json
...
"x-auth-request-email": "<your github email>",
"x-auth-request-user": "<your github user id>",
"x-auth-request-roles": "admin, monitoring, user",
...
```

We can further utilize the generated headers either for more detailed request routing in objects like [_HTTPRoute_](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1alpha2.HTTPRoute), or we can deploy the [OPA Envoy Plugin](https://www.openpolicyagent.org/docs/latest/envoy-introduction/) as a sidecar to one of our services and continue to control access policies within our service mesh. For dynamic management of assigned roles, we would need to expand our cluster with a service that the [OPA Envoy Plugin](https://www.openpolicyagent.org/docs/latest/envoy-introduction/) could use to dynamically load access policies.
