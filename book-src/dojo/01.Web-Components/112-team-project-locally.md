# Deployment of the team project on the local cluster

>info:> This chapter is just a summary of the procedure for student work and will not be covered in the exercises.

As part of the semester project, it needs to be deployed on the shared cluster. The process is essentially similar to the deployment of the _<pfx>-ambulance-ufe_ application.

1. Create a separate repository for the web component of your application, later also for the web API of your application, and implement these services as shown in the exercises. Prepare a new release of the application.

2. Create a new repository for the deployment - gitops - of your application. If you are working individually, you can use the `ambulance-gitops` repository created in the exercises and just expand it - add configurations for your application in the respective directories `${WAC_ROOT}/ambulance-gitops/apps`, `${WAC_ROOT}/ambulance-gitops/clusters/localhost/install`, and `${WAC_ROOT}/ambulance-gitops/clusters/wac-aks/install`. In the case of team collaboration, we recommend creating a new repository with shared access for team members.

3. Similarly to the process described in the chapter [Deployment of the application on a production Kubernetes cluster](./111-production-deployment), deploy your project to the shared cluster.

>warning:> The shared cluster has strict rules in the [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) header. If you want to use external resources - for example, a library available from CDN - the browser will block access to such an address. Therefore, all resources must be part of your code.
