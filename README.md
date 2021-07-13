# OpenShift All-in-One

Welcome to our OpenShift All-in-One deployment automation repository. Here you'll find Ansible playbooks that will allow you to deploy an "all in one" configuration of OpenShift with a wide variety of options and extensions via [operators](https://docs.openshift.com/container-platform/4.7/operators/understanding/olm-what-operators-are.html). The reason it's known as an "all-in-one" is because the automation expects just a **single** baremetal host to be available; the automation will then virtualise all OpenShift infrastructure on-top, allowing you to get started with almost zero external requirements.

We do this for simplicity and for ease of scale when we need to - we have full control over the baremetal machine and we can virtualise networks, storage, host everything we need with no bandwidth/latency concerns, and also means we don't have to worry about sharing access to hardware and the potential conflicts that may arise from doing so. The purpose of this repo is **not** for production usage, simply to help build an environment for learning/enablement, demonstrations, or reproducing configurations very quickly.

The following diagram helps to visualise the configuration:

(WIP ADD DIAGRAM)



With the current implementation it's possible to achieve the following-

* Deploy against an existing RHEL 8 / CentOS 8 baremetal host as a "*prebuilt*" system
* Utilise the dynamic [Packet/EquinixMetal](https://metal.equinix.com/) provisioner for on-demand access to baremetal
* Deploy either a [user-provisioned infrastructure](https://docs.openshift.com/container-platform/4.7/architecture/architecture-installation.html) (UPI) or [installer-provisioned infrastructure](https://docs.openshift.com/container-platform/4.7/architecture/architecture-installation.html) (IPI) cluster-
  * UPI deploys the nodes via DHCP/PXE, with no provider integration
  * IPI deploy the nodes through the Baremetal IPI interface via IPMI and OpenStack Ironic ([Metal3](https://metal3.io/))
* Select the exact version of OpenShift that you want, e.g. "4.6.8", or a latest release, "latest-4.7"
* Specify whether you want a [compact cluster](https://www.openshift.com/blog/delivering-a-three-node-architecture-for-edge-deployments), i.e. a master-only 3-node configuration with no workers
* Specify the number of workers that you want, 1-3, depending on the host specification
* Deploy **just** the base infrastructure required to support OpenShift installation, i.e. don't run the install
* Deploy an OpenShift cluster, but leave it bare, i.e. no further customisation post-deployment
* Deploy an OpenShift cluster with a wide variety of additional operators deployed, including -
  * OpenShift Virtualization (CNV)
  * OpenShift Container Storage / OpenShift Data Foundations (OCS/ODF)
  * Advanced Cluster Manager (ACM)
  * Advanced Cluster Security (ACS)
* Deploy a Red Hat OpenStack Platform Plus deployment (combination of OpenShift, ACM, and ACS)
* Deploy an optional Apache Guacamole instance for easy browser-based interaction with the cluster
* (**WIP**) Deploy the cluster in a pre-determined state to support self-paced labs and demonstration for-
  * Baremetal IPI cluster deployment & post-deployment utilisation
  * OpenShift Virtualization (CNV) deployment and utilisation

> **NOTE**: If you're using the dynamic Packet/EquinixMetal provisioner, you're entirely responsible for any costs associated with utilising resources from this vendor. We provide the automation and can help set a decommissioning date so the are pruned after a certain deadline, but if for whatever reason this fails, the authors, nor our employers can be held responsible.



# Getting Started

Note, these playbooks are expected to be operated from your workstation/laptop, and not executed on the target host directly, therefore please don't clone this repository on the baremetal system you want to use, clone them locally and execute **against** a target machine.

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
| baremetal_provider        | Select the baremetal provider that you want to use, i.e. which platform is providing your infrastructure. <br /><br />Choices are between "**prebuilt**" and "**packet**", where "prebuilt" will take a machine that you've already deployed with an EL8 compatible distribution and pre-configured SSH-keys, or "packet" where it will dynamically provision a new baremetal system on Packet, at your own cost.<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| prebuilt_ip               | Enter the IP address of the target baremetal system if you're using "prebuilt" with `baremetal_provider`. Do not enter a hostname for this. This has not been tested with IPv6 but it should work just fine.<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_api_token          | If you're using "packet" for `baremetal_provider`, enter your API token so that the dynamic provisioner can deploy a baremetal instance with your credentials. Your API token can be found here: https://metal.equinix.com/developers/api/<br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_project_name       | If you're using "packet" for `baremetal_provider`, enter the name of the project you would like to use; if it doesn't exist it will be created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_ondemand_type      | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_spot_type          | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_spot_bid           | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_deploy_type        | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_facility           | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_runtime_hours      | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| packet_delete_project     | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| ssh_key                   | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| pull_secret               | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_guacamole          | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_compact            | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_ocp                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_ocp_plus           | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_cnv                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_ocs                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_acm                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_acs                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| deploy_nfs                | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| ocp_version               | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |
| ocp_workers               | created. <br /><br />**Default**: '2' - we'd rather save capacity on the baremetal host, and two is the minimum number of workers in a non-compact cluster. |
| deploy_type               | created. <br /><br />**Default**: There is <u>no default</u>. You must specify an option here manually. |



If you wish to deploy the Guacamole UI, you'll need to make sure you've either got Ansible 2.10+, or you've installed the Podman module:

~~~bash
$ ansible-galaxy collection install containers.podman
~~~

When you're ready to deploy a cluster, call `main.yml` making sure you specify your variables and the dynamic inventory (don't worry that it doesn't exist yet):

~~~bash
$ ansible-playbook playbooks/main.yml -e @my_vars.yml -i inventory.yml
~~~

To destroy your cluster, call `destroy.yml`, again making sure to specify your variables and dynamic inventory:

~~~bash
$ ansible-playbook playbooks/destroy.yml -e @my_vars.yml -i inventory.yml
~~~
