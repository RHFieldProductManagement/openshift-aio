In this section we will deploy and connect a MongoDB database where the
`nationalparks` application will store the location information.

This time we are going to deploy the MongoDB application in a Virtual Machine 
by leveraging OpenShift Virtualization.   


### 1.  Search for the Virtual Machine Template


In this module we will create MongoDB from a *Template*, which contains all the necessary Kubernetes resources and configuration to deploy and run MongoDB in a VM which is based on Centos.

Please go back to the [Web Console](http://console-openshift-console.{{cluster_subdomain}}/k8s/cluster/projects)

If you are in the in the Administrator perspective, switch to Developer perspective and go to the *{{PARKSMAP_NAMESPACE}}* project. 

> From the left menu, click *+Add*. You will see a screen where you have multiple options to deploy application. 

> Then Click *All Services* and in the *Search* text box and enter *mongo*  to find the MongoDB VM template. 

 <br/><br/> 

![Search Template](img/parksmap-mongodb-search.png)  

<br/>

### 2. Instantiate the Virtual Machine Template

In order to instantiate the temaplate, first click on the `MongoDB Virtual Machine` template to open the popup menu 
and then click on the *Instantiate Template* button as you did when you deployed the parksmap application components.
This will open a dialog that will allow you to configure the template. This template allows you to configure the following parameters:

- *MongoDB Application Name*
- *Virtual Machine Username*
- *Virtual Machine User Password*
- *Database Name*
- *Database Username*
- *Database Admin Password*
  
 > Enter *mongodb-nationalparks* in  **MongoDB Application Name** field and leave other parameter values as-is.
 
 <br/><br/> 

![Configure Template](img/parksmap-mongodb-nationalparks.png)  

 <br/>

> Next click the blue *Create* button. 

You will be directed to the *Topology* page, where you should see the visualization for the `mongodb-nationalparks` virtual machine in the `workshop` application. 
This will make OpenShift to create both *VirtualMachine* and *Service* objects. `nationalparks` backend application will use this *mongodb-nationalparks service* to communicate with MongoDB.  

### 3. Verify the Database Service in Virtual Machine  

It will take some time MongoDB VM to start and initialize. You can check the status of VM in the Web Console by clicking on the VM  details in the Topology View or execute following command in the terminal 

```execute
oc get vm
```

~~~bash
NAME                    AGE   STATUS     READY
mongodb-nationalparks   45s   Running    True
~~~

After MongoDB Virtual Machine started, open *Virtual Machine Console* as shown in the figure below 

 <br/><br/> 

![Open VM Console](img/parksmap-nationalparks-mongodb-console.png)  

 <br/>

> Switch to *Serial Console* and wait for the login prompt.

> On the login screen, enter the following credentials:

~~~bash
  Login: redhat
  Password: openshift 
~~~

> Check whether *mongod* service is running by executing following:

```execute
systemctl status mongod
```

Please verify whether *mongod* service is up and running as shown in the figure below

 <br/><br/> 

![MongoDB Service Status](img/parksmap-mongodb-nationalparks-check.png)  

 <br/>

### 4. Verify Nationalparks Application

Now that we have a database deployed, we can again visit the `nationalparks` web
service to query for data:


> [Nationalparks Data All](http://nationalparks-{{project_namespace}}.{{cluster_subdomain}}/ws/data/all)

And the result?

> []

Where's the data? Think about the process you went through. You deployed the
application and then deployed the database. Nothing actually loaded anything
*INTO* the database, though.

The application provides an endpoint to do just that:

> [Nationalparks Data Load](http://nationalparks-{{project_namespace}}.{{cluster_subdomain}}/ws/data/load)

And the result?

> Items inserted in database: 2893

If you then go back to `/ws/data/all` you will see tons of JSON data now.

If you check your browser now:

> [Parksmap](http://parksmap-{{project_namespace}}.{{cluster_subdomain}})

 You'll notice that the parks suddenly are showing up as below. 
 <br/> 

![Parksmap](img/parksmap-nationalparks-ui.png)  


