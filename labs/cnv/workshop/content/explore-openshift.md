#### Command Line Interface

OpenShift includes a feature-rich web console with both an Administrator perspective and a Developer perspective. In addition to the web console, OpenShift includes command line tools
to provide users with a nice interface to work with applications deployed to the
platform.  The `oc` command line tool is an executable written in the Go
programming language and is available for the following operating systems:

* Microsoft Windows
* MacOS 10
* Linux

This lab environment has the `oc` command line tool installed, and your lab user is already logged in to the OpenShift cluster.

Issue the following command to see help information:

```execute-1
oc help
```

#### Using a Project

Projects are a top level concept to help you organize your deployments. An
OpenShift project allows a community of users (or a user) to organize and manage
their content in isolation from other communities. Each project has its own
resources, policies (who can or cannot perform actions), and constraints (quotas
and limits on resources, etc). Projects act as a "wrapper" around all the
application services and endpoints you (or your teams) are using for your work.

#### Web Console

OpenShift ships with a web-based console that will allow users to perform various tasks via a browser.  

From within the lab guide window you'll see a button in the middle at the top that allows you to switch between the terminal and console options. Select the console and you should see the OpenShift dashboard:

<img  border="1" src="img/console-button.png"/>

<BR/>

> **Hint**: Please use Web Console in a seperate tab for the full functionality. 

To get a feel for how the web console works, click on this  [Web Console](http://console-openshift-console.%cluster_subdomain%/k8s/cluster/projects)

To login to Web Console, you need to first get the *kubeadmin-password*.

You can get it from the OpenShift installation directory in the bastion node.

First ssh to bastion node:

```execute-1
ssh %bastion-username%@%bastion-host%
```

When you see the prompt enter **%bastion-password%** as password

Then you can execute following to get the  password

```execute-1
cat %kubeadmin-password-file%
```

Note the password and exit 

```execute-1
exit
```

On Web Console login screen, enter the following credentials:

- Username: *kubeadmin*
- Password: *$kubeadmin-password*

The first time you access the web console, you will most likely be in the Administrator perspective. 

At the top of the left navigation menu, you can toggle between the Administrator perspective and the Developer perspective.

Select **Developer** to switch to the Developer perspective. Once the Developer perspective loads, you should be in the **Topology** view. Right now, there are no applications or components to view, but once you begin working on the lab, you'll be able to visualize and interact with the components in your application here.

We will be using a mix of command line tooling and the web console for the labs.
Get ready!