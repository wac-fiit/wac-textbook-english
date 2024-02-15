## Infrastructure Configuration Archiving

---

>info:>
Template for pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-070`

---

A prerequisite for [GitOps] and a requirement for [Flux] to work is that the deployment manifest of our system is stored in a Git repository. In the previous section, we created a directory called _webcloud-gitops_, which contains the deployment description of our project. Now, let's archive it.

1. On the [GitHub] page, log in to your account, and create a new repository with the name `ambulance-gitops`. Uncheck _Add a README_ in the menu and select _None_ for the _.gitignore_ file, then press `Create`.

![GitOps Repository](./img/070-01-GitOpsRepo.png)

2. In the command line, navigate to the `${WAC_ROOT}/ambulance-gitops` folder and initialize a local Git repository with the command:

```ps
git init
```

3. Add and submit all local files to the archive

```ps
git add .
git commit -m 'initial version of gitops folder'
```

4. Connect the local repository to the [GitHub] repository.
>info:> You can use the command generated on your project's GitHub page.

```ps
git remote add origin https://github.com/<your-account>/ambulance-gitops.git
```

_origin_ is the name we assigned to the remote repository.

5. Synchronize your local repository with the remote repository. When prompted, enter your login credentials.

```ps
git push --set-upstream origin main
```

In your browser, check that your files are stored in the remote repository.

![Remote repository gitops](./img/070-02-RepoContent.png)

