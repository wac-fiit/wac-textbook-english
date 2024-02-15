## Manual Deployment of the Application in Microsoft Azure

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-042`

Development in short cycles and fast feedback are the essence of agile development. The ability to work with a real application - instead of a PowerPoint presentation - and access the application at any time allows for relevant feedback. Our goal is to deploy this application to a public data center as quickly as possible.

The simplest way to deploy the application is to manually copy files to the cloud. This is suitable, for example, when continuous integration is not available or when developing the application locally. Another increasingly used option is containerizing the application and deploying such a container. We will show this process in the following steps, and in the next chapter, we will configure an automated continuous deployment of the application.

1. Go to the [Visual Studio Dev Essentials][vs-essentials] page, connect to the services, and activate a free account for _Microsoft Azure_ services. During activation, you will be asked for credit card information, but no charges will be applied unless you explicitly decide to do so. We will only use free services for the exercises.

![Free Azure Services](./img/042-01-AzureFree.png)

After activation, you will be redirected to the [Azure Portal][azure-portal], where you can manage your resources in the Microsoft Azure public data center.

2. In the portal, select _+ Create a resource_ in the left panel and choose _Web App_.

3. Choose a new resource group - _Resource Group - Create New_ - and name it `WebInCloud-Dojo`.

The resource group groups resources belonging to one logical application and allows you to manage multiple resources as a single entity (deletion, creation, movement between accounts, access control, etc.).

4. In the _Name_ field, enter the application name, for example, `<pfx>-ambulance`. This name will be used as the server name where your application will be exposed, so it must be a unique name.
  
5. In the _Instance Details_ section:

* In the _Publish_ row, select _Docker Container_
* Set the _Operating System_ to _Linux_.
* In the _Region_ row, choose `West Europe` (data center in the Netherlands).

6. In the _Pricing plan_ section, choose a new _Linux plan_ (_Create new_). Name it `WebInCloud-DojoServicePlan` and change the standard setting in the _Pricing Plan_ field to _Free F1_.

![Creating a Web App resource](./img/042-01-CreateWebApp.png)

7. Press the _Next: Docker_ button and change the _Image Source_ option to `Docker Hub`. Enter `<your-account>/ambulance-ufe:latest` in the _Image and tag_ field.

![Setting up a Docker image for Azure Web App](./img/042-02-CreateWebAppDocker.png)

8. Now press the _Review + create_ button, review the settings for the web application, and confirm by selecting _Create_.

After a while, a new message _Deployment succeeded_ will appear in your notifications in the portal. Choose _Pin to Dashboard_ (optional) and then _Go to Resource_. In the overview of the web application (_Overview_), you will see a link - _URL_ - to the new application. Open this link in a new window.

![Resource Overview Web App](./img/042-03-WebAppOverview.png)

In the browser, you should see the familiar list of waiting patients, this time served from the Microsoft Azure data center.

Your application is deployed.
As mentioned earlier, this is the most basic manual way to deploy an application to the cloud. In the next steps, we will show you how to automate this process.

>$apple:> If you have an arm64 architecture processor, your application may be non-functional at this point due to processor architecture incompatibility. Continue to the next chapter, where the continuous integration specification will be modified to create an image for different platforms and processor architectures.
