In this lab, we will use OpenShift Web Console to deploy the frontend and backend components of the ParksMap application. 
Parksmap application consists of one frontend web application, two backend applications and 2 databases. 

- ParksMap frontend web application which is also called `parksmap` and uses OpenShift's service discovery mechanism to discover the backend services deployed and shows their data on the map.

- Nationalparks backend application queries for national parks information (including its
coordinates) that is stored in a MongoDB database. 

- MLBParks backend application queries Major League Baseball stadiums in the US that are stored in an another MongoDB database.

Parksmap frontend and backend components are shown in the diagram below:
 <br/>

![Application Architecture](img/roadshow-app-architecture-main.png)  

 <br/>

### 1. Creating the Project

As a first step, we need to create a project where Parksmap application will be deployed. 
You can create the project with the following command:

```execute
oc new-project %parksmap-project-namespace%
```

### 2.  Grant Service Account View Permissions

The parksmap frontend application continously monitors the **routes** of the backend applications. So that's it require permissions to access OpenShift API to learn about other **Pods**, **Services**, and **Route** within the **Project**. 


```execute
oc policy add-role-to-user view -z default
```

The *oc policy* command above is giving a defined _role_ (*view*) to default user so that applications in current project can access OpenShift API.

### 3. Login to OpenShift Web Console

We will use OpenShift Web Console to deploy Parksmap Web Application components. 

Please go to the [Web Console](http://console-openshift-console.%cluster_subdomain%/k8s/cluster/projects) 
and on the login screen, enter the following credentials:

- Username: *kubeadmin*

- Password: **$KUBEADMIN_PASSWORD** 

### 4.  Search for the Application Template

If you are in the in the Administrator perspective, switch to Developer perspective and go to the `%parksmap-project-namespace%` project. 

From the left menu, click `+Add`. You will see a screen where you have multiple options to deploy application to OpenShift. Click `All Services` as shown below.

 <br/>

![Service Catalog](img/parksmap-all-services.png)  

 <br/> 

We will be using `Templates` to deploy the application components. A template describes a set of objects that can be parameterized 
and processed to produce a list of objects for creation by OpenShift Container Platform. 
A template can be processed to create anything you have permission to create within a project, for example services, build configurations, and deployment configurations. A template can also define a set of labels to apply to every object defined in the template.

You can create a list of objects from a template using the CLI or, if a template has been uploaded to your project or the global template library, using the web console.

In the `Search` text box, enter *parksmap* to find the application template. 

 <br/>

![Search Template](img/parksmap-search-template.png)  

 <br/>

### 5. Instantiate the Application Template

Then click on the `Parksmap` template to open the popup menu and then click on the `Instantiate Template` button as shown below.

 <br/>

![Instantiate Template](img/parksmap-instantiate-template.png)  

 <br/>

This will open a dialog that will allow you to configure the template. This template allows you to configure the following parameters:

- Parksmap Web Application Name
- Mlbparks Application Name
- Mlbparks MongoDB Application Name
- Nationalparks Application Name
- Nationalparks MongoDB Application Name
 <br/>

![Configure Template](img/parksmap-application-template.png)  
 
 <br/>

Next click the blue *Create* button without changing default parameters. You will be directed to the *Topology* page, where you should see the visualization for the `parksmap` deployment config in the `workshop` application. 
This will make OpenShift to create all Kubernetes resources to deploy the application which includes *Deployment*, *Service*, and *Route*.


### 6. Check the Application

These few steps are the only ones you need to run to all 3 application components of `parksmap` on OpenShift. Getting the `parksmap` application running may take a little while to complete. 
Each OpenShift node that is asked to run the images of applications has to pull (download) it, if the node does not already have it cached locally. You can check on the status of the image download and deployment in the *Pod* details page, or from the command line with the `oc get pods` command to check the readiness of pods or you can monitor it from the Developer Console.

Your screen will end up looking something like this:
 <br/> 

![Configure Template](img/parksmap-topology-1.png)   
 
 <br/>

This is the *Topology* page, where you should see the visualization for the `parksmap` ,`nationalparks`  and `mlbparks` deployments in the `workshop` application.


### 7. Access the Application

If you click on the `parksmap` entry in the Topology view, you will see some information about that deployment. 
The *Resources* tab may be displayed by default. If so, click on the *Details* tab.  
 <br/>

![Details Tab image](img/parksmap-topology-route.png)

 <br/>

On that panel, you will see that there is a single *Route* which allows external access to `parksmap` application. While *Services* provide internal abstraction and load balancing within an OpenShift environment, sometimes clients (users, systems, devices, etc.) **outside** of OpenShift need to access an application. 

The way that external clients are able to access applications running in OpenShift is through the OpenShift routing layer. And the data object behind that is a *Route*.

Also note that there is a decorator icon on the `parksmap` visualization now. If you click that, it will open the URL for your *Route* in a browser. 

This application is now available at the URL shown in the Developer Perspective. Click the link and you will see the following:

 <br/>

![Parksmap UI](img/parksmap-view-not-working.png)

 <br/>

You can notice that `parksmap` application does not show any parks as we haven't deployed database servers for the backends yet. 




