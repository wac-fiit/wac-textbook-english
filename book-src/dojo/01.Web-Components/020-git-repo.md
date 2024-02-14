## Repository and Code Archiving

---

>info:>
Template for the pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-020`

---

In this chapter, we will create a new repository and archive the code. We will use the services provided on [GitHub]. It is recommended to create the repository as public, but it's not a requirement.

1. On the [GitHub] page, log in to your account, and in the top panel, click the "+" button and choose _New Repository_.

![Create a new repository](./img/020-01-NewRepo.png)

2. In the displayed window, choose the repository name `ambulance-ufe`, and leave the other options in their default state. Check that all options in the _Initialize this repository with:_ section are empty. Then click the _Create repository_ button.

![Create a new repository](./img/020-02-CreateRepo.png)

After creating the repository, you will see a page with instructions to create a local repository and synchronize it with the remote repository.
In our case, we will use the commands shown in the __...or push an existing repository from the command line__ section.

3. If you chose to create a private repository, on the displayed page, choose the _Invite Collaborators_ option and add the exercise teachers as collaborators to give them access to the repository. This step allows them to analyze any issues in your code.

4. In VS Code, navigate to the `${WAC_ROOT}/ambulance-ufe` folder and initialize the local Git repository using the commands:

```ps
git config --global init.defaultBranch main
git init
```

>info:> With the first command, we changed the main branch's name to `main`.

5. Open the file `${WAC_ROOT}/ambulance-ufe/.gitignore` and make sure it contains lines with the entries `node_modules/`, `dist/`, `www/`, `loader/`. This file specifies which files and subdirectories should not be archived, which in most cases includes files generated during the compilation of source files and packages that can be obtained automatically from available sources and other archives.

```ps
dist/ @_important_@
www/ @_important_@
loader/  @_important_@
...
node_modules/ @_important_@
...
```

6. Add and commit all local files, and then push them to the repository.

```ps
git add .
git commit -m 'initial version of ambulance waiting list web component'
```

7. Link your local repository to the GitHub repository. In the following command, replace `<account>` with your GitHub username.

>info:> You can use the command generated on your repository page on [GitHub].

```ps
git remote add origin https://github.com/<account>/ambulance-ufe.git
```

_origin_ is the name we assigned to the remote repository.

8. Synchronize your local repository with the remote repository. When prompted, enter your login credentials.

>info:> You can use the command generated on your project page on [GitHub].

```ps
git push --set-upstream origin main
```

Check in your browser that your files are stored in the remote repository.

![Synchronized Repository](./img/020-03-GitRepository.png)

During the exercises, we will use a simplified development process and work directly on the `main` branch of the repository. However, when working in a team, it is recommended to use the [_Fork and Pull Requests_](https://gist.github.com/Chaser324/ce0505fbed06b947d962) development workflow.

You can create a Git repository on other servers as well, such as popular platforms like [Azure DevOps][azure-devops], [GitLab][gitlab], or [Bitbucket][bitbucket]. The criteria for choosing a platform include support for automated continuous integration and deployment, professional team support, and easy resource management for the development team itself. In the context of this tutorial, we will work with services provided on [GitHub].
