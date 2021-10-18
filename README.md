# OpenShift All-in-One

Welcome to our OpenShift All-in-One deployment automation repository. Here you'll find Ansible playbooks that will allow you to deploy an "all in one" configuration of OpenShift with a wide variety of options and extensions via [operators](https://docs.openshift.com/container-platform/4.7/operators/understanding/olm-what-operators-are.html). The reason it's known as an "all-in-one" is because the automation expects just a **single** baremetal host to be available; the automation will then virtualise all OpenShift infrastructure on-top, allowing you to get started with almost zero external requirements.

We do this for simplicity and for ease of scale when we need to - we have full control over the baremetal machine and we can virtualise networks, storage, host everything we need with no bandwidth/latency concerns, and also means we don't have to worry about sharing access to hardware and the potential conflicts that may arise from doing so. The purpose of this repo is **not** for production usage, simply to help build an environment for learning/enablement, demonstrations, or reproducing configurations very quickly.

The following diagram helps to visualise the configuration:

(**WIP** ADD DIAGRAM)



With the current implementation it's possible to achieve the following-

* Deploy against an existing RHEL 8 / CentOS 8 baremetal host as a "*prebuilt*" system
* Utilise the dynamic [Packet/EquinixMetal](https://metal.equinix.com/) provisioner for on-demand access to baremetal
* Deploy either a [user-provisioned infrastructure](https://docs.openshift.com/container-platform/4.7/architecture/architecture-installation.html) (UPI) or [installer-provisioned infrastructure](https://docs.openshift.com/container-platform/4.7/architecture/architecture-installation.html) (IPI) cluster-
  * UPI deploys the nodes via DHCP/PXE, with no provider integration
  * IPI deploy the nodes through the Baremetal IPI interface via IPMI and OpenStack Ironic ([Metal3](https://metal3.io/))
* Select the exact version of OpenShift that you want, e.g. "*4.6.8*", or a latest release, "*latest-4.7*"
* Specify whether you want a [compact cluster](https://www.openshift.com/blog/delivering-a-three-node-architecture-for-edge-deployments), i.e. a master-only 3-node configuration with no workers
* Specify the number of workers that you want, 1-3, depending on the host specification
* Deploy **just** the base infrastructure required to support OpenShift installation, i.e. don't run the install
* Deploy an OpenShift cluster, but leave it bare, i.e. no further customisation post-deployment
* Deploy an OpenShift cluster with a wide variety of additional operators deployed, including -
  * [OpenShift Virtualization](https://www.openshift.com/learn/topics/virtualization/) (CNV)
  * [OpenShift Container Storage / OpenShift Data Foundations](https://www.redhat.com/en/technologies/cloud-computing/openshift-data-foundation) (OCS/ODF)
  * [Advanced Cluster Manager](https://www.redhat.com/en/technologies/management/advanced-cluster-management) (ACM)
  * [Advanced Cluster Security](https://www.redhat.com/en/resources/advanced-cluster-security-for-kubernetes-datasheet) (ACS)
* Deploy a [Red Hat OpenShift Platform Plus](https://www.openshift.com/products/platform-plus) deployment (combination of OpenShift, ACM, and ACS)
* Deploy an optional [Apache Guacamole](https://guacamole.apache.org/) instance for easy browser-based interaction with the cluster
* (**WIP**) Deploy a *disconneted* cluster in which the OpenShift cluster relies on a dedicated image registry
* Deploy NFS-based "basic" persistent volume storage for when OCS/ODF is not enabled
* (**WIP**) Deploy the cluster in a pre-determined state to support self-paced labs and demonstration for-
  * Baremetal IPI cluster deployment & post-deployment utilisation
  * OpenShift Virtualization (CNV) deployment and utilisation

> **NOTE**: If you're using the dynamic Packet/EquinixMetal provisioner, you're entirely responsible for any costs associated with utilising resources from this vendor. We provide the automation and can help set a decommissioning date so the are pruned after a certain deadline, but if for whatever reason this fails, the authors, nor our employers can be held responsible.



# Getting Started

Note, these playbooks are expected to be operated from your workstation/laptop, and not executed on the target host directly, however, if you clone this repository on the baremetal system you want to use, make sure that your `prebuilt_ip` is the IP or hostname of the machine, and *not* `127.0.0.1`; when you're executing against a remote server or running the playbooks locally, you will need to make sure that the user in which you execute the playbooks with has password-free root access to the target by ensuring that ssh-keys are exchanged prior to operation.

## Software prerequisites

Installation requires either Ansible 2.10+ or you'll need to install the **Podman module** for lower versions. This can be done via `ansible-galaxy` or an RPM:

~~~bash
$ ansible-galaxy collection install containers.podman

-or-

$ sudo dnf install -y ansible-collection-containers-podman.noarch
~~~

## Configure  

To get started, first clone this repository:

~~~bash
$ git clone https://github.com/RHFieldProductManagement/openshift-aio.git
~~~

Next, enter the directory and make a copy of the `sample_vars.yml` file, as we'll use this to customise the deployment:

~~~bash
$ cd openshift-aio
$ cp sample_vars.yml my_vars.yml
~~~

Now you can edit your variables to suit your expected configuration:

~~~bash
$ vi my_vars.yml
~~~

The list of available options (exposed as Ansible variables) is described below-

| Name of Option / Variable | Purpose                                                      |
| ------------------------- | ------------------------------------------------------------ |
| `baremetal_provider`      | Select the baremetal provider that you want to use, i.e. which platform is providing your infrastructure. <br /><br />Choices are between "**prebuilt**" and "**packet**", where "prebuilt" will take a machine that you've already deployed with an EL8 compatible distribution and pre-configured SSH-keys, or "packet" where it will dynamically provision a new baremetal system on Packet, at your own cost.<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `prebuilt_ip`             | Enter the IP address (or hostname) of the target baremetal system if you're using "prebuilt" with `baremetal_provider`. This has not been tested with IPv6 but it should work just fine.<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `packet_api_token`        | If you're using "packet" for `baremetal_provider`, enter your API token so that the dynamic provisioner can deploy a baremetal instance with your credentials. Your API token can be found here: https://metal.equinix.com/developers/api/<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `packet_project_name`     | If you're using "*packet*" for `baremetal_provider`, enter the name of the project you would like to use; if it doesn't exist it will be created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `packet_delete_project`   | Select whether you want the destroy process to clean up and remove the Packet project. Warning, if this is set to true, existing instances that aren't part of this deployment may be removed when the project is deleted; use with caution.<br /><br />**Default**: false |
| `packet_deploy_type`      | Packet provisioning supports two different pricing models; "*spot*" and "*ondemand*". Spot pricing is where you bid for access to a system at a given price (see `packet_spot_bid`), which can be a way of getting much cheaper access to available systems, but it comes at the risk of your instance being deleted if a higher-bidder comes along and there's a resource shortage. On-demand pricing is where you pay the full advertised price with no risk of your instance being pulled by another higher-paying customer. For this reason we default to on demand. Full details on spot pricing can be found [here](https://metal.equinix.com/developers/docs/deploy/spot-market/). <br /><br />**Default**: "ondemand" |
| `packet_ondemand_type`    | Select the instance type/size that you wish to provision on Packet. There are varying options, and it may depend on the datacentre that you deploy into (see `packet_facility`); but we typically require 128GB memory (minimum) and NVMe-based storage. We also currently only support x86_64 architectures. A list of available instance types can be found [here](https://metal.equinix.com/product/servers/).<br /><br />**Default**: "s3.xlarge.x86" |
| `packet_spot_type`        | As with `packet_ondemand_type`, we can request a specific instance type, but within a spot pricing context. <br /><br />**Default**:  "s3.xlarge.x86" |
| `packet_spot_bid`         | Select the cost per hour that you're bidding when using `packet_deploy_type: spot`. If your bid is accepted by Packet, your instance will provision, but as mentioned above, be aware of the instance revoking rules.<br /><br />**Default**: "0.70", i.e. $0.70/hour (US), which based on "s3.xlarge.x86" instance type is significantly under market pricing and would be considered fairly risky. |
| `packet_facility`         | Specify the Packet facility (datacentre) that you want to utilise for deployment of your instance. There are many to choose from, but not all facilities have access to all of the instance types, so please ensure availability of the type via their [website](https://metal.equinix.com/developers/docs/locations/facilities/). <br /><br />**Default**: "am6" (Amsterdam, NL) |
| `packet_runtime_hours`    | Specify how long you want the Packet instance to run for; this helps take some of the risk out of accidentally leaving the Packet system running after you're done. This flag is used to set a termination date for the instance. Note that whilst this flag should be honoured, we can take no responsibility for any costs incurred. <br /><br />**Default**: "3", i.e. 3 hours from instance request, not from Playbook completion; sometimes the playbooks can take >1hr to run, depending on the configuration requested. |
| `ocp4_aio_ssh_key`                 | Specify the public ssh-key that you want to use, **not** the location to the file, e.g. `ocp4_aio_ssh_key: 'ssh-rsa AAAA....'`, making sure that you place single quotes around the entry. This key will be injected into the instance if using `baremetal_provider: packet` and also be used for authentication during the installation. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `pull_secret`             | Specify your OpenShift *pull secret* so we can pull OpenShift images. This is a **json** formatted string with all of the necessary authentication and authorisation secrets and is linked to your specific username/email-address. This can be downloaded from [cloud.redhat.com](https://cloud.redhat.com/) (with instructions [here](https://access.redhat.com/solutions/4844461)). <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `ocp4_aio_deploy_guacamole`        | Specify whether you want to deploy [Apache Guacamole](https://guacamole.apache.org/) as part of the deployment. This will provide access to the remote cluster via a web-browser, both CLI and GUI access, without having to configure a proxy server locally. This is useful for demo or lab purposes, but is completely optional.<br /><br />**Default**: false |
| `ocp4_aio_deploy_disconnected`     | Specify whether you want to use a *disconnected* image registry as part of the installation to mimic a disconnected installation  - this can speed up the deployment time and also limit the amount of bandwidth consumed as the image pull for the OpenShift bits themselves only happens once. If set to *true*, a registry will be deployed for you and all images for the specified version (`ocp4_aio_ocp_version`) will be pulled.<br /><br />**Default**: false |
| `ocp4_aio_deploy_compact`          | Specify whether you want to deploy a compact cluster, i.e. three nodes that are simultaneously master and worker nodes, i.e. no separate worker nodes. This is useful when you want to minimise resource utilisation or mimic small "edge type" clusters. <br /><br />**Default**: false |
| `ocp4_aio_deploy_ocp`              | Specify whether you want the automation to actually deploy an OpenShift cluster as part of the playbook run. By default this is set to *true*, but you can set this to *false* if you only want the base infrastructure to be configured; this is useful in scenarios in which you want to run through the deployment of an OpenShift cluster yourself.<br /><br />**Default**: true |
| `ocp4_aio_deploy_ocp_plus`         | This is the same variable as `ocp4_aio_deploy_ocp` but it also enforces the automated deployment of Red Hat Advanced Cluster Security (ACS) and Red Hat Advanced Cluster Manager (ACM), via `ocp4_aio_deploy_acs` and `ocp4_aio_deploy_acm`. So if this option is enabled, settings for OCP/ACM/ACS are overridden.  **NOTE**: We're working to add Quay support. <br /><br />**Default**: false |
| `ocp4_aio_deploy_cnv`              | Specify whether you want to automatically deploy the OpenShift Virtualization (CNV) operator as part of the OpenShift installation. This does nothing other than deploying the operator and any required core components, e.g. CNV HyperConverged.<br /><br />**Default**: false |
| `ocp4_aio_deploy_ocs`              | Specify whether you want to automatically deploy the OpenShift Container Storage (OCS), now known as OpenShift Data Foundations (ODF), operator as part of the OpenShift installation. This does nothing other than deploying the operator and any required core components, e.g. OCS Storage Cluster. <br /><br />**Default**: false |
| `ocp4_aio_deploy_acm`              | Specify whether you want to automatically deploy the Red Hat Advanced Cluster Manager (ACM) operator as part of the OpenShift installation. This does nothing other than deploying the operator and any required core components, e.g. ACM MultiClusterHub. <br /><br />**Default**: false |
| `ocp4_aio_deploy_acs`              | Specify whether you want to automatically deploy the Red Hat Advanced Cluster Security operator as part of the OpenShift installation. This does nothing other than deploying the operator and any required core components, e.g. ACS Central. <br /><br />**Default**: false |
| `ocp4_aio_deploy_nfs`              | Specify whether you want to deploy basic NFS-based storage for your cluster; this will create an NFS storage class as well as some empty PV's to use as NFS doesn't use a dynamic provisioner. Note, if `ocp4_aio_deploy_ocs` is set to *false*, `ocp4_aio_deploy_nfs` is set to *true* by default. If both are enabled, OCS becomes the default storage class, but both will be available in the resulting cluster.<br /><br />**Default**: false |
| `ocp4_aio_ocp_version`             | Specify the version of OpenShift that you wish to use, this can be an exact version of OpenShift, e.g. *"4.7.19"*, or it can be specified as a latest major release, e.g. *"latest-4.6"*. If  `ocp4_aio_deploy_ocp: true` (or `ocp4_aio_deploy_ocp_plus: true`) a cluster with this version will be deployed, otherwise just the client/installer binaries and any required images will be pulled for you. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| `ocp_workers`             | Specify the number of workers that you'd like to use between 1-3; there's some work in progress to expand this further, but right now the minimum is *'2'*, although *'1'* can be used if you **manually** enable schedulable masters during installation. <br /><br />* If you specify *'0'* here, `ocp4_aio_deploy_compact` is forced to *true*. <br />* If `ocp4_aio_deploy_ocs` is set to *true*, `ocp_workers` will be forced to *'3'*  <br /><br />**Default**: '2' - we'd rather save capacity on the baremetal host, and two is the minimum number of workers in a non-compact cluster. |
| `ocp4_aio_deploy_type`             | Specify the deployment type that you want to use; you can choose from a UPI or an IPI type deployment, in which the "baremetal" type will be used for both. With a UPI installation, there's no provider integration and is considered a more agnostic deployment approach, whereas IPI uses the IPMI-based management to provide a more integrated baremetal management interface via [Metal3](https://metal3.io/). <br /><br />**Default**: "ipi" (Installer Provisioned Infrastructure) |

## Deployment

Before being able to deploy, make sure to fetch the required roles with the following command :

~~~bash
$ ansible-galaxy -r playbools/roles/requirements.yml -p playbooks/roles
~~~

When you're ready to deploy a cluster, call `main.yml` making sure you specify your variables and the dynamic inventory (don't worry that it doesn't exist yet):

~~~bash
$ ansible-playbook playbooks/main.yml -e @my_vars.yml
~~~

To destroy your cluster, call `destroy.yml`, again making sure to specify your variables and dynamic inventory:

~~~bash
$ ansible-playbook playbooks/destroy.yml -e @my_vars.yml -i inventory.yml
~~~



# Feedback

If you have any questions or other feedback, please raise a GitHub issue for this, or alternatively submit a PR, we always welcome contributions! Thanks a lot for giving it a try.
