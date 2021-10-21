[**<u>Work in Progress</u>**]

Welcome to the OpenShift Virtualization self-paced lab guide/workbook. We've put this together to give you an overview and technical deep dive into how OpenShift Virtualization works, and how it will bring virtualisation capabilities to OpenShift over the next few months.

**OpenShift Virtualization** is now the official *product name* for the Container-native Virtualization operator for OpenShift. This has more commonly been referred to as "**CNV**" and is the downstream offering for the upstream [Kubevirt project](https://kubevirt.io/). While some aspects of this lab will still have references to "CNV", any reference to "CNV", "Container-native Virtualization" and "OpenShift Virtualization" can be used interchangeably.

In these labs you'll utilise a virtual environment that mimic as close as (feasibly) possible to a real [baremetal IPI](https://metal3.io/) OpenShift 4.9 deployment to help get you up to speed with the concepts of OpenShift Virtualization. In this hands-on lab you won't need to deploy OpenShift, that'll already be deployed for you as part of the [OpenShift](https://github.com/RHFieldProductManagement/openshift-aio) AIO concept, but aside from this lab guide, it's a completely empty cluster, ready to be configured and used with OpenShift Virtualization.


This is the self-hosted lab guide that will run you through the following:

* **Validating the OpenShift deployment**
* **Deploying OpenShift Virtualization**
* **Setting up Storage for OpenShift Virtualization**
* **Setting up Networking for OpenShift Virtualization**
* **Deploying some CentOS8 / RHEL8 / Fedora 34 Test Workloads**
* **Performing Live Migrations and Node Maintenance**
* **Utilising network masquerading on pod networking for VM's**
* **Using the OpenShift Web Console extensions for OpenShift Virtualization**
* **Upgrading OpenShift Virtualization via the Operator Lifecycle**

Within the lab you can cut and paste commands directly from the instructions; but also be sure to review all commands carefully both for functionality and syntax!

> **NOTE**: In some browsers and operating systems you may need to use Ctrl-Shift-C / Ctrl-Shift-V to copy/paste within this lab environment!

Please be very aware, this is a work in progress, there may be some bugs/typo's, and we very much welcome feedback on the content, what's missing, what would be good to have, etc. Please feel free to submit PR's or raise GitHub issues, we're glad of the feedback!



When you're ready...

