# Exercise: Web Application using Stencil JS

## <a name="goal"></a>Exercise Goal

In this exercise, you will learn to create a simple application based on the [WebComponents][webc] technology. You will learn how to encapsulate this application into a standalone software container and how to deploy it to an existing web page running on the Kubernetes platform. You will also learn to use the [Material Design Web Components][md-webc] library to style the visual appearance of the application and create an API specification, generating a client for this specification using the [OpenAPI][openapi] tool.

## Used Technologies

The application:

* is implemented in [TypeScript][typescript] using the [Stencil JS][stencil] application library in the form of [web components][webc],
* is created using the micro Front End technique and integrated into an existing web application interface through dynamically loaded web components,
* is deployed as a set of software containers in the [Kubernetes][kubernetes] system,
* uses the [Material Design Web Components][md-webc] library for styling the visual appearance of the application.

When choosing technologies, we aimed to use those that are currently used in practice and meet the goals of the exercise. For most technologies, there are alternative options, and if interested, you can use them instead of those used in this exercise. The condition is that the resulting application is created as a [web component][webc] and deployed as a software container in the [Kubernetes][kubernetes] system. Especially if you decide to use a technology other than [Stencil JS][stencil] to create a web component, the resulting implementation must use the [Shadow DOM][shadow-dom] and [Custom Elements][custom-elements] specifications.

In the case of the chosen technologies, the question often arises, why we chose [Stencil JS][stencil] instead of other alternatives such as [Angular][angular], [React][react], or [Vue][vue]. The reason is that [Stencil JS][stencil] is a library primarily focused on creating web components and is based on the [WebComponents][webc] standards, while in other alternatives, the development of web components is just an "added option." Therefore, although it is possible to use other libraries, in the case of [Stencil JS][stencil], the advantage is that it is a relatively simple overlay on the [WebComponents][webc] technology, allowing us to focus more on other aspects of full-stack development while fully mastering the [WebComponents][webc] technology.

## <a name="assignment"></a>Exercise Assignment

Create a web interface for managing the waiting room of a clinic. Basic functionality:

Two types of users will use the application: Patient and Clinic Nurse

* Patient
  * As a patient, I want to come to the clinic, enter my ID number (or patient number), and join the queue. I want the system to inform me of my position and the approximate time when I will be examined.
  * As a patient with an acute illness (injury, high temperature) or with a claim to priority treatment (e.g., pregnancy), I want to enter my ID number upon arrival at the waiting room and join the list of priority patients.
  * As a patient waiting in the clinic, I want to have a visual overview of the current status of my queue.
* Clinic Nurse
  * As a clinic nurse, I want an overview of the number and identity of waiting patients and the next patient in line.
  * As a clinic nurse, I want to know how many patients are waiting for a medical examination, how many are waiting for priority treatment, and how many are waiting for administrative processing.
  * As a clinic nurse, in the case of assessing a patient's serious condition, I want the option to change the order of waiting patients.

## Technical Limitations:

* The application is implemented in the TypeScript programming language using the [Stencil JS][stencil] application library in the form of [web components][webc].
* The application is created using the micro Front End technique and integrated into an existing web application interface through dynamically loaded web components.
* The application is deployed as a set of software containers in the [Kubernetes][kubernetes] system.
* The application uses the [Material Design Web Components][md-webc] library for styling the visual appearance of the application.

## <a name="preparation"></a>Preparation for Exercise

* Installation of [Node JS][nodejs], latest version
  * We recommend checking the **nvm** tool for managing NodeJS versions (in practice, you may encounter projects requiring different versions).
* Installation of [Visual Studio Code][vscode]
* Installed extensions in Visual Studio Code:
  * [ESLint (dbaeumer.vscode-eslint)](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
* Installation of [Git][git]
* Created an account on [GitHub]
* Created an account on [Microsoft Azure Cloud](https://azure.microsoft.com/en-us/free/students/)
* Preliminary familiarization with the [TypeScript][typescript] language and the [Stencil JS][stencil] library.
* Installed [Docker Desktop][docker-desktop] with the enabled [kubernetes][kubernetes] subsystem, or a functional installation of the docker package and minikube on the Linux system.
* Created an account on [Docker Hub][docker-hub]

## Working Environment

In the exercises, we assume that all activities will be carried out under one directory, which is referred to in the text as `${WAC_ROOT}`. In this directory, all repositories created during the exercise will be stored, along with auxiliary files. We recommend keeping a workspace for Visual Studio Code in this folder.

The commands we use on the command line assume the use of the [PowerShell] environment, which is standardly available on the Windows platform. Although most commands work without modification in the [Bash] environment, we recommend using the [PowerShell] environment on Linux and MacOS as well. The installation procedure is described on the [Install PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell) page.

## Development Containers

At the beginning of each chapter, commands for applying pre-created [Development Containers] template containers are provided. These are used to initialize the project at the beginning of the chapter to the expected state. The command given is not normally necessary and serves mainly to synchronize the project's state between individual exercises in case of technical problems. A detailed procedure is provided in the [Problem Solving](../99.Problems-Resolutions/01.development-containers.md) chapter.
